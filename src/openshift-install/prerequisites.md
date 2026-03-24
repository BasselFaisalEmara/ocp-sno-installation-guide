---
title: Prerequisites & Client Tools
---

# :material-download: Phase 1 — Prerequisites & Client Tools

Before creating the installation configuration, you need to download the OpenShift client (`oc`) and installer binary, and generate an SSH key pair for node access.

---

## 1.1 — Download OpenShift Client

Download the OCP 4.14 client tools from the Red Hat mirror:

```bash
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.14.0/openshift-client-linux.tar.gz
```

!!! tip "Choosing a Version"

    To use a different version, browse the available releases at:  
    `https://mirror.openshift.com/pub/openshift-v4/clients/ocp/`

    For the latest stable release, use:
    ```bash
    wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz
    ```

---

## 1.2 — Install the Client

Extract the binaries directly into the system PATH:

```bash
tar xvf openshift-client-linux.tar.gz -C /usr/bin
```

This installs two binaries:

| Binary | Purpose |
|--------|---------|
| `oc` | OpenShift CLI (superset of `kubectl`) |
| `kubectl` | Standard Kubernetes CLI |

### Verify Installation

```bash
oc version
kubectl version --client
```

---

## 1.3 — Enable Shell Auto-Completion

Set up bash auto-completion for `oc` commands:

```bash
# Generate completion script
oc completion bash > /etc/bash_completion.d/openshift

# Load immediately in current session
source /etc/bash_completion.d/openshift
```

!!! info "Auto-Completion"

    After this, pressing ++tab++ will auto-complete `oc` commands, resource types, and even resource names. This significantly speeds up CLI operations.

---

## 1.4 — Download the OpenShift Installer

```bash
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.14.0/openshift-install-linux.tar.gz
```

Extract:

```bash
tar xvf openshift-install-linux.tar.gz -C /usr/bin
```

### Verify

```bash
openshift-install version
```

Expected output:
<div class="cmd-output">
openshift-install 4.14.0<br/>
built from commit ...<br/>
release image quay.io/openshift-release-dev/ocp-release@sha256:...
</div>

---

## 1.5 — Generate SSH Key Pair

An SSH key is needed to access the CoreOS-based SNO node for debugging:

```bash
ssh-keygen -t ed25519 -N ''
ls .ssh/
cat .ssh/id_ed25519.pub
```

Expected output and troubleshooting:

```text
[root@bastion serveradmin]# ssh-keygen -t ed25519 -N ''
ls .ssh/
cat .ssh/id_ed25519.pub
Generating public/private ed25519 key pair.
Enter file in which to save the key (/root/.ssh/id_ed25519): 
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:4m1BntQDLp8dDiReSRAyYEIv2r2T7QDEOZe5u/rJPYI root@bastion.ocp.local
The key's randomart image is:
+--[ED25519 256]--+
|o.o.o ++=.       |
| +.. * =.o       |
| .=.+ o = +      |
|.o.+ . * * o     |
|. o o . S o      |
|   . * o .       |
|   .* o o        |
|  E..B..         |
|  .o=.o.         |
+----[SHA256]-----+
ls: cannot access '.ssh/': No such file or directory
cat: .ssh/id_ed25519.pub: No such file or directory
```

If you receive the `No such file or directory` error as shown above, it means you are executing the command outside of the root home directory. You must explicitly reference the correct `~/.ssh` path.

### View the Generated Files and Public Key

Execute the following commands to locate and view the key:

```bash
ls -lrth .ssh
ls -la ~/.ssh
cat ~/.ssh/id_ed25519.pub
```

Expected output:

```text
[root@bastion serveradmin]# ls -lrth .ssh
ls: cannot access '.ssh': No such file or directory
[root@bastion serveradmin]# ls -la ~/.ssh
total 12
drwx------.  2 root root   46 Mar 24 12:50 .
dr-xr-x---. 17 root root 4096 Mar 24 12:44 ..
-rw-------.  1 root root  419 Mar 24 12:50 id_ed25519
-rw-r--r--.  1 root root  104 Mar 24 12:50 id_ed25519.pub
[root@bastion serveradmin]# cat ~/.ssh/id_ed25519.pub
ssh-ed25519 AAAA... root@bastion.ocp.local
```

!!! warning "Save This Key"

    You will need the contents of `id_ed25519.pub` when creating the `install-config.yaml` in the next step. Copy it now.

---

## 1.6 — Obtain Red Hat Pull Secret

1. Navigate to [console.redhat.com/openshift/install/pull-secret](https://console.redhat.com/openshift/install/pull-secret)
2. Log in with your Red Hat account
3. Click **Download pull secret** or **Copy pull secret**
4. Save it to the Bastion:

```bash
vim ~/pull-secret.json
# Paste the pull secret contents and save
```

!!! danger "Required"

    The pull secret is **mandatory**. Without it, the installer cannot pull container images from `quay.io` and `registry.redhat.io`.

---

## Checklist

Before proceeding, verify all prerequisites:

- [x] `oc` CLI installed and on PATH
- [x] `openshift-install` installed and on PATH
- [x] Bash auto-completion configured
- [x] SSH key pair generated (`~/.ssh/id_ed25519.pub`)
- [x] Red Hat pull secret saved to `~/pull-secret.json`

!!! success "Checkpoint"

    All tools and credentials are ready. Proceed to create the installation configuration.

---

**Next:** [:octicons-arrow-right-24: Phase 2 — Installation Configuration](install-config.md)
