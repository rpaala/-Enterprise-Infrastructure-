# Phase 1 Progress: First VLAN

## Done
- Backed up the Proxmox network config before making changes
- Made bridge vmbr0 VLAN-aware
- Created a "Lab" network on the UniFi router (VLAN <LAB_VLAN_ID>, subnet <LAB_SUBNET>, DHCP)
- Cloned a test VM from the cloud-init template and tagged it VLAN <LAB_VLAN_ID>

## Exit checks
| Check | Result |
|---|---|
| Test VM gets an address from the Lab subnet | Pass |
| Proxmox web UI still reachable after the change | Pass |

## Lessons learned
- "Checksum mismatch" on VM edit means a clone task was still running. Refresh and retry.
- Cloud-init templates need a user and password set before console login.

## Next
- Firewall rule: Lab VLAN must not reach the management network