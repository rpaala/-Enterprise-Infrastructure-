# Architecture Overview

A sanitized version of the home lab architecture plan. Addresses, hostnames and device identifiers are placeholders; see the [redaction policy](redaction-policy.md).

## Summary

The lab is one k3s cluster on a repurposed iMac (three Proxmox VMs) plus a Raspberry Pi 4, with a Raspberry Pi 3 as an independent watchdog, behind segmented VLANs on a UniFi gateway. A NAS holds cluster volumes and backups, and a nightly encrypted offsite copy covers its single disk.

### Design principles

- **Everything as code.** Terraform builds the VMs, Ansible configures every host, and Git drives the cluster, so any node can be wiped and rebuilt from the repo.
- **Watch from outside.** The Pi 3 runs monitoring outside the cluster, so a hypervisor outage still raises an alert.
- **Nothing exposed to the internet.** Remote work goes through a WireGuard VPN; no ports are forwarded to lab hosts.
- **Each piece shows a skill.** Infrastructure as code, configuration management, GitOps, backups with tested restores, and AI services on the cluster.

## Hardware and roles

| Device | Role | Network | Watch out for |
|---|---|---|---|
| iMac 12,1 (2011), quad-core i5, 20 GB RAM, 500 GB SSD, headless | Proxmox VE host for three k3s VMs and an ops VM | Thunderbolt-to-Gigabit adapter over powerline to the gateway; later a trunk (Management + Lab) | Wired link only; Linux fan control (applesmc + mbpfan) |
| Powerline AV2 kit | Carries the iMac's wired link over house wiring | Transparent bridge | Wall outlets only; pair for encryption; real speed is a few hundred Mbps |
| Raspberry Pi 4 | k3s agent for arm64 workloads | Lab VLAN, access port | Boot from USB SSD, not SD |
| Raspberry Pi 3 | Watchdog: Uptime Kuma and backup DNS in Docker | Lab VLAN, access port | 1 GB RAM, too small for a k3s server |
| NAS (single disk) | NFS volumes for the cluster; backup target for Proxmox and etcd | Lab VLAN; admin UI reachable only from Trusted | No RAID: a copy, not a safe one |
| UniFi gateway | VLANs, zone firewall, DHCP, local DNS, WireGuard | All VLANs | May need a small VLAN-aware switch for more wired devices |
| Windows laptop | Workstation (VS Code, Python, kubectl, SSH) | Trusted, VPN when away | Holds no state; everything lives in Git and the lab |
| Personal and IoT devices | Desktop, printer, VoIP phone, smart displays | Trusted or IoT | Kept out of the lab's blast radius |

## Network layout

Five VLANs plus a VPN subnet keep the lab away from home devices and management interfaces.

| VLAN | Subnet | Members | Can reach |
|---|---|---|---|
| Management | `<MGMT_SUBNET>` | Proxmox host, network gear | Internet, NAS for backups |
| Lab | `<LAB_SUBNET>` | k3s VMs, Pis, ops VM, NAS | Internet; Proxmox API from the ops VM only |
| Trusted | `<TRUSTED_SUBNET>` | Laptop and personal computers | Everything |
| IoT | `<IOT_SUBNET>` | TVs, speakers, cameras, printer, appliances | Internet only |
| Guest | `<GUEST_SUBNET>` | Visitors' devices | Internet only, client isolation |
| VPN | `<VPN_SUBNET>` | WireGuard clients | Same as Trusted |

Lab address plan (DHCP reservations), by role rather than address:

| Slot | Host |
|---|---|
| `<K3S_API_VIP>` | k3s API virtual IP (kube-vip) |
| `<CP1_IP>` | k3s-cp-1 |
| `<PI4_IP>`, `<PI3_IP>` | pi4 (agent), pi3 (watchdog) |
| `<WK1_IP>`, `<WK2_IP>` | k3s-wk-1, k3s-wk-2 |
| `<OPS_IP>` | ops VM |
| `<NAS_IP>` | NAS |
| DHCP pool | A block in the upper half of the Lab subnet |
| LoadBalancer pool | A separate block for Kubernetes services and ingress |

### Firewall rules (zone-based)

1. Trusted and VPN may reach Management and Lab.
2. Lab may not reach Management, except the ops VM to the Proxmox API (TCP 8006) for Terraform. The Proxmox host may reach the NAS over NFS for backups.
3. Lab may not reach Trusted or IoT, except one narrow exception for automated patch checks over SSH.
4. IoT and Guest reach the internet only. Trusted may start connections to IoT (casting, printing); IoT may never start connections inward.
5. From the WAN, only the WireGuard port is open. UPnP is off.

Switching: the hypervisor's port is a trunk (native Management, Lab tagged) feeding a VLAN-aware Proxmox bridge, so each VM picks its VLAN by tag. Pi and NAS ports are Lab access ports.

DNS: local records on the gateway under a private zone (`<LOCAL_DOMAIN>`), with app hostnames pointing at the ingress address. AdGuard Home on the Pi 3 filters DNS for every network, and outbound DNS from other devices is blocked so IoT can't bypass it.

### IoT isolation

Smart devices run firmware that can't be inspected and is rarely patched, so they live on their own VLAN with its own Wi-Fi (WPA2/WPA3 mixed, 2.4 GHz on). If one is compromised it can reach the internet but not computers, the lab, or the gateway's admin pages. mDNS forwarding is enabled only between Trusted and IoT so casting and AirPrint still work. Every device gets its default password changed, auto-updates on, and an entry in the device inventory.

## Proxmox on the iMac

Proxmox VE is the base OS: lightweight, well supported on Sandy Bridge with VT-x, and the most common open-source hypervisor in home labs. Alternatives considered: XCP-ng (heavier, extra management VM), Incus on Debian (lighter, less familiar to reviewers), bare-metal k3s (loses VMs, snapshots and the Terraform demo).

| VM | vCPU | RAM | Disk | Runs |
|---|---|---|---|---|
| k3s-cp-1 | 2 | 3 GB | 40 GB | k3s server with embedded etcd |
| k3s-wk-1 | 4 | 7 GB | 100 GB | Agent: AI services and apps |
| k3s-wk-2 | 2 | 4 GB | 80 GB | Agent: apps, data, monitoring |
| ops | 2 | 2 GB | 40 GB | Docker, Terraform, Ansible, Python, self-hosted runner |

- VMs are clones of one Ubuntu Server 24.04 cloud-init template, created by Terraform with the bpg/proxmox provider.
- VM disks live on local SSD; the NAS is NFS storage for backups only.
- Ten vCPUs on four cores is a 2.5:1 overcommit, fine while mostly idle. The GPU is unusable for AI, so models run on CPU (small quantized models via Ollama) or through hosted APIs.

### Installation and unattended reinstall

The host was installed by hand once from a checksum-verified Proxmox VE ISO (ext4 on the single SSD, no dual boot), then hardened: no-subscription repositories, a separate admin account with TOTP, key-only SSH, VLAN-aware bridge, fan control and NFS backup storage. See [build log 01](build-log/01-proxmox-install-imac.md).

Later reinstalls are unattended using Proxmox's automated installer: an `answer.toml` kept in the private working repo (with a hashed root password and placeholder filters) is baked into the ISO, the target disk is pinned by serial number so the stick can't wipe the wrong drive, and Ansible's `proxmox_host` role replaces the manual hardening list. A full rebuild becomes: boot the USB, wait, run one playbook, `terraform apply`.

## k3s cluster

| Node | Host | Arch | Role |
|---|---|---|---|
| k3s-cp-1 | iMac VM | amd64 | Server with embedded etcd (`--cluster-init`), tainted for system pods |
| k3s-wk-1 | iMac VM | amd64 | Agent: AI services and apps |
| k3s-wk-2 | iMac VM | amd64 | Agent: apps, data, monitoring |
| pi4 | Raspberry Pi 4 | arm64 | Agent: arm64 and edge workloads |
| pi3 | Raspberry Pi 3 | arm64 | Not in cluster: Uptime Kuma and AdGuard Home |

- **API endpoint:** kube-vip holds a virtual IP from day one, so adding servers later doesn't change the kubeconfig.
- **Ingress:** built-in Traefik; ServiceLB disabled in favor of kube-vip's cloud provider (or MetalLB).
- **Storage classes:** `local-path` for fast node-pinned data; `nfs` via csi-driver-nfs for anything that must move.
- **Mixed architectures:** amd64-only images pinned with `nodeSelector: kubernetes.io/arch: amd64`.
- **First workloads:** Flux, kube-prometheus-stack, Ollama with Open WebUI, n8n, Homepage.
- **Trade-off:** the cluster goes down with the iMac. An HA control plane demo later would add two small server VMs.

## Storage and backup (3-2-1)

| What | How | When | Where | Keep |
|---|---|---|---|---|
| Proxmox VMs | vzdump, snapshot mode, zstd | Nightly | NAS | 7 daily, 4 weekly |
| k3s etcd | Built-in etcd-snapshot | Every 12 hours | Local, then NAS | 10 snapshots |
| Cluster volumes | NFS storage class | Continuous | NAS | Covered by offsite copy |
| Config and manifests | Git | Every push | GitHub | Full history |
| NAS backup shares | restic, encrypted | Nightly | Backblaze B2 with Object Lock | 30 daily, 12 monthly |

Quarterly restore test: one VM into Proxmox and one etcd snapshot into a throwaway cluster. SMART alerts on the NAS disk; a mirror disk when budget allows.

## Remote access

The laptop works through VS Code Remote-SSH into the ops VM, so tools and state stay in the lab. Away from home it connects through WireGuard first, with split tunneling for only the Management and Lab subnets. One ed25519 key, password logins off everywhere. A Cloudflare Tunnel with Access login can publish a single read-only demo app without opening any port.

## Automation workflow

1. Edit in VS Code on the ops VM and open a pull request.
2. GitHub Actions checks formatting, validates Terraform, lints Ansible and scans for secrets. A self-hosted runner inside the Lab VLAN posts the `terraform plan`.
3. On merge, the runner applies Terraform and runs Ansible against changed hosts.
4. Flux reconciles the cluster from the new commit.

Cluster secrets are encrypted with SOPS and age; host secrets use Ansible Vault; Terraform state is remote. Kubeconfigs, keys and state never get committed.

## Security baseline

| Area | Controls |
|---|---|
| Accounts | Password manager, passkeys or TOTP everywhere, recovery codes stored offline, non-admin daily accounts |
| Edge | Auto firmware updates, WPA3 on Trusted, IPS on, no port forwards except VPN, UPnP off |
| DNS | AdGuard Home filtering with threat-intel and DoH-bypass blocklists; outbound DNS locked to the resolver |
| Backups | 3-2-1 with an immutable offsite copy and quarterly restore drills |
| Monitoring | Logs from every host and the gateway in Loki; alerts to phone; daily AI security digest from a local model |
| Inventory | Device inventory in the private repo; weekly job alerts on unknown devices; patch status per host |
| Hardening | Disk encryption, screen lock and firewall on every computer; automatic security updates on Linux hosts |
| Secrets | SOPS/age, Ansible Vault, a secrets manager for API keys, gitleaks in pre-commit and CI, GitHub secret scanning |
| Incident response | A one-page offline plan: isolate, preserve, rotate, rebuild, review |

### AI security digest

Every morning an n8n workflow pulls the last 24 hours of security events from Loki, asks a local Ollama model for a plain-language summary, and sends it to a phone. The model runs locally, so logs never leave the house.

## Risks

| Risk | Effect | Mitigation |
|---|---|---|
| 2011 iMac hardware fails | Cluster and ops VM down | Pi 3 watchdog alerts; rebuild from Terraform and backups |
| 20 GB RAM ceiling | Few more VMs | New services go in the cluster, not new VMs |
| Single NAS disk fails | Volumes and local backups lost | Offsite copy, SMART alerts, mirror later |
| SD card wear | Pi stops booting | Both Pis boot from USB SSDs |
| Mixed amd64/arm64 | Pods fail on wrong node | Multi-arch images, nodeSelector |
| One 1 GbE link (and slow powerline NAS writes) | Shared bandwidth | Keep NFS light; monitoring data on local disks first |
