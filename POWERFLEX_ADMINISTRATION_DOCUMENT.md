# Dell PowerFlex Administration Document
## PowerFlex 4.x — Operations, Management, and Maintenance

**Version:** 1.0  
**Based on:** Dell PowerFlex 4.5.x Technical Overview, PowerFlex 4.6 Complete Guide, Implementation/Upgrade Guides  

*Note: The PowerFlex 4.0 Administration PDF (c:\powerflex documentation\) was not found. This document consolidates administration content from available PowerFlex documentation.*

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [PowerFlex Manager Administration](#2-powerflex-manager-administration)
3. [Storage Administration](#3-storage-administration)
4. [Volume and Snapshot Management](#4-volume-and-snapshot-management)
5. [Protection Domain and Storage Pool](#5-protection-domain-and-storage-pool)
6. [MDM Cluster Administration](#6-mdm-cluster-administration)
7. [Rebuild and Rebalance](#7-rebuild-and-rebalance)
8. [Maintenance Mode](#8-maintenance-mode)
9. [Backup and Restore](#9-backup-and-restore)
10. [Monitoring and Alerts](#10-monitoring-and-alerts)
11. [SCLI Reference](#11-scli-reference)
12. [Replication Administration](#12-replication-administration)
13. [Lifecycle and Upgrades](#13-lifecycle-and-upgrades)

---

## 1. Introduction

This document covers day-to-day administration of Dell PowerFlex systems. PowerFlex Manager (PFMP) provides the primary UI; SCLI remains available for advanced and scripted operations.

### Key Administration Interfaces

| Interface | Use Case |
|-----------|----------|
| PowerFlex Manager UI | Day-to-day management, monitoring, lifecycle |
| SCLI (ScaleIO CLI) | Scripting, bulk operations, advanced queries |
| REST API | Automation, integration with third-party tools |

---

## 2. PowerFlex Manager Administration

### 2.1 Accessing PowerFlex Manager

- URL: `https://<powerflex-manager-ip>`
- Default roles: Admin, Operator, Viewer
- Integrate with AD/LDAP for enterprise auth (if configured)

### 2.2 Dashboard and Monitoring

- **Dashboard** — System health, capacity, alerts, rebuild/rebalance status
- **Block** — Protection domains, storage pools, volumes, hosts
- **File** — NAS file services (if deployed)
- **Monitoring** — Events, diagnostics, metrics

### 2.3 Management Diagnostics

- **Monitoring > Management Diagnostics > Diagnostics**
- Run diagnostics before upgrades or troubleshooting
- Review raw output for failures

### 2.4 Inventory

- Run inventory before upgrades or major changes
- Resolve any inventory errors before proceeding
- Inventory syncs hardware, software, and configuration

---

## 3. Storage Administration

### 3.1 Adding an SDS

1. PowerFlex Manager: **Block > Storage** or add node workflow
2. SCLI: `scli --add_sds --sds_name <name> --sds_ip <IP> --protection_domain_name <PD> ...`

### 3.2 Adding a Device to SDS

```bash
scli --add_sds_device --sds_name <name> --device_path <path> --storage_pool_name <SP>
```

### 3.3 Removing SDS (Maintenance)

1. Place SDS in maintenance mode
2. Drain data (rebalance)
3. Remove SDS from protection domain
4. Uninstall SDS software from node

### 3.4 Storage Pool Operations

- **Create** — PowerFlex Manager or SCLI
- **Expand** — Add devices from SDS in same protection domain
- **Remove** — Requires no volumes; drain first

---

## 4. Volume and Snapshot Management

### 4.1 Create Volume

- PowerFlex Manager: **Block > Volumes > Create**
- SCLI: `scli --add_volume --storage_pool_name <SP> --volume_name <name> --size_gb <size>`

### 4.2 Map Volume to SDC

```bash
scli --map_volume_to_sdc --volume_name <name> --sdc_name <sdc_name> [--allow_multiple_mappings]
```

### 4.3 Unmap Volume

```bash
scli --unmap_volume_from_sdc --volume_name <name> --sdc_name <sdc_name>
```

### 4.4 Host Groups

- Use host groups for consistent mapping across clusters
- Map volume to host group instead of individual SDC
- `scli --map_volume_to_sdc --volume_name <name> --sdc_name <host_group_name>`

### 4.5 Snapshots

- **Create snapshot:** PowerFlex Manager or SCLI
- **Snapshot policies** — Automated schedules (if supported)
- **vTrees** — Volume trees for snapshot chains
- **Consistency groups** — For application-consistent snapshots

### 4.6 Volume Migration

- Migrate volumes between storage pools (same protection domain)
- Use for rebalancing or before pool removal

---

## 5. Protection Domain and Storage Pool

### 5.1 Protection Domain

- Logical grouping of SDS nodes
- Activate/Deactivate for maintenance or power-off
- **Activate:** PowerFlex Manager > Block > Protection Domain > Activate
- **Deactivate:** Required before powering off nodes

### 5.2 Fault Sets

- Optional grouping for replica placement
- Ensures copies on different fault sets (e.g., different racks)
- Configure during initial setup or expansion

### 5.3 Storage Pool Best Practices

- At least 2 nodes per protection domain for RF2
- Balance capacity across pools
- Use fine granularity only when NVDIMM available (compression, snapshots)

---

## 6. MDM Cluster Administration

### 6.1 MDM Roles

| Role | Count | Purpose |
|------|-------|---------|
| Primary | 1 | Active metadata manager |
| Secondary | 1–2 | Standby failover |
| Tie-Breaker | 1–2 | Quorum |

### 6.2 Query Cluster

```bash
scli --query_cluster
```

### 6.3 Switch Primary MDM

```bash
scli --switch_mdm_ownership --new_primary_mdm_ip <IP>
```

Use for planned maintenance or rebalancing.

### 6.4 Add/Remove MDM

- Add Secondary or Tie-Breaker via PowerFlex Manager or SCLI
- Remove only after redistributing roles
- Maintain quorum (odd number of MDM nodes recommended)

---

## 7. Rebuild and Rebalance

### 7.1 Rebuild

- Occurs when SDS or device fails
- Recreates replicas from surviving copies
- **Policy:** Unlimited (recommended during failure) or Limit concurrent I/O

### 7.2 Rebalance

- Occurs when adding/removing devices or nodes
- Redistributes data for even layout
- **Policy:** Unlimited (during expansion) or Limit concurrent I/O (steady state)

### 7.3 Throttling

- Rebuild throttling: `scli --set_rebuild_io_priority_policy`
- Rebalance throttling: `scli --set_rebalance_io_priority_policy`
- Limit concurrent I/O to reduce impact on production workloads

### 7.4 Policy Settings (Recommended)

| State | Rebuild | Rebalance | Maintenance Mode |
|-------|---------|-----------|-------------------|
| Normal | Unlimited | Limit concurrent I/O=10 | Limit concurrent I/O=10 |
| Upgrade/Expansion | Unlimited | Unlimited | Limit concurrent I/O=10 |

---

## 8. Maintenance Mode

### 8.1 Instant Maintenance Mode (IMM)

- Node continues to serve data during maintenance
- Use for short operations (e.g., driver update)
- Limited I/O during maintenance

### 8.2 Protected Maintenance Mode (PMM)

- Node is removed from data path
- Data rebalanced to other nodes
- Use for hardware replacement, OS upgrade

### 8.3 SDS Maintenance

1. Place SDS in maintenance mode
2. Wait for rebalance to complete
3. Perform maintenance
4. Exit maintenance mode

---

## 9. Backup and Restore

### 9.1 PowerFlex Manager Backup

- **What is backed up:** PxFM configuration, credentials, topology, resource groups
- **Not backed up:** User data, PowerFlex volumes, vCenter VMs

### 9.2 Backup Destinations

| Type | Support |
|------|---------|
| NFS | Yes (primary) |
| SMB | Yes (4.5.x+) |
| Local | Yes (copy off-host for DR) |

### 9.3 Backup Procedure

1. **Settings > Backup and Restore**
2. Configure path: `host:/path` (NFS) or `\\server\share` (SMB)
3. **Test Connection**
4. **Backup Now**

### 9.4 Restore Procedure

1. Deploy fresh PowerFlex Manager or restore VM
2. **Settings > Backup and Restore > Restore**
3. Select backup file
4. Restore; verify connectivity to MDM and nodes

### 9.5 Best Practices

- Backup before upgrades and major changes
- Store backups off-host (remote NFS/SMB)
- Test restore periodically

---

## 10. Monitoring and Alerts

### 10.1 Events and Alerts

- **PowerFlex Manager > Monitoring > Events**
- Configure notification rules
- Webhook for ServiceNow, PagerDuty, etc.

### 10.2 Secure Connect Gateway (SCG)

- Deploy SCG for Dell Technologies connectivity
- Proactive support, call home
- Configure Policy Manager for alert routing

### 10.3 SNMP and Webhook

- Configure SNMP community and trap destination
- Webhook URL for custom integrations
- Verify firewall ports for Dell connectivity services

### 10.4 Key Metrics to Monitor

- MDM/SDS/SDC status
- Rebuild and rebalance progress
- Storage pool capacity and spare
- I/O latency and throughput
- Component health (disks, NICs)

---

## 11. SCLI Reference

### 11.1 Login

```bash
scli --login --management_system_ip <MDM_IP> --username admin
```

### 11.2 Common Commands

| Task | Command |
|------|---------|
| Cluster topology | `scli --query_cluster` |
| All SDSs | `scli --query_all_sds` |
| Specific SDS | `scli --query_sds --sds_name <name>` |
| All SDCs | `scli --query_all_sdc` |
| Storage pool | `scli --query_storage_pool --protection_domain_name <PD> --storage_pool_name <SP>` |
| SDS devices | `scli --query_sds_device_info --sds_name <name> --all_devices` |
| All volumes | `scli --query_all_volumes` |
| Volume details | `scli --query_volume --volume_name <name>` |
| System capacity | `scli --query_system` |

---

## 12. Replication Administration

### 12.1 Replication Consistency Group (RCG)

- Create RCG for volume groups
- Configure RPO, direction
- Activate or terminate RCG

### 12.2 Replication Pairs

- Map source volume to target volume
- Initial copy; then continuous replication

### 12.3 Pause and Resume

- Pause for maintenance
- Resume when ready

### 12.4 Failover

- Access target volumes after failover
- Document RTO/RPO for DR procedures

---

## 13. Lifecycle and Upgrades

### 13.1 Upgrade Order (PowerFlex Manager)

1. PowerFlex Manager
2. vCSA (if applicable)
3. Management VM OS packages
4. CloudLink (if used)
5. PFMC (vSAN or PowerFlex)
6. Storage-only, hyperconverged, compute-only nodes
7. Switches
8. Firmware (e.g., IPI G5)

### 13.2 LIA (Light Installation Agent)

- Enables automated maintenance
- Config: `/opt/emc/scaleio/lia/cfg/conf.txt`
- Token: update `lia_token`, restart LIA service

### 13.3 RCM Trains

- Stay on engineered RCM for stability
- Skip up to 2 RCM versions per train
- Contact Dell Support for hops > 2

---

## References

- Dell PowerFlex 4.6.x Administration Guide
- Dell PowerFlex Rack Administration Guide
- PowerFlex Manager Backup and Restore (KBA 000184942)
- Dell PowerFlex Networking Best Practices
