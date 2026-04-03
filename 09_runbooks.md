# Runbooks

Operational procedures for common and incident-response tasks. Each runbook states its trigger, prerequisites, steps, and expected outcome. Follow steps in order; deviations should be documented.

---

## RB-01 — Add a New Wired Device to the Network

**Trigger:** A new device needs network access.
**Prereqs:** Physical ethernet run terminated at patch panel; switch port available.

**Steps:**

1. Determine the appropriate VLAN based on device role:
   - Admin workstations → VLAN 10 (Trusted)
   - Lab servers, cluster nodes → VLAN 40 (Lab)
   - IoT or smart devices → VLAN 20 (IoT)
   - Temporary guest access → VLAN 30 (Guest)

2. Configure the switch port. SSH to the switch from VLAN 10:
   ```
   conf t
   interface GigabitEthernet1/0/<port>
    description <DeviceName_Role>
    switchport mode access
    switchport access vlan <VLAN-ID>
    spanning-tree portfast
   exit
   write memory
   ```

3. If the device requires a stable IP address, create a static DHCP mapping in pfSense:
   - **Services > DHCP Server > [VLAN interface] > Static Mappings**
   - Enter MAC address, desired IP (below .50 in the pool), and hostname.

4. Verify connectivity from the device:
   ```bash
   ping <VLAN-gateway>      # gateway should respond
   curl -s https://one.one.one.one/cdn-cgi/trace   # internet should resolve
   ```

5. Verify isolation from VLAN 10 (if device is not on VLAN 10):
   ```bash
   ping 192.0.2.1           # VLAN 10 gateway — should fail
   ```

6. Update this repository: add the device to `01_rack_plan_and_inventory.md` and `02_physical_topology.md`.

**Expected outcome:** Device receives IP, has internet access, cannot reach VLANs outside its own.

---

## RB-02 — Decommission a Device

**Trigger:** A device is being removed from the network permanently.

**Steps:**

1. Remove or reassign the switch port:
   ```
   conf t
   interface GigabitEthernet1/0/<port>
    description UNUSED
    switchport access vlan 99
    shutdown
   exit
   write memory
   ```
   Shutting down and moving to VLAN 99 prevents any device plugged into that port from getting network access until deliberately re-provisioned.

2. Remove the static DHCP mapping in pfSense if one exists.

3. If the device had any firewall rules referencing its IP, remove or update those rules in pfSense.

4. If the device was a Kubernetes node, drain it first:
   ```bash
   kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
   kubectl delete node <node-name>
   ```

5. Update `01_rack_plan_and_inventory.md` and `02_physical_topology.md`. Commit the change.

---

## RB-03 — Respond to a Suspected Compromised Host

**Trigger:** Firewall logs show unexpected inter-VLAN traffic from a host, anomalous outbound connections, or a host is behaving unexpectedly.

**Steps:**

1. **Identify the host.** Check pfSense firewall logs (**Status > System Logs > Firewall**) and filter by the source IP to understand what traffic it is generating.

2. **Isolate the host immediately.** Shut down its switch port to cut network access without touching the host itself (preserves volatile memory state for investigation):
   ```
   conf t
   interface GigabitEthernet1/0/<port>
    shutdown
   exit
   ```

3. **Do not log into the suspected host remotely.** Console access only if investigation requires live access. Remote login may trigger attacker persistence mechanisms or contaminate forensic evidence.

4. **Preserve evidence before remediation:**
   - Take a memory dump if tooling is available and the host is still running.
   - Capture the current process list, network connections, and cron jobs via console if accessible.
   - Review `/var/log/auth.log` (Ubuntu) or `/var/log/secure` (Rocky Linux) on the host.
   - Export pfSense firewall logs for the host's IP for the previous 7 days.

5. **Review blast radius.** Determine what the host could have reached:
   - If on VLAN 10: check pfSense logs for lateral movement to other VLAN 10 hosts and management interfaces.
   - If on VLAN 40: confirm pfSense blocked any VLAN 10 or VLAN 99 access attempts; check whether the RKE2 or kubeadm cluster API was accessed.
   - If a Kubernetes node is suspected: rotate cluster certificates and review API server audit logs.

6. **Remediate.** Rebuild the host from scratch. Do not attempt to clean a compromised system in-place.

7. **Restore service.** Re-image, re-provision, re-add to switch and DHCP. Run verification tests from `06_testing_and_verification.md`.

8. **Review and harden.** Identify the initial access vector. Update firewall rules, RBAC policies, or host configuration to close the gap. Document the incident and changes in git.

---

## RB-04 — pfSense Configuration Backup and Restore

**Trigger:** Scheduled backup, pre-change backup, or recovery from failed pfSense change.

### Backup

pfSense AutoConfigBackup runs automatically on every configuration save. For a manual backup before a significant change:

1. **Diagnostics > Backup & Restore > Backup Configuration**
2. Download the XML backup to the admin workstation.
3. Store in a secure location off the pfSense device.
4. Update the relevant documentation in this repository to reflect the intended post-change state, then commit before making the change. This creates a reviewable record of what changed and why.

### Restore

1. Boot pfSense (fresh install or existing).
2. **Diagnostics > Backup & Restore > Restore Configuration**
3. Upload the XML backup file.
4. pfSense will reboot and apply the configuration.
5. Verify all VLAN interfaces come up and firewall rules are present.
6. Run the connectivity and isolation tests from `06_testing_and_verification.md`.

---

## RB-05 — Add a New Kubernetes Node to the RKE2 Cluster

**Trigger:** Scaling the production RKE2 cluster with an additional worker node.

**Prereqs:** New server provisioned with Rocky Linux, network access on VLAN 40, static DHCP assignment configured.

**Steps:**

1. Complete RB-01 to add the node to VLAN 40 with a static IP.

2. On the RKE2 control plane (Bosgame), retrieve the node join token:
   ```bash
   cat /var/lib/rancher/rke2/server/node-token
   ```

3. On the new node, install RKE2 in agent mode:
   ```bash
   curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
   ```

4. Configure the agent to join the cluster:
   ```bash
   mkdir -p /etc/rancher/rke2
   cat > /etc/rancher/rke2/config.yaml <<EOF
   server: https://<control-plane-ip>:9345
   token: <node-token>
   EOF
   ```

5. Start and enable the agent:
   ```bash
   systemctl enable rke2-agent --now
   ```

6. Verify from the control plane:
   ```bash
   kubectl get nodes
   # New node should appear as Ready within ~60 seconds
   ```

7. Update `01_rack_plan_and_inventory.md` with the new node's details. Commit.

---

## RB-06 — Rotate Cloudflare Tunnel Credentials

**Trigger:** Scheduled rotation, suspected credential leak, or Cloudflare Access policy change.

**Steps:**

1. Log into the Cloudflare Zero Trust dashboard.
2. Navigate to **Networks > Tunnels**, select the active tunnel.
3. Generate a new tunnel token (this does not immediately invalidate the old one).
4. Update the Kubernetes secret in the RKE2 cluster:
   ```bash
   kubectl create secret generic cloudflare-tunnel-token \
     --from-literal=token=<new-token> \
     --namespace <tunnel-namespace> \
     --dry-run=client -o yaml | kubectl apply -f -
   ```
5. Restart the cloudflared deployment to pick up the new token:
   ```bash
   kubectl rollout restart deployment/cloudflared -n <tunnel-namespace>
   ```
6. Verify the tunnel reconnects:
   ```bash
   kubectl logs -n <tunnel-namespace> deployment/cloudflared | grep -i "connection"
   ```
7. Confirm external access is restored via the expected hostname.
8. Delete the old tunnel token from the Cloudflare dashboard.

---

## RB-07 — Switch IOS Upgrade

**Trigger:** Security advisory, new IOS release, or scheduled maintenance.

**Prereqs:** New IOS image verified against Cisco's published MD5/SHA512 hash. Maintenance window agreed.

**Steps:**

1. Back up the running configuration before any change:
   ```bash
   ssh admin@<switch-mgmt-ip> "show running-config" > backups/switch-pre-upgrade-$(date +%F).conf
   ```

2. Copy the new IOS image to the switch flash via TFTP or SCP:
   ```
   copy scp: flash:
   ```

3. Verify the image MD5 on the switch matches Cisco's published hash:
   ```
   verify /md5 flash:<image-filename>
   ```

4. Set the boot variable:
   ```
   conf t
   boot system flash:<new-image-filename>
   write memory
   end
   ```

5. Reload the switch (note: this will drop all connections for ~3 minutes):
   ```
   reload
   ```

6. After reload, verify IOS version and confirm all VLANs and trunk are operational:
   ```
   show version
   show vlan brief
   show interfaces trunk
   ```

7. Run the switch verification steps from `06_testing_and_verification.md`.
