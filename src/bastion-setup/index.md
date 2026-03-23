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

---

## Next Steps

Proceed to configure each service in order:

1. [:octicons-arrow-right-24: Firewall Configuration](firewall.md)
2. [:octicons-arrow-right-24: DNS Server (BIND)](dns.md)
3. [:octicons-arrow-right-24: DHCP Server](dhcp.md)
4. [:octicons-arrow-right-24: HAProxy Load Balancer](haproxy.md)
