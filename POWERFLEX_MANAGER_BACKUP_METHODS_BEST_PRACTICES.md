# PowerFlex Manager (PxFM) — Backup Methods and Best Practices

A focused reference on PowerFlex Manager backup methods, restore procedures, and operational best practices.

---

## Table of Contents

1. [What PxFM Backup Covers](#1-what-pxfm-backup-covers)
2. [Backup Methods](#2-backup-methods)
3. [Backup Destinations](#3-backup-destinations)
4. [Backup Procedure (Step-by-Step)](#4-backup-procedure-step-by-step)
5. [Restore Procedure](#5-restore-procedure)
6. [Best Practices](#6-best-practices)
7. [When to Backup](#7-when-to-backup)
8. [Reference](#8-reference)

---

## 1. What PxFM Backup Covers

| Included | Not Included |
|----------|--------------|
| PxFM configuration | User data (volumes, snapshots) |
| Resource groups, templates | PowerFlex cluster data |
| Credentials (encrypted) | vCenter/ESXi VMs |
| Network settings | CloudLink Center config |
| System topology metadata | MDM/SDS/SDC runtime state |

**Purpose:** Recover PxFM appliance state after failure, migration, or upgrade. PowerFlex storage data is protected by PowerFlex replication (mesh-mirror or erasure coding), not by PxFM backup.

---

## 2. Backup Methods

| Method | Trigger | Use Case |
|--------|---------|----------|
| **Manual (Backup Now)** | User-initiated from PxFM GUI | Before upgrades, changes, or on-demand |
| **Scheduled** | Automated (if supported in your version) | Regular retention; check PxFM UI for schedule options |
| **CLI** | Script/cron | Automation; integrate with backup tools |

### GUI Path

- **Settings** → **Backup and Restore** (or equivalent)
- Configure backup path, test connection, **Backup Now**

---

## 3. Backup Destinations

| Protocol | Support | Notes |
|----------|---------|-------|
| **NFS** | Yes | Primary method; local or remote NFS share |
| **SMB** | Yes (4.5.x+) | Windows/Samba share; per Administration Guide |
| **Local** | Yes | Local path on PxFM (e.g., `/var/nfs/backup`) — **copy off-host for safety** |

### Local NFS Example (3.x / 4.x)

- Path: `127.0.0.1:/var/nfs/backup` (local loopback NFS)
- Directory: `/var/nfs/backup`
- Permissions: `chown tomcat:delladmin backup`; `chmod 755 backup`

### Remote NFS / SMB

- Use remote NFS export or SMB share for disaster recovery
- Ensure PxFM has network access and credentials (if required)
- Test connection before first backup

---

## 4. Backup Procedure (Step-by-Step)

### Prerequisites

- NFS/SMB share configured and accessible from PxFM
- Sufficient space for backup (typically hundreds of MB to low GB)
- `delladmin` or equivalent admin access to PxFM

### Step-by-Step (Local NFS Example)

1. **Create backup directory (if local)**
   ```bash
   ssh delladmin@<pxfm-ip>
   sudo su -
   cd /var/nfs/
   mkdir backup
   chown tomcat:delladmin backup
   chmod 755 backup
   ```

2. **Configure backup in PxFM**
   - Login to PxFM GUI
   - **Settings** → **Backup and Restore**
   - **Edit** under Settings and Details
   - **Backup path:** `127.0.0.1:/var/nfs/backup` (or remote NFS/SMB path)
   - **Test Connection** — must succeed
   - **Save**

3. **Run backup**
   - Click **Backup Now**
   - Check **Use backup settings**
   - Click **Backup Now**
   - Wait for completion

4. **Verify and copy off-host**
   - Confirm backup files in `/var/nfs/backup`
   - Copy to jump host or backup server (e.g., WinSCP, rsync)

---

## 5. Restore Procedure

### Prerequisites

- Backup files available
- New or replacement PxFM VM deployed
- Same or compatible PxFM version (check compatibility matrix)

### Step-by-Step

1. **Bring new PxFM online**
   - Deploy PxFM VM; initial setup
   - Configure IP; enable SSH

2. **Prepare restore destination**
   ```bash
   ssh delladmin@<new-pxfm-ip>
   sudo su -
   cd /var/nfs/
   mkdir backup
   # Copy backup files to /var/nfs/backup (e.g., via WinSCP)
   chown tomcat:delladmin backup
   chmod 755 backup
   ```

3. **Restore from GUI**
   - **Settings** → **Backup and Restore**
   - **Restore Now**
   - Provide filename and path; **Test Connection**
   - **Restore Now** → wait for completion

4. **Verify**
   - Login to PxFM
   - Check version (**About**)
   - Verify resource groups, connectivity to PowerFlex cluster

---

## 6. Best Practices

| Practice | Reason |
|----------|--------|
| **Backup before every upgrade** | Single point of recovery if upgrade fails |
| **Store backups off the PxFM host** | Protect against PxFM VM loss or corruption |
| **Use remote NFS/SMB when possible** | Avoid single-point-of-failure (local disk) |
| **Do not store backups on PowerFlex storage** | If PowerFlex is down, backup may be inaccessible; use separate storage |
| **Copy to multiple locations** | Secondary copy for disaster recovery |
| **Document backup path and credentials** | Needed for restore; store securely |
| **Test restore periodically** | Validate backup integrity and procedure |
| **Align with CloudLink backup** | If using CloudLink, back up CloudLink Center separately before upgrades |
| **Retention:** Keep at least 2–3 restore points | Version before last major change + current |
| **Verify backup completion** | Check logs/GUI; ensure no errors |

### Storage Recommendations

| Location | Use |
|----------|-----|
| **Remote NFS on separate storage** | Primary backup destination |
| **Jump host / backup server** | Secondary copy after manual transfer |
| **Avoid** | PxFM local disk only; PowerFlex datastore used by PxFM |

---

## 7. When to Backup

| Event | Action |
|-------|--------|
| Before PowerFlex Manager upgrade | Backup PxFM |
| Before CloudLink Center upgrade | Backup PxFM and CloudLink |
| Before vCenter upgrade | Backup PxFM |
| Before major configuration change | Backup PxFM |
| Before adding/removing resource groups | Backup PxFM |
| Scheduled (e.g., weekly) | Backup PxFM |
| Before node replacement or OS upgrade | Backup PxFM |

---

## 8. Reference

| Resource | URL |
|----------|-----|
| PowerFlex Manager backup and restore (NFS) | [KBA 000184942](https://www.dell.com/support/kbdoc/en-us/000184942/powerflex-manager-how-to-take-backup-and-restore-of-the-system-successfully-using-local-share-storage) |
| PowerFlex Manager appliance backup | [KBA 000183009](https://www.dell.com/support/kbdoc/en-us/000183009/powerflex-manager-how-to-backup-powerflex-manager-appliance) |
| Upgrade using backup/restore (3.7.0) | [KBA 000189267](https://www.dell.com/support/kbdoc/en-ng/000189267/powerflex-manager-how-to-upgrade-to-version-3-7-0-using-backup-and-restore-method) |
| PowerFlex 4.6.x Administration Guide - Restoring | [Dell Support](https://www.dell.com/support/manuals/en-us/scaleio/powerflex_sw_admin_guide_46/restoring) |
| PowerFlex Appliance - Back up using PowerFlex Manager | [Dell Support](https://www.dell.com/support/manuals/en-us/vxflex-appliance-r740xd/flex-appliance-administration/back-up-using-powerflex-manager) |

---

*Document version: 1.0 | For latest procedures, refer to Dell Administration Guide for your PxFM version.*
