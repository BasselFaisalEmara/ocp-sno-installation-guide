---
title: Installation Configuration
---

# :material-file-cog: Phase 2 — Installation Configuration

This phase creates the `install-config.yaml`, the primary input to the OpenShift installer, and generates the Ignition configs that the SNO node will consume during bootstrap.

---

## 2.1 — Create Installation Directory

```bash
mkdir -p ~/ocp-install
cd ~/ocp-install
```

!!! warning "Installer Consumes the Config"

    The `openshift-install` command **deletes** `install-config.yaml` after generating manifests. Always back it up before running the installer:

    ```bash
    cp install-config.yaml install-config.yaml.bak
    ```

---

## 2.2 — Create `install-config.yaml`

```bash
vim ~/ocp-install/install-config.yaml
```

```yaml title="install-config.yaml" linenums="1"
apiVersion: v1
baseDomain: ocp.local                        # (1)!
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0                                # (2)!
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 1                                # (3)!
metadata:
  name: sno                                  # (4)!
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14                      # (5)!
    hostPrefix: 23
  networkType: OVNKubernetes                 # (6)!
  serviceNetwork:
  - 172.30.0.0/16                            # (7)!
platform:
  none: {}                                   # (8)!
fips: false
pullSecret: '<YOUR_PULL_SECRET_HERE>'        # (9)!
sshKey: '<YOUR_SSH_PUBLIC_KEY_HERE>'          # (10)!
```

1.  :material-domain: Base domain — combined with `metadata.name` to form `sno.ocp.local`
2.  :fontawesome-solid-triangle-exclamation: **Must be `0` for SNO** — the single node runs both control plane and worker workloads
3.  :material-numeric-1-circle: **Must be `1` for SNO** — single control plane node
4.  :material-tag: Cluster name — becomes the subdomain (`sno.ocp.local`)
5.  :material-lan: Pod network CIDR — internal overlay network for pods
6.  :material-network: OVN-Kubernetes is the default SDN for OCP 4.14+
7.  :material-server-network: Service network CIDR — internal ClusterIP range
8.  :material-cancel: `none` platform — bare metal / no cloud provider integration
9.  :material-key: Paste your Red Hat pull secret (single-line JSON)
10. :material-ssh: Paste the contents of `~/.ssh/id_ed25519.pub`

---

## Key Configuration Explained

### Why `replicas: 0` for workers?

In a Single Node OpenShift deployment, the control plane node is **automatically schedulable** for workloads. Setting worker replicas to `0` tells the installer not to expect any dedicated worker nodes. The SNO node fulfills both roles.

### Cluster Name + Base Domain

The combination of `metadata.name` and `baseDomain` forms the cluster's FQDN:

```
<metadata.name>.<baseDomain> = sno.ocp.local
```

This must match your DNS configuration:

| Record | Resolves To |
|--------|-------------|
| `api.sno.ocp.local` | `192.168.83.10` (Bastion/HAProxy) |
| `api-int.sno.ocp.local` | `192.168.83.10` (Bastion/HAProxy) |
| `*.apps.sno.ocp.local` | `192.168.83.10` (Bastion/HAProxy) |

---

## 2.3 — Generate Manifests (Optional Inspection)

If you want to inspect or customize manifests before creating Ignition configs:

```bash
openshift-install create manifests --dir ~/ocp-install
```

This generates:

```
~/ocp-install/
├── manifests/
│   ├── cluster-config.yaml
│   ├── cluster-dns-02-config.yml
│   ├── cluster-infrastructure-02-config.yml
│   ├── cluster-ingress-02-config.yml
│   └── ...
└── openshift/
    ├── 99_openshift-cluster-api_master-machines-0.yaml
    └── ...
```

!!! note

    If you only want Ignition configs and don't need to customize manifests, you can skip this step and go directly to creating Ignition configs (the installer generates manifests internally).

---

## 2.4 — Generate Ignition Configs

```bash
openshift-install create single-node-ignition-config --dir ~/ocp-install
```

This produces:

```
~/ocp-install/
├── bootstrap-in-place-for-live-iso.ign    # (1)!
├── metadata.json
└── auth/
    ├── kubeadmin-password                  # (2)!
    └── kubeconfig                          # (3)!
```

1.  :material-fire: The Ignition config that will be injected into the CoreOS ISO
2.  :material-lock: Initial admin password for the web console
3.  :material-key-variant: Kubeconfig for CLI access

!!! danger "Backup These Files"

    The `auth/` directory contains credentials you **cannot regenerate**. Back them up immediately:

    ```bash
    cp -r ~/ocp-install/auth ~/ocp-install-auth-backup
    ```

---

## 2.5 — Prepare the Boot ISO

Download the RHCOS live ISO:

```bash
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.14/latest/rhcos-live.x86_64.iso
```

Embed the Ignition config into the ISO using `coreos-installer`:

```bash
# Install coreos-installer if not present
dnf install coreos-installer -y

# Embed ignition into the ISO
coreos-installer iso ignition embed \
  -i ~/ocp-install/bootstrap-in-place-for-live-iso.ign \
  rhcos-live.x86_64.iso
```

!!! tip "Alternative: Serve via HTTP"

    Instead of embedding, you can serve the Ignition file over HTTP and pass it as a kernel argument during boot:

    ```bash
    # Start a simple HTTP server
    python3 -m http.server 8080 --directory ~/ocp-install

    # Then use this kernel argument when booting the ISO:
    # coreos.inst.ignition_url=http://192.168.83.10:8080/bootstrap-in-place-for-live-iso.ign
    ```

---

## Checklist

- [x] `install-config.yaml` created with correct domain, replicas, and credentials
- [x] Backup of `install-config.yaml` saved
- [x] Ignition configs generated
- [x] `auth/` directory backed up
- [x] Boot ISO prepared with embedded Ignition

!!! success "Checkpoint"

    The installation media is ready. The next step is to boot the SNO node.

---

**Next:** [:octicons-arrow-right-24: Phase 3 — Bootstrap & Installation](bootstrap.md)
