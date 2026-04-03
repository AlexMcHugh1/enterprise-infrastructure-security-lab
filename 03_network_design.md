# Network Design

> **Note:** IP addresses in this document use RFC 5737 documentation ranges (192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24) for illustration. Live subnet addressing is not published in this repository.

---

## VLAN Architecture

Five VLANs provide security segmentation across the network. Inter-VLAN routing is handled entirely by pfSense; the Cisco switch operates as a pure Layer 2 device.

| VLAN | Name | Purpose | Example Subnet | Gateway | DHCP |
|------|------|---------|----------------|---------|------|
| 10 | Trusted | Admin workstations, internal services | 192.0.2.0/24 | 192.0.2.1 | .50–.199 |
| 20 | IoT | Wi-Fi AP, isolated smart devices | 198.51.100.0/24 | 198.51.100.1 | .50–.199 |
| 30 | Guest | Temporary access, internet-only | 203.0.113.0/24 | 203.0.113.1 | .50–.199 |
| 40 | Lab | Bare-metal Kubernetes, security research | *(not published)* | *(not published)* | Yes |
| 99 | Mgmt | Switch SVI, infrastructure management | *(not published)* | *(not published)* | Static only |

---

## Routing and Firewall Policy

All inter-VLAN routing decisions are enforced at the pfSense firewall. The switch has no Layer 3 capability beyond the management SVI on VLAN 99.

| Source | Destination | Policy |
|--------|-------------|--------|
| VLAN 10 (Trusted) | Internet | Permit |
| VLAN 10 (Trusted) | VLAN 99 (Mgmt) | Permit — management access only |
| VLAN 10 (Trusted) | pfSense GUI | Permit |
| VLAN 20 (IoT) | Internet | Permit |
| VLAN 20 (IoT) | Any LAN VLAN | Deny |
| VLAN 30 (Guest) | Internet | Permit |
| VLAN 30 (Guest) | Any LAN VLAN | Deny |
| VLAN 40 (Lab) | Internet | Permit |
| VLAN 40 (Lab) | VLAN 10 (Trusted) | Deny |
| VLAN 40 (Lab) | VLAN 20 (IoT) | Deny |
| Any | VLAN 99 (Mgmt) | Deny (except VLAN 10) |

---

## Kubernetes Cluster Network

Both Kubernetes clusters run on VLAN 40 (Lab), providing full network isolation from trusted and IoT segments.

**RKE2 Cluster (production)**
- Control plane: Bosgame Mini PC
- Worker: Dell PowerEdge R220
- External access: Cloudflare Tunnel (outbound-only; no inbound NAT or exposed firewall ports)

**kubeadm Cluster (security research)**
- Single host: Dell PowerEdge R410 (128 GB RAM)
- Used for CKA exam preparation, security tooling evaluation, and running deliberately misconfigured workloads for security research
- Isolated within VLAN 40; no routing path to production cluster nodes at the application layer

---

## External Access — Cloudflare Tunnel

External access to cluster workloads is provided exclusively via Cloudflare Tunnel, running as a daemon within the RKE2 cluster. This eliminates the need for inbound firewall port mappings or dynamic DNS, and shifts authentication and access control to Cloudflare Access policies.

```
[External Client]
        │ HTTPS (Cloudflare edge)
        ▼
[Cloudflare Edge Network]
        │ encrypted tunnel (outbound from cluster)
        ▼
[cloudflared — running in RKE2 cluster, VLAN 40]
        │
        ▼
[Cluster Services]
```

---

## DNS

Internal DNS resolution is provided for VLAN 10 and VLAN 40. IoT and Guest VLANs resolve via upstream public resolvers enforced by pfSense DNS resolver policy.

---

## Management Plane

Switch management is accessible only via the VLAN 99 SVI, reachable from VLAN 10. The pfSense web GUI is accessible from VLAN 10 only. SSH and console access to all servers are restricted to VLAN 10 source addresses by firewall rule.
