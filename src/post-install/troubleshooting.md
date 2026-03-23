---
title: Troubleshooting
---

# :material-bug: Troubleshooting Guide

Common issues encountered during SNO installation and their resolutions.

---

## DNS Issues

### Problem: `dig` returns no results or wrong IP

**Symptoms:**

- `dig api.sno.ocp.local` returns `NXDOMAIN` or `SERVFAIL`
- Installation hangs waiting for node to check in

**Resolution:**

=== "Check 1: BIND is running"

    ```bash
    systemctl status named
    ```

=== "Check 2: Firewall allows DNS"

    ```bash
    firewall-cmd --list-all --zone=internal | grep dns
    ```

=== "Check 3: Zone files have no syntax errors"

    ```bash
    named-checkconf /etc/named.conf
    named-checkzone ocp.local /etc/named/zones/forward.sno.ocp.local
    named-checkzone 83.168.192.in-addr.arpa /etc/named/zones/reverse.sno.ocp.local
    ```

=== "Check 4: Bastion uses itself for DNS"

    ```bash
    cat /etc/resolv.conf
    ```
    
    Should contain:
    ```
    nameserver 127.0.0.1
    # or
    nameserver 192.168.83.10
    ```

---

### Problem: Wildcard DNS (`*.apps`) not resolving

**Cause:** Missing wildcard A record in forward zone.

**Fix:** Ensure this line exists in `/etc/named/zones/forward.sno.ocp.local`:

```dns
*.apps.sno.ocp.local.     IN    A    192.168.83.10
```

After editing, restart BIND:

```bash
systemctl restart named
```

---

## DHCP Issues

### Problem: SNO node gets no IP address

**Symptoms:**

- Node hangs at network configuration during boot
- No lease entry in `/var/lib/dhcpd/dhcpd.leases`

**Resolution:**

1. Verify DHCP service is running:
    ```bash
    systemctl status dhcpd
    journalctl -u dhcpd -f
    ```

2. Confirm the MAC address in `dhcpd.conf` matches the node's actual NIC:
    ```bash
    grep "hardware ethernet" /etc/dhcp/dhcpd.conf
    ```

3. Ensure DHCP is listening on the correct interface:
    ```bash
    # Check if dhcpd is bound to ens224
    ss -ulnp | grep :67
    ```

4. Verify the subnet declaration matches the interface's IP range.

---

## HAProxy Issues

### Problem: API not reachable on port 6443

**Symptoms:**

- `oc login` or `oc get nodes` times out
- `curl -k https://api.sno.ocp.local:6443/version` fails

**Resolution:**

```bash
# Check HAProxy status
systemctl status haproxy

# Check SELinux
getsebool haproxy_connect_any
# Should return: haproxy_connect_any --> on

# Check firewall
firewall-cmd --list-ports --zone=internal
firewall-cmd --list-ports --zone=external
# Should include 6443/tcp

# Validate config
haproxy -c -f /etc/haproxy/haproxy.cfg

# Check backend health via stats
curl -s http://localhost:9000/stats | grep -i "sno1"
```

---

### Problem: HAProxy fails to start — "cannot bind socket"

**Cause:** SELinux is blocking HAProxy from binding to non-standard ports.

**Fix:**

```bash
setsebool -P haproxy_connect_any 1
systemctl restart haproxy
```

---

## Bootstrap Issues

### Problem: Installation stalls at "Waiting for bootstrap to complete"

**Symptoms:**

- `openshift-install wait-for bootstrap-complete` hangs for >45 minutes
- No progress in logs

**Resolution:**

1. **SSH into the node** and check:
    ```bash
    ssh core@192.168.83.20
    sudo journalctl -b -f -u bootkube.service
    ```

2. **Common sub-issues:**

    | Log Message | Cause | Fix |
    |-------------|-------|-----|
    | `connection refused on :6443` | API server not started | Wait, check etcd logs |
    | `etcd timeout` | DNS SRV record incorrect | Verify `_etcd-server-ssl._tcp` SRV record |
    | `failed to pull image` | No internet / pull secret issue | Check NAT, verify pull secret |
    | `x509: certificate` | Clock skew | Sync NTP on both Bastion and node |

3. **Check etcd specifically:**
    ```bash
    sudo crictl ps | grep etcd
    sudo crictl logs <etcd-container-id>
    ```

---

### Problem: Node can't reach the internet (failed to pull images)

**Cause:** NAT/masquerade not working on the Bastion.

**Fix:**

```bash
# On the Bastion
cat /proc/sys/net/ipv4/ip_forward    # Should be 1
firewall-cmd --query-masquerade --zone=external
firewall-cmd --query-masquerade --zone=internal

# Test from the node
ssh core@192.168.83.20
curl -s https://quay.io
```

If IP forwarding is disabled:

```bash
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
firewall-cmd --reload
```

---

## Certificate / Authentication Issues

### Problem: `x509: certificate signed by unknown authority`

**Cause:** The client doesn't trust the cluster's self-signed CA.

**Fix:** Use the `--insecure-skip-tls-verify` flag or export the kubeconfig:

```bash
export KUBECONFIG=~/ocp-install/auth/kubeconfig
oc get nodes
```

---

### Problem: Forgot kubeadmin password

**Fix:** It's stored in the auth directory:

```bash
cat ~/ocp-install/auth/kubeadmin-password
```

If the file is lost, you can reset it by creating a new `htpasswd` identity provider.

---

## General Debugging Commands

```bash
# Cluster events (recent issues)
oc get events -A --sort-by='.lastTimestamp' | tail -30

# Describe a failing pod
oc describe pod <pod-name> -n <namespace>

# Pod logs
oc logs <pod-name> -n <namespace>

# Node conditions
oc describe node sno1 | grep -A5 Conditions

# Resource usage
oc adm top nodes
oc adm top pods -A

# Must-gather (Red Hat support bundle)
oc adm must-gather
```

---

## Quick Reference: Service Restart Commands

| Service | Restart | Status | Logs |
|---------|---------|--------|------|
| DNS | `systemctl restart named` | `systemctl status named` | `journalctl -u named` |
| DHCP | `systemctl restart dhcpd` | `systemctl status dhcpd` | `journalctl -u dhcpd` |
| HAProxy | `systemctl restart haproxy` | `systemctl status haproxy` | `journalctl -u haproxy` |
| Firewall | `firewall-cmd --reload` | `firewall-cmd --state` | `journalctl -u firewalld` |
