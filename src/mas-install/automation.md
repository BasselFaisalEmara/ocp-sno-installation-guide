---
title: Automated MAS Deployment
---

# :material-robot: 3. Automated MAS Deployment (IBM CLI)

Deploying MAS manually through OperatorHub requires installing over 15 individual operators (Cert Manager, SLS, UDS, MongoDB, DB2, Core, Manage, etc.) and precisely sequencing their custom resources. 

To simplify this, IBM provides an official DevOps automation container (`quay.io/ibmmas/cli`) which uses Ansible and Tekton Pipelines to orchestrate the entire deployment gracefully.

---

## 3.1 — Install Container Runtime

The IBM MAS CLI runs inside a container. Ensure either **Podman** or **Docker** is installed on the Bastion host:

```bash
# Install Podman (recommended for RHEL)
dnf install podman -y

# Verify
podman --version
```

!!! tip "Docker vs Podman"
    All `podman` commands below are interchangeable with `docker`. On RHEL 8/9, Podman is pre-installed or available natively.

---

## 3.2 — Prepare the MAS CLI Container

Pull the official IBM MAS CLI container image and create a working instance:

```bash
mkdir ~/sno
cd ~/sno
podman pull quay.io/ibmmas/cli
```

Start the container in detached mode:

```bash
podman run -dit --name mas-installer quay.io/ibmmas/cli:latest bash
```

Create a configuration directory inside the container:

```bash
podman exec -it mas-installer bash -c "mkdir /mascli/masconfig"
```

---

## 3.3 — Inject Credentials

The automated installer requires the Red Hat Pull Secret and the IBM MAS License to proceed. We will inject the files you downloaded during the previous steps directly into the container.

```bash
# Assuming you saved the license in ~/mas-install/licenses/mas-license.dat
# Assuming your pull secret is at ~/pull-secret.json

podman cp ~/pull-secret.json mas-installer:/mascli/masconfig/pull-secret
podman cp ~/mas-install/licenses/mas-license.dat mas-installer:/mascli/masconfig/license.dat
```

---

## 3.4 — Authenticate with OpenShift

The container needs authorized access to the OpenShift cluster API. You can extract the `kube:admin` token from the web console.

1. Log into the **OpenShift Web Console**.
2. Click your username (`kubeadmin`) in the top right corner.
3. Select **Copy login command** and authenticate.
4. Copy the resulting `oc login --token=...` command.

Execute that command directly inside the container instance:

```bash
podman exec -it mas-installer bash
# Inside the container:
oc login --token=sha256~... --server=https://api.sno.ocp.local:6443 --insecure-skip-tls-verify
```

---

## 3.5 — Execute the Installer

While still inside the container's bash shell, trigger the interactive installation wizard:

```bash
mas install
```

### Critical Prompts to Expect:

The installer will ask questions to tailor your deployment. Answer them carefully:

* **MAS Instance ID:** `sno`
* **Use online catalog?** `y`
* **MAS Version:** Select `8.10` (or `8.9`/`8.11` as instructed)
* **Install Manage:** `y`
* **Customize database settings:** `y`
  * Choose to generate a dedicated JDBC configuration for Manage.
* **Configure Storage Class Usage:**
  * When prompted for the ReadWriteOnce (RWO) storage class, enter **`lvm-vg1`** (from our LVM operator).
* **Configure IBM Container Registry:** Paste the **Entitlement Key** you generated earlier.
* **Configure Product License:** Provide the path to the injected license (`/mascli/masconfig/license.dat`).

---

## 3.6 — Monitor via Tekton Pipelines

The CLI will install the **OpenShift Pipelines Operator (Tekton)** automatically and spin up PipelineRuns to execute your configuration.

To monitor progress live from the OpenShift Web Console:

1. Log into the **OpenShift Web Console**.
2. Navigate to **Pipelines** → **PipelineRuns** from the left sidebar.
3. At the top of the page, switch the **Project** dropdown to `mas-sno-pipelines` (the namespace format is `mas-<instance-id>-pipelines`).
4. Click on the active PipelineRun to see the task breakdown.
5. Click on any individual task to view its real-time logs.

!!! info "Expected Duration"
    Depending on your hardware and internet speed, a full MAS Manage installation on SNO takes approximately **2 to 3 hours**.

    | Stage | Approximate Time |
    |-------|-----------------|
    | Cert Manager + Common Services | 10–15 min |
    | MongoDB + SLS | 15–20 min |
    | DB2 Warehouse | 30–45 min |
    | MAS Core | 20–30 min |
    | MAS Manage (with demo data) | 30–60 min |

---

!!! success "MAS Deployment Complete!"
    Once the pipelines finish successfully, you will receive the Super User credentials in the terminal to access the MAS Administration portal.
