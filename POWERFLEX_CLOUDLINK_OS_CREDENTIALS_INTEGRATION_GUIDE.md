# Dell PowerFlex — CloudLink Upgrade, Compute OS Upgrade, Password Rotation & CyberArk/Splunk Integration Guide

A comprehensive reference covering CloudLink upgrade with PowerFlex lifecycle manager, compute OS upgrade via lifecycle manager, password rotation, credential management, and integration with CyberArk or Splunk.

---

## Table of Contents

1. [CloudLink Upgrade with PowerFlex Lifecycle Manager](#1-cloudlink-upgrade-with-powerflex-lifecycle-manager)
2. [Compute OS Upgrade with Lifecycle Manager](#2-compute-os-upgrade-with-lifecycle-manager)
3. [Password Rotation and Credential Management](#3-password-rotation-and-credential-management)
4. [Integration with CyberArk or Splunk](#4-integration-with-cyberark-or-splunk)
5. [Upgrade Sequence Overview](#5-upgrade-sequence-overview)
6. [Reference Diagram](#6-reference-diagram)

---

## 1. CloudLink Upgrade with PowerFlex Lifecycle Manager

### Overview

CloudLink upgrade is a **prerequisite step** in the PowerFlex lifecycle upgrade sequence. PowerFlex Lifecycle Manager (or upgrade procedures) coordinates the CloudLink Center upgrade before PowerFlex Manager and node upgrades.

### Supported Upgrade Path

| From | To |
|------|-----|
| CloudLink Center 7.1.9 or later (7.1.x) | CloudLink Center 8.1.x or later |

### Important: In-Place Replacement (Not Standard Upgrade)

- CloudLink 8.1.x uses a **different operating system** than 7.1.x
- Upgrade requires **in-place replacement** of CloudLink Center VMs, not a traditional in-place upgrade
- Standard deployment: **2 instances**; up to **4 instances** supported

### CloudLink Center vs CloudLink Agent

| Component | Upgrade When |
|-----------|--------------|
| **CloudLink Center** | Upgraded first, as part of management stack (manual replacement procedure) |
| **CloudLink Agent** | Upgraded during **resource group upgrade** (automated with PowerFlex Manager) |

### Compatibility Notes

- **CloudLink Agent 7.1.x** can connect to **CloudLink Center 8.1.x** and unlock disks
- **Limitation:** New encryption or decryption of existing disks is **not** possible with Agent 7.1.x + Center 8.1.x mix (default PowerFlex Manager deployment)
- For custom configurations, contact Dell Technologies Support before upgrade

### Pre-Upgrade Checklist

- [ ] Halt or pause automated interactions with CloudLink Center cluster
- [ ] Ensure existing CloudLink Center and agents are **7.1.9 or later**
- [ ] Record CloudLink Center VM info (see below)
- [ ] Generate backup from CloudLink 7.1.9 console (key + backup file)
- [ ] Download CloudLink 8.1.x OVA from Dell Support or compliance file

### CloudLink Center Information to Record

| Parameter | Location |
|-----------|----------|
| Backup prefix, backup store | System > Backup |
| SNMP | Server > SNMP |
| Syslog | Server > Syslog |
| DNS | Server > DNS |
| NTP | Server > Time |
| TLS/certificates | Server > TLS |
| E-mail notifications | System > Email Notifications |

### Upgrade Procedure (High-Level)

1. **Disable anti-affinity rules** on CloudLink Center (prevent DRS from moving VMs)
2. **Record** IP, netmask, gateway for all CloudLink Center VMs
3. **Backup** CloudLink 7.1.9 (Generate New Backup → download key + backup file)
4. **Replace initial CloudLink cluster:**
   - Deploy CloudLink 8.1.x OVA
   - Configure same static IP as 7.1.9 member being replaced
   - Power on → Restore from backup (key + backup file)
5. **Replace additional CloudLink Center VMs** one at a time:
   - Power off 7.1.9 VM → Deploy 8.1.x OVA → Configure IP → Power on → Join cluster
   - Reconfigure SNMP, Syslog, DNS, NTP, TLS, Backup, Email
6. **Enable anti-affinity rules** on CloudLink Center

### Caution

- Modifications during replacement may cause deltas to not be captured → data unavailability or data loss
- Verify PowerFlex node connectivity after each replacement (`svm status` on encrypted node)

### Reference

- [KBA 000315084: PowerFlex 4.x Upgrade CloudLink Center 7.1.9 to 8.1.x](https://www.dell.com/support/kbdoc/en-us/000315084/powerflex-upgrade-cloudlink-center-7-1-9-or-later-to-8-1-x)
- [KBA 000202412: Devices in error state when upgrading CloudLink with PxFM](https://www.dell.com/support/kbdoc/en-us/000202412/issue-when-upgrading-cloudlink-with-powerflex-manager)
- [KBA 000203759: CloudLink 7.1.5 upgrade path for PowerFlex](https://www.dell.com/support/kbdoc/en-lb/000203759/cloudlink-upgrade-for-powerflex-systems)

---

## 2. Compute OS Upgrade with Lifecycle Manager

### Overview

PowerFlex supports **compute OS upgrades** (ESXi, Linux, Windows) orchestrated through PowerFlex Manager and resource group lifecycle workflows.

### Supported Platforms

| OS | Upgrade Support |
|----|-----------------|
| **VMware ESXi** | Yes — upgrade ESXi + SDC together on nodes |
| **RHEL/CentOS** | Yes — e.g., upgrade to RHEL 8.3 |
| **Windows** | Yes — compute-only nodes |
| **SLES** | Yes — per PowerFlex support matrix |

### Upgrade Methods

| Method | Use Case |
|--------|----------|
| **Automated (PowerFlex Manager)** | Resource group upgrade; PxFM orchestrates node-by-node |
| **Manual** | When automated path not available or for edge cases |

### ESXi + SDC Upgrade

- ESXi and SDC are upgraded together on each node
- Prerequisites: compatibility matrix, CPLD firmware, storage policies
- Workflow: prepare for upgrade → upload packages → upgrade nodes

### Linux OS Upgrade (RHEL 8.3+)

- Documentation covers RHEL upgrade to 8.3 and later
- Automated upgrade for Linux-based nodes supported
- Post-upgrade steps required (verify SDC, network, etc.)

### PowerFlex Management Controller Upgrade

- PowerFlex Manager 4.5+ can **automatically upgrade** the PowerFlex management controller (vCenter, ESXi hosts, vDS, Tools)
- Reference: [KBA 000231441](https://www.dell.com/support/kbdoc/en-us/000231441/upgrade-powerflex-management-controller-automatically-with-powerflex-manager-4-5-and-later)

### Lifecycle Manager Workflow

1. Prepare OS images and upgrade packages
2. Configure storage policies for upgrades
3. Run health checks
4. Upgrade via resource group (PxFM) or manual procedure
5. Verify cluster health and SDC connectivity

### Reference

- [PowerFlex 4.6.x: Prepare for VMware ESXi upgrade](https://www.dell.com/support/manuals/en-us/scaleio/pfx_p_software-install-upgrade-46x/prepare-for-the-vmware-esxi-upgrade)
- [PowerFlex 4.6.x: Upgrading operating systems on nodes](https://www.dell.com/support/manuals/en-us/scaleio/pfx_p_software-install-upgrade-46x/upgrading-operating-systems-on-powerflex-nodes)
- [PowerFlex Appliance Admin: Upgrade Windows and Linux compute-only nodes](https://www.dell.com/support/manuals/en-us/vxflex-appliance-r640/vxf-app_ag/upgrade-windows-and-linux-compute-only-nodes)

---

## 3. Password Rotation and Credential Management

### PowerFlex Credential Manager

PowerFlex Manager includes a **Credential Manager** that stores and manages credentials for:

| Component | Purpose |
|-----------|---------|
| **VMware vCenter** | Management cluster access |
| **iDRAC** | Node out-of-band management |
| **OS (Linux/Windows)** | Node login for automation |
| **CloudLink** | secadmin, configuration |

### Password Change Procedures

| Component | Method |
|-----------|--------|
| **vCenter** | Credential Manager in PxFM; or VMware vCenter password management |
| **iDRAC** | Credential Manager; or manual via iDRAC UI → iDRAC Settings → Users |
| **PowerFlex Gateway** | CLI or GUI reset |
| **PowerFlex admin** | SCLI: `scli --set_password --old_password <old> --new_password <new>` |

### Known Issue: iDRAC Password Change with Custom SSL

- **PFMP 4.6.x and earlier:** When using **custom SSL certificates**, credential manager may fail to change iDRAC passwords (SSL SAN mismatch).
- **Workaround:**
  1. Change password manually in iDRAC (iDRAC Settings → Users → Local Users → Edit)
  2. Update iDRAC credentials in PowerFlex Manager to match new password
- **Alternative:** Contact Dell Support to revert to default Ingress PFMP certificate (requires professional support)

### Password Rotation Best Practices

| Practice | Reason |
|----------|--------|
| Rotate vCenter and iDRAC credentials periodically | Compliance; reduce credential exposure |
| Update credential manager after manual changes | Keep PxFM in sync for automation |
| Use strong, unique passwords per component | Defense in depth |
| Document rotation schedule | Audit and compliance |

### Reference

- [KBA 000184515: Password change issue with credential manager](https://www.dell.com/support/kbdoc/en-us/000184515/powerflex-manager-password-change-issue-with-credential-manager)
- [KBA 000258783: iDRAC password cannot be changed with custom SSL certs](https://www.dell.com/support/kbdoc/en-us/000258783/powerflex-management-platform-idrac-password-can-t-be-changed-when-using-custom-ssl-certificates)
- [PowerFlex 3.6 CLI: Reset admin password](https://www.dell.com/support/manuals/en-sg/scaleio/pfx_cli_reference_guide_3.6.x/reset-the-admin-user-password)

---

## 4. Integration with CyberArk or Splunk

### Splunk Integration

#### Dell EMC PowerFlex Add-on and App for Splunk

| Capability | Details |
|------------|---------|
| **Purpose** | Monitor PowerFlex storage; log aggregation; dashboards |
| **Data collection** | PowerFlex Gateway; syslog; REST API |
| **Deployment** | Add-on on Splunk forwarders; App on search head |
| **Config** | Systems, proxy, log settings, data inputs, SSL (secure or non-secure) |
| **Use case** | Centralized logging, metrics, alerting, compliance |

#### Splunk Data Inputs

- Configure data inputs in PowerFlex Add-on
- PowerFlex can forward syslog to Splunk
- Options: SSL certificate setup, index configuration

### CyberArk Integration

#### Splunk Add-on for CyberArk (Separate from PowerFlex)

| Capability | Details |
|------------|---------|
| **Purpose** | Ingest privileged account activity from CyberArk |
| **Sources** | CyberArk PTA (Privileged Threat Analytics), EPV (Enterprise Password Vault) |
| **Format** | Syslog, CEF (Common Event Format) |
| **Integration** | Splunk Enterprise Security, PCI Compliance apps |

#### PowerFlex and CyberArk Together

- **No documented native integration** between PowerFlex credential manager and CyberArk for automatic credential injection
- **Possible approaches:**
  1. **Splunk as correlation layer:** PowerFlex Add-on + CyberArk Add-on → unified view in Splunk
  2. **Custom integration:** CyberArk Application Identity Manager (AIM) or Conjur with custom scripting to push credentials to PowerFlex (consult Dell Professional Services or CyberArk)
  3. **Manual rotation with CyberArk:** Rotate credentials in CyberArk; manually update in PowerFlex Credential Manager

### Integration Architecture (Conceptual)

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  PowerFlex      │     │  CyberArk       │     │  Splunk         │
│  Add-on        │────►│  Add-on         │────►│  Enterprise     │
│  (logs/metrics)│     │  (PTA/EPV logs) │     │  Security       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                        │                        │
        └────────────────────────┴────────────────────────┘
                    Unified monitoring & alerting
```

### Reference

- [Dell EMC PowerFlex Add-on and App for Splunk](https://infohub.delltechnologies.com/en-us/l/dell-emc-powerflex-add-on-and-app-for-splunk-user-guide-1/)
- [Splunk Add-on for CyberArk](https://splunkbase.splunk.com/app/2891)
- [Configure data inputs - PowerFlex Splunk](https://infohub.delltechnologies.com/en-us/l/dell-emc-powerflex-add-on-and-app-for-splunk-user-guide-1/configure-data-inputs/)

---

## 5. Upgrade Sequence Overview

### Full Lifecycle Upgrade Order (PowerFlex 4.x with CloudLink)

| Step | Component |
|------|-----------|
| 1 | Backup PowerFlex Manager; disable HA/DRS |
| 2 | **CloudLink Center** 7.1.9+ → 8.1.x (in-place replacement) |
| 3 | **vCenter** (remove HA, snapshot, upgrade, re-enable HA) |
| 4 | **PowerFlex Manager** |
| 5 | **PowerFlex management controller** (ESXi, vDS, Tools) |
| 6 | **Resource group upgrades** (CloudLink Agent upgraded here) |
| 7 | **Storage software** / PowerFlex nodes |
| 8 | **Compute OS** (ESXi, Linux) on nodes |
| 9 | Re-enable HA/DRS; verify health |

---

## 6. Reference Diagram

### CloudLink Upgrade Flow (7.1.9 → 8.1.x)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CloudLink Center Upgrade (In-Place Replacement)       │
└─────────────────────────────────────────────────────────────────────────┘

  CloudLink 7.1.9                    CloudLink 8.1.x
  ┌──────────────┐                    ┌──────────────┐
  │ Center VM 1  │  Backup (key+     │ Center VM 1  │
  │ Center VM 2  │  backup file)     │ Center VM 2  │  (Replace one
  └──────┬───────┘  ──────────────►  └──────┬───────┘   at a time)
         │                                   │
         │     PowerFlex Nodes               │
         │     (encrypted)                   │
         └─────────────┬─────────────────────┘
                       │
              CloudLink Agent 7.1.x  (upgraded during RG upgrade)
```

### Credential Manager and External Tools

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PowerFlex Manager Credential Manager                  │
│  • vCenter    • iDRAC    • OS (Linux/Windows)    • CloudLink            │
└─────────────────────────────┬───────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
   Manual rotation     CyberArk (custom)      Splunk (monitoring)
   in PxFM UI          integration           PowerFlex Add-on
   or iDRAC/vCenter    (if implemented)      + CyberArk Add-on
```

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| CloudLink 7.1.9 → 8.1.x upgrade | [KBA 000315084](https://www.dell.com/support/kbdoc/en-us/000315084/powerflex-upgrade-cloudlink-center-7-1-9-or-later-to-8-1-x) |
| CloudLink upgrade (devices error) | [KBA 000202412](https://www.dell.com/support/kbdoc/en-us/000202412/issue-when-upgrading-cloudlink-with-powerflex-manager) |
| PowerFlex Management Controller auto upgrade | [KBA 000231441](https://www.dell.com/support/kbdoc/en-us/000231441/upgrade-powerflex-management-controller-automatically-with-powerflex-manager-4-5-and-later) |
| Credential manager password issue | [KBA 000184515](https://www.dell.com/support/kbdoc/en-us/000184515/powerflex-manager-password-change-issue-with-credential-manager) |
| iDRAC password + custom SSL | [KBA 000258783](https://www.dell.com/support/kbdoc/en-us/000258783/powerflex-management-platform-idrac-password-can-t-be-changed-when-using-custom-ssl-certificates) |
| PowerFlex Add-on for Splunk | [Info Hub](https://infohub.delltechnologies.com/en-us/l/dell-emc-powerflex-add-on-and-app-for-splunk-user-guide-1/) |
| Splunk Add-on for CyberArk | [Splunkbase](https://splunkbase.splunk.com/app/2891) |

---

*Document version: 1.0 | For latest details, refer to Dell official manuals.*
