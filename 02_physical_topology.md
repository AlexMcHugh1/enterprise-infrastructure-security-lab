# Physical Topology

This document describes the physical cabling layout and logical connectivity between the patch panel, firewall, switch, and end devices. Specific patch panel port assignments are not published here; the table below reflects the device-to-VLAN mapping that the cabling implements.

---

## Device → Switch VLAN Assignment

| Device | VLAN | Role | Notes |
|--------|------|------|-------|
| Protectli FW4C (LAN) | Trunk | 802.1Q uplink | Carries VLANs 10, 20, 30, 40, 99 |
| Bosgame Mini PC | 40 — Lab | RKE2 control plane | Rack shelf |
| Dell PowerEdge R220 | 40 — Lab | RKE2 worker node | 1U rack |
| Dell PowerEdge R410 | 40 — Lab | kubeadm security cluster | 2U rack |
| Dell OptiPlex | 10 — Trusted | File storage / media | Rack shelf |
| Desktop PC | 10 — Trusted | Admin workstation | External (desk) |
| ThinkPad | 40 — Lab | Pentesting / security research | External |
| TP-Link EAP653 | 20 — IoT | Wi-Fi 6 AP (isolated VLAN) | External, passive PoE+ |

All physical ethernet runs terminate at the 24-port patch panel (U12). Short patch leads connect the panel to the switch. The firewall trunk port carries all VLANs over a single physical link.

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

---

## Cabling Notes

- All rack-to-switch runs use Cat6 terminated at the patch panel; short patch leads to the switch for clean cable management.
- The firewall LAN interface connects to the switch trunk port via a single Cat6 run through the patch panel.
- The TP-Link EAP653 AP is cabled back to the switch via the patch panel and powered by a passive PoE+ injector; it carries VLAN 20 traffic only.
- Desktop PC and ThinkPad connect via ethernet runs to the patch panel when docked. ThinkPad also connects wirelessly through EAP653 on a dedicated SSID mapped to VLAN 40 when needed.
