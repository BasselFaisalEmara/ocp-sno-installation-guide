---
title: Assisted Installer Setup
---

# :material-cloud-check: Phase 2 — Assisted Installer Setup

This phase uses the Red Hat Hybrid Cloud Console to generate the cluster configuration and the discovery ISO that the SNO node will use to boot.

---

## 2.1 — Create Cluster via Assisted Installer

Instead of constructing YAML files manually, we will use the **Red Hat Hybrid Cloud Console** to generate our cluster configuration and discovery ISO.

1. Navigate to [console.redhat.com/openshift](https://console.redhat.com/openshift) and log in.
2. From the main menu, navigate to **Clusters** and click **Create cluster**.
3. Select the **Datacenter** tab.
4. Under **Assisted Installer**, click **Create cluster**.

---

## 2.2 — Cluster Details

Fill out the initial cluster specifications to match your DNS and network layout:

* **Cluster name:** `sno`
* **Base domain:** `ocp.local` 
  *(This will make the full cluster address `sno.ocp.local`)*
* **OpenShift version:** Select `OpenShift 4.14.x` (e.g., `4.14.36`)
* **CPU architecture:** `x86_64`
* **Number of control plane nodes:** `1` (This tells the installer it's a Single Node OpenShift deployment)
* **Hosts' network configuration:** `DHCP only`

Click **Next** to proceed.

---

## 2.3 — Networking Configuration

The Assisted Installer may present a **Networking** configuration step. Verify or set the following values:

| Setting | Value |
|---------|-------|
| **Machine CIDR** | `192.168.83.0/24` (auto-detected from DHCP) |
| **Service Network** | `172.30.0.0/16` (default — leave unchanged) |
| **Cluster Network** | `10.128.0.0/14` (default — leave unchanged) |
| **Network Type** | `OVNKubernetes` |

!!! note
    For SNO, the Assisted Installer typically auto-populates these from the DHCP-assigned address. Only modify them if your network layout differs from the defaults.

Click **Next** to proceed.

---

## 2.4 — Add Host and Generate Discovery ISO

1. On the **Host discovery** step, click the **Add hosts** button.
2. When prompted for SSH keys, paste the contents of your `id_ed25519.pub` public key that you generated on the Bastion host in the previous step.
3. Click **Generate Discovery ISO**.
4. Once generated, **Download the ISO** to your local machine.

---

## 2.5 — Upload ISO to VMware ESXi

We now need to attach this Discovery ISO to the SNO virtual machine.

1. Log in to your **VMware ESXi Host Client**.
2. Navigate to **Storage** → Select your datastore (e.g., `datastore1`) and click **Datastore browser**.
3. Create an `ISOs` directory if you don't have one, and click **Upload**.
4. Upload the `*-discovery.iso` file that you just downloaded from Red Hat.

---

## 2.6 — Boot the SNO Node

1. In the ESXi Client, select your SNO virtual machine (e.g., `SNO-Bassel`) and click **Edit settings**.
2. Locate the **CD/DVD drive 1**.
3. Change the setting to **Datastore ISO file** and select the discovery ISO you just uploaded.
4. Ensure the **Connect at power on** box is checked.
5. Save the settings and **Power On** the virtual machine.

---

## 2.7 — Troubleshooting Discovery Connectivity

The SNO node will boot the ISO, acquire an IP via DHCP, and attempt to register itself back to the Red Hat Hybrid Cloud Console. 

**Wait a few minutes.** If the host does not appear in the web console under the "Host discovery" page, it likely cannot reach the internet because of Bastion routing.

To verify, open the VM console in ESXi and test connectivity:

```bash
curl -4 -I https://console.redhat.com
```

If it fails to connect, the `firewalld` NAT policy on your Bastion host is not bridging the zones correctly.

!!! tip "Already configured?"
    If you followed the [Firewall Configuration](../bastion-setup/firewall.md#13--create-routing-policy-internal--external) guide earlier and created the `in_out_policy`, this should already be working. Verify with:

    ```bash
    firewall-cmd --info-policy=in_out_policy
    ```

    If the policy does not exist, go back and create it as described in [Step 1.3 — Create Routing Policy](../bastion-setup/firewall.md#13--create-routing-policy-internal--external).

After verifying the firewall policy, the SNO node should successfully reach the internet, phone home, and appear as a **Ready** host in your Red Hat web console!

!!! success "Checkpoint"

    The SNO node has been successfully discovered by the Assisted Installer. You are now ready to finalize the network and storage settings in the web UI and trigger the installation.

---

**Next:** [:octicons-arrow-right-24: Phase 3 — Bootstrap & Installation](bootstrap.md)
