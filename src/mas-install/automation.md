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

The installer will walk you through **26 steps**. Below is the complete walkthrough with recommended answers for our SNO environment.

---

### Step 1: Set Target OpenShift Cluster

If you already logged in via `oc login` in step 3.3, the CLI will detect it:

```text
Already connected to OCP Cluster:
 https://console-openshift-console.apps.sno.ocp.local

Proceed with this cluster? [y/n] y
```

---

### Step 2: Choose Install Mode

Choose **Advanced** to have full control over all configuration options:

```text
Show advanced installation options? [y/n] y
```

---

### Step 3: IBM Maximo Operator Catalog Selection

Select the latest supported catalog. The installer will show available catalogs with their release dates:

```text
Select catalog v9-250624-amd64
Select release 9.1
```

!!! tip
    Always use the latest catalog unless your training instructions specify otherwise.

---

### Step 4: License Terms

Accept the IBM license agreements:

```text
Do you accept the license terms? [y/n] y
```

---

### Step 5: Configure Storage Class Usage

Enter our LVM StorageClass for **both** RWO and RWX:

```text
ReadWriteOnce (RWO) storage class lvms-vg1
ReadWriteMany (RWX) storage class lvms-vg1
```

!!! note
    On SNO with LVM, we use `lvms-vg1` for both access modes since there is only a single node.

---

### Step 6: Configure AppPoint Licensing

```text
SLS Mode 1
License file /mascli/license.dat
Contact e-mail address b.emara@maximo.com.sa
Contact first name bassel
Contact last name emara
IBM Data Reporter Operator (DRO) Namespace redhat-marketplace
```

* **SLS Mode:** `1` (Cluster-Shared License)
* **License file:** `/mascli/license.dat` (the file you copied via `scp` in step 3.4)
* Fill in your **contact details** as required.

---

### Step 7: Configure IBM Container Registry

Paste your **IBM Entitlement Key** (generated from the [IBM Container Library](https://myibm.ibm.com/products-services/containerlibrary)):

```text
IBM entitlement key ****************************************************
```

---

### Step 8: Configure MAS Instance

```text
Instance ID sno
Workspace ID sno
Workspace name single instance
```

---

### Step 9: Configure Operational Mode

For training/demo environments, select **Non-Production**:

```text
Operational Mode 2
```

---

### Step 10: Certificate Authority Trust

```text
Trust default CAs? [y/n] y
```

---

### Step 11: Cluster Ingress Secret Override

Leave empty (press Enter) to use the default:

```text
Cluster ingress certificate secret name
```

---

### Step 12: Configure Domain & Certificate Management

```text
Configure domain & certificate management? [y/n] n
```

---

### Step 13: Single Sign-On (SSO)

```text
Configure SSO properties? [y/n] n
```

---

### Step 14: Special Characters for User IDs

```text
Allow special characters for user IDs and usernames? [y/n] n
```

---

### Step 15: Enable Guided Tour

```text
Enable Guided Tour? [y/n] n
```

---

### Step 16: Application Selection

Select **only Manage** for our SNO deployment:

```text
Install IoT? [y/n] n
Install Manage? [y/n] y
Install Assist? [y/n] n
Install Optimizer? [y/n] n
Install Visual Inspection? [y/n] n
Install Real Estate and Facilities? [y/n] n
```

---

### Step 17: Configure Maximo Manage

#### 17.1 — Components

```text
Select components to enable? [y/n] n
```

This installs the default: **Base + Health**.

#### 17.2 — Server Bundles

```text
Select a server bundle configuration 1
```

Option `1` deploys the **"all"** server pod only (lower resource usage, ideal for SNO).

#### 17.3 — Database

```text
Customize database settings? [y/n] n
```

#### 17.4 — Customization

```text
Include customization archive? [y/n] n
```

#### 17.5 — Other Settings

```text
Configure Additional Settings? [y/n] n
```

---

### Step 18: Configure MongoDB

```text
Create MongoDb cluster using MongoDb Community Edition Operator? [y/n] y
MongoDb namespace mongoce
```

---

### Step 19: Configure Databases

#### 19.1 — Manage Database

Create a **dedicated Db2 Warehouse** instance for Manage:

```text
Create Manage dedicated Db2 instance using the IBM Db2 Universal Operator? [y/n] y
Select the Manage dedicated DB2 instance type 1
```

#### 19.2 — Installation Namespace

```text
Install namespace db2u
```

#### 19.3–19.5 — Node Affinity, CPU/Memory, Storage

Accept defaults for all:

```text
Configure node affinity? [y/n] n
Configure node tolerations? [y/n] n
Customize CPU and memory request/limit? [y/n] n
Customize storage capacity? [y/n] n
```

---

### Step 20: Configure Grafana

```text
Install namespace grafana5
Grafana storage size 10Gi
```

---

### Step 21: Configure Turbonomic

```text
Configure IBM Turbonomic integration? [y/n] n
```

---

### Step 22: Additional Configuration

```text
Use additional configurations? [y/n] n
```

---

### Step 23: Configure Pod Templates

```text
Use pod templates? [y/n] n
```

---

### Step 24: Non-Interactive Install Command

The CLI will display a reusable non-interactive command. **Save this for future re-installs:**

```bash
export IBM_ENTITLEMENT_KEY=x
mas install --mas-catalog-version v9-250624-amd64 \
  --ibm-entitlement-key $IBM_ENTITLEMENT_KEY \
  --mas-channel 9.1.x --mas-instance-id sno \
  --mas-workspace-id sno --mas-workspace-name "single instance" \
  --non-prod --disable-walkme \
  --storage-class-rwo "lvms-vg1" --storage-class-rwx "lvms-vg1" \
  --storage-pipeline "lvms-vg1" --storage-accessmode "ReadWriteOnce" \
  --license-file "/mascli/license.dat" \
  --uds-email "b.emara@maximo.com.sa" \
  --uds-firstname "bassel" --uds-lastname "emara" \
  --dro-namespace "redhat-marketplace" \
  --mongodb-namespace "mongoce" \
  --manage-channel "9.1.x" \
  --manage-jdbc "workspace-application" \
  --manage-components "base=latest,health=latest" \
  --manage-server-bundle-size "dev" \
  --db2-manage --db2-channel "v110509.0" \
  --db2-namespace "db2u" --db2-type "db2wh" \
  --accept-license --no-confirm
```

!!! tip "Save This Command"
    Copy this command and save it as a script on the Bastion. If you ever need to reinstall, you can skip all 23 prompts by running it directly.

---

### Step 25: Review Settings

The CLI will display a full summary of all your chosen settings. Review carefully, then proceed:

```text
Proceed with these settings? [y/n] y
```

---

### Step 26: Launch Install

The CLI submits a Tekton PipelineRun to orchestrate the entire installation:

```text
✅️ OpenShift Pipelines Operator is installed and ready to use
✅️ Namespace is ready (mas-sno-pipelines)
✅️ MAS CLI image deployment test completed
✅️ Latest Tekton definitions are installed (v14.1.0)
✅️ PipelineRun for sno install submitted
```

---

## 3.6 — Monitor via Tekton Pipelines

The CLI will provide a direct URL to track the PipelineRun:

```text
View progress:
  https://console-openshift-console.apps.sno.ocp.local/k8s/ns/mas-sno-pipelines/tekton.dev~v1beta1~PipelineRun/sno-install-260325-0900
```

You can also monitor from the OpenShift Web Console manually:

1. Log into the **OpenShift Web Console**.
2. Navigate to **Pipelines** → **PipelineRuns** from the left sidebar.
3. Switch the **Project** dropdown to `mas-sno-pipelines`.
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

