# Monitoring & Logging

This document describes the observability stack for the homelab network: what is logged, where logs are collected, what is monitored, and what conditions trigger investigation.

---

## Logging Architecture

```
[pfSense firewall logs]  ─────────────────────┐
[Cisco switch syslog]    ──────────────────────┤
[Rocky Linux / Ubuntu journald] ───────────────┼──► [Centralised syslog — Dell OptiPlex]
[Kubernetes events & audit logs] ──────────────┤
[cloudflared tunnel logs] ─────────────────────┘
```

All infrastructure components forward logs to a centralised syslog receiver on the Dell OptiPlex (VLAN 10, Trusted). This host has no role in the data path, so a logging failure does not affect network or cluster availability.

---

## pfSense Firewall Logging

### What is logged

All pfSense firewall rules that **block** traffic are logged by default. Selected permit rules are also logged for audit purposes.

| Rule / Event | Logged | Rationale |
|---|---|---|
| All blocked packets (any interface) | Yes | Primary threat indicator; lateral movement attempts are visible here |
| VLAN 40 → VLAN 10 deny hits | Yes | Any hit indicates a potentially compromised lab host attempting pivot |
| VLAN 20 → any internal deny hits | Yes | IoT device attempting LAN access is anomalous |
| WAN inbound deny hits | Yes | Port scan / active reconnaissance baseline |
| DHCP lease events | Yes | Device tracking; unexpected MACs visible immediately |
| pfSense auth events (GUI login) | Yes | Admin access audit trail |
| Cloudflare Tunnel egress | No | Tunnel health is monitored separately via cloudflared logs |

### Log format and retention

pfSense logs are forwarded via syslog (UDP) to the centralised collector on VLAN 10. Local circular buffer on pfSense retains recent events for immediate triage. The collector retains 90 days of firewall logs.

---

## Switch Logging

The Cisco WS-C2960S sends syslog messages to the centralised collector.

```
logging host <SYSLOG-SERVER-IP>
logging trap informational
logging source-interface Vlan99
service timestamps log datetime msec localtime
```

### Logged switch events

| Event | Severity | Notes |
|---|---|---|
| Link up / link down | Informational | Unexpected port state changes visible immediately |
| STP topology changes | Warning | Indicates possible loop or rogue switch |
| MAC address flapping | Warning | Potential ARP/MAC spoofing indicator |
| Authentication failures (if 802.1X configured) | Error | |
| Config changes | Critical | `archive log config` captures before/after diffs |

---

## Host Logging (Rocky Linux / Ubuntu Server)

All servers forward systemd journal entries to the centralised syslog via rsyslog or systemd-journal-remote. Key sources:

| Source | What is captured |
|---|---|
| `sshd` | All auth attempts, successes, and failures — primary access audit |
| `sudo` | All privilege escalation events |
| `kernel` | OOM events, hardware errors, USB device insertion |
| `firewalld` / `ufw` | Host-level firewall events (defence in depth beyond VLAN segmentation) |
| `auditd` | Syscall auditing on servers that handle sensitive operations |
| `dnf` / `apt` | Package installation and removal events |

SSH authentication failures are an expected background noise for any internet-connected host; the useful signal is failures on internal-only hosts where no legitimate external access should occur.

---

## Kubernetes Monitoring

### RKE2 Cluster

**Cluster health** is monitored via Prometheus and Grafana running within the cluster, scraped via Cloudflare Tunnel for external dashboard access.

| Metric | Alert threshold |
|---|---|
| Node NotReady | Immediate |
| Pod CrashLoopBackOff (system namespace) | Immediate |
| etcd leader elections | Investigate |
| API server request error rate | > 5% over 5 minutes |
| Node disk pressure | > 85% usage |
| Node memory pressure | > 90% usage |

**Kubernetes audit logging** is enabled. The audit policy captures:
- All requests to the `kube-system` namespace
- All secret reads and writes
- All RBAC binding creates/deletes
- Failed authentication attempts against the API server

**Network policy violations** are logged by the CNI.

### kubeadm Cluster (R410)

Monitored for cluster health only. No production alerting — this cluster is expected to have failing, misconfigured, or intentionally broken workloads as part of security research. Node availability is tracked; everything else is treated as signal in a research context.

---

## Alerting

Alerting is intentionally kept simple — alert fatigue defeats the purpose.

| Condition | Channel | Priority |
|---|---|---|
| VLAN 40 → VLAN 10 firewall deny hit | Immediate review | High |
| New MAC address on VLAN 10 | Review | Medium |
| SSH auth failure spike on any server | Review | Medium |
| RKE2 node NotReady | Immediate | High |
| WAN deny spike (> 10× baseline) | Investigate | Medium |
| pfSense GUI login from unexpected host | Immediate review | High |
| Switch link down (unexpected) | Investigate | Medium |

---

## Log Review Cadence

| Review | Frequency | Scope |
|---|---|---|
| Firewall block log | Daily | Scan for anomalous inter-VLAN traffic |
| SSH auth log | Weekly | Check for unexpected access patterns |
| Kubernetes audit log | Weekly | RBAC changes, secret access |
| DHCP lease table | Weekly | Unknown MACs |
| Switch syslog | As-triggered | Link events, STP changes |

---

## Configuration Backup Logging

pfSense configuration is backed up automatically on every change via the AutoConfigBackup package. Backups are encrypted and stored off-device. The git repository in this project captures the *intended* state; AutoConfigBackup captures the *actual* running state including timestamps of every change.

The Cisco switch running configuration is backed up to the centralised host weekly via SCP:

```bash
# Run from VLAN 10 management host
ssh admin@<switch-mgmt-ip> "show running-config" > backups/switch-$(date +%F).conf
```

Backup files are retained for 30 days and compared against the previous version on each run. Any diff is flagged for review.
