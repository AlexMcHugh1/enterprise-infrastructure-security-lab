# Rack Plan & Inventory

This document records the physical rack layout and full hardware inventory for the homelab. The rack is a 12U wall-mount unit. All configurations are version-controlled in this repository.

---

## Rack Elevation (Front View)

> Top-to-bottom. U positions reflect current physical layout.

```
U12 ┌────────────────────────────────────────────────────────┐
    │  24-Port Patch Panel (Cat6)                            │
U11 ├────────────────────────────────────────────────────────┤
    │  Cable Manager / Brush Panel                           │
U10 ├────────────────────────────────────────────────────────┤
    │  Cisco WS-C2960S-24TS-L (Managed Switch)               │
U09 ├────────────────────────────────────────────────────────┤
    │  Dell PowerEdge R220                                   │
U08 ├────────────────────────────────────────────────────────┤
    │  Dell PowerEdge R410                           (2U)    │
U07 ├────────────────────────────────────────────────────────┤
    │  (R410 continued)                                      │
U06 ├────────────────────────────────────────────────────────┤
    │  1U Fan Shelf (exhaust)                                │
U05 ├────────────────────────────────────────────────────────┤
    │  Front-mount PDU                                       │
U04 ├────────────────────────────────────────────────────────┤
    │  Shelf — Protectli FW4C / Bosgame Mini PC              │
U03 ├────────────────────────────────────────────────────────┤
    │  Dell OptiPlex + Storage                               │
U02 ├────────────────────────────────────────────────────────┤
    │                                                        │
U01 └────────────────────────────────────────────────────────┘
```

---

## Hardware Inventory

| Item | Make / Model | Role | Mount | OS / Firmware |
|------|--------------|------|-------|---------------|
| Patch Panel | 24-port Cat6 | Cable termination | U12 | — |
| Cable Manager | Brush panel | Cable hygiene | U11 | — |
| Managed Switch | Cisco WS-C2960S-24TS-L | L2 switching, VLAN segmentation | U10 | IOS 15.x |
| Server | Dell PowerEdge R220 (32 GB RAM) | RKE2 worker node | U09 | Rocky Linux |
| Server | Dell PowerEdge R410 (128 GB RAM) | Dedicated kubeadm cluster — security research | U08–U07 | Rocky Linux |
| Fan Shelf | 1U fan tray | Active cooling / airflow | U06 | — |
| PDU | Front-mount PDU | Power distribution | U05 | — |
| Firewall | Protectli FW4C | pfSense CE — router, firewall, VLAN termination | U04 (shelf) | pfSense CE |
| Mini PC | Bosgame (Ryzen 7 5825U, 32 GB RAM, 1 TB NVMe) | RKE2 control plane | U04 (shelf) | Rocky Linux |
| Desktop | Dell OptiPlex | File storage and media server | U03 | Ubuntu Server |
| SATA Dock | 2-bay USB | Bulk storage (8 TB + 2 TB HDD) | U03 (with OptiPlex) | — |
| AP | TP-Link EAP653 (AX3000, Wi-Fi 6) | Wireless — VLAN 20 (IoT), isolated | External | Omada |
| Workstation | Desktop PC | Admin workstation | External (desk) | Arch Linux |
| Laptop | ThinkPad | Pentesting and security research | External | Arch Linux |
| ISP Router | Home router | Internet uplink | External | — |

---

## Power & Cabling Notes

- **PDU** feeds the switch, Protectli, Bosgame, both PowerEdge servers, SATA dock, and fan shelf. All adapters labeled.
- **Trunk** from Protectli LAN → Switch carries VLANs 10, 20, 30, 40, 99 via 802.1Q.
- **WAN** uplink runs from ISP router to Protectli WAN port via patch panel.
- **TP-Link EAP653** connects to the switch via patch panel, powered by a passive PoE+ injector; assigned to VLAN 20.
- **Airflow:** Fan shelf set to exhaust. R410 and R220 use internal fans; leave gap between dense rack units.
- **All patch leads** are labeled at both ends. Short (~0.3 m) leads from patch panel to switch for clean dressing.

---

## Logical Role Summary

```
[ISP Router] ──WAN──► [Protectli FW4C — pfSense]
                              │ 802.1Q Trunk
                       [Cisco WS-C2960S]
                              │
              ┌───────────────┼──────────────────┐
              │               │                  │
         VLAN 10          VLAN 40            VLAN 20
         Trusted          Lab / K8s          IoT / Wi-Fi
         ─────────        ─────────────      ───────────
         Desktop PC       Bosgame (CP)       EAP653 AP
         OptiPlex         R220 (worker)
                          R410 (security)
                          ThinkPad (pentest)
```
