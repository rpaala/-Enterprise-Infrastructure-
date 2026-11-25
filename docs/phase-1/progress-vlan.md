# Phase 1 Progress: First VLAN

## Done

- Backed up the Proxmox network config before making changes
- Made bridge vmbr0 VLAN-aware
- Created a "Lab" network on the UniFi router (VLAN `<LAB_VLAN_ID>`, subnet `<LAB_SUBNET>`, DHCP)
- Cloned a test VM from the cloud-init template and tagged it with the Lab VLAN ID

## Exit checks

| Check | Result |
|---|---|
| Test VM gets an address from the Lab subnet | Pass |
| Proxmox web UI still reachable after the change | Pass |
| Lab VLAN has internet access | Pass |
| Lab VLAN blocked from management network | Pass |

## Lessons learned

- "Checksum mismatch" on VM edit means a clone task was still running. Refresh and retry.
- Cloud-init templates need a user and password set before console login.

## Next

- Phase 1 (foundation) is in progress: the Lab VLAN and NAS work are done; the remaining VLANs and WireGuard are still to do.
- Management, Trusted, IoT and Guest VLANs are to be completed.
- Run the remaining Phase 1 exit tests.
- Start Phase 2 after the remaining tasks from Phase 1 are complete.
