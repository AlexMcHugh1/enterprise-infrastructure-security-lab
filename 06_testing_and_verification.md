# Testing & Verification

Verification procedures for network segmentation, VLAN isolation, Kubernetes cluster health, and Cloudflare Tunnel connectivity. Run these after any configuration change to confirm the network behaves as designed.

> **Note:** IP addresses below use RFC 5737 documentation ranges as examples. Substitute actual addresses from the network design document.

---

## 1. VLAN Gateway Reachability

Confirm each VLAN interface on pfSense responds to its gateway address from a host on that VLAN.

```bash
# From a host on VLAN 10 (Trusted)
ping 192.0.2.1       # VLAN 10 gateway — expect: success

# From a host on VLAN 20 (IoT / Wi-Fi)
ping 198.51.100.1    # VLAN 20 gateway — expect: success

# From a host on VLAN 30 (Guest)
ping 203.0.113.1     # VLAN 30 gateway — expect: success

# From a host on VLAN 40 (Lab)
ping <VLAN40-GW>     # VLAN 40 gateway — expect: success
```

---

## 2. Internet Connectivity

Confirm outbound internet access from each VLAN that should have it.

```bash
# From VLAN 10, VLAN 20, VLAN 30, VLAN 40
curl -s https://one.one.one.one/cdn-cgi/trace | grep ip
# Expect: external IP from ISP — success for all four VLANs
```

---

## 3. VLAN Isolation Tests

These tests confirm that inter-VLAN firewall rules are enforced. All cross-VLAN attempts below must fail.

```bash
# From VLAN 20 (IoT) — attempt to reach Trusted gateway
ping 192.0.2.1       # expect: no response (blocked by pfSense)

# From VLAN 20 (IoT) — attempt to reach Lab gateway
ping <VLAN40-GW>     # expect: no response (blocked by pfSense)

# From VLAN 30 (Guest) — attempt to reach Trusted gateway
ping 192.0.2.1       # expect: no response (blocked by pfSense)

# From VLAN 40 (Lab) — attempt to reach Trusted gateway
ping 192.0.2.1       # expect: no response (blocked by pfSense)

# From VLAN 40 (Lab) — attempt to reach switch management SVI
ping <VLAN99-ADDR>   # expect: no response (blocked by pfSense)
```

---

## 4. Management Plane Access

Confirm management access is restricted to VLAN 10.

```bash
# From VLAN 10 (Trusted)
ping <VLAN99-SWITCH-SVI>    # expect: success
curl -sk https://<pfSense-GW>/ | grep -i pfsense   # expect: pfSense login page

# From VLAN 40 (Lab)
ping <VLAN99-SWITCH-SVI>    # expect: no response (denied)
```

---

## 5. RKE2 Cluster Health

Run from the admin workstation on VLAN 10 via SSH to the Bosgame control plane (VLAN 40).

```bash
# Node status
kubectl get nodes -o wide
# Expect: bosgame=Ready (control-plane), r220=Ready (worker)

# System pods
kubectl get pods -n kube-system
# Expect: all pods Running or Completed, no CrashLoopBackOff

# Cluster info
kubectl cluster-info
```

---

## 6. kubeadm Cluster Health (R410)

```bash
# SSH to R410, then:
kubectl get nodes
# Expect: r410=Ready

kubectl get pods -A
# Expect: all system pods healthy
```

---

## 7. Cloudflare Tunnel

```bash
# Check cloudflared pod status in RKE2 cluster
kubectl get pods -n <tunnel-namespace> -l app=cloudflared
# Expect: Running

# Check tunnel connection from Cloudflare dashboard or:
kubectl logs -n <tunnel-namespace> deployment/cloudflared | grep -i "connection"
# Expect: "Connection registered" entries, no error reconnects

# External access test
curl -I https://<your-tunnel-hostname>
# Expect: HTTP 200 (or appropriate service response)
# No inbound ports on pfSense WAN should be open for this to succeed
```

---

## 8. Switch Verification

```bash
# SSH to switch (from VLAN 10)
show vlan brief
# Expect: VLANs 10, 20, 30, 40, 99 active

show interfaces trunk
# Expect: Gi1/0/1 in trunking state, VLANs 10,20,30,40,99 allowed and active

show spanning-tree summary
# Expect: portfast active on all access ports
```

---

## 9. pfSense Firewall Rule Verification

From the pfSense GUI (accessible from VLAN 10):

- **Firewall > Rules** — review per-VLAN rules match documented policy
- **Diagnostics > Packet Capture** — use to confirm blocked traffic is being dropped at the expected interface
- **Status > DHCP Leases** — confirm static assignments for servers are resolving correctly

---

## Verification Checklist

| Test | Expected Result | Status |
|------|-----------------|--------|
| VLAN 10 gateway reachable | Success | |
| VLAN 20 gateway reachable | Success | |
| VLAN 30 gateway reachable | Success | |
| VLAN 40 gateway reachable | Success | |
| Internet from VLAN 10 | Success | |
| Internet from VLAN 20 | Success | |
| Internet from VLAN 30 | Success | |
| Internet from VLAN 40 | Success | |
| VLAN 20 → VLAN 10 blocked | Blocked | |
| VLAN 30 → VLAN 10 blocked | Blocked | |
| VLAN 40 → VLAN 10 blocked | Blocked | |
| VLAN 40 → VLAN 99 blocked | Blocked | |
| VLAN 10 → switch SVI reachable | Success | |
| RKE2 nodes Ready | Both nodes Ready | |
| kubeadm node Ready | R410 Ready | |
| Cloudflare Tunnel connected | Running, registered | |
| Switch VLANs active | 10,20,30,40,99 | |
