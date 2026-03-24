---
title: MAS Installation Overview
---

# :material-ibm: MAS Installation Preparations

Now that the Single Node OpenShift (SNO) cluster is operational, the next major objective is to prepare the environment for the **IBM Maximo Application Suite (MAS)**.

MAS is an enterprise-grade suite of applications that has strict prerequisites regarding persistent storage and software licensing. Before we can deploy the MAS core or any of its applications (like Manage, Monitor, or Health), we must bridge the gap between a vanilla OpenShift cluster and an IBM-ready environment.

!!! quote "Reference Repository"
    The workflows detailed in this section draw inspiration and best practices from the official IBM repository for deploying MAS on Single Node OpenShift:  
    [:octicons-mark-github-16: `ibm-mas-manage/sno`](https://github.com/ibm-mas-manage/sno/blob/master/docs/index.md)

---

## Preparation Phases

```mermaid
graph LR
    A["1. Persistent<br/>Storage"] --> B["2. IBM Licensing<br/>& Entitlements"]
    B --> C["MAS Core<br/>Installation Ready ✓"]

    style A fill:#7c3aed,stroke:#6d28d9,color:#fff
    style B fill:#6366f1,stroke:#4f46e5,color:#fff
    style C fill:#10b981,stroke:#059669,color:#fff
```

---

## Phase Overview

| Phase | What Happens | Why It's Needed |
|-------|-------------|-----------------|
| **LVM Storage Operator** | Add a 500GB disk and configure the OpenShift LVM Storage Operator. | MAS and its underlying databases (DB2, MongoDB, Kafka) require dynamic block and file storage provisioning. |
| **Entitlement & License** | Obtain the IBM Entitlement key and the AppPoint `.dat` license file. | Required to pull proprietary MAS container images from the IBM registry and activate the suite. |

---

## Proceed to Each Step

1. [:octicons-arrow-right-24: 1. LVM Storage Operator Setup](storage.md)
2. [:octicons-arrow-right-24: 2. Entitlement & License File](licensing.md)
