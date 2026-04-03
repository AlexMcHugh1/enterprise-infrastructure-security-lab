# Cisco WS-C2960S Switch Configuration

Reference configuration for the Cisco WS-C2960S-24TS-L. This documents the VLAN definitions, trunk uplink to pfSense, and per-device access port assignments. IP addresses use RFC 5737 documentation ranges.

---

## VLAN Definitions

```
vlan 10
 name Trusted
vlan 20
 name IoT
vlan 30
 name Guest
vlan 40
 name Lab
vlan 99
 name Mgmt
exit
```

---

## Trunk Port — Uplink to pfSense

```
interface GigabitEthernet1/0/1
 description Uplink_to_pfSense_LAN
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
 switchport trunk native vlan 10
 spanning-tree portfast trunk
exit
```

---

## Access Ports — Lab Servers (VLAN 40)

```
interface GigabitEthernet1/0/2
 description Bosgame_RKE2_ControlPlane
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
exit

interface GigabitEthernet1/0/3
 description DellR220_RKE2_Worker
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
exit

interface GigabitEthernet1/0/4
 description DellR410_Security_Cluster
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
exit

interface GigabitEthernet1/0/5
 description ThinkPad_Pentesting
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
exit
```

---

## Access Ports — Trusted (VLAN 10)

```
interface GigabitEthernet1/0/6
 description DellOptiPlex_Storage
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
exit

interface GigabitEthernet1/0/7
 description DesktopPC_AdminWorkstation
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
exit
```

---

## Access Port — IoT AP (VLAN 20)

```
interface GigabitEthernet1/0/8
 description TPLink_EAP653_IoT_AP
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
exit
```

---

## Management Interface (VLAN 99 SVI)

The switch management interface is assigned a static address on VLAN 99. The address below uses an RFC 5737 documentation range as an example.

```
interface Vlan99
 description Management_SVI
 ip address 198.51.100.2 255.255.255.0
 ! Example address — RFC 5737 documentation range. Replace with actual mgmt address.
 no shutdown
exit

ip default-gateway 198.51.100.1
! Example gateway — matches pfSense VLAN 99 interface address.

end
write memory
```

---

## Global Hardening Settings

```
no ip http server
no ip http secure-server
service password-encryption
logging buffered 16384
no cdp run
```
