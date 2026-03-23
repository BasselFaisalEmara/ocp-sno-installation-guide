---
title: OpenShift Installation
---

# :material-kubernetes: OpenShift Installation

With all Bastion infrastructure services operational, the next phase is to download the OpenShift tools, prepare the installation configuration, and deploy the SNO cluster.

---

## Installation Phases

```mermaid
graph LR
    A["1. Prerequisites<br/>& Client Tools"] --> B["2. Install Config<br/>& Manifests"]
    B --> C["3. Bootstrap<br/>& Installation"]
    C --> D["Cluster<br/>Ready ✓"]

    style A fill:#7c3aed,stroke:#6d28d9,color:#fff
    style B fill:#6366f1,stroke:#4f46e5,color:#fff
    style C fill:#06b6d4,stroke:#0891b2,color:#fff
    style D fill:#10b981,stroke:#059669,color:#fff
```

---

## Phase Overview

| Phase | What Happens | Duration |
|-------|-------------|----------|
| **Prerequisites** | Download `oc` CLI and `openshift-install`, generate SSH keys | ~5 min |
| **Install Config** | Create `install-config.yaml`, generate Ignition configs | ~5 min |
| **Bootstrap** | Boot SNO node, wait for bootstrap and installation to complete | ~45-90 min |

---

## Proceed to Each Phase

1. [:octicons-arrow-right-24: Prerequisites & Client Tools](prerequisites.md)
2. [:octicons-arrow-right-24: Installation Configuration](install-config.md)
3. [:octicons-arrow-right-24: Bootstrap & Installation](bootstrap.md)
