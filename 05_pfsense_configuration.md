# pfSense CE Configuration Reference

Reference configuration for the Protectli FW4C running pfSense CE. Covers interface assignments, VLAN termination, DHCP scopes, firewall rules, and Cloudflare Tunnel egress. IP addresses use RFC 5737 documentation ranges for illustration.

---

## Physical Interfaces

| Interface | Port | Role |
|-----------|------|------|
| igc0 | WAN | DHCP from ISP/home router |
| igc1 | LAN | 802.1Q trunk to Cisco switch |

The LAN interface carries all VLANs as 802.1Q sub-interfaces. No IP is assigned to the physical igc1 interface — addressing lives on the VLAN sub-interfaces.

---

## VLAN Interfaces

| VLAN | Tag | Interface Name | Example Gateway | DHCP | Notes |
|------|-----|----------------|-----------------|------|-------|
| 10 | 10 | LAN_Trusted | 192.0.2.1/24 | Yes (.50–.199) | Admin default gateway; pfSense GUI accessible from here |
| 20 | 20 | LAN_IoT | 198.51.100.1/24 | Yes (.50–.199) | Internet-only; isolated from all LAN VLANs |
| 30 | 30 | LAN_Guest | 203.0.113.1/24 | Yes (.50–.199) | Internet-only; no LAN access |
| 40 | 40 | LAN_Lab | *(not published)* | Yes | Kubernetes clusters and security research; no path to VLAN 10 |
| 99 | 99 | LAN_Mgmt | *(not published)* | No — static only | Switch SVI; reachable from VLAN 10 only |

---

## DHCP Scopes

Each VLAN interface runs an independent DHCP server scoped to that subnet. Static DHCP assignments (by MAC) are used for all servers and infrastructure devices on VLANs 10 and 40 to maintain stable addressing.

- **VLAN 10:** dynamic pool .50–.199; static assignments below .50 for OptiPlex and Desktop PC
- **VLAN 20:** dynamic pool .50–.199 (Wi-Fi clients via EAP653)
- **VLAN 30:** dynamic pool .50–.199
- **VLAN 40:** dynamic pool .50–.199; static assignments below .50 for Bosgame, R220, R410
- **VLAN 99:** no DHCP — switch management address is statically configured on the switch SVI

---

## Firewall Rules Summary

Rules are applied per-interface on ingress. Default deny is implicit on all interfaces.

### VLAN 10 — Trusted
```
Pass    TCP/UDP  VLAN10 net → any:53         DNS
Pass    TCP      VLAN10 net → pfSense:443    GUI management
Pass    TCP      VLAN10 net → VLAN99 net     Switch management
Pass    any      VLAN10 net → WAN            Internet
Block   any      VLAN10 net → RFC1918        Private ranges not explicitly allowed
```

### VLAN 20 — IoT
```
Pass    any      VLAN20 net → WAN            Internet only
Block   any      VLAN20 net → !WAN           No LAN access
```

### VLAN 30 — Guest
```
Pass    any      VLAN30 net → WAN            Internet only
Block   any      VLAN30 net → !WAN           No LAN access
```

### VLAN 40 — Lab
```
Pass    any      VLAN40 net → WAN            Internet (including Cloudflare Tunnel egress)
Block   any      VLAN40 net → VLAN10 net     No access to trusted segment
Block   any      VLAN40 net → VLAN99 net     No management access
Block   any      VLAN40 net → VLAN20 net     No access to IoT
```

### WAN
```
Block   any      WAN → any                   Default deny all inbound
```

No inbound NAT rules exist on the WAN interface. External access to cluster workloads is handled entirely by Cloudflare Tunnel (outbound-only).

---

## Cloudflare Tunnel

Cloudflare Tunnel (`cloudflared`) runs as a deployment within the RKE2 cluster on VLAN 40. It establishes an outbound-only encrypted connection to the Cloudflare edge, through which external requests are proxied to internal services. This removes the need for any inbound firewall rules or port forwarding on the WAN interface.

The pfSense WAN firewall has no NAT or inbound permit rules for cluster traffic. All external access is mediated by Cloudflare Access policies.

---

## DNS Resolver

pfSense Unbound provides DNS resolution for internal hosts on VLANs 10 and 40. IoT and Guest VLANs are served by upstream resolvers via pfSense; they have no visibility into internal DNS entries.

DNS rebind protection is enabled. Private reverse zones are configured for VLANs 10, 40, and 99.

---

## NTP

pfSense operates as the NTP server for all internal VLANs. Upstream sync uses pool.ntp.org. All servers on VLAN 40 sync to pfSense to maintain consistent timestamps for log correlation.
