---
title: LVM Storage Operator
---

# :material-harddisk: 1. Add Storage & LVM Storage Operator

IBM Maximo Application Suite (MAS) and its underlying databases (`DB2`, `MongoDB`, `Kafka`) require dynamic block and file storage provisioning. In a Single Node OpenShift environment, the easiest way to provide this is by adding a secondary virtual disk and deploying the **LVM Storage Operator**.

---

## 1.1 — Add a 500GB Disk to the SNO VM

1. Open your **VMware ESXi Host Client** (or vCenter).
2. Right-click the **SNO-Bassel** virtual machine and select **Edit settings**.
3. Under the **Virtual Hardware** tab, click **Add hard disk** -> **New standard hard disk**.
4. Set the size to **500 GB** and ensure it uses "Thin provisioned" allocation (if preferred for lab environments).
5. Click **Save**.

The SNO node will instantly detect the new block device without requiring a reboot.

---

## 1.2 — Verify the Disk from the CLI

SSH into your SNO node (or use the web console terminal) to ensure the 500GB disk is visible:

```bash
# Log in using the Bastion SSH key
ssh -i ~/.ssh/id_ed25519 core@192.168.83.20

# Run lsblk to list the drives
lsblk
```

Expected output:
```text
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0  400G  0 disk                      <- The primary OS disk
├─sda1   8:1    0    1M  0 part 
├─sda2   8:2    0  127M  0 part 
├─sda3   8:3    0  384M  0 part /boot
└─sda4   8:4    0  399G  0 part /var/lib/containers
sdb      8:16   0  500G  0 disk                      <- The new unformatted disk
```

Note the identifier for the new disk. It should be `sdb` or `nvme1n1`, depending on your hypervisor controller.

---

## 1.3 — Install the LVM Storage Operator

1. Log into the **OpenShift Web Console** (`https://console-openshift-console.apps.sno.ocp.local`) as `kubeadmin`.
2. On the left menu, navigate to **Operators** → **OperatorHub**.
3. Search for **LVM Storage**.
4. Click on the Red Hat-provided **LVM Storage** operator and click **Install**.
   - **Update Channel:** `stable-4.14` (or whatever matches your cluster)
   - **Installation Mode:** `All namespaces on the cluster`
   - **Installed Namespace:** `openshift-storage`
5. Click **Install** and wait for the status to show **Succeeded**.

---

## 1.4 — Create the LVMCluster to Manage the Disk

Once the Operator is running, we need to instruct it to format and divide that 500GB disk into dynamically provisionable Persistent Volumes.

1. Navigate to **Operators** → **Installed Operators** and click the **LVM Storage** operator.
2. In the top tabs, click **LVMCluster**.
3. Click **Create LVMCluster**.
4. Switch to the **YAML view** instead of the form view and paste the following manifest:

```yaml title="lvmcluster.yaml"
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: lvmcluster
  namespace: openshift-storage
spec:
  storage:
    deviceClasses:
      - name: vg1
        deviceSelector:
          paths:
            - /dev/sdb                     # (1)!
        thinPoolConfig:
          name: thin-pool-1
          sizePercent: 90
          overprovisionRatio: 10
```

1.  :material-alert: **Modify this path** to match the 500GB disk you found in step 1.2 (e.g., `/dev/sdb` or `/dev/nvme1n1`).

5. Click **Create**.

---

## 1.5 — Verify the Storage Classes

The `LVMCluster` will automatically format `/dev/sdb`, create a Logical Volume thin-pool, and install two Kubernetes StorageClasses required by MAS.

Verify the installation via the CLI or web console:

```bash
oc get storageclasses
```

Expected output:
```text
NAME                 PROVISIONER         RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
lvm-vg1 (default)    topolvm.io          Delete          WaitForFirstConsumer   true                   2m
lvm-vg1-snapshot     snap.topolvm.io     Delete          WaitForFirstConsumer   true                   2m
```

!!! success "Storage Ready"
    The cluster now can fulfill dynamic PersistentVolumeClaims. When MAS installs its databases, OpenShift will automatically carve chunks of the 500GB disk for those databases. 

---

## 1.6 — Create the Image Registry PVC

Before enabling the registry, we must prepare a PersistentVolumeClaim backed by our LVM storage so the registry has somewhere to store images.

1. Create the PVC manifest file:

    ```bash
    vi registry-pv.yaml
    ```

2. Paste the following content:

    ```yaml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: image-registry-storage
      namespace: openshift-image-registry
      annotations:
        imageregistry.openshift.io: 'true'
        volume.beta.kubernetes.io/storage-provisioner: topolvm.cybozu.com
        volume.kubernetes.io/storage-provisioner: topolvm.cybozu.com
      finalizers:
        - kubernetes.io/pvc-protection
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 200Gi
      storageClassName: lvms-vg1
      volumeMode: Filesystem
    ```

3. Apply it:

    ```bash
    oc create -f registry-pv.yaml
    ```

    Expected output:
    ```text
    persistentvolumeclaim/image-registry-storage created
    ```

---

## 1.7 — Enable the OpenShift Image Registry

By default, Single Node OpenShift deployments on bare metal/vSphere have the local image registry completely disabled. **MAS requires this registry to be active for its internal container pipelines.**

Now that the PVC is ready, edit the registry operator configuration to enable it:

```bash
oc edit configs.imageregistry.operator.openshift.io
```

Make three specific modifications in the YAML editor:

* **Set `managementState`** from `Removed` to `Managed`:

    ```yaml
    managementState: Removed
    ```
    to
    ```yaml
    managementState: Managed
    ```

* **Set `rolloutStrategy`** from `RollingUpdate` to `Recreate`:

    ```yaml
    rolloutStrategy: RollingUpdate
    ```
    to
    ```yaml
    rolloutStrategy: Recreate
    ```

* **Set `storage`** — replace the empty block with a PVC claim:

    ```yaml
    storage: {}
    ```
    to
    ```yaml
    storage:
      pvc:
        claim: ''
    ```

Save and quit the editor. The registry operator will bind the PVC we created and start the registry pods.

Verify:

```bash
oc get pods -n openshift-image-registry
oc get pvc -n openshift-image-registry
```

The PVC should show as **Bound** and the registry pods should be **Running**.

---

## 1.8 — Set LVM StorageClass as Default

MAS and its dependencies expect a **default** `StorageClass` to be available. If the LVM StorageClass is not already marked as default, set it now:

```bash
oc patch storageclass lvms-vg1 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Verify:

```bash
oc get storageclasses
```

You should see `(default)` next to `lvms-vg1`.

---

## 1.9 — Troubleshoot Image Registry PVC

If the `image-registry-storage` PVC is stuck in **Pending** status after enabling the registry, it is usually caused by the PVC requesting `ReadWriteMany` (RWX) access mode, which LVM does not support.

1. Check the PVC status:
    ```bash
    oc get pvc -n openshift-image-registry
    ```

2. If status is `Pending`, delete the existing PVC:
    ```bash
    oc delete pvc image-registry-storage -n openshift-image-registry
    ```

3. Recreate it using the corrected `registry-pv.yaml` from section 1.6 above:
    ```bash
    oc create -f registry-pv.yaml
    ```

The PVC should immediately transition to **Bound** status.

!!! warning "Still Not Bound?"
    If the PVC remains in Pending, the LVM thin pool may be full or misconfigured. Try uninstalling and reinstalling the LVM Operator, then wiping and re-adding the disk.

---

**Next:** [:octicons-arrow-right-24: 2. Entitlement & License File](licensing.md)

