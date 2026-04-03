# Hardware Inventory

Full inventory of homelab hardware with roles, OS, and networking notes.

---

## Inventory

| Item | Make / Model | Role | Location | OS / Firmware |
|------|--------------|------|----------|---------------|
| Managed Switch | Cisco WS-C2960S-24TS-L | L2 switching, VLAN segmentation | Rack | IOS 15.x |
| Patch Panel | 24-port Cat6 | Cable termination | Rack | — |
| Cable Manager | Brush panel | Cable hygiene | Rack | — |
| Fan Shelf | 1U fan tray | Active cooling / airflow | Rack | — |
| PDU | Front-mount PDU | Power distribution | Rack | — |
| Firewall | Protectli FW4C | pfSense CE — router, firewall, VLAN termination | Rack shelf | pfSense CE |
| Mini PC | Bosgame (Ryzen 7 5825U, 32 GB RAM, 1 TB NVMe) | RKE2 control plane | Rack shelf | Rocky Linux |
| Server | Dell PowerEdge R220 (32 GB RAM) | RKE2 worker node | Rack | Rocky Linux |
| Server | Dell PowerEdge R410 (128 GB RAM) | Dedicated kubeadm cluster — security research | External (top of rack) | Rocky Linux |
| Desktop | Dell OptiPlex | File storage and media server | Rack | Ubuntu Server |
| SATA Dock | 2-bay USB | Bulk storage (8 TB + 2 TB HDD) | With OptiPlex | — |
| AP | TP-Link EAP653 (AX3000, Wi-Fi 6) | Wireless — VLAN 20 (IoT), isolated | External | Omada |
| Workstation | Desktop PC | Admin workstation | Desk | Arch Linux |
| Laptop | ThinkPad | Pentesting and security research | External | Arch Linux |
| ISP Router | Home router | Internet uplink | External | — |

---

## Power & Cabling Notes

- **PDU** feeds the switch, Protectli, Bosgame, R220, R410, SATA dock, and fan shelf. All adapters labeled.
- **Trunk** from Protectli LAN → Switch carries VLANs 10, 20, 30, 40, 99 via 802.1Q.
- **WAN** uplink runs from ISP router to Protectli WAN port via patch panel.
- **TP-Link EAP653** connects to the switch via patch panel, powered by a passive PoE+ injector; assigned to VLAN 20.
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
