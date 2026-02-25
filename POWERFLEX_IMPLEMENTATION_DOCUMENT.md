# Dell PowerFlex Implementation Document
## PowerFlex 4.x / 5.x Rack and Appliance

**Version:** 1.0  
**Based on:** Dell PowerFlex Rack Field Implementation Guides 4x/5x, Network Planning Guide, Technical Overview  

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Implementation Overview](#2-implementation-overview)
3. [Deployment Architectures](#3-deployment-architectures)
4. [Network Implementation](#4-network-implementation)
5. [Switch Configuration Scenarios](#5-switch-configuration-scenarios)
6. [MDM Redistribution](#6-mdm-redistribution)
7. [Spare Capacity Verification](#7-spare-capacity-verification)
8. [Events and Alerts](#8-events-and-alerts)
9. [Secure Connect Gateway](#9-secure-connect-gateway)
10. [Finalization Checklist](#10-finalization-checklist)

---

## 1. Introduction

This document provides implementation guidance for Dell PowerFlex Rack and Appliance deployments. It covers design decisions, network connectivity scenarios, post-deployment configuration, and system finalization.

---

## 2. Implementation Overview

### 2.1 Logical PowerFlex Configuration

PowerFlex supports multiple network architectures:

- **Access-aggregation** — Access switches connect to aggregation switches
- **Leaf-spine** — Leaf switches connect to spine and border-leaf
- **Layer 2** — Direct VLAN connectivity
- **Layer 3** — Routed connectivity via SVI

### 2.2 Implementation Phases

1. **Pre-deployment** — Network design, switch config, VLAN provisioning
2. **Deployment** — PowerFlex Manager deployment, node addition
3. **Post-deployment** — License upload, MDM redistribution, spare capacity check
4. **Finalization** — Alerts, Secure Connect Gateway, test plan

---

## 3. Deployment Architectures

### 3.1 PowerFlex Rack (Integrated)

- Factory-built, single SKU
- PowerFlex management cluster (KVM or ESXi) included
- Cisco Nexus or Dell PowerSwitch
- PowerFlex Manager pre-deployed

### 3.2 PowerFlex Appliance

- Bring-your-own networking (optional)
- Bring-your-own management (optional)
- Minimum 4 nodes; 6 recommended
- PowerFlex Manager for deployment and lifecycle

### 3.3 Node Types and Network Templates

| Node Type | Template | Storage Data Networks | LACP | VLANs (Example) |
|-----------|----------|------------------------|------|-----------------|
| PFMC 2.0 | LACP | 6 | Yes | 101, 103, 105, 130, 140–143, 150–154 |
| Hyperconverged | LACP | 4 | Yes | 105–106, 150, 151–154, 161–162 |
| Compute-only | LACP | 4 | Yes | 105–106, 150, 151–154 |
| Storage-only | LACP / Trunk | 4 | Yes / No | 150–154, 161–162 |
| File nodes | LACP / Trunk | 4 | Yes / No | 150–154, 250–252 |

---

## 4. Network Implementation

### 4.1 Pre-Deployment Network Requirements

- Redundant connections to access switches (MLAG: VLT or vPC)
- Management interface IPs configured
- VLT/vPC on switch interconnects
- **MTU=9216** on uplinks and server-facing ports carrying PowerFlex data
- **LLDP** enabled on switch ports connected to PowerFlex nodes
- **SNMP** enabled, community string `public`, trap destination = PowerFlex Manager
- Layer-2 connectivity of PowerFlex data and management VLANs to PFMC

### 4.2 Multi-VLAN and Multi-Subnet

- Supported for all network types except data, vSAN, and NSX overlay
- Aggregation: SVI + trunk allowed VLAN list
- Multi-subnet: additional IP per subnet, first-hop redundancy as required
- Management VM on PFMC must have VLAN 130 for multi-VLAN
- All PowerFlex controller nodes must have VLAN 130
- OOB management for multi-VLAN on all access/leaf switches

### 4.3 Post-Network Activity

- Clear temporary ports and SVIs after cutover
- Register Cisco Nexus on Cisco Smart Account portal
- Configure Smart Call Home for Cisco Nexus
- Dell PowerSwitch: configure service port, interconnect

---

## 5. Switch Configuration Scenarios

### 5.1 Cisco Nexus Management Switch

**Scenario A: Layer 2 — Management switch only**  
- Direct connection to customer network via management switch  
- SVIs on management switch for VLAN routing  

**Scenario B: Layer 2 — Management aggregation switch**  
- Management switch uplinks to aggregation switch  
- L2 VLAN extension to customer network  

**Scenario C: Layer 3 — Management aggregation switch**  
- Routed connectivity via aggregation switch  
- SVI and routing configuration on aggregation  

### 5.2 Cisco Nexus Access Switch

- **Hyperconverged:** Access switches connect nodes; vMotion, management, data VLANs
- **Storage-only:** Aggregation switches for storage network; access for compute
- **Layer 2 connectivity:** Port channels, VLAN trunking, vPC

### 5.3 Dell PowerSwitch

- **Management switch:** Service port configuration, embedded jump server
- **Access switch:** Two-layer or hyperconverged; LACP, trunk
- **Dual network:** Separate management and data networks; aggregation and access roles

### 5.4 PowerFlex Rack Service Port

- Used for initial setup before customer network integration
- Configure on Cisco Nexus or Dell PowerSwitch per guide
- Embedded OS jump server can use service port for setup
- Configure service laptop Ethernet port for field use

---

## 6. MDM Redistribution

### 6.1 Purpose

- Distribute MDM roles across nodes for optimal resilience
- Avoid concentrating MDMs on same fault domain

### 6.2 MDM Cluster Layouts

- **3-node PFMC:** 1 Primary, 1 Secondary, 1 Tie-Breaker
- **5-node PFMC:** 1 Primary, 2 Secondary, 2 Tie-Breaker
- MDMs can run on storage-only or hyperconverged nodes
- Use PowerFlex Manager or SCLI to redistribute

### 6.3 Best Practice

- Ensure Primary and Secondary MDMs are on different fault sets
- Tie-Breaker can be standalone or collocated with SDS

---

## 7. Spare Capacity Verification

Before finalizing:

1. Check spare capacity in each protection domain
2. Verify sufficient capacity for rebuild scenarios
3. Use PowerFlex Manager Dashboard or Block > Storage Pools
4. Ensure no "Low capacity" or "Critical" alerts

---

## 8. Events and Alerts

### 8.1 Prerequisites

- Verify firewall ports for Dell Technologies connectivity services
- Obtain Software ID and License Authorization Code

### 8.2 Dell Technologies Connectivity Services

- Enable on PowerFlex management cluster
- Replaces legacy SupportAssist
- Required for proactive support and call home

### 8.3 Secure Connect Gateway (SCG)

- Deploy using QCOW2 image
- Register and log in to SCG UI
- Configure for event forwarding

### 8.4 Policy Manager

- Deploy for Secure Connect Gateway
- Configure policies for alert routing

### 8.5 SNMP and Webhook

- Configure SNMP on resources for webhook integration
- Set community string and trap destination
- Use for integration with enterprise monitoring (e.g., ServiceNow, PagerDuty)

### 8.6 Managing Events and Alerts

- PowerFlex Manager: Monitoring > Events
- Configure notification rules
- Webhook for automation

---

## 9. Secure Connect Gateway

### 9.1 Deployment

- Deploy SCG VM using QCOW2 from Dell Support
- Configure network (VLAN 105 or 130 typically)
- Register with Dell Technologies

### 9.2 Registration

- Obtain registration code from Dell Support portal
- Complete registration in SCG UI
- Link to PowerFlex system

### 9.3 Policy Manager

- Deploy Policy Manager VM
- Connect to SCG
- Define alert routing and escalation

---

## 10. Finalization Checklist

### 10.1 System Finalization

- [ ] All nodes powered and healthy
- [ ] Protection domains activated
- [ ] Licenses uploaded (PowerFlex, CloudLink, SmartFabric, VMware)
- [ ] MDM redistribution complete
- [ ] Spare capacity verified
- [ ] Events and alerts configured
- [ ] Secure Connect Gateway deployed and registered
- [ ] SNMP/webhook configured (if needed)

### 10.2 Test Plan

- PowerFlex rack test cases (see Field Implementation Guide Chapter 16):
  - PFMC (KVM) health
  - PFMC (ESXi) health
  - Asynchronous replication (if used)
  - Failure scenarios
  - NVMe over TCP (if used)
  - Secure Connect Gateway and Policy Manager

### 10.3 Secure Component Verification

- Optional: Enable Secure Boot on PowerFlex nodes
- Optional: Disable TPM attestation alarm per security policy

---

## References

- Dell PowerFlex Rack with PowerFlex 4.x / 5.x Field Implementation Guide
- Dell PowerFlex Appliance Network Planning Guide 4.x
- Dell PowerFlex Rack Administration Guide
