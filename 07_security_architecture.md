# Security Architecture

This document describes the security design rationale behind the homelab network. It covers trust zones, threat model, defence-in-depth controls, and attack surface analysis. Configuration details are in the relevant per-component documents; this document explains the *why*.

---

## Design Principles

**1. Default deny everywhere.**
All inter-VLAN traffic is blocked unless explicitly permitted. pfSense evaluates rules on ingress per interface; the implicit deny-all at the end of every ruleset is the foundation everything else builds on.

**2. Least-privilege network access.**
Each device is on the least-privileged VLAN that satisfies its operational requirements. IoT devices get internet access; they get nothing else. Lab servers get internet access for package management and Cloudflare Tunnel egress; they cannot reach the trusted segment under any circumstances.

**3. Isolation of experimental from operational.**
The R410 kubeadm cluster exists specifically to run misconfigured, vulnerable, or untrusted workloads in a controlled environment without any risk to the production RKE2 cluster or trusted network. VLAN segmentation enforces this at the network layer; it is not dependent on application-layer controls.

**4. No inbound attack surface on WAN.**
There are no inbound NAT rules, no port forwarding, and no exposed services on the WAN interface. External access to cluster services is exclusively via Cloudflare Tunnel — an outbound-only encrypted connection. An attacker scanning the WAN IP finds nothing.

**5. Management plane is separate and locked down.**
Infrastructure management (switch, pfSense GUI, SSH) is reachable only from VLAN 10 (Trusted). The switch management SVI is on a dedicated VLAN 99 that no lab or IoT device can reach. This means a compromised lab server cannot pivot to reconfigure network infrastructure.

**6. Everything is version-controlled.**
All network configuration is documented in this repository. Changes are tracked, diffs are reviewable, and the intended state of the network is always explicit. This is the operational baseline against which drift is measured.

---

## Trust Zones

```
┌─────────────────────────────────────────────────────────────────┐
│  INTERNET (untrusted)                                           │
│  ─────────────────────────────────────────────────────────────  │
│  │ WAN — no inbound rules; Cloudflare Tunnel egress only        │
│  ▼                                                              │
│  ┌──────────────────────┐                                       │
│  │ pfSense CE (FW4C)    │  ← Trust boundary enforcement         │
│  └──────────────────────┘                                       │
│           │                                                     │
│  ┌────────┴─────────────────────────────────────────────────┐   │
│  │  VLAN 10 — TRUSTED                                       │   │
│  │  Admin workstations, internal services                   │   │
│  │  Can reach: internet, VLAN 99, pfSense GUI               │   │
│  │  Cannot reach: VLAN 20, VLAN 30, VLAN 40 (by design)    │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  VLAN 20 — IOT (untrusted devices)                       │   │
│  │  Wi-Fi AP, smart devices                                 │   │
│  │  Can reach: internet only                                │   │
│  │  Cannot reach: any internal VLAN                         │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  VLAN 30 — GUEST (untrusted users)                       │   │
│  │  Temporary internet access                               │   │
│  │  Can reach: internet only                                │   │
│  │  Cannot reach: any internal VLAN                         │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  VLAN 40 — LAB (semi-trusted, isolated)                  │   │
│  │  Kubernetes clusters, security research, pentesting      │   │
│  │  Can reach: internet, Cloudflare Tunnel egress            │   │
│  │  Cannot reach: VLAN 10, VLAN 20, VLAN 99                 │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  VLAN 99 — MANAGEMENT (infrastructure only)              │   │
│  │  Switch SVI                                              │   │
│  │  Reachable from: VLAN 10 only                            │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Threat Model

### Threats Addressed

| Threat | Mitigating Control |
|--------|-------------------|
| Compromised IoT device pivoting to trusted hosts | VLAN 20 has no route to VLAN 10; enforced by pfSense deny rule |
| Compromised lab server reaching admin workstations | VLAN 40 → VLAN 10 explicitly denied; no exceptions |
| Compromised lab server reconfiguring network infrastructure | VLAN 40 → VLAN 99 denied; switch and pfSense GUI unreachable from lab |
| External attacker scanning WAN IP | No inbound NAT; no open ports; nothing to find |
| External attacker brute-forcing exposed services | No services exposed on WAN; Cloudflare Access enforces authentication before traffic reaches the cluster |
| Lateral movement from security research workloads | R410 kubeadm cluster is on VLAN 40, physically separate from RKE2 cluster; no application-layer trust between them |
| DNS rebinding attacks | pfSense Unbound DNS rebind protection enabled; IoT/Guest VLANs have no visibility into internal DNS |
| Rogue DHCP server on a VLAN | Single authoritative DHCP server per VLAN on pfSense; no DHCP relay between VLANs |
| Configuration drift going undetected | Intended state is documented and version-controlled; deviations are visible in git diff |

### Threats Accepted / Out of Scope

| Threat | Rationale |
|--------|-----------|
| Physical access to rack | Physical security is out of scope for this network design; assumed trusted |
| Compromise of pfSense itself | pfSense is updated regularly; GUI is restricted to VLAN 10; WAN management access is disabled |
| ISP-level traffic interception | WAN traffic is not end-to-end encrypted beyond TLS; VPN egress is available from VLAN 10 if required |
| Supply chain compromise of OS packages | Rocky Linux and Ubuntu Server use signed packages; addressed at the OS layer, not the network layer |

---

## Attack Surface Analysis

### WAN Interface
- DHCP client only — acquires address from upstream router
- No inbound NAT rules
- No services listening on WAN
- Default deny-all inbound
- **Effective inbound attack surface: zero**

### VLAN 10 — Trusted
- pfSense GUI exposed within this VLAN (HTTPS only)
- SSH to servers is source-restricted to VLAN 10 addresses
- Workstations are managed endpoints (patched, no unnecessary services)

### VLAN 40 — Lab
- Highest internal attack surface by design — this is where security research happens
- Kubernetes API server, etcd, and node services are accessible within VLAN 40
- R410 cluster intentionally runs misconfigured workloads; this is expected
- Firewall policy ensures any compromise is contained to VLAN 40
- Cloudflare Access policies provide authentication for any externally-reachable services

### Management Plane
- Switch SSH accessible only from VLAN 10
- pfSense GUI accessible only from VLAN 10
- No remote management from WAN or any other VLAN
- CDPdisabled on switch; HTTP server disabled

---

## Defence-in-Depth Layers

```
Layer 1 — Perimeter
  pfSense WAN: default deny inbound, no exposed services

Layer 2 — Network Segmentation
  VLAN isolation enforced at pfSense; switch is pure L2
  Five distinct trust zones with explicit inter-zone policy

Layer 3 — Management Plane Isolation
  VLAN 99 dedicated to infrastructure management
  Reachable only from VLAN 10; locked behind firewall rules

Layer 4 — Zero-Trust External Access
  Cloudflare Tunnel replaces port forwarding and VPN exposure
  Authentication enforced at Cloudflare edge before traffic enters the network

Layer 5 — Cluster Isolation
  Production RKE2 cluster and research kubeadm cluster are separate systems
  No shared credentials, no shared storage, no application-layer trust

Layer 6 — Configuration as Code
  Intended network state is version-controlled
  Changes are deliberate and traceable; drift is detectable
```

---

## Kubernetes Security Posture

### RKE2 Cluster

RKE2 ships with CIS Kubernetes Benchmark hardening applied by default:
- API server anonymous authentication disabled
- RBAC enforced; no cluster-admin bindings outside of system components
- etcd data encryption at rest enabled
- Network policies restrict pod-to-pod communication
- Cloudflare Tunnel handles ingress; no NodePort or LoadBalancer services expose ports on the host network

### kubeadm Cluster (R410)

This cluster is intentionally used to practice with misconfigured workloads for security research. The isolation model treats the entire R410 cluster as untrusted:
- Network boundary: VLAN 40, no route to VLAN 10 or management plane
- Physical boundary: separate bare-metal host
- No shared credentials or CA with the RKE2 cluster
- Misconfigured workloads here cannot affect production cluster stability or security
