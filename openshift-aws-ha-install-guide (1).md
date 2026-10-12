# OpenShift HA Cluster on AWS (IPI) - Complete Guide

This guide installs a **highly available OpenShift cluster** (3 control plane + 3 worker nodes) on **AWS EC2** using the **Installer-Provisioned Infrastructure (IPI)** method.

> **Cost warning:** A 6-node HA cluster runs continuously and costs real money. Estimated cost is roughly **USD 2 per hour (about 1,400-1,600 per month)** if left running. Check the AWS Pricing Calculator for your region. **Destroy the cluster as soon as you finish practice** (see Section 7). Set an AWS Budget alarm before you start.

---

## 1. What you will build

| Item | Details |
|------|---------|
| Installation method | IPI (installer creates the infrastructure). This is the default method for AWS. |
| Topology | HA cluster: 3 control plane (master) nodes + 3 worker nodes |
| Availability | Nodes spread across 3 Availability Zones (AZs) |
| Region (example) | `ap-south-1` (Mumbai). Use the region closest to you. |

The installer creates these AWS resources automatically:
- VPC, subnets, route tables, NAT gateways, internet gateway
- Security groups
- EC2 instances (bootstrap, control plane, workers)
- Load balancers for the Kubernetes API and the application Ingress
- Route 53 DNS records in your hosted zone
- IAM roles and instance profiles
- An S3 bucket for ignition configuration files

---

## 2. Machine selection

| Role | Count | Instance type | vCPU | RAM | Root volume |
|------|-------|---------------|------|-----|-------------|
| Bootstrap (temporary, auto-deleted) | 1 | `m6i.xlarge` | 4 | 16 GiB | 100 GiB gp3 (default) |
| Control plane (master) | 3 | `m6i.2xlarge` | 8 | 32 GiB | 120 GiB gp3 |
| Worker (compute) | 3 | `m6i.xlarge` | 4 | 16 GiB | 120 GiB gp3 |

**Notes:**
- Red Hat docs list `m6i.*` as a tested instance family for AWS. The sample install-config uses `m6i.xlarge` for control plane and `c5.4xlarge` for compute as examples only.
- Newer Red Hat docs (openshift-docs change) show a minimum of **8 vCPU and 16 GB RAM** for HA topology nodes. The per-role table could not be read directly, so verify it for your version (Section 9). If you want to be safe, use `m6i.2xlarge` for all 6 nodes. That roughly doubles the worker cost.
- Confirm the instance type exists in your region and AZs:

```bash
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters Name=instance-type,Values=m6i.2xlarge \
  --region ap-south-1
```

- Total vCPU needed at peak: 3 x 8 (control plane) + 3 x 4 (workers) + 4 (bootstrap) = **40 vCPU**.

---

## 3. Prerequisites

### 3.1 AWS account and IAM user

1. Use an AWS account where you can create EC2, VPC, ELB, IAM, Route 53 and S3 resources.
2. Create a dedicated IAM user for the installer.
   - **Lab shortcut:** attach `AdministratorAccess`.
   - **Production:** use only the permissions the installer needs. Red Hat docs list the required EC2, IAM, Route 53, ELB and S3 permission sets.
3. Create an access key for that user (IAM -> Users -> Security credentials).

Do not commit access keys to Git.

### 3.2 Route 53 public hosted zone (mandatory)

The AWS account must have a **dedicated public hosted zone** in Route 53 for the base domain. Red Hat docs state this is required.

Example: base domain `aws.example.com`, cluster name `ocp-ha`. The API becomes `api.ocp-ha.aws.example.com`.

Steps:
1. Route 53 -> Hosted zones -> **Create hosted zone**, name `aws.example.com`, type **Public**.
2. Copy the 4 NS records of the new zone.
3. If `example.com` is managed outside AWS, add those 4 NS records as an NS delegation for `aws` in your DNS provider.
4. Verify delegation:

```bash
dig NS aws.example.com +short
```

Installation fails without working delegation.

### 3.3 Service quota

Check the EC2 quota for "Running On-Demand Standard (A, C, D, H, I, M, R, T, Z) instances" in your region. Red Hat docs note that default service limits can block installation.

```bash
aws service-quotas list-service-quotas --service-code ec2 \
  --query "Quotas[?QuotaName=='Running On-Demand Standard (A, C, D, H, I, M, R, T, Z) instances'].[QuotaName,Value]" \
  --region ap-south-1
```

Keep at least **64 vCPU** quota for headroom. Request an increase in Service Quotas if needed.

### 3.4 Tools on your workstation (Linux / macOS / WSL)

| Tool | Purpose | Get it from |
|------|---------|-------------|
| `aws` CLI v2 | AWS authentication and checks | aws.amazon.com/cli |
| `openshift-install` | Creates the cluster | console.redhat.com/openshift/install/aws/installer-provisioned |
| `oc` | Cluster CLI | Same download page as the installer |
| `ssh-keygen` | SSH key for node access | Built in |

Download `openshift-install` and `oc` for the **same version** you want to install.

```bash
aws configure
# AWS Access Key ID: <your key>
# AWS Secret Access Key: <your secret>
# Default region name: ap-south-1
# Default output format: json

aws sts get-caller-identity
```

### 3.5 Pull secret and SSH key

1. Open console.redhat.com -> OpenShift -> Downloads -> **Pull secret**. Copy it.
2. Create an SSH key:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ocp-ha -N ""
cat ~/.ssh/ocp-ha.pub
```

---

## 4. Step-by-step installation

### Step 1: Create the installation directory

```bash
mkdir -p ~/ocp-ha-install
cd ~/ocp-ha-install
```

Use `--dir` for every install and destroy command. This keeps state files (including `auth/`) in one place.

### Step 2: Create install-config.yaml

Create `~/ocp-ha-install/install-config.yaml`:

```yaml
apiVersion: v1
baseDomain: aws.example.com            # Must match your Route 53 public hosted zone
metadata:
  name: ocp-ha                         # Cluster name
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      type: m6i.2xlarge                # Control plane instance type
      rootVolume:
        size: 120
        type: gp3
  replicas: 3
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    aws:
      type: m6i.xlarge                 # Worker instance type
      rootVolume:
        size: 120
        type: gp3
  replicas: 3
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: ap-south-1
publish: External                      # Use Internal only if you have VPN/Direct Connect
pullSecret: '<PASTE PULL SECRET JSON HERE>'
sshKey: 'ssh-ed25519 AAAA... your-public-key'
```

Replace:
- `baseDomain` with your hosted zone name
- `region` with your region
- `pullSecret` with the JSON from Section 3.5 (single line, inside single quotes)
- `sshKey` with the contents of `~/.ssh/ocp-ha.pub`

Keep `replicas: 3` for both control plane and compute to get an HA cluster.

### Step 3: Back up install-config.yaml

The installer **consumes** `install-config.yaml` during `create cluster`. Keep a copy:

```bash
cp ~/ocp-ha-install/install-config.yaml ~/install-config.backup.yaml
```

### Step 4: Create the cluster

```bash
cd ~
./openshift-install create cluster --dir ~/ocp-ha-install --log-level=info
```

Expected flow:
1. Installer creates infrastructure (VPC, load balancers, IAM, Route 53 records).
2. Bootstrap node starts and brings up the control plane.
3. Control plane nodes join, then workers join.
4. Bootstrap is removed once the control plane is ready.
5. Final output shows the console URL and `kubeadmin` login details.

Expected time: about **35-60 minutes**. Errors are written to `~/ocp-ha-install/.openshift_install.log`.

When successful, you will see:

```
INFO Install Complete!
INFO Run 'export KUBECONFIG=.../auth/kubeconfig' to manage the cluster with 'oc'
INFO Login to the console with user: kubeadmin, password: xxxxx-xxxxx-xxxxx-xxxxx
```

### Step 5: Log in

```bash
export KUBECONFIG=~/ocp-ha-install/auth/kubeconfig
oc whoami
oc get nodes
```

Console login:
- URL: `https://console-openshift-console.apps.ocp-ha.aws.example.com`
- User: `kubeadmin`
- Password: `cat ~/ocp-ha-install/auth/kubeadmin-password`

Install the `oc` CLI from the same download page if `oc` is missing.

---

## 5. Verification checklist

Run these after install. Every item should pass before you consider the cluster healthy.

```bash
# 1. Nodes: 3 control plane + 3 worker, all Ready
oc get nodes -o wide

# 2. Cluster version: Available=True, Progressing=False
oc get clusterversion

# 3. Cluster operators: all Available=True, Degraded=False
oc get co

# 4. Machines (created by machine API)
oc get machines -n openshift-machine-api

# 5. Pods not running cleanly (should be empty or only Completed)
oc get pods -A | grep -vE 'Running|Completed'

# 6. Ingress controller
oc get ingresscontroller -n openshift-ingress-operator

# 7. Etcd members (should show 3 healthy members)
oc get etcd cluster -o jsonpath='{.status.conditions[?(@.type=="EtcdMembersAvailable")].status}{"\n"}'
```

Expected results:
- `oc get nodes`: 6 nodes, STATUS `Ready`. Control plane nodes show `control-plane,master`.
- `oc get co`: every operator `AVAILABLE=True`, `DEGRADED=False`.
- Console opens in browser and login works.

### Deploy a test app

```bash
oc new-project hello-ha
oc create deployment hello --image=docker.io/nginxinc/nginx-unprivileged:latest --port=8080
oc expose deployment hello --port=8080
oc expose svc/hello
oc get route hello
```

Open the route hostname in a browser. You should see the nginx welcome page.

Clean up the test:

```bash
oc delete project hello-ha
```

---

## 6. Post-install basics (recommended)

1. **Add an identity provider** (for example, HTPasswd) and create a real admin user.
   - Remove `kubeadmin` only after the new admin user works.
2. **Back up the kubeconfig and installer directory** to a secure location. Do not share them.
3. **Set up a Budget alarm** in AWS Billing so you know when costs rise.
4. Learn these next:
   - `oc adm must-gather` for support data
   - Cluster upgrades (`oc adm upgrade`)
   - Adding worker nodes with MachineSets

---

## 7. Cleanup (destroy the cluster)

When practice is done, destroy the cluster to stop charges:

```bash
cd ~
./openshift-install destroy cluster --dir ~/ocp-ha-install --log-level=info
```

Then check the AWS console for leftovers:
- EC2 instances (should be zero)
- Load balancers (ELB/NLB)
- NAT gateways and Elastic IPs
- Route 53 records in your hosted zone (installer records should be removed; the hosted zone itself stays)
- S3 bucket for ignition files
- IAM roles and instance profiles created for the cluster

After confirming cleanup, you may delete the installer state folder:

```bash
rm -rf ~/ocp-ha-install
```

Do not delete the Route 53 hosted zone if you plan to reinstall.

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `InsufficientInstanceCapacity` or instance type not found | Instance type not offered in that AZ or region | Check with `describe-instance-type-offerings` and change type |
| `VcpuLimitExceeded` | EC2 vCPU quota too low | Increase quota in Service Quotas |
| Install stops at "waiting for Kubernetes API" | API DNS record or NS delegation wrong | Check `dig api.ocp-ha.aws.example.com` and NS records |
| `AccessDenied` in log | IAM user lacks permissions | Attach required permissions or AdministratorAccess (lab only) |
| Bootstrap never completes | Network or time sync issue, or machine too small | Read `.openshift_install.log`; check instance sizing |
| Certificate or auth errors after install | Clock drift or wrong pull secret | Confirm pull secret is valid; check NTP on nodes |
| Cluster stuck during create | Quota, networking, or operator errors | Run `oc get co`, `oc get nodes`, and read installer log |

Useful commands:

```bash
# Read installer log
tail -n 200 ~/ocp-ha-install/.openshift_install.log

# Re-run wait commands if the install was interrupted
./openshift-install wait-for install-complete --dir ~/ocp-ha-install --log-level=info
```

---

## 9. Verification status of this guide

Some details could not be confirmed directly from the official page, so this table shows what was checked.

| Item | Status | Source |
|------|--------|--------|
| IPI is the default AWS installation method | Verified | Red Hat OpenShift 4.19 and 4.21 "Installing on AWS" |
| Public Route 53 hosted zone is required | Verified | Red Hat OpenShift "Installing on AWS" (Configuring Route 53) |
| IAM permissions required for installer | Verified (categories). Full list must be copied from docs. | Red Hat OpenShift "Installing on AWS" |
| Default service limits can block install | Verified | Red Hat OpenShift 4.19 and 4.21 |
| `m6i.*` is a tested AWS instance family | Verified | OKD 4.16-4.19 "Installing on AWS customizations" |
| install-config.yaml is consumed during install; back it up | Verified | OKD 4.16 "Installing on AWS customizations" |
| `openshift-install destroy cluster` and `--dir` usage | Verified | Installer project README |
| Kubeconfig at `auth/kubeconfig`; kubeadmin login | Verified | Installer project README |
| Minimum CPU/RAM per role for HA | **Not fully verified.** Newer docs show 8 vCPU / 16 GB for HA topology; per-role table not read. | openshift-docs change (github.com/openshift/openshift-docs) |
| Cost estimate | **Approximate.** Not checked against live AWS pricing. | Estimate |
| Mumbai (`ap-south-1`) instance availability | **Not verified.** Run the `describe-instance-type-offerings` command. | Check yourself |
| Install time (35-60 min) and `oc` checks | General experience, not from docs | General knowledge |
| `kubeadmin-password` file path | General knowledge, commonly used | General knowledge |

Before running this in production, confirm everything against the Red Hat documentation for your exact OpenShift version.

---

## 10. Sources

- Red Hat OpenShift Container Platform 4.21, Installing on AWS: https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html-single/installing_on_aws/index
- Red Hat OpenShift Container Platform 4.19, Installing on AWS: https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html-single/installing_on_aws
- OKD 4.16 Installing on AWS customizations: https://docs.okd.io/4.16/installing/installing_aws/ipi/installing-aws-customizations.html
- OKD 4.18 Installing on AWS network customizations: https://docs.okd.io/4.18/installing/installing_aws/ipi/installing-aws-network-customizations.html
- OpenShift Installer README (create and destroy commands): https://pkg.go.dev/github.com/openshift/installer
- Installer download and pull secret: https://console.redhat.com/openshift/install/aws/installer-provisioned
