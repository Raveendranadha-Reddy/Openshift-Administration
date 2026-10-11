# OpenShift Installation on AWS – Step-by-Step Notes

> **Installation method used in this guide:** Installer-Provisioned Infrastructure (IPI) on AWS.
> **Region used throughout:** `us-east-1`
> **User:** all OpenShift installer steps are run as `root` (`sudo su -`), so `aws configure` and `ssh-keygen` must also be done as `root`.

---

## Prerequisites (check before starting)

- A **Route 53 public hosted zone** for your base domain (or a delegated subdomain) in the same AWS account. Without it the installer fails at the base-domain prompt.
- Sufficient **EC2 vCPU / Elastic IP / VPC quotas** in `us-east-1`. A default cluster (3 control-plane + 3 worker nodes) is not free-tier friendly, so **destroy the cluster when you finish** (Step 19).
- A Red Hat account and an AWS account.

---

## Step 1: Create an RHEL 9 Virtual Machine

1. Create one virtual machine (VM) with Red Hat Enterprise Linux (RHEL) 9.
2. Verify the operating system:

```bash
cat /etc/redhat-release
```

---

## Step 2: Create a Red Hat Developer Account

1. Open https://developers.redhat.com/
2. Create a Red Hat Developer account.
3. Verify your account credentials.

---

## Step 3: Register RHEL 9 with Red Hat Subscription Management

Run the following command:

```bash
sudo subscription-manager register
```

Enter your Red Hat Developer account username and password when prompted.

Verify registration:

```bash
sudo subscription-manager status
```

Verify repository access:

```bash
sudo dnf repolist
```

> **Note:** Registration alone does not guarantee that software repositories are available. Verify repository access before installing packages.

---

## Step 4: Create an AWS Account

1. Open https://aws.amazon.com/
2. Create an AWS account.
3. Sign in to the AWS Management Console.
4. Select the AWS region you want to use. In this lab, the region is **us-east-1**.

---

## Step 5: Create an IAM User and Group

1. Open the AWS Management Console.
2. Navigate to **IAM → User groups**.
3. Create a group named `openshift-admin-group`.
4. Attach the required permissions. For a lab, `AdministratorAccess` is the simplest option. For tighter security, use the IAM permissions listed in the OpenShift documentation for IPI on AWS.
5. Create a user named `openshift-admin` and add the user to the group.
6. Open the user → **Security credentials** → **Create access key** (use case: *Command Line Interface*). Save the **Access Key ID** and **Secret Access Key** securely.

> Do **not** use root-account access keys.

---

## Step 6: Install AWS CLI (v2 recommended)

Download the AWS CLI v2 installation package:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

Extract the downloaded ZIP file:

```bash
unzip awscliv2.zip
```

Install AWS CLI:

```bash
sudo ./aws/install
```

Verify the installation:

```bash
aws --version
```

<details>
<summary>Legacy method (AWS CLI v1 bundle – not recommended for new installations)</summary>

```bash
curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"
unzip awscli-bundle.zip
sudo ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws
aws --version
```

</details>

---

## Step 7: Configure AWS CLI

Configure your AWS credentials and default settings (run as the same user that will run the installer, i.e. `root`):

```bash
aws configure
```

Provide the following details when prompted:

| Prompt | Value |
|---|---|
| AWS Access Key ID | `<your-access-key>` |
| AWS Secret Access Key | `<your-secret-key>` |
| Default region name | `us-east-1` |
| Default output format | `json` |

Check the AWS CLI configuration:

```bash
aws configure list
```

Example output:

```
Name       Value                Type
----       -----                ----
profile    <not set>            None
access_key ****************IWED shared-credentials-file
secret_key ****************tcqD shared-credentials-file
region     us-east-1            config-file
```

Test AWS authentication:

```bash
aws sts get-caller-identity
```

Check S3 access:

```bash
aws s3 ls
```

If the command succeeds, the configured identity can authenticate and perform the S3 listing operation permitted by its IAM policies. An empty result can simply mean that no S3 buckets are available to list.

---

## Step 8: Generate an SSH Key Pair

Create an ED25519 SSH key pair on your RHEL 9 system (as `root`):

```bash
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
```

The command creates two files:

```
/root/.ssh/id_ed25519      # Private key
/root/.ssh/id_ed25519.pub  # Public key
```

Verify the generated files:

```bash
ls -l ~/.ssh/id_ed25519*
```

Display the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

> **Important:** Keep the private key secure. Never share it or upload it to GitHub.
> **Note:** The key is saved as `id_ed25519` so the filename matches the ED25519 algorithm (not `id_rsa`).

---

## Step 9: Prepare the OpenShift Installation Directory

Switch to the root user (if you are not already root):

```bash
sudo su -
```

Create the installation directory:

```bash
mkdir -p /root/openshift
```

Navigate to the directory:

```bash
cd /root/openshift
```

Verify the current location:

```bash
pwd
```

Expected output:

```
/root/openshift
```

---

## Step 10: Download the OpenShift Installation Program

1. Open the Red Hat OpenShift installation portal: https://console.redhat.com/openshift/install
2. Select **Run it yourself**.
3. Select **AWS** as the platform.
4. Select the installation type: **Automated (Installer-Provisioned Infrastructure, IPI)**.
   > UPI (User-Provisioned Infrastructure) requires you to build the VPC, load balancers, and bootstrap/control-plane nodes manually, and uses a different workflow. It is not covered by `create cluster` in Step 13.
5. Download the OpenShift installation program or copy the download link provided on the page.

Navigate to the installation directory:

```bash
cd /root/openshift
```

Download the installer using the copied URL:

```bash
wget "<PASTE_INSTALLER_DOWNLOAD_URL>"
```

Extract the downloaded archive. Replace the filename with the actual downloaded file name:

```bash
tar -xvf openshift-install-linux.tar.gz
```

Verify the installer:

```bash
./openshift-install version
```

> **Note:** The downloaded archive name may differ depending on the OpenShift version and platform.

---

## Step 11: Download and Install the OpenShift CLI

From the same Red Hat installation page, download the OpenShift CLI package for Linux.

Navigate to the installation directory:

```bash
cd /root/openshift
```

Extract the CLI archive:

```bash
tar -xvf openshift-client-linux.tar.gz
```

Copy the CLI binaries into a directory in the system PATH:

```bash
install -m 0755 oc /usr/local/bin/oc
install -m 0755 kubectl /usr/local/bin/kubectl
```

Verify the installation:

```bash
oc version --client
kubectl version --client
```

The `oc` command is the OpenShift CLI. It supports OpenShift-specific operations and most standard Kubernetes CLI operations. The `kubectl` command is the standard Kubernetes CLI.

> **Note:** The exact archive filename depends on the version you download. Use `ls -l` to identify the extracted files.

---

## Step 12: Generate the OpenShift Installation Configuration (install-config.yaml)

Run the following command from the installation directory:

```bash
cd /root/openshift
./openshift-install create install-config --dir=/root/openshift
```

The installer prompts you for the following details.

**a) SSH Public Key**
Select the SSH public key created in Step 8. This key allows you to access cluster machines for troubleshooting when SSH access is configured.

```bash
cat /root/.ssh/id_ed25519.pub
```

**b) Platform**
Choose **aws** as the target platform for deploying the OpenShift cluster.

**c) AWS Region**
Select:

```
us-east-1
```

> Make sure this matches the region configured in Step 7.

**d) Base Domain**
Enter a domain that you own or control (a Route 53 public hosted zone), or a delegated subdomain configured for the installation.

```
example.com
```

The cluster's DNS name is based on the cluster name and base domain.

**e) Cluster Name**
Choose a name for your OpenShift cluster. Example:

```
openshift-cluster
```

The resulting cluster domain will look like:

```
openshift-cluster.example.com
```

The web console will be available at:

```
https://console-openshift-console.apps.openshift-cluster.example.com
```

**f) Pull Secret**
Obtain the pull secret from the Red Hat Hybrid Cloud Console: https://console.redhat.com/openshift/install

The pull secret authenticates the cluster against the required Red Hat container image registries. Copy the complete pull-secret JSON into the installer prompt. **Do not share it publicly or commit it to Git.**

**Other settings:** the installer may ask for additional settings depending on the OpenShift version.

> **Back up `install-config.yaml` now.** The installer consumes (deletes) this file when you run `create cluster`:
>
> ```bash
> cp /root/openshift/install-config.yaml /root/install-config.yaml.bak
> ```

---

## Step 13: Create the OpenShift Cluster

Run the following command from the directory containing the OpenShift installer:

```bash
cd /root/openshift

./openshift-install create cluster \
  --dir=/root/openshift \
  --log-level=info
```

This command starts the cluster installation process. The installer provisions the required AWS infrastructure and configures the OpenShift cluster according to the installation configuration.

> **Important:** `create cluster` is the **IPI** workflow. If you chose **UPI**, you must provision the AWS infrastructure separately (VPC, load balancers, bootstrap and control-plane nodes, ignition files, etc.). Do not assume this command completes a UPI installation.

Wait for the installation to finish. It can take 40–60 minutes or longer, depending on the environment.

> Keep the whole `/root/openshift` directory. It contains the state files required by `destroy cluster` and the `auth/` credentials.

---

## Step 14: Configure the KUBECONFIG Environment Variable

`KUBECONFIG` tells the OpenShift CLI which cluster configuration and credentials to use.

**A) Permanent configuration**

Append the following line to the root user's `.bashrc` file:

```bash
echo 'export KUBECONFIG=/root/openshift/auth/kubeconfig' >> /root/.bashrc
```

Apply the configuration immediately:

```bash
source /root/.bashrc
```

**B) Temporary configuration**

This setting applies only to the current shell session:

```bash
export KUBECONFIG=/root/openshift/auth/kubeconfig
```

---

## Step 15: Verify OpenShift Cluster Nodes

List the cluster nodes using `oc`:

```bash
oc get nodes
```

Alternatively, use `kubectl`:

```bash
kubectl get nodes
```

Display additional node details:

```bash
oc get nodes -o wide
```

Check the cluster operators:

```bash
oc get clusteroperators
```

Check the cluster version:

```bash
oc get clusterversion
```

---

## Step 16: Retrieve the OpenShift Console Administrator Password

Display the initial `kubeadmin` password:

```bash
cat /root/openshift/auth/kubeadmin-password
```

Use this password with the username `kubeadmin` to log in to the web console, provided the account is still enabled.

> Treat this password as sensitive information.

---

## Step 17: Find the OpenShift Web Console Route

Run:

```bash
oc get routes -n openshift-console | grep console
```

To retrieve the console URL directly:

```bash
oc get route console \
  -n openshift-console \
  -o jsonpath='{.spec.host}{"\n"}'
```

Open the resulting HTTPS URL in your browser and log in with your configured credentials.

---

## Step 18: Check OpenShift Installation Logs

View the installation log:

```bash
cat /root/openshift/.openshift_install.log
```

Follow the log while troubleshooting:

```bash
tail -f /root/openshift/.openshift_install.log
```

Check the current installer version:

```bash
./openshift-install version
```

---

## Step 19: Destroy the OpenShift Cluster

> **Warning:** Destroying a cluster deletes AWS infrastructure and causes data loss. Back up important data and verify the target cluster before proceeding.

Navigate to the installation directory:

```bash
cd /root/openshift
```

For a cluster created using installer-provisioned infrastructure (IPI), run:

```bash
./openshift-install destroy cluster \
  --dir=/root/openshift \
  --log-level=info
```

The installer uses the installation configuration and state files in the specified directory to identify resources to remove.

For UPI deployments, cleanup may require deleting AWS resources manually because the installer does not necessarily manage all infrastructure resources.

---

## Final Verification Checklist

1. Confirm all nodes are `Ready` using `oc get nodes`.
2. Confirm cluster operators are healthy using `oc get clusteroperators`.
3. Verify the console route using `oc get routes -n openshift-console`.
4. Store the kubeconfig and administrator credentials securely.
5. Confirm the installation type (IPI vs UPI) before creating or destroying the cluster.

---

## Summary of Corrections Made

| # | Issue in original notes | Fix |
|---|---|---|
| 1 | AWS CLI configured before AWS account / IAM user existed | Reordered: AWS account → IAM user + access keys → install CLI → `aws configure` |
| 2 | "Step 8" duplicated; "Configure AWS CLI" repeated | Merged into one configure step (Step 7) |
| 3 | Step 13 duplicated (short + detailed) | Merged into one Step 12 |
| 4 | Step 14 had no title; cluster domain line was empty | Titled "Create the OpenShift Cluster"; added example domain and console URL |
| 5 | UPI selected but `create cluster` (IPI command) used | Changed selection to IPI; kept UPI notes as warnings |
| 6 | Region mismatch (`us-east-1` vs `us-east-2`) | `us-east-1` used throughout |
| 7 | Root vs non-root path mix for SSH key / AWS credentials | All steps run as `root` consistently |
| 8 | Legacy AWS CLI v1 bundle | AWS CLI v2 as main method; v1 kept in a collapsible block |
| 9 | Missing prerequisites | Added Route 53 hosted zone, quotas, IAM access keys, `install-config.yaml` backup |
| 10 | Leftover correction note in KUBECONFIG step | Removed |
