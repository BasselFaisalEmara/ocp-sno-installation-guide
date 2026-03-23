---
title: DNS Server (BIND)
---

# :material-dns: Step 2 — DNS Server (BIND)

OpenShift **requires** proper DNS resolution for API, etcd, and application ingress endpoints. The Bastion runs a BIND DNS server that serves both forward and reverse zones for the cluster.

---

## 2.1 — Install BIND

```bash
dnf install bind bind-utils -y
```

---

## 2.2 — Create Zone Directory

```bash
mkdir -p /etc/named/zones
```

---

## 2.3 — Configure BIND (`named.conf`)

Edit the main BIND configuration file:

```bash
vim /etc/named.conf
```

Replace the contents with the following:

```dns title="/etc/named.conf" linenums="1"
//
// named.conf — SNO Cluster DNS
//

acl AllowQuery {
        192.168.83.0/24;
        10.1.1.0/24;
        localhost;
        localnets;
};

options {
        listen-on port 53 { localnets; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { AllowQuery; };

        recursion yes;
        forwarders {
            10.1.1.1;           // (1)!
        };
        dnssec-validation no;
        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

// Forward zone for ocp.local
zone "ocp.local" {
    type master;
    file "/etc/named/zones/forward.sno.ocp.local";
};

// Reverse zone for 192.168.83.0/24
zone "83.168.192.in-addr.arpa" {
    type master;
    file "/etc/named/zones/reverse.sno.ocp.local";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

1.  :material-information: Replace `10.1.1.1` with your upstream DNS server / gateway IP for recursive resolution of external domains.

!!! info "Key Configuration Decisions"

    - **`allow-query`** — Restricts DNS queries to the internal subnet, external subnet, and localhost.
    - **`recursion yes`** — Enables recursive lookups so the SNO node can resolve external names (e.g., `quay.io`, `registry.redhat.io`).
    - **`forwarders`** — Points to the upstream DNS/gateway for names not in our zones.
    - **`dnssec-validation no`** — Disabled for lab simplicity. Enable in production.

---

## 2.4 — Forward Zone File

Create the forward zone file for `ocp.local`:

```bash
vim /etc/named/zones/forward.sno.ocp.local
```

```dns title="/etc/named/zones/forward.sno.ocp.local" linenums="1"
$TTL    604800
@       IN      SOA     bastion.ocp.local. root.ocp.local (
                  1     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800     ; Minimum
)
        IN      NS      bastion
bastion.ocp.local.          IN      A       192.168.83.10

; SNO Nodes
sno1.sno.ocp.local.         IN      A      192.168.83.20

; OpenShift Internal - Load balancer (pointed to Bastion/HAProxy)
api.sno.ocp.local.        IN    A    192.168.83.10
api-int.sno.ocp.local.    IN    A    192.168.83.10
*.apps.sno.ocp.local.     IN    A    192.168.83.10

; ETCD Cluster
etcd-0.sno.ocp.local.    IN    A     192.168.83.20

; OpenShift Internal SRV records (cluster name = sno)
_etcd-server-ssl._tcp.sno.ocp.local.    86400     IN    SRV     0    10    2380    etcd-0.sno

; Explicit application records
oauth-openshift.apps.sno.ocp.local.     IN     A     192.168.83.10
console-openshift-console.apps.sno.ocp.local.     IN     A     192.168.83.10
```

!!! warning "Critical Records"

    The following records are **mandatory** for OpenShift installation:

    | Record | Why It's Needed |
    |--------|-----------------|
    | `api.sno.ocp.local` | Kubernetes API server endpoint |
    | `api-int.sno.ocp.local` | Internal API used by nodes during bootstrap |
    | `*.apps.sno.ocp.local` | Wildcard for all application Routes (Ingress) |
    | `_etcd-server-ssl._tcp` SRV | etcd cluster member discovery |

---

## 2.5 — Reverse Zone File

Create the reverse zone file:

```bash
vim /etc/named/zones/reverse.sno.ocp.local
```

```dns title="/etc/named/zones/reverse.sno.ocp.local" linenums="1"
$TTL    604800
@       IN      SOA     bastion.ocp.local. root.ocp.local (
                  1     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800     ; Minimum
)

  IN      NS      bastion.ocp.local.

10      IN    PTR    bastion.sno.ocp.local.
10      IN    PTR    api.sno.ocp.local.
10      IN    PTR    api-int.sno.ocp.local.
;
20    IN    PTR    sno1.sno.ocp.local.
;
```

---

## 2.6 — Validate Configuration

Always validate DNS configuration files before starting the service:

=== "Check named.conf syntax"

    ```bash
    named-checkconf /etc/named.conf
    ```

    No output means success. ✅

=== "Check Reverse Zone"

    ```bash
    named-checkzone 83.168.192.in-addr.arpa /etc/named/zones/reverse.sno.ocp.local
    ```

    Expected:
    <div class="cmd-output">
    zone 83.168.192.in-addr.arpa/IN: loaded serial 1<br/>
    <span class="success">OK</span>
    </div>

=== "Check Forward Zone"

    ```bash
    named-checkzone sno.ocp.local /etc/named/zones/forward.sno.ocp.local
    ```

    Expected:
    <div class="cmd-output">
    zone sno.ocp.local/IN: loaded serial 1<br/>
    <span class="success">OK</span>
    </div>

---

## 2.7 — Open Firewall Ports

```bash
firewall-cmd --add-service=dns --zone=external --permanent
firewall-cmd --add-port=53/udp --zone=external --permanent
firewall-cmd --reload
```

!!! note

    DNS is already allowed on the **internal** zone by default. We add it to the **external** zone so the Bastion can also respond to DNS queries from the external network if needed.

---

## 2.8 — Enable and Start the Service

```bash
systemctl enable --now named.service
systemctl status named
```

Expected:
<div class="cmd-output">
● named.service - Berkeley Internet Name Domain (DNS)<br/>
&nbsp;&nbsp;&nbsp;Loaded: loaded<br/>
&nbsp;&nbsp;&nbsp;Active: <span class="success">active (running)</span>
</div>

---

## 2.9 — Verify DNS Resolution

Test forward resolution:

```bash
dig sno.ocp.local
dig api.sno.ocp.local +short
dig api-int.sno.ocp.local +short
dig console-openshift-console.apps.sno.ocp.local +short
```

Expected for `api.sno.ocp.local`:
<div class="cmd-output">
<span class="success">192.168.83.10</span>
</div>

Test reverse resolution:

```bash
dig -x 192.168.83.20 +short
```

Expected:
<div class="cmd-output">
<span class="success">sno1.sno.ocp.local.</span>
</div>

Test SRV record:

```bash
dig _etcd-server-ssl._tcp.sno.ocp.local SRV +short
```

Expected:
<div class="cmd-output">
0 10 2380 <span class="success">etcd-0.sno.ocp.local.</span>
</div>

!!! success "Checkpoint"

    DNS is fully operational. All forward, reverse, and SRV records resolve correctly.

---

**Next:** [:octicons-arrow-right-24: Step 3 — DHCP Server](dhcp.md)
