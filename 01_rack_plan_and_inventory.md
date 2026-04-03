# Hardware Inventory

| Item | Make / Model | Role | OS / Firmware |
|------|--------------|------|---------------|
| Firewall | Protectli FW4C | pfSense CE — router, firewall, VLAN termination | pfSense CE |
| Managed Switch | Cisco WS-C2960S-24TS-L | L2 switching, VLAN segmentation | IOS 15.x |
| Mini PC | Bosgame (Ryzen 7 5825U, 32 GB RAM, 1 TB NVMe) | RKE2 control plane | Rocky Linux |
| Server | Dell PowerEdge R220 (32 GB RAM) | RKE2 worker node | Rocky Linux |
| Server | Dell PowerEdge R410 (128 GB RAM) | Dedicated kubeadm cluster — security research | Rocky Linux |
| Desktop | Dell OptiPlex | File storage and media server | Ubuntu Server |
| AP | TP-Link EAP653 (AX3000, Wi-Fi 6) | Wireless — VLAN 20 (IoT), isolated | Omada |
| Workstation | Desktop PC | Admin workstation | Arch Linux |
| Laptop | ThinkPad | Pentesting and security research | Arch Linux |

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
