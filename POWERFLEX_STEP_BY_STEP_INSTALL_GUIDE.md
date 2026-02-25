# Dell PowerFlex Step-by-Step Install Guide
## PowerFlex 4.x / 5.x Rack and Appliance

**Version:** 1.0  
**Based on:** Dell PowerFlex Rack Field Implementation Guide 4x/5x  

---

## Overview

This guide provides a sequential, step-by-step procedure to bring up a PowerFlex Rack from factory delivery to operational state. Adjust steps based on your configuration (KVM vs ESXi PFMC, hyperconverged vs two-layer, with/without PowerScale, etc.).

---

## Phase 1: Preparation

### Step 1.1 — Verify Shipment and Connections

1. Confirm servers are not damaged from shipment.
2. Verify all cables are seated properly.
3. Check manufacturing handoff notes for items to complete.
4. Ensure PDUs, switches, and nodes are physically installed per rack design.

### Step 1.2 — Gather Prerequisites

- [ ] Customer services: Active Directory (optional), DNS, NTP
- [ ] Physical infrastructure: reliable power, cooling, core network access
- [ ] Serial cable and laptop for Dell networking (if applicable)
- [ ] RCM bundle downloaded from Dell Support
- [ ] Passwords from RCM portal for encrypted ZIP files
- [ ] VMware ESXi and vCSA images (Broadcom) if using ESXi PFMC

### Step 1.3 — PowerFlex Rack Files

1. Log in to Dell Technologies Support: **PowerFlex rack RCM software**.
2. Click **Drivers & Downloads**, filter for RCM files, select RCM to download.
3. Log in to RCM portal: **Converged Platforms & Solutions Code Matrix**.
4. Under **Select System**, choose **PowerFlex rack**, ensure **Releases** selected.
5. Filter for RCM, click link, then **Download Guide**.
6. Read instructions and obtain passwords for ZIP files.
7. Extract ZIPs and provide passwords when prompted.

---

## Phase 2: Power-On Sequence

### Step 2.1 — Verify PDU Breakers

1. Confirm PDU breakers are in **OPEN (OFF)**.
2. Use enclosed tool to press white tab below switch if needed.
3. Connect external AC feeds to PDUs.
4. Verify power available; LEDs display number on each PDU.

### Step 2.2 — Power On Zone A

1. Close PDU circuit breakers for **Zone A** (press ON side).
2. Verify all components connected to Zone A PDUs light up.

### Step 2.3 — Power On Zone B

1. Close PDU circuit breakers for **Zone B** (press ON side).
2. Verify all components connected to Zone B PDUs light up.

### Step 2.4 — Power On Network Components

**Order and wait times:**

1. **Management switches** — Power on; wait until PS1 and PS2 LEDs are solid green (~10 min).
2. **Cisco Nexus** aggregation/leaf-spine — Wait until system status LED is green.
3. **Dell PowerSwitch** — Wait until system status LED is solid green.

---

## Phase 3: PowerFlex Management Cluster (KVM)

### Step 3.1 — Power On PFMC Nodes

1. Log in to iDRAC for each management node.
2. Power on all PowerFlex management nodes.
3. Monitor virtual console; wait for SLES server to appear.

### Step 3.2 — Verify PowerFlex Services

On each SLES host:

```bash
sudo systemctl status lia
sudo systemctl status mdm
sudo systemctl status sds
sudo systemctl status scini
sudo systemctl status activemq   # MDMs only
```

Ensure all show **active (running)**.

### Step 3.3 — Verify Management VMs

```bash
sudo crm cluster run "virsh list"
```

Confirm VMs are running.

### Step 3.4 — Check RKE2 Server

1. Log in to management VMs running PowerFlex management platform.
2. On each node:

```bash
systemctl status rke2-server
```

- **active** → Proceed.
- **activating** → Wait and recheck.
- **failed** → Run: `systemctl start rke2-server`

### Step 3.5 — Verify Kubernetes Nodes

```bash
kubectl get nodes
```

Wait until all nodes show **Ready**. Retry if errors.

### Step 3.6 — Start Database (if needed)

```bash
export NAMESPACE=powerflex
export DBSA_IP=$(kubectl get service/postgres-monitor-leader -n $NAMESPACE -o jsonpath='{.spec.clusterIP}')
curl -i -H "Accept: application/json" "http://$DBSA_IP:8080/pgclusterstartup"
```

Check status:

```bash
curl -i -H "Accept: application/json" "http://$DBSA_IP:8080/pgclusterstartupstatus"
```

Repeat until status returns `"startup_status":"verified"`.

### Step 3.7 — PowerFlex Manager Diagnostics

1. Log in to PowerFlex Manager.
2. Go to **Monitoring > Management Diagnostics > Diagnostics**.
3. Run diagnostics; resolve any failures.

### Step 3.8 — Activate Management Protection Domain

1. Log in to PowerFlex Manager.
2. Go to **Block > Protection Domain**.
3. Select protection domain (checkbox).
4. Click **Protection Domain** > **More Actions**.
5. Set **Force Activate** = No.
6. Click **Activate**; wait for status **Active**.

### Step 3.9 — CRM Startup (if resources stopped)

1. SSH to management cluster.
2. Run: `sudo crm status` — verify nodes online.
3. Start resources:

```bash
sudo crm resource start stonith-sbd
sudo crm resource start cl-dlm
sudo crm status   # Verify stonith-sbd and cl-dlm Started
sudo crm resource start cl-shared-lock
sudo crm resource start cl-shared-fs1   # Repeat for all shared FS
sudo crm resource start <resource name>   # Repeat for all resources
sudo crm status   # All resources running
```

---

## Phase 4: Storage and Production Nodes

### Step 4.1 — Power On Storage-Only Nodes

1. Power on all PowerFlex storage-only nodes.
2. In PowerFlex Manager 5.x: Activate production protection domain (if separate).

### Step 4.2 — Power On Hyperconverged Nodes

1. Power on all PowerFlex hyperconverged nodes.
2. Verify in PowerFlex Manager: Block > Nodes; all SDS/SDC online.

### Step 4.3 — Power On Compute-Only Nodes (if applicable)

1. Power on PowerFlex compute-only nodes.
2. Verify SDC connectivity in PowerFlex Manager.

### Step 4.4 — Power On vCenter / NSX (if applicable)

1. Power on VMware vCenter VMs.
2. Power on VMware NSX Edge nodes (if used).

### Step 4.5 — Power On Workload VMs

1. Power on customer workload VMs.
2. Verify applications and storage access.

### Step 4.6 — Health Check

1. PowerFlex Manager: **Dashboard** — verify no errors.
2. **Block** tab: check rebuild status, rebalance.
3. Resolve any ongoing rebuild or rebalance before heavy use.

---

## Phase 5: Licenses and Switch Configuration

### Step 5.1 — Upload PowerFlex License

1. PowerFlex Manager: **Settings** or license configuration.
2. Upload PowerFlex license file.
3. Activate if required.

### Step 5.2 — CloudLink License (if used)

1. Verify CloudLink license valid.
2. Configure per CloudLink documentation.

### Step 5.3 — Dell SmartFabric OS10 License (Dell PowerSwitch)

1. Install license on each Dell PowerSwitch per guide.
2. Troubleshoot if installation fails (see Field Implementation Guide).

### Step 5.4 — Cisco Nexus Service Port

1. Configure PowerFlex rack service port on Cisco Nexus.
2. Configure embedded OS jump server to use service port if needed.

### Step 5.5 — Dell PowerSwitch Service Port

1. Configure service port for Dell PowerSwitch.
2. Configure service laptop Ethernet port for field use.

### Step 5.6 — Customer Network Integration

Follow applicable scenario:

- **Scenario A:** Layer 2 — Management switch
- **Scenario B:** Layer 2 — Management aggregation switch
- **Scenario C:** Layer 3 — Management aggregation switch

Apply VLANs, trunking, and uplinks per Network Planning Guide.

---

## Phase 6: Post-Deployment

### Step 6.1 — Redistribute MDM Cluster

1. PowerFlex Manager or SCLI: redistribute MDM roles.
2. Ensure Primary/Secondary on different fault sets.
3. Verify: `scli --query_cluster`

### Step 6.2 — Verify Spare Capacity

1. PowerFlex Manager: **Block > Storage Pools**.
2. Check spare capacity per protection domain.
3. Ensure sufficient for rebuild.

### Step 6.3 — Change Default Passwords

1. Change PowerFlex rack default passwords per security policy.
2. Document in secure location.

### Step 6.4 — Configure Events and Alerts

1. Verify firewall ports for Dell Technologies connectivity services.
2. Deploy Secure Connect Gateway (QCOW2).
3. Deploy Policy Manager.
4. Register and log in to SCG UI.
5. Configure SNMP for webhook (if needed).
6. Configure event/alert rules in PowerFlex Manager.

---

## Phase 7: Final Validation

### Step 7.1 — Run Test Plan

Execute PowerFlex rack test cases from Field Implementation Guide:

- PFMC (KVM) test cases
- PFMC (ESXi) test cases (if applicable)
- Asynchronous replication (if used)
- Failure scenarios
- NVMe over TCP (if used)
- Secure Connect Gateway and Policy Manager

### Step 7.2 — Document Handoff

- Record IP addresses, hostnames, credentials (secure store).
- Document network topology and VLAN assignments.
- Provide customer with administration guide and support contacts.

---

## Quick Reference: Power-On Order

1. PDUs (Zone A, then Zone B)
2. Switches (management, then aggregation/access)
3. PowerFlex management cluster nodes
4. Management VMs (BOSS then PowerFlex)
5. Activate protection domain
6. Storage-only nodes
7. Activate production protection domain (5.x)
8. Hyperconverged nodes
9. Compute-only nodes
10. vCenter, NSX Edge
11. Workload VMs

---

## Troubleshooting

| Issue | Action |
|-------|--------|
| rke2-server failed | `systemctl start rke2-server` |
| Nodes not Ready | Wait; check networking, DNS |
| Protection domain not activating | Check SDS status; verify MDM cluster |
| Rebuild/rebalance stuck | Check policy settings; contact Dell Support |
| Switch not discovered | Verify SNMP, LLDP; check compatibility matrix |

---

## References

- Dell PowerFlex Rack with PowerFlex 4.x / 5.x Field Implementation Guide
- PowerFlex Rack Administration Guide
