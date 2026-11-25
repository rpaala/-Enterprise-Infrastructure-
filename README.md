# Enterprise Infrastructure Home Lab

Step-by-step process of building a home lab using readily available hardware and open-source software. This project takes into factor what it's like to build a platform from the ground up, taking into factor the hardware, software, security, and general requirements. It is meant to be a scalable project, constantly evolving based on the needs of the stakeholder.

The lab is built like a small enterprise platform: a hypervisor, a Kubernetes cluster, segmented networking, backed-up storage, infrastructure as code, a CI/CD pipeline, security tooling, monitoring, and AI services that run on top of it all. Azure and AWS Cloud are also integrated in the project. Azure serves as the main cloud platform while side labs of AWS are featured, first created using console and then Terraform.

> **Status:** Proxmox host, Ubuntu VM template, 3-node k3s cluster and NAS-backed storage are built. Phase 1 (foundation) is in progress: the Lab VLAN and NAS work are done; the remaining VLANs and WireGuard are still to do. See the [phase plan](docs/phase-plan.md) for the full roadmap.

## Architecture at a glance

```text
                 Internet
                    │  (only the VPN port is open; nothing else forwarded)
            ┌───────┴────────┐
            │  Edge router   │  VLANs · zone firewall · DHCP · DNS · WireGuard
            └───────┬────────┘
   ┌────────────┬───┴────────┬──────────────┬─────────────┐
 Management    Lab          Trusted        IoT           Guest
   │            │             │              │
 Proxmox VE   k3s VMs, Pis,  Workstations   Smart devices,
 host         ops VM, NAS    (admin)        printer, VoIP
   │
   └─ VMs: k3s-cp-1 · k3s-wk-1 · k3s-wk-2 · ops
```

Archi Home Lab Architecture

![Home lab network architecture](diagrams/Network%20Architecture.png)

| Layer | Technology |
|---|---|
| Hypervisor | Proxmox VE on a repurposed 2011 iMac (headless) |
| Compute | k3s: 1 control-plane VM, 2 worker VMs, 1 Raspberry Pi 4 (arm64) |
| Watchdog | Raspberry Pi 3 outside the cluster: Uptime Kuma, backup DNS |
| Storage | NAS over NFS for cluster volumes and backups, encrypted offsite copy |
| Network | VLAN segmentation, zone-based firewall, WireGuard VPN, no port forwards |
| Automation | Terraform (bpg/proxmox), Ansible, Flux GitOps, GitHub Actions |
| Security | gitleaks, Trivy, cert-manager, Falco, SOPS/age, key-only SSH, 2FA |
| Observability | Prometheus, Grafana, Loki, Alertmanager |
| AI | Ollama and hosted model APIs, a RAG assistant over the lab's own docs |

Details: [Architecture overview](docs/architecture.md).

## Roadmap

| Phase | What it adds | Status |
|---|---|---|
| 0 | Role focus: Platform 70% / Systems Eng 20% / DevSecOps 10% | Done |
| 1 | Foundation: VLANs, WireGuard, NAS restore test and offsite backup | In progress |
| 2 | Git repo and Terraform (VMs as code, Azure VNet and VM) | Planned |
| 3 | Ansible configuration management | Planned |
| 4 | Docker and a small Python API on k3s | Planned |
| 5 | CI/CD with GitHub Actions and a self-hosted runner | Planned |
| 6 | Security tools: Trivy, cert-manager, secrets, Falco | Planned |
| 7 | Monitoring: Prometheus, Grafana, Loki, Alertmanager | Planned |
| 8 | RAG engineering assistant over the lab's docs and runbooks | Planned |
| 9 | Digital engineering: SysML v2 model with requirement traceability in CI | Planned |

## Skills demonstrated

- **Platform engineering:** Proxmox virtualization, Kubernetes (k3s) on mixed amd64/arm64 nodes, GitOps
- **Infrastructure as code:** Terraform, Ansible, unattended Proxmox reinstalls from an answer file
- **Networking and security:** VLAN design, zone firewalls, VPN-only remote access, IoT isolation, DNS filtering, secrets management, secret scanning in CI
- **Reliability:** 3-2-1 backups with tested restores, out-of-band monitoring, incident runbooks; 3-2-1 backup prevents single point of failure.
- **AI and automation:** local and hosted LLMs, an AI security digest, retrieval-augmented generation
- **Systems engineering:** requirements, architecture views, interfaces and verification evidence (MBSE) using Archi, an open source cross platform modeling toolkit to create ArchiMate models

## Repository layout

```text
docs/
  architecture.md      Sanitized architecture overview
  phase-plan.md        Phases 0 to 9 with goals, steps and interview evidence
  phases/              One write-up per phase (start from _template.md)
  cloud/               Azure add-ons, AWS SAA-C03 side labs, cloud cost guardrails
  build-log/           Step-by-step build logs
  decisions/           Architecture decision records (ADRs)
  redaction-policy.md  What never goes in this repo, and the placeholders to use
diagrams/              Network and system diagrams (no real addresses)
runbooks/              Operational procedures: restore, rebuild, incident response
terraform/ ansible/ kubernetes/ helm/   Added as each phase begins
```

## Privacy and security

This repo is public-facing, so it never contains real IP addresses, subnets, hostnames, MAC addresses, domains, serial numbers, usernames, keys or tokens. Every value like that is a placeholder, as described in the [redaction policy](docs/redaction-policy.md). A [gitleaks](https://github.com/gitleaks/gitleaks) scan runs on every push and pull request, with extra rules for private IP and MAC addresses.

---

© 2026 rpaala. All rights reserved. Shared for portfolio review.
