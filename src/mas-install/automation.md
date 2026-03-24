---
title: Automated MAS Deployment
---

# :material-robot: 3. Automated MAS Deployment (IBM CLI)

Deploying MAS manually through OperatorHub requires installing over 15 individual operators (Cert Manager, SLS, UDS, MongoDB, DB2, Core, Manage, etc.) and precisely sequencing their custom resources. 

To simplify this, IBM provides an official DevOps automation container (`quay.io/ibmmas/cli`) which uses Ansible and Tekton Pipelines to orchestrate the entire deployment gracefully.

---

## 3.1 — Install Podman

The IBM MAS CLI runs inside a container. Install **Podman** on the Bastion host:

```bash
dnf install podman -y
```

Expected output:
```text
Package podman-2:5.1.2-1.el9.x86_64 is already installed.
Dependencies resolved.
================================================================================
 Package          Architecture     Version              Repository
================================================================================
Upgrading:
 podman           x86_64           2:5.2.2-1.el9        appstream

Transaction Summary
================================================================================
Upgrade  1 Package

...
Complete!
```

---

## 3.2 — Get the OpenShift Login Token

The MAS CLI container needs to authenticate with the OpenShift cluster API. Retrieve, in advance, the login token from the Web Console.

1. Log into the **OpenShift Web Console** (`https://console-openshift-console.apps.sno.ocp.local`).
2. Click your username (`kubeadmin`) in the top right corner.
3. Select **Copy login command**.
4. Authenticate again if prompted, then click **Display Token**.
5. Copy the **full `oc login` command** shown under "Log in with this token":

```bash
oc login --token=sha256~xxxxxxxxxxxxxxxxxxxxxxxxxxxx --server=https://api.sno.ocp.local:6443
```

!!! tip "Keep this token handy"
    You will need the **Server URL** and **Login Token** in the next step when the `mas install` wizard prompts you.

---

## 3.3 — Launch the MAS CLI Container

Run the official IBM MAS CLI container interactively. The `-v ~/:/mnt/home` flag mounts your home directory into the container so it can access your license and pull-secret files:

```bash
podman run -ti --rm -v ~/:/mnt/home --pull always quay.io/ibmmas/cli
```

!!! warning "Network Isolation Issue"
    If the container **cannot reach the OpenShift API** (e.g., connection timeouts or DNS failures during `mas install`), the default Podman network namespace is isolated from the host network.

    **Fix:** Exit the container and restart it with `--network host`:

    ```bash
    exit
    podman run -ti --rm --network host -v ~/:/mnt/home --pull always quay.io/ibmmas/cli
    ```

    This allows the container to use the Bastion's network stack directly, resolving DNS and reaching `api.sno.ocp.local:6443`.

---

## 3.4 — Copy the License File into the Container

Once inside the container, copy the MAS license file from your Bastion host using `scp`. The Bastion's **external IP** is used because the container (with `--network host`) shares the host's network:

```bash
scp 10.1.1.30:/root/license.dat .
```

You will be prompted to accept the SSH host key and enter the Bastion root password:

```text
The authenticity of host '10.1.1.30 (10.1.1.30)' can't be established.
ED25519 key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.1.1.30' (ED25519) to the list of known hosts.
root@10.1.1.30's password:
license.dat                                     100%  XXX     X.XKB/s   00:00
```

!!! note
    Replace `10.1.1.30` with your Bastion's actual external IP address. The license file will be copied to the current directory inside the container.

---

## 3.5 — Execute the MAS Installer

Once inside the container shell, trigger the interactive installation wizard:

```bash
mas install
```

### Step 1: Set Target OpenShift Cluster

The installer will first prompt you to connect to your cluster:

```text
IBM Maximo Application Suite Admin CLI v19.2.1
Powered by https://github.com/ibm-mas/ansible-devops/ and https://tekton.dev/

1) Set Target OpenShift Cluster
Server URL: https://api.sno.ocp.local:6443
Login Token: ****************************************
Disable TLS Verify? [y/n] y
```

* **Server URL:** `https://api.sno.ocp.local:6443`
* **Login Token:** Paste the token you copied in step 3.2
* **Disable TLS Verify:** `y` (required for self-signed certificates on SNO)

### Step 2: MAS Configuration Prompts

The installer will continue with questions to tailor your deployment. Answer them carefully:

* **MAS Instance ID:** `sno`
* **Use online catalog?** `y`
* **MAS Version:** Select `8.10` (or `8.9`/`8.11` as instructed)
* **Install Manage:** `y`
* **Customize database settings:** `y`
  * Choose to generate a dedicated JDBC configuration for Manage.
* **Configure Storage Class Usage:**
  * When prompted for the ReadWriteOnce (RWO) storage class, enter **`lvms-vg1`** (from our LVM operator).
* **Configure IBM Container Registry:** Paste the **Entitlement Key** you generated earlier.
* **Configure Product License:** Provide the path to the license file (`/mnt/home/license.dat` — since your home directory is mounted at `/mnt/home`).

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
