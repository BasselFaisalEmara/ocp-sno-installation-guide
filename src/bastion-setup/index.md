---
title: Bastion Host Setup
---

# :material-server-network: Bastion Host Setup

The Bastion Host is the **foundation of the SNO deployment**. It provides all infrastructure services required by both the OpenShift installer and the running cluster.

---

## Overview

Before the SNO node can be installed, the Bastion must be configured with the following services in order:

```mermaid
graph LR
    A["1. Firewall<br/>Zone & NAT"] --> B["2. DNS<br/>BIND Server"]
    B --> C["3. DHCP<br/>IP Assignment"]
    C --> D["4. HAProxy<br/>Load Balancer"]
    D --> E["Ready for<br/>OCP Install"]

    style A fill:#7c3aed,stroke:#6d28d9,color:#fff
    style B fill:#6366f1,stroke:#4f46e5,color:#fff
    style C fill:#06b6d4,stroke:#0891b2,color:#fff
    style D fill:#10b981,stroke:#059669,color:#fff
    style E fill:#f59e0b,stroke:#d97706,color:#fff
```

!!! warning "Order Matters"

    The services must be configured **in the order listed above**. DNS must be operational before DHCP, and all services must be running before the OpenShift installer is executed.

---

## What You'll Configure

| Step | Service | Purpose | Config Files |
|------|---------|---------|--------------|
| **1** | firewalld | Network zones, NAT, port forwarding | Runtime (firewall-cmd) |
| **2** | BIND DNS | Name resolution for cluster FQDNs | `named.conf`, zone files |
| **3** | ISC DHCP | Static IP via MAC reservation | `dhcpd.conf` |
| **4** | HAProxy | API & Ingress load balancing | `haproxy.cfg` |

---

## System Preparation

Before configuring services, set the hostname and verify network interfaces:

```bash
# Set the Bastion hostname
hostnamectl set-hostname bastion.ocp.local

# Verify
cat /etc/hostname
```

Expected output:
<div class="cmd-output">
bastion.ocp.local
</div>

Verify both NICs are present:

```bash
nmcli device status
```

You should see both `ens192` (external) and `ens224` (internal) interfaces listed.

### Verify NTP Time Synchronization

Clock skew between the Bastion and the SNO node can cause `x509: certificate has expired` errors during installation. Ensure Chrony (NTP) is syncing correctly:

```bash
# Check chrony status
chronyc tracking
timedatectl
```

If NTP is not synchronized, enable it:

```bash
systemctl enable --now chronyd
timedatectl set-ntp true
```

### Configure Network Interfaces (`nmtui`)

It is critical to configure the interfaces correctly so the Bastion uses its own DNS server and maintains static IPs. Run the NetworkManager Text User Interface:

```bash
nmtui
```

**1. Configure the External Interface (`ens192`)**

1. Select **Edit a connection** from the main menu.
2. Select the external interface (`ens192`) from the list and choose **&lt;Edit...&gt;**.
3. Under **IPv4 CONFIGURATION**, change the **DNS servers** to point to the local loopback address: `127.0.0.1`.
4. Scroll down and check the box for **`[X] Ignore automatically obtained DNS parameters`**. This prevents DHCP from overwriting your custom DNS settings.
5. Select **&lt;OK&gt;** to save.

**2. Configure the Internal Interface (`ens224`)**

1. Back in the connection list, select the internal interface (`ens224`) and choose **&lt;Edit...&gt;**.
2. Change the **IPv4 CONFIGURATION** to **&lt;Manual&gt;**.
3. Select **&lt;Show&gt;** to expand the IPv4 settings and add the internal IP address (`192.168.83.10/24`).
4. Select **&lt;OK&gt;** to save.
5. Select **&lt;Back&gt;** and then **&lt;Quit&gt;** to exit the utility.

**3. Apply the Changes**

Restart the network connections to apply the new configurations:

```bash
nmcli connection down ens192 && nmcli connection up ens192
nmcli connection down ens224 && nmcli connection up ens224
```

---

## Next Steps

Proceed to configure each service in order:

1. [:octicons-arrow-right-24: Firewall Configuration](firewall.md)
2. [:octicons-arrow-right-24: DNS Server (BIND)](dns.md)
3. [:octicons-arrow-right-24: DHCP Server](dhcp.md)
4. [:octicons-arrow-right-24: HAProxy Load Balancer](haproxy.md)
