# PowerFlex Manager (PxFM) — Kubernetes Components, Keycloak & Important Pods/Deployments

A reference on PowerFlex Manager (PxFM) / PowerFlex Management Platform (PFMP) architecture, including Kubernetes-based components, Keycloak, and other important services.

---

## Table of Contents

1. [PxFM/PFMP Deployment Models](#1-pxfmpfmp-deployment-models)
2. [Kubernetes-Based Architecture](#2-kubernetes-based-architecture)
3. [Keycloak — Identity & Access Management](#3-keycloak--identity--access-management)
4. [REST API Services in PxFM Kubernetes Pods](#4-rest-api-services-in-pxfm-kubernetes-pods)
5. [Important Pods and Deployments](#5-important-pods-and-deployments)
6. [PFMP_config.json and Cluster Parameters](#6-pfmp_configjson-and-cluster-parameters)
7. [PowerFlex CSI on Kubernetes (Consumer Side)](#7-powerflex-csi-on-kubernetes-consumer-side)
8. [Troubleshooting](#8-troubleshooting)
9. [Reference](#9-reference)

---

## 1. PxFM/PFMP Deployment Models

| Model | Infrastructure | K8s? |
|-------|----------------|------|
| **VMware vSphere** | 3 MVMs on ESXi | Yes (K8s on MVMs) |
| **PowerFlex Management Cluster (KVM)** | KVM hypervisor on dedicated nodes | Yes |
| **Integrated / Non-KVM** | Co-resident on PowerFlex storage nodes (boot device) | Yes |
| **Linux Management** | BYO Linux VMs or bare metal | Yes |

### MVM (Management Virtual Machine)

- Typical: **3 MVMs** per PFMP cluster
- Each MVM: distinct hostname, NTP-synced, IP on PowerFlex management VLAN
- Installer VM deploys Kubernetes and PowerFlex services onto MVMs
- Minimum NVMe boot: 960 GB (e.g., R650 BOSS-S2, R660 BOSS-N1) for co-resident

---

## 2. Kubernetes-Based Architecture

### Overview

PowerFlex Manager runs as a **containerized platform on Kubernetes**. The installer deploys:

1. **Kubernetes cluster** on the MVMs
2. **PowerFlex management services** as pods/deployments
3. **Docker** (or Podman) on the installer node

### Deployment Flow

```
Installer VM  →  Deploy K8s + PFMP  →  MVMs run PFMP cluster
                      │
                      ├── Keycloak (IAM)
                      ├── Database
                      ├── Web/Gateway
                      └── Other PFMP services
```

### Infrastructure Options

- **KVM:** PowerFlex management cluster with KVM; PFMP runs in K8s on that cluster
- **Non-KVM:** PFMP on boot device of 3 PowerFlex storage nodes (co-resident)
- **vSphere:** PFMP VMs on vCenter; K8s runs inside those VMs

---

## 3. Keycloak — Identity & Access Management

### Role

PowerFlex Manager uses **Keycloak** for:

| Function | Details |
|----------|---------|
| **Authentication** | User login, SSO |
| **Authorization** | Role-based access; LDAP group mapping |
| **LDAP/AD integration** | Federation with Active Directory, LDAP |
| **Session management** | Sessions, tokens |

### Keycloak Admin Console

- Access: PFxM Keycloak Admin Console (URL and access method per Dell docs)
- Use: Configure LDAP, realms, clients, group mappings
- **LDAP Group Search Filter:** Applied here for large AD environments

### LDAP Configuration (Large Environments)

| Issue | Mitigation |
|-------|------------|
| **20,000+ AD groups** | Keycloak may timeout during login |
| **Solution** | Apply LDAP group search filter in Keycloak Admin Console to limit which groups are processed |
| **Reference** | [KBA 000224858](https://www.dell.com/support/kbdoc/en-ie/000224858/4-x-how-to-apply-ldap-group-search-filter-in-pfxm-keycloak-admin-console), [KBA 000243118](https://www.dell.com/support/kbdoc/en-uk/000243118/powerflex-ldap-configuration-for-large-environments) |

### Keycloak in PFMP vs Customer K8s

| Context | Keycloak Role |
|---------|---------------|
| **Inside PxFM/PFMP** | Built-in IAM for PowerFlex Manager UI and API |
| **Customer K8s + PowerFlex CSI** | Optional; some designs use Keycloak for IAM-as-a-service alongside PowerFlex storage |

---

## 4. REST API Services in PxFM Kubernetes Pods

### Overview

PowerFlex 4.x (and 5.x) exposes REST APIs through **multiple microservices** running as pods in the PFMP Kubernetes cluster. These services replaced the single PowerFlex Gateway from 3.x.

### REST API Service (Pod) Mapping

| Service / Pod | Role | API Endpoints / Use |
|---------------|------|---------------------|
| **block-legacy-gateway** | Storage administration | Volumes, mapping, storage pools, protection domains, SDS, SDC. Replaces PowerFlex 3.x Gateway. |
| **asmui** | Lifecycle management (LCM) | Add nodes, manage resource groups, services, deployment. |
| **PowerAPI / SSO** | Authentication & user management | `/rest/auth/login`, token issue/refresh. OAuth2/OIDC. |

### Authentication Flow

1. **Login:** `POST https://<pfxm-host>/rest/auth/login` with JSON `{"username":"...","password":"..."}`
2. **Response:** `access_token` (expires in ~5 min; refresh via OIDC)
3. **Subsequent calls:** Include `Authorization: Bearer <access_token>` (or equivalent)

### Block API Examples (block-legacy-gateway)

| Operation | Method | Endpoint |
|-----------|--------|----------|
| Get config | GET | `/api/Configuration` |
| Get system info | GET | `/api/types/System/instances` |
| List volumes | GET | `/api/types/Volume/instances` |
| Create volume | POST | `/api/types/Volume/instances` |
| Remove volume | POST | `/api/instances/Volume::{id}/action/removeVolume` |
| Map volume to host | POST | `/api/instances/Volume::{id}/action/addMappedHost` |

### API Categories (PowerFlex 4.x+)

| Category | Service | Status |
|----------|---------|--------|
| **Block API** | block-legacy-gateway | Legacy; to be merged into PowerAPI |
| **PowerFlex Manager API** | asmui | LCM; to be merged into PowerAPI |
| **PowerAPI** | PowerAPI service | SSO, NAS, file, events, alerts; Dell standard API model |

### Access

- All API calls (except login and token refresh) require a valid access token
- Host: Same URL as PowerFlex Manager GUI (e.g., `https://pfmp.vdi.example.com`)

### Reference

- [PowerFlex REST API Introduction](https://infohub.delltechnologies.com/en-us/t/powerflex-rest-api-introduction/)
- [PowerFlex 4.x RestAPI Calls](https://powerflex.me/2024/09/10/an-introduction-to-powerflex-4-x-restapi-calls/)
- [Dell Developer - PowerFlex API](https://developer.dell.com/apis) → Storage → PowerFlex

---

## 5. Important Pods and Deployments

### Known / Likely PFMP Components

Dell does not publish a full public list of PFMP pod names. Based on documentation and typical designs:

| Component | Likely Role |
|-----------|-------------|
| **Keycloak** | Identity and access management |
| **block-legacy-gateway** | REST API for block storage (volumes, mapping, pools) |
| **asmui** | REST API for LCM (resource groups, nodes, deployment) |
| **PowerAPI / SSO** | REST API for auth (`/rest/auth/login`), tokens |
| **Database** | Configuration and state (PostgreSQL or similar) |
| **Web/Gateway** | PowerFlex Manager UI and API gateway |
| **Backend services** | Orchestration, node management, storage operations |
| **Support/Connect** | SupportAssist, Secure Connect Gateway integration |

### Querying PFMP Cluster (If Accessible)

If you have `kubectl` access to the PFMP cluster:

```bash
kubectl get pods -A
kubectl get deployments -A
kubectl get services -A
```

- Namespace may be product-specific (e.g., `powerflex-manager`, `pfmp`, or similar)
- Exact names depend on PFMP version and packaging

### Support Data Collection

- Use Dell procedures to collect PFMP support data
- Reference: [KBA 000203573: Collect PFMP and PFMP Installer support data](https://www.dell.com/support/kbdoc/en-us/000203573/powerflex-how-to-collect-powerflex-manager-platform-pfmp-and-pfmp-installer-support-data)

---

## 6. PFMP_config.json and Cluster Parameters

### Purpose

- **PFMP_config.json** (or equivalent) defines cluster deployment parameters
- Used during "Deploy the PowerFlex management platform cluster"
- Contains: MVM hostnames, IPs, network config, storage, etc.

### Verification

- Install guide: "Verify the PFMP_config.json file"
- "Verify the PFMP_config.json key input restrictions"
- Ensure values match environment (DNS, NTP, networks)

### Reference

- [PowerFlex management platform cluster configuration parameters](https://www.dell.com/support/manuals/en-us/scaleio/pfx_p_software_install_guide_5x/powerflex-management-platform-cluster-configuration-parameters)

---

## 7. PowerFlex CSI on Kubernetes (Consumer Side)

### Distinction

| Component | Purpose |
|-----------|---------|
| **PxFM/PFMP** | Manages PowerFlex storage infrastructure |
| **PowerFlex CSI driver** | Lets Kubernetes clusters *consume* PowerFlex storage as PersistentVolumes |

### PowerFlex CSI (Separate from PxFM)

- Deployed via **Dell CSM Operator** or Helm
- Namespace: `vxflexos` (typical)
- Requires SDC on all K8s nodes running the node component
- CRD: `ContainerStorageModule` — `kubectl get csm -A`

### Keycloak with PowerFlex CSI (Optional)

- Customer designs may use Keycloak for IAM when running apps on K8s with PowerFlex CSI
- Example: Keycloak + MetalLB + NGINX Ingress + PowerFlex CSI for shared IAM and storage

---

## 8. Troubleshooting

### PxFM UI Login Failures

| Symptom | Possible Cause |
|---------|----------------|
| "Failure to initialize" | Keycloak or backend service issues |
| "User has no permissions" | Keycloak DB connectivity, LDAP/group config, or session |
| Timeout on login | LDAP group enumeration (large AD) — apply group search filter |

### LDAP Large Environments

- Use LDAP group search filter in Keycloak
- Reduces groups Keycloak processes; avoids login timeouts
- See [KBA 000243118](https://www.dell.com/support/kbdoc/en-uk/000243118/powerflex-ldap-configuration-for-large-environments)

### Pod/Service Health

- If `kubectl` is available: `kubectl get pods -A` and inspect `STATUS`
- Check logs: `kubectl logs <pod> -n <namespace>`
- Collect PFMP support data per Dell procedure

---

## 9. Reference

| Topic | URL |
|-------|-----|
| LDAP group search filter (Keycloak) | [KBA 000224858](https://www.dell.com/support/kbdoc/en-ie/000224858/4-x-how-to-apply-ldap-group-search-filter-in-pfxm-keycloak-admin-console) |
| LDAP large environments | [KBA 000243118](https://www.dell.com/support/kbdoc/en-uk/000243118/powerflex-ldap-configuration-for-large-environments) |
| PFMP UI login failure | [KBA 000225511](https://www.dell.com/support/kbdoc/en-in/000225511/powerflex-4-x-pfmp-ui-login-failure) |
| Collect PFMP support data | [KBA 000203573](https://www.dell.com/support/kbdoc/en-us/000203573/powerflex-how-to-collect-powerflex-manager-platform-pfmp-and-pfmp-installer-support-data) |
| PowerFlex 5.0 Install Guide (PFMP deployment) | [Dell Support](https://www.dell.com/support/manuals/en-us/scaleio/pfx_p_software_install_guide_5x/deploying-the-powerflex-management-platform) |
| PowerFlex CSI (K8s consumer) | [CSM Docs](https://dell.github.io/csm-docs/docs/getting-started/installation/openshift/powerflex/csmoperator/) |
| PFMP_config.json verification | [Dell Support](https://www.dell.com/support/manuals/en-us/scaleio/pfx_p_software_install_guide_5x/verifying-the-pfmpconfigjson-file) |
| PowerFlex REST API Introduction | [Info Hub](https://infohub.delltechnologies.com/en-us/t/powerflex-rest-api-introduction/) |
| PowerFlex 4.x RestAPI Calls | [powerflex.me](https://powerflex.me/2024/09/10/an-introduction-to-powerflex-4-x-restapi-calls/) |
| Dell Developer - PowerFlex API | [developer.dell.com/apis](https://developer.dell.com/apis) |

---

## Architecture Diagram (Conceptual)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PowerFlex Management Platform (PFMP)                   │
│                         Kubernetes Cluster (on MVMs)                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌─────────────┐  ┌─────────────────────┐  ┌─────────────┐  ┌─────────────┐   │
│   │  Keycloak   │  │ block-legacy-gateway│  │   asmui     │  │  PowerAPI   │   │
│   │  (IAM)      │  │ (Block REST API)    │  │ (LCM API)   │  │ (Auth/SSO)  │   │
│   └──────┬──────┘  └──────┬──────────────┘  └──────┬──────┘  └──────┬──────┘   │
│          │                │                │                │           │
│          └────────────────┴────────────────┴────────────────┘         │
│                                    │                                     │
│                          PFMP_config.json                                │
│                          Cluster parameters                              │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────┐
                    │  PowerFlex Cluster (MDM/SDS/   │
                    │  SDC, Storage Pools)           │
                    └─────────────────────────────────┘
```

---

*Document version: 1.0 | Dell may change PFMP internals; refer to current Dell documentation for your version.*
