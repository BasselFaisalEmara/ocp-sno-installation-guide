---
title: Validation & Smoke Tests
---

# :material-clipboard-check: Validation & Smoke Tests

After successful installation, run through these validation checks to confirm every component of the cluster is healthy.

---

## Pre-Flight: Set KUBECONFIG

```bash
export KUBECONFIG=~/ocp-install/auth/kubeconfig
```

---

## 1. Node Health

```bash
oc get nodes -o wide
```

??? success "Expected Output"

    ```
    NAME   STATUS   ROLES                         AGE   VERSION    INTERNAL-IP      OS-IMAGE
    sno1   Ready    control-plane,master,worker    1h    v1.27.x   192.168.83.20    RHCOS 4.14
    ```

Key checks:

- [x] Status is `Ready`
- [x] Roles include `control-plane`, `master`, **and** `worker`
- [x] Internal IP matches DHCP assignment (`192.168.83.20`)

---

## 2. Cluster Version

```bash
oc get clusterversion
```

- [x] `AVAILABLE` = `True`
- [x] `PROGRESSING` = `False`
- [x] No error messages in `STATUS`

---

## 3. Cluster Operators

```bash
oc get clusteroperators
```

All operators should show:

| Column | Expected |
|--------|----------|
| `AVAILABLE` | `True` |
| `PROGRESSING` | `False` |
| `DEGRADED` | `False` |

!!! warning "Common Slow Operators"

    The following operators may take additional time after installation:

    - `image-registry` — Needs storage configuration
    - `monitoring` — Prometheus stack initialization
    - `console` — Web console deployment

    Wait 10-15 minutes and re-check if any show `PROGRESSING=True`.

### Check for degraded operators

```bash
oc get co | grep -v "True.*False.*False"
```

If this returns any rows, those operators need attention.

---

## 4. Pod Health

### All Pods

```bash
oc get pods -A | grep -v Running | grep -v Completed
```

This shows any pods **not** in a healthy state. Investigate any that appear.

### Critical Namespaces

```bash
# API Server
oc get pods -n openshift-kube-apiserver

# etcd
oc get pods -n openshift-etcd

# Ingress (Router)
oc get pods -n openshift-ingress

# DNS
oc get pods -n openshift-dns

# Console
oc get pods -n openshift-console
```

---

## 5. Networking

### Internal DNS resolution

```bash
oc debug node/sno1 -- chroot /host nslookup api.sno.ocp.local
oc debug node/sno1 -- chroot /host nslookup kubernetes.default.svc.cluster.local
```

### External connectivity

```bash
oc debug node/sno1 -- chroot /host curl -sI https://quay.io
```

Expected: HTTP `200` or `301` — confirms outbound internet access through the Bastion's NAT.

---

## 6. Ingress & Console Access

### Verify Router Pods

```bash
oc get pods -n openshift-ingress
```

Should show `router-default-*` pods in `Running` state.

### Test Console URL

From a machine that can resolve `*.apps.sno.ocp.local`:

```bash
curl -skI https://console-openshift-console.apps.sno.ocp.local
```

Expected:
<div class="cmd-output">
HTTP/1.1 200 OK
</div>

---

## 7. Storage (Image Registry)

By default, the image registry operator is set to `Removed` on bare-metal/SNO. To enable it with `emptyDir` (lab/dev only):

```bash
oc patch configs.imageregistry.operator.openshift.io cluster \
  --type merge \
  --patch '{"spec":{"managementState":"Managed","storage":{"emptyDir":{}}}}'
```

!!! danger "Not for Production"

    `emptyDir` storage is **ephemeral** — registry data is lost on pod restart. For production, configure persistent storage (NFS, local-storage, etc.).

### Verify

```bash
oc get pods -n openshift-image-registry
```

---

## 8. Deploy a Test Application

Deploy a simple test app to validate the full stack:

```bash
# Create a test project
oc new-project smoke-test

# Deploy a test app
oc new-app --name=hello \
  --image=quay.io/openshift/origin-hello-openshift

# Expose the service
oc expose svc/hello

# Get the route
oc get route hello
```

### Test the route

```bash
curl http://hello-smoke-test.apps.sno.ocp.local
```

Expected:
<div class="cmd-output">
<span class="success">Hello OpenShift!</span>
</div>

### Cleanup

```bash
oc delete project smoke-test
```

---

## Validation Summary

| Check | Command | Expected |
|-------|---------|----------|
| Node Ready | `oc get nodes` | `Ready` |
| Cluster Version | `oc get clusterversion` | Available, not progressing |
| All Operators healthy | `oc get co` | All True/False/False |
| No failed pods | `oc get pods -A` | All Running/Completed |
| DNS working | `dig api.sno.ocp.local` | `192.168.83.10` |
| Console accessible | `curl -skI https://console...` | HTTP 200 |
| App route works | `curl http://hello-...` | `Hello OpenShift!` |

!!! success "🎉 All Validations Passed"

    Your SNO cluster is fully operational and ready for workloads.

---

**Next:** [:octicons-arrow-right-24: Troubleshooting Guide](troubleshooting.md)
