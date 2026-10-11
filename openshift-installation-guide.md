# OpenShift Installation Guide

OpenShift installation is best understood from three perspectives:

| # | Perspective | Question it answers |
|---|-------------|---------------------|
| 1 | Installation Methods | **How** do we install the cluster? |
| 2 | Deployment Platforms | **Where** do we install the cluster? |
| 3 | Cluster Architectures | **How** do we design the cluster? |

## Installation vs Cluster Architecture

Installation methods (IPI, UPI, Assisted Installer, Agent-based Installer) describe **how OpenShift is installed**.
Cluster architectures (SNO, Compact, Highly Available) describe **how nodes and their roles are organized**.

SNO (Single Node OpenShift) is therefore a cluster architecture, not an installation method. Any installation approach can be combined with any supported architecture.

![Installation vs Cluster Architecture](images/04-installation-vs-architecture.png)

| Example combination | Result |
|---------------------|--------|
| IPI + SNO | Single Node cluster installed with IPI (on supported platforms) |
| Agent-based Installer + Compact | Compact cluster installed with the Agent-based Installer |
| UPI + Highly Available | HA cluster installed with UPI |

---

## 1. Installation Methods

![OpenShift Installation Methods](images/03-installation-methods.png)

| Method | Meaning | Best use case |
|--------|---------|---------------|
| **IPI** – Installer-Provisioned Infrastructure | Installer provisions the infrastructure automatically | Production, AWS, supported platforms |
| **UPI** – User-Provisioned Infrastructure | User creates and configures the infrastructure manually | Advanced production, custom networking |
| **Assisted Installer** | Guided installation through a Web UI/API | Bare metal, supported platforms |
| **Agent-based Installer** | Generate an Agent ISO, boot the hosts from it, and install | Restricted networks, bare metal |
| **Disconnected / local workflows** | Workflows for environments with limited or no internet access (typically using a local mirror registry) | Restricted enterprise environments |

### IPI – Installer-Provisioned Infrastructure

The OpenShift installer creates and manages most of the required infrastructure.

1. Provisions infrastructure (VMs, networking, load balancers) depending on the platform.
2. Configures the cluster components.
3. Installs and brings up the cluster.

Example: installing on AWS.

```bash
openshift-install create cluster --dir=ocp-cluster
```

### UPI – User-Provisioned Infrastructure

The user or infrastructure team prepares everything before installing.

1. Create VMs or physical servers.
2. Configure networking, DNS, load balancers, and storage.
3. Generate the installation config (`install-config.yaml`), manifests, and ignition files.
4. Boot the machines (bootstrap, control plane, workers) and complete the installation.

Example: VMware or bare-metal servers with manually provisioned infrastructure.

### IPI vs UPI

![IPI vs UPI](images/05-ipi-vs-upi.png)

| Feature | IPI | UPI |
|---------|-----|-----|
| Full form | Installer-Provisioned Infrastructure | User-Provisioned Infrastructure |
| Infrastructure setup | Mostly automated | Mostly manual |
| VM provisioning | Installer-managed | User-managed |
| Networking | Installer configures supported components | User prepares required networking |
| Load balancer | Installer-managed where supported | User generally configures it |
| Installation complexity | Lower | Higher |
| Infrastructure control | Less granular | More granular |
| Best suited for | Standardized deployments | Custom enterprise environments |

---

## 2. Deployment Platforms

![OpenShift Deployment Platforms](images/02-deployment-platforms.png)

| Platform | Installation options / model |
|----------|------------------------------|
| AWS | IPI, UPI |
| Bare Metal Servers | IPI, UPI, Assisted, Agent-based |
| VMware vSphere | IPI, UPI, Assisted Installer, Agent-based |
| Microsoft Azure | IPI, UPI |
| Google Cloud Platform (GCP) | IPI, UPI |
| Red Hat OpenStack Platform | IPI, UPI |
| Nutanix | Depends on platform/version support |
| IBM Power / IBM Z | Architecture-specific installation methods |
| OpenShift Local (CRC) | Single-node environment on a developer laptop/local machine |
| ROSA | Red Hat OpenShift Service on AWS (managed service) |
| ARO | Azure Red Hat OpenShift (managed service) |
| OpenShift Dedicated | Red Hat-managed OpenShift service |

---

## 3. Cluster Architectures and Deployment Models

![OpenShift Cluster Deployment Models](images/01-cluster-deployment-models.png)

| Model | Description | Typical use cases |
|-------|-------------|-------------------|
| **Single Node OpenShift (SNO)** | Control plane and workloads on one node | Development/testing, edge, low-resource environments |
| **Compact cluster** | 3 nodes that run control plane and workloads together | Small production, space/cost-optimized sites |
| **Standard HA cluster** | 3 control-plane nodes + worker nodes | Production, high availability, scalability |
| **Hosted Control Planes (HCP)** | Control plane runs separately in a hosting cluster; workers run in other clusters | Multi-cluster management, large-scale enterprise |
| **Edge deployment** | Small clusters (SNO/compact) at remote or branch locations | Branch sites, IoT, low-resource environments |
| **Disconnected / air-gapped** | No internet or restricted network | Government, regulated enterprises |
| **Managed OpenShift** | Control plane and management shared with the cloud provider and Red Hat (ROSA, ARO, OpenShift Dedicated) | Reduced operational overhead, enterprise SLA, large-scale production |

> Note: SNO, Compact, Standard HA, and HCP are true architectures. Edge, Disconnected, and Managed describe the deployment scenario or operating model and can be combined with those architectures.
