# Dell PowerFlex — Lifecycle Management, vCenter Upgrade, Metrics, CloudLink+K8s, Licensing & Backup Guide

A comprehensive reference covering PowerFlex lifecycle management (4.6 and 5.0), vCenter upgrade sequence, single vs multi-component upgrades, metrics scraping (Grafana/Prometheus/CSM), CloudLink with Kubernetes/OpenShift, CloudLink licensing, and CloudLink/PxFM backup.

---

## Table of Contents

1. [PowerFlex Lifecycle Management (4.6 vs 5.0)](#1-powerflex-lifecycle-management-46-vs-50)
2. [vCenter Upgrade — Can PowerFlex Do It? Sequence](#2-vcenter-upgrade--can-powerflex-do-it-sequence)
3. [Single Component vs Multiple Component Upgrade](#3-single-component-vs-multiple-component-upgrade)
4. [Metrics Scraping, Data Collection & Charting](#4-metrics-scraping-data-collection--charting)
5. [CloudLink Encryption and Kubernetes/OpenShift](#5-cloudlink-encryption-and-kubernetesopenshift)
6. [CloudLink Licensing](#6-cloudlink-licensing)
7. [CloudLink and PxFM Backup](#7-cloudlink-and-pxfm-backup)
8. [Reference Diagram](#8-reference-diagram)

---

## 1. PowerFlex Lifecycle Management (4.6 vs 5.0)

### Overview

PowerFlex Lifecycle Management covers upgrades, patches, maintenance, and operational procedures for PowerFlex clusters and PowerFlex Manager.

### PowerFlex 4.6 Lifecycle

| Activity | Details |
|----------|---------|
| **Upgrade path** | 3.6.x → 4.5.x (core) + 4.6.x (management); 4.5.x → 4.6.x |
| **RCM (Rack Config Module)** | Full RCM/IC bundle upgrade typically required for storage/node updates |
| **Single component** | PxFM 4.5.x+ supports single component upgrade (see Section 3) |
| **Maintenance mode** | SDS can enter maintenance mode for drain, work, exit |
| **LIA** | Light Installation Agent enables automated maintenance/upgrades |

### PowerFlex 5.0 Lifecycle

| Activity | Details |
|----------|---------|
| **Upgrade path** | 4.5.x/4.6.x → 5.0.x; consult Dell upgrade guide for 4.6→5.0 |
| **Single component upgrade** | **New in 5.0:** Upgrade individual components (e.g., security patches) without full RCM/IC upgrade |
| **Storage node** | Can add storage nodes for scale-out without adding compute |
| **Maintenance** | PDS/DGWT maintenance mode; rebalance/rebuild monitoring |

### Lifecycle Workflow (General)

1. **Pre-upgrade**
   - Health check
   - Backup PowerFlex Manager
   - Disable VMware HA and DRS (if applicable)
   - Upgrade CloudLink Center first (if integrated)

2. **Upgrade order** (typical)
   - CloudLink Center
   - VMware vCenter
   - PowerFlex Manager
   - Storage software / PowerFlex nodes
   - ESXi and SDC on nodes

3. **Post-upgrade**
   - Verify cluster health
   - Re-enable HA/DRS
   - Test workloads

---

## 2. vCenter Upgrade — Can PowerFlex Do It? Sequence

### Can PowerFlex Upgrade vCenter?

**Yes.** PowerFlex lifecycle management includes guidance and procedures for upgrading the **PowerFlex management controller** VMware vCenter. The upgrade is performed as part of the overall PowerFlex management stack upgrade, following a prescribed sequence.

### Prerequisites

- Remove VMware vCenter HA configuration (consult Broadcom documentation)
- Take snapshot of PowerFlex management controller vCenter
- Disable VMware HA and DRS in vCenter before management cluster upgrade

### vCenter Upgrade Sequence (PowerFlex 4.x)

| Step | Action |
|------|--------|
| 1 | **Remove** VMware vCenter HA configuration |
| 2 | **Snapshot** PowerFlex management controller vCenter |
| 3 | **Upgrade** PowerFlex management controller vCenter (e.g., 7.0.x → 8.0.x) — follow Broadcom upgrade docs |
| 4 | **Upgrade** PowerFlex management controller VMware ESXi hosts |
| 5 | **Upgrade** distributed virtual switches |
| 6 | **Upgrade** VMware Tools and hardware version |
| 7 | **Configure** VMware vCenter HA |

### Important Notes

- **IC bundles 46.375.00, 46.380.00+:** Do not include VMware ESXi and vCSA images. Customers must procure standard images and patch depot from Broadcom support portal. Custom/Dell_customized images are not supported.
- Full upgrade guide: [Dell PowerFlex Appliance with PowerFlex 4.x Upgrade Guide](https://www.dell.com/support/manuals/en-us/powerflex-appliance-r650/flex-app-upgrade-guide-4x/)

### Reference

- [KBA 000315070: PowerFlex 4.x Upgrading VMware vCenter](https://www.dell.com/support/kbdoc/en-us/000315070/powerflex-4-x-upgrading-vmware-vcenter)

---

## 3. Single Component vs Multiple Component Upgrade

### Single Component Upgrade

| Aspect | Details |
|--------|---------|
| **Availability** | PowerFlex Manager 4.5.x and later |
| **Use case** | Apply security patches or specific component updates without full RCM/IC upgrade |
| **Benefit** | Reduce risk, faster patch deployment, less downtime |
| **5.0 enhancement** | Explicit support for single component upgrades in PowerFlex 5.0 |

### Multiple Component Upgrade (Full RCM/IC)

| Aspect | Details |
|--------|---------|
| **Use case** | Major version upgrade, firmware, ESXi, SDC, storage software |
| **Scope** | Full RCM (Rack Configuration Module) or IC (Intelligent Catalog) bundle |
| **Order** | Must follow prescribed sequence (CloudLink, vCenter, PxFM, nodes, etc.) |
| **Validation** | Confirm RCM/IC bundle support in Managed Mode before upgrade |

### When to Use Which

| Scenario | Approach |
|----------|----------|
| Security patch for PxFM | Single component |
| CPLD/firmware on nodes | Multiple component (or node-level) |
| vCenter 7→8 upgrade | Multiple component (vCenter is part of stack) |
| ESXi + SDC upgrade on nodes | Multiple component |
| Storage software minor patch | Single component (if supported) |

### Reference

- [KBA 000223004: Single Component Upgrade on PxFM 4.5.x+](https://www.dell.com/support/kbdoc/en-us/000223004/pfxm-how-to-perform-single-component-upgrade-on-pfxm-4-5-x-and-later-versions)
- [KBA 000206391: Confirm RCM/IC bundle support in Managed Mode](https://www.dell.com/support/kbdoc/en-us/000206391/powerflex-manager-how-to-confirm-rcm-or-ic-bundle-on-a-rack-or-appliance-is-supported-in-managed-mode)

---

## 4. Metrics Scraping, Data Collection & Charting

### Options for PowerFlex Metrics

| Option | Stack | Use Case |
|--------|-------|----------|
| **Dell CSM Observability** | OpenTelemetry → Prometheus → Grafana | Kubernetes/container environments; official Dell integration |
| **sio2prom** | Prometheus exporter (Rust) | Standalone Prometheus; ScaleIO/VxFlex/PowerFlex |
| **PowerFlex GUI** | Built-in | Real-time monitoring in PowerFlex Manager |
| **Splunk** | PowerFlex Add-on for Splunk | Log aggregation + metrics; compliance/audit |

### Dell CSM for Observability (Recommended for K8s)

**Architecture:**
```
PowerFlex → OpenTelemetry Agent → OpenTelemetry Collector → Prometheus → Grafana
```

| Component | Role |
|------------|------|
| **karavi-metrics-powerflex** | Collects PowerFlex array-level metrics |
| **OpenTelemetry Collector** | Processes and exports in Prometheus format |
| **Prometheus** | Scrapes metrics |
| **Grafana** | Dashboards, alerting |

### Available Metrics (CSM)

| Metric Category | Metrics | Config Flag |
|-----------------|---------|-------------|
| **I/O Performance** | Read/write bandwidth (MB/s), latency (ms), IOPS by export node and volume | `sdc_metrics_enabled` (default: true) |
| **Storage Capacity** | Total logical capacity, available, in-use, provisioned (GB) per storage pool | `storage_class_pool_metrics_enabled` |

### Grafana Dashboards

- **Pre-built dashboards:** [dell/karavi-observability](https://github.com/dell/karavi-observability/blob/main/grafana/dashboards/powerflex)
- **Capabilities:** IOPS, bandwidth, latency, capacity utilization, rebuild/rebalance, SDS/SDC/volume metrics
- **Alerting:** Email (and other channels) for threshold alerts

### sio2prom (Third-Party)

- Prometheus exporter for PowerFlex/ScaleIO/VxFlex
- Written in Rust; includes Grafana dashboards
- Alternative when not using CSM/Kubernetes

### Deployment (CSM Observability)

- Deployed via Helm
- Requires cert-manager for TLS
- CSI driver config secrets copied to observability namespace
- Reference: [PowerFlex Metrics | CSM Docs](https://dell.github.io/csm-docs/v1/observability/metrics/powerflex/)

---

## 5. CloudLink Encryption and Kubernetes/OpenShift

### Dell CloudLink for Containers

Dell CloudLink supports **encryption for containers** on **Kubernetes** clusters.

| Feature | Details |
|---------|---------|
| **Platform** | Kubernetes (including OpenShift) |
| **Use case** | Encrypt container workloads and persistent data |
| **Integration** | CloudLink components deploy alongside Kubernetes |

### Prerequisites (CloudLink 8.0)

- Kubernetes cluster with CloudLink-compatible version
- KMIP-compliant external KMS for key management
- Administrator access for CloudLink deployment

### Licensing Change (CloudLink 8.0+)

- **Encryption for Containers** moved from LIC format to **Instance-based licensing (XML)**
- Upgrades from 7.1.5+ require **license conversion** (LIC → XML)
- Without conversion, Encryption for Containers may stop working after upgrade
- Contact: https://licensing.emc.com/ for conversion

### OpenShift

- OpenShift runs on Kubernetes; CloudLink for Containers can be used on OpenShift
- Ensure CloudLink support matrix includes your OpenShift version
- For OpenShift-specific encryption (e.g., etcd, disk), refer to Red Hat docs (user-managed encryption, Azure Key Vault, etc.); these are separate from Dell CloudLink

---

## 6. CloudLink Licensing

### Overview

CloudLink uses license files for activation. Licenses are platform- and feature-specific.

### Purchasing Options (CloudLink 8.0.2+)

| Option | Details |
|--------|---------|
| **Activate licenses** | Upload and activate license files |
| **Split licenses** | Split activated licenses for multiple deployments |
| **Convert formats** | LIC → XML (especially for Encryption for Containers 8.0+) |

### License Types

| License | Purpose |
|---------|---------|
| **PowerFlex encryption** | CloudLink with PowerFlex |
| **Encryption for Containers** | CloudLink with Kubernetes; instance-based (XML) in 8.0+ |
| **SED management** | Self-Encrypting Drive key management |

### Reference

- [Dell CloudLink 8.0.2 Deployment Guide - Licensing](https://www.dell.com/support/manuals/en-us/cloudlink-securevm/cl_8-0-2_deployment_guide/dell-cloudlink-licenses-purchasing-options)
- [Dell PowerFlex Licensing Guide](https://www.dell.com/support/kbdoc/en-us/000242214/dell-powerflex-licensing-guide)

---

## 7. CloudLink and PxFM Backup

### PowerFlex Manager (PxFM) Backup

| Aspect | Details |
|--------|---------|
| **Destination** | NFS share (local or remote) |
| **Contents** | PxFM configuration, not user data |
| **Restore** | Recovers PxFM appliance state |
| **Versions** | 3.x, 4.x, 5.x — backup procedures in Administration Guide |

### Backup Procedure (High-Level)

1. Configure NFS share accessible to PowerFlex Manager
2. In PowerFlex Manager: navigate to backup/restore settings
3. Set backup destination and schedule (if supported)
4. Trigger backup or use scheduled backup
5. Verify backup completion and retention

### CloudLink Backup

| Aspect | Details |
|--------|---------|
| **Automatic backups** | CloudLink can be configured for automatic backups |
| **Backup store** | Configurable; change via "Change the backup store for automatic backups" (CloudLink Admin Guide) |
| **Integration** | CloudLink Center backup is separate from PxFM backup |

### Best Practice

- **Backup before any upgrade:** PxFM and CloudLink Center
- Store backups on separate storage (e.g., NFS outside PowerFlex cluster)
- Test restore procedure periodically

### Reference

- [KBA 000184942: PowerFlex Manager backup and restore using Local Share Storage](https://www.dell.com/support/kbdoc/en-us/000184942/powerflex-manager-how-to-take-backup-and-restore-of-the-system-successfully-using-local-share-storage)
- [KBA 000183009: PowerFlex Manager appliance backup](https://www.dell.com/support/kbdoc/en-us/000183009/powerflex-manager-how-to-backup-powerflex-manager-appliance)
- [CloudLink 7.1.3 Admin Guide - Change backup store](https://www.dell.com/support/manuals/en-us/cloudlink-securevm/cl_7.1.3_administrator_guide/change-the-backup-store-for-automatic-backups)

---

## 8. Reference Diagram

### Lifecycle and Upgrade Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PowerFlex Lifecycle Upgrade Sequence                   │
└─────────────────────────────────────────────────────────────────────────┘

 1. Pre-Upgrade                    2. Management Stack                3. Storage/Nodes
    ┌──────────────┐                  ┌──────────────┐                  ┌──────────────┐
    │ Health Check │                  │ CloudLink    │                  │ PowerFlex    │
    │ Backup PxFM  │───────────────► │ Center       │───────────────► │ Manager     │
    │ Disable HA   │                  └──────┬───────┘                  └──────┬───────┘
    └──────────────┘                         │                                 │
                                             ▼                                 ▼
                                      ┌──────────────┐                  ┌──────────────┐
                                      │ vCenter      │                  │ Storage SW   │
                                      │ (remove HA,  │                  │ ESXi + SDC   │
                                      │  snapshot,   │                  │ on nodes     │
                                      │  upgrade)    │                  └──────────────┘
                                      └──────┬───────┘
                                             │
                                             ▼
                                      ┌──────────────┐
                                      │ ESXi hosts   │
                                      │ vDS, Tools   │
                                      │ Re-enable HA │
                                      └──────────────┘
```

### Metrics Pipeline (CSM Observability)

```
┌─────────────┐     ┌─────────────────────┐     ┌─────────────────┐
│ PowerFlex   │────►│ OpenTelemetry Agent │────►│ OTel Collector  │
│ (MDM/SDS/   │     │ (karavi-metrics-    │     │                 │
│  SDC)      │     │  powerflex)          │     └────────┬────────┘
└─────────────┘     └─────────────────────┘              │
                                                         │ Prometheus format
                                                         ▼
┌─────────────┐     ┌─────────────────────┐     ┌─────────────────┐
│ Grafana     │◄────│ Prometheus          │◄────│ Metrics export  │
│ Dashboards  │     │ (scrape)             │     │                 │
└─────────────┘     └─────────────────────┘     └─────────────────┘
```

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| PowerFlex 4.x Upgrading vCenter | [KBA 000315070](https://www.dell.com/support/kbdoc/en-us/000315070/powerflex-4-x-upgrading-vmware-vcenter) |
| Single Component Upgrade (PxFM) | [KBA 000223004](https://www.dell.com/support/kbdoc/en-us/000223004/pfxm-how-to-perform-single-component-upgrade-on-pfxm-4-5-x-and-later-versions) |
| RPS Upgrade Customer Prep | [KBA 000340894](https://www.dell.com/support/kbdoc/en-us/000340894/rps-general-procedure-powerflex-upgrade-customer-preparation-guide) |
| PowerFlex Metrics (CSM) | [CSM Docs](https://dell.github.io/csm-docs/v1/observability/metrics/powerflex/) |
| Grafana for PowerFlex | [powerflex.me](https://powerflex.me/2020/07/27/historical-reporting-for-your-powerflex-cluster-with-grafana/) |
| CloudLink K8s Prerequisites | [CloudLink 8.0 Deployment](https://www.dell.com/support/manuals/en-us/cloudlink-securevm/cl_8-0_deployment-guide/prerequisites-to-set-up-encryption-for-containers-on-kubernetes-cluster) |
| CloudLink Licensing | [CloudLink 8.0.2 Licensing](https://www.dell.com/support/manuals/en-us/cloudlink-securevm/cl_8-0-2_deployment_guide/dell-cloudlink-licenses-purchasing-options) |
| PxFM Backup and Restore | [KBA 000184942](https://www.dell.com/support/kbdoc/en-us/000184942/powerflex-manager-how-to-take-backup-and-restore-of-the-system-successfully-using-local-share-storage) |

---

*Document version: 1.0 | For latest details, refer to Dell official manuals.*
