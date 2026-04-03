# Enterprise Infrastructure Security Lab

A production-grade homelab running five isolated VLANs, a two-node bare-metal RKE2 Kubernetes cluster, and a dedicated kubeadm cluster for security research. Built around a **Protectli FW4C** running **pfSense CE** and a **Cisco WS-C2960S-24TS-L** managed switch. External access is zero-trust via Cloudflare Tunnel — no inbound firewall ports exposed. Every configuration is documented and version-controlled in this repository.

> **Note:** IP addresses throughout this documentation use RFC 5737 reserved ranges for illustration. Live addressing is not published in this repository.

---

## Design Principles

- **Default deny.** All inter-VLAN traffic is blocked unless explicitly permitted by pfSense firewall rules. There is no implicit trust between network segments.
- **No inbound attack surface.** The WAN interface has zero open ports. External access to cluster services runs exclusively via Cloudflare Tunnel — an outbound-only encrypted connection. An attacker scanning the WAN IP finds nothing to target.
- **Least-privilege segmentation.** Each device lives on the least-privileged VLAN its function requires. IoT devices cannot reach lab servers. Lab servers cannot reach the admin network. Security research workloads run on a physically separate cluster with no path to production.
- **Management plane isolation.** Switch management and pfSense GUI are reachable only from VLAN 10. A compromised lab host cannot pivot to reconfigure network infrastructure.
- **Configuration as code.** The intended state of the network is always explicit, version-controlled, and reviewable. Drift from the documented state is detectable.

---


## Hardware Overview

| Device | Role | OS |
|--------|------|----|
| Protectli FW4C | pfSense CE — firewall, router, VLAN termination | pfSense CE |
| Cisco WS-C2960S-24TS-L | Managed Layer 2 switch — VLAN trunk & segmentation | IOS 15.x |
| Dell PowerEdge R220 (32 GB RAM) | RKE2 worker node | Rocky Linux |
| Bosgame Mini PC | RKE2 control plane | Rocky Linux |
| Dell PowerEdge R410 (128 GB RAM) | Dedicated kubeadm cluster — security tooling & research | Rocky Linux |
| Dell OptiPlex | File storage and media server | Ubuntu Server |
| Desktop PC | Admin workstation | Arch Linux |
| ThinkPad | Pentesting and security research | Arch Linux |
| TP-Link EAP653 | Wi-Fi 6 access point — isolated IoT VLAN | — |
| 24-Port Patch Panel (Cat6) | Physical cable termination | — |

---

## High-Level Topology

```
[ISP / Home Router]
        │ WAN
[Protectli FW4C — pfSense CE]
        │ 802.1Q Trunk (VLANs 10, 20, 30, 40, 99)
[Cisco WS-C2960S-24TS-L]
        │
 ┌──────┴──────────────────────────────────────┐
 │                                             │
VLAN 10 — Trusted          VLAN 40 — Lab / Kubernetes
 ├─ Desktop PC (admin)       ├─ Bosgame (RKE2 control plane)
 └─ Dell OptiPlex (storage)  ├─ Dell R220  (RKE2 worker)
                             ├─ Dell R410  (kubeadm / security cluster)
VLAN 20 — IoT               └─ ThinkPad   (pentesting)
 └─ TP-Link EAP653 (Wi-Fi 6)

VLAN 30 — Guest (internet-only)
VLAN 99 — Management (static, switch SVI)

[Cloudflare Tunnel] ← outbound-only from VLAN 40 → secure external access
```

---

## VLAN Summary

| VLAN | Name | Purpose | Example Subnet | DHCP |
|------|------|---------|----------------|------|
| 10 | Trusted | Admin workstations, internal services | 192.0.2.0/24 | Yes |
| 20 | IoT | Wireless AP, isolated smart devices | 198.51.100.0/24 | Yes |
| 30 | Guest | Internet-only, no LAN access | 203.0.113.0/24 | Yes |
| 40 | Lab | Bare-metal Kubernetes, security research | *(not published)* | Yes |
| 99 | Mgmt | Switch and infrastructure management | *(not published)* | Static only |

All inter-VLAN routing is enforced by pfSense firewall rules. VLANs 20, 30, and 40 have no access to VLAN 10. VLAN 99 is reachable only from VLAN 10.

---

## Kubernetes Clusters

**RKE2 cluster (production)**
- Control plane: Bosgame Mini PC (Rocky Linux)
- Worker node: Dell PowerEdge R220 (32 GB RAM, Rocky Linux)
- External access: Cloudflare Tunnel — no inbound firewall ports exposed

**kubeadm cluster (security research)**
- Runs on Dell PowerEdge R410 (128 GB RAM, Rocky Linux)
- Used for CKA exam preparation, security tooling evaluation, and deliberately misconfigured workloads for security research
- Isolated on VLAN 40 — no path to production cluster or trusted network

---

## Network Isolation Summary

| Source VLAN | Can Reach |
|-------------|-----------|
| VLAN 10 (Trusted) | Internet, VLAN 99 (switch mgmt), pfSense GUI |
| VLAN 20 (IoT) | Internet only |
| VLAN 30 (Guest) | Internet only |
| VLAN 40 (Lab) | Internet, Cloudflare Tunnel egress |
| VLAN 99 (Mgmt) | Management plane only |

---

## Documentation

| File | Description |
|------|-------------|
| `01_rack_plan_and_inventory.md` | Rack elevation and full hardware inventory |
| `02_physical_topology.md` | Device-to-VLAN assignment and logical topology diagram |
| `03_network_design.md` | VLAN plan, subnets, IP addressing, and routing policy |
| `04_switch_configuration.md` | Cisco WS-C2960S trunk and access port reference config |
| `05_pfsense_configuration.md` | pfSense VLAN interfaces, DHCP, and firewall rules |
| `06_testing_and_verification.md` | Connectivity, isolation, and cluster health verification |
| `07_security_architecture.md` | Threat model, trust zones, and defence-in-depth rationale |
| `08_monitoring_and_logging.md` | Logging architecture, monitored events, and alert policy |
| `09_runbooks.md` | Operational procedures for provisioning, incidents, and maintenance |
