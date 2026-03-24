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

## 2.3 — Add Host and Generate Discovery ISO

1. On the **Host discovery** step, click the **Add hosts** button.
2. When prompted for SSH keys, paste the contents of your `id_ed25519.pub` public key that you generated on the Bastion host in the previous step.
3. Click **Generate Discovery ISO**.
4. Once generated, **Download the ISO** to your local machine.

---

## 2.4 — Upload ISO to VMware ESXi

We now need to attach this Discovery ISO to the SNO virtual machine.

1. Log in to your **VMware ESXi Host Client**.
2. Navigate to **Storage** → Select your datastore (e.g., `datastore1`) and click **Datastore browser**.
3. Create an `ISOs` directory if you don't have one, and click **Upload**.
4. Upload the `*-discovery.iso` file that you just downloaded from Red Hat.

---

## 2.5 — Boot the SNO Node

1. In the ESXi Client, select your SNO virtual machine (e.g., `SNO-Bassel`) and click **Edit settings**.
2. Locate the **CD/DVD drive 1**.
3. Change the setting to **Datastore ISO file** and select the discovery ISO you just uploaded.
4. Ensure the **Connect at power on** box is checked.
5. Save the settings and **Power On** the virtual machine.

---

## 2.6 — Troubleshooting Discovery Connectivity

The SNO node will boot the ISO, acquire an IP via DHCP, and attempt to register itself back to the Red Hat Hybrid Cloud Console. 

**Wait a few minutes.** If the host does not appear in the web console under the "Host discovery" page, it likely cannot reach the internet because of Bastion routing.

To verify, open the VM console in ESXi and test connectivity:

```bash
curl -4 -I https://console.redhat.com
```

If it fails to connect, the `firewalld` NAT policy on your Bastion host is not bridging the zones correctly. Apply the following fix on your **Bastion host**:

```bash
# 1. Create a new routing policy bridging the zones
sudo firewall-cmd --new-policy int_to_ext --permanent

# 2. Tell the policy that traffic comes IN from the internal zone (SNO)
sudo firewall-cmd --policy int_to_ext --add-ingress-zone internal --permanent

# 3. Tell the policy that traffic goes OUT to the external zone (Internet)
sudo firewall-cmd --policy int_to_ext --add-egress-zone external --permanent

# 4. Set the policy to ACCEPT and route the traffic
sudo firewall-cmd --policy int_to_ext --set-target ACCEPT --permanent

# 5. Reload the firewall to apply the new bridge
sudo firewall-cmd --reload
```

After applying these firewall rules, the SNO node should successfully reach the internet, phone home, and appear as a **Ready** host in your Red Hat web console!

!!! success "Checkpoint"

    The SNO node has been successfully discovered by the Assisted Installer. You are now ready to finalize the network and storage settings in the web UI and trigger the installation.

---

**Next:** [:octicons-arrow-right-24: Phase 3 — Bootstrap & Installation](bootstrap.md)
