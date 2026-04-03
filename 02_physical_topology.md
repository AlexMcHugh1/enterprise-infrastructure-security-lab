# Network Topology

## Device → VLAN Assignment

| Device | VLAN | Role |
|--------|------|------|
| Protectli FW4C (LAN) | Trunk | 802.1Q uplink — carries VLANs 10, 20, 30, 40, 99 |
| Bosgame Mini PC | 40 — Lab | RKE2 control plane |
| Dell PowerEdge R220 | 40 — Lab | RKE2 worker node |
| Dell PowerEdge R410 | 40 — Lab | kubeadm security cluster |
| Dell OptiPlex | 10 — Trusted | File storage / media server |
| Desktop PC | 10 — Trusted | Admin workstation |
| ThinkPad | 40 — Lab | Pentesting / security research |
| TP-Link EAP653 | 20 — IoT | Wi-Fi 6 AP — isolated VLAN |

---

## Logical Topology

```
[ISP / Home Router]
        │ WAN
        ▼
[Protectli FW4C — pfSense CE]
  igc0: WAN
  igc1: LAN (802.1Q trunk → switch)
        │
        ▼
[Cisco WS-C2960S-24TS-L]
        │
 ┌──────┴───────────────────────────────────────────────┐
 │                                                      │
 │  VLAN 10 (Trusted)         VLAN 40 (Lab)             │
 │  ─────────────────         ─────────────             │
 │  Desktop PC                Bosgame (RKE2 CP)         │
 │  Dell OptiPlex             Dell R220 (RKE2 worker)   │
 │                            Dell R410 (kubeadm)       │
 │  VLAN 20 (IoT)             ThinkPad (pentest)        │
 │  ─────────────                                       │
 │  TP-Link EAP653            VLAN 30 (Guest)           │
 │  [Wi-Fi clients]           ───────────────           │
 │                            Internet-only             │
 │                                                      │
 │  VLAN 99 (Mgmt)                                      │
 │  ──────────────                                      │
 │  Switch SVI (static)                                 │
 │  Reachable from VLAN 10 only                         │
 └──────────────────────────────────────────────────────┘
        │
        ▼
[Cloudflare Tunnel — outbound egress from VLAN 40]
  Zero-trust external access to cluster workloads.
  No inbound firewall ports required.
```
