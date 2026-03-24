---
title: Entitlement & License
---

# :material-key: 2. Entitlement Key & License Data

Deploying the **IBM Maximo Application Suite (MAS)** requires two distinct licensing artifacts:

1. **IBM Entitlement Key** — Used to authenticate against the IBM Container Registry (`cp.icr.io`) to download the proprietary container images.
2. **MAS License File (`.dat`)** — Loaded into the Suite Service to allocate and track your AppPoints (Concurrent or Authorized users).

---

## 2.1 — Obtain Your IBM Entitlement Key

You must have an active IBM account associated with your organization's Passport Advantage or Container Library entitlements.

1. Navigate to the **IBM Container Library**:
   [https://myibm.ibm.com/products-services/containerlibrary](https://myibm.ibm.com/products-services/containerlibrary)
2. Log in using your IBMid.
3. On the left navigation pane, click **Get entitlement key**.
4. Read and accept the license agreements if prompted.
5. Locate the **Access your container software** section and click **Copy key**.

!!! warning "Secure the Key"
    Treat this key like a master password. It provides access to download any IBM software your organization is licensed for. We will store this key in an OpenShift `Secret` in the `mas-[instance-id]-core` namespace during installation.

---

## 2.2 — Download the MAS License File (`.dat`)

The MAS cluster operates on an **AppPoint** model. A specific file containing these AppPoint allocations must be generated for the cluster. This file relies on the MAC address or host ID of the cluster/OpenShift node.

1. Navigate to the **IBM Rational License Key Center (LKC)**.
2. Log in using your IBMid.
3. Generate a new license key specific to Maximo Application Suite. 
   *(Note: The system may ask you for a Host ID. Since we are deploying MAS in OpenShift, the license is typically bound to the **Cluster ID** or the specific host ID of the internal license server).*
4. Once generated, **Download the `.dat` file** to your Bastion host.

---

## 2.3 — Upload License to the Bastion Host

To prepare for the MAS CLI installation script, place the `.dat` file in an accessible directory on your Bastion host:

```bash
# Example directory structure for MAS installation
mkdir -p ~/mas-install/licenses
```

Use `scp` or a file transfer tool to move the downloaded `.dat` file into `~/mas-install/licenses/mas-license.dat`.

```bash
scp -i ~/.ssh/id_ed25519 ~/Downloads/mas-license.dat core@192.168.83.20:~/mas-install/licenses/
```

Verify the file is present:

```bash
ls -l ~/mas-install/licenses/
```

---

## Checklist

Before running any MAS deployment scripts or operators, ensure you have:

- [x] A 500GB volume mounted and managed by the **LVM Storage Operator** (producing valid `StorageClasses`).
- [x] A valid **IBM Entitlement Key** saved securely.
- [x] A valid MAS **AppPoint License `.dat` file** saved on the Bastion or accessible during installation.

!!! success "Ready for MAS!"
    The infrastructure is now 100% capable of supporting IBM Maximo Application Suite. You can proceed with running the MAS CLI installer or deploying the MAS Operators via the OpenShift Web Console.
