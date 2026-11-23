# Phase Plan

The roadmap for the lab, from what's already built through an AI assistant and MBSE-as-code. Each phase lists its goal, what it builds on, the main steps, effort, and the **interview evidence** it should produce.

Effort: **Small** = 1 to 2 evenings, **Medium** = 1 to 2 weekends, **Large** = 3 or more weekends.

## Where the lab is today

| Piece | Status | Notes |
|---|---|---|
| Proxmox host (iMac) | Built | VMs start at boot in a set order |
| Ubuntu VM template | Built | Base for every new VM |
| 3-node k3s cluster | Built | Ready to host workloads |
| NAS storage | Mostly built | NFS shares for backups and k3s volumes; nightly Proxmox backups |
| NAS leftovers | Not started | Restore test, SMART alerts, offsite copy, workstation backups |
| Router VLANs | Not started | Segmentation |
| WireGuard VPN | Not started | Secure remote admin |
| Terraform, Ansible, Docker, CI/CD, security tools, monitoring | Not started | Phases 2 to 7 |

Known limit: NAS writes are slow over powerline (about 2 MB/s), which matters for monitoring data and app volumes later.

## Phase 0: Role focus

**Goal:** Decide what kind of engineer the lab shows off. Split: Platform Engineering 70%, Systems Engineering 20%, DevSecOps 10%.

- Platform work (k3s, Terraform, Docker, CI/CD) gets the most depth.
- Systems Engineering shows through MBSE notes: requirements, architecture diagrams, interfaces and verification.
- DevSecOps shows up in Phases 5 and 6.

**Effort:** None. **Evidence:** A one-paragraph "why I built this" statement for the README.

## Phase 1: Finish the foundation (VLANs, WireGuard, NAS)

**Goal:** Finish VLAN-segmented networking, NAS-backed storage and secure remote administration.

1. Plan the VLANs on paper first (Management, Lab, Trusted, IoT, Guest), each with a placeholder range such as `<MGMT_SUBNET>`.
2. Create the VLANs on the router, then move one device at a time. Keep a cable or console way back in.
3. Add firewall rules between VLANs. Start simple: Trusted reaches Lab and Management; IoT reaches only the internet.
4. Set up WireGuard with one peer for the laptop and one for the phone. Test that only what you intend is reachable from outside.
5. Finish the NAS leftovers: restore a template VM under a new ID to prove backups work, turn on SMART alerts, set up the offsite copy, add workstation backups.

**Effort:** Large (VLANs can lock you out; go slowly and log each change). **Evidence:** A sanitized network diagram, a WireGuard runbook, and a restore-test write-up ("restored a VM from backup in X minutes").

## Phase 2: Git repo and Terraform

**Goal:** Stop building VMs by hand. Describe them with Terraform and keep everything in Git from day one.

1. Set up the repo layout: `terraform/`, `ansible/`, `kubernetes/`, `helm/`, `docs/`, `diagrams/`, `runbooks/`, with a `.gitignore` that blocks secrets and state.
2. Move existing notes into `docs/` and `runbooks/` (sanitized).
3. Install Terraform and connect it to Proxmox with a dedicated API token stored in a password manager, never in the repo.
4. Write Terraform for one test VM cloned from the template. `plan`, `apply`, `destroy` until it's boring.
5. Describe the k3s VMs in Terraform; import or recreate the existing nodes once confident.
6. Add a small GCP project with a budget alert; create and destroy one tiny resource with Terraform.
7. Commit after each working step, with clear messages.

**Effort:** Medium. **Evidence:** The repo's commit history, a "one command builds a VM, one removes it" demo, and GCP code for cloud provisioning.

## Phase 3: Ansible

**Goal:** Terraform creates machines; Ansible configures them, so a rebuilt VM comes back the same every time.

1. Write an inventory listing the k3s VMs by name with placeholder addresses.
2. A base playbook: updates, a standard user, key-only SSH, firewall, time sync.
3. A playbook that installs k3s (one server, agents). Test it on fresh Terraform VMs.
4. Add the Raspberry Pis.
5. Save playbooks under `ansible/`.

**Effort:** Medium. **Evidence:** "I can tear down and rebuild my whole cluster from Git in under an hour", timed.

## Phase 4: Docker and a small app

**Goal:** Build a small app, containerize it, and run it on k3s.

1. A small Python API (FastAPI or Flask) with `/health` and a lab-info endpoint.
2. A Dockerfile; test with Docker Compose.
3. Kubernetes manifests (Deployment, Service) under `kubernetes/`, deployed with `kubectl apply`.
4. Optional: a Helm chart under `helm/`.

**Effort:** Medium. **Evidence:** The running app, its Dockerfile and manifests, and a screenshot or short video.

## Phase 5: CI/CD with GitHub Actions

**Goal:** Every push tests, scans, builds and deploys automatically.

1. Add pytest tests to the API.
2. A workflow: test, security scan, build the image, push to GitHub Container Registry, deploy to k3s.
3. Deploy through a **self-hosted runner** inside the Lab VLAN, so the cluster is never exposed to the internet.
4. Store tokens as GitHub Secrets, never in files.

**Effort:** Medium. **Evidence:** A green pipeline badge in the README, and an explanation of why the runner sits inside the network.

## Phase 6: Security tools

**Goal:** Image scanning, runtime alerts, TLS and secrets handling.

1. **Trivy** as the pipeline's scan stage, failing builds on critical vulnerabilities.
2. **cert-manager** for internal HTTPS certificates.
3. **Secrets:** Kubernetes Secrets first; Sealed Secrets or SOPS later.
4. **Falco** for runtime alerts (test by opening a shell in a container).
5. A short security decision note in `docs/decisions/` for each tool.

**Effort:** Medium to Large. **Evidence:** A build failed by a planted vulnerable image, a Falco alert screenshot, and the decision notes.

## Phase 7: Monitoring

**Goal:** See what the lab is doing and get told when something breaks.

1. kube-prometheus-stack (Prometheus, Grafana, Alertmanager).
2. Loki for logs.
3. One dashboard for the cluster, one for the API.
4. Two or three real alerts (node down, disk nearly full, backup missing) to email or phone.
5. Keep monitoring data on local disks until NAS write speed is fixed.

**Effort:** Medium. **Evidence:** Dashboard screenshots and an incident note in `runbooks/` for an alert you received and fixed.

## Phase 8: RAG engineering assistant

**Goal:** An AI assistant that answers questions about the lab ("how do I restore a VM?") from its own notes, ArchiMate models, runbooks and docs, with links to sources. RAG (retrieval-augmented generation) means it first finds the relevant pages, then a language model writes an answer from them.

Builds on: the repo (Phase 2) as the knowledge source, the documentation habit from every phase, Docker and k3s (Phase 4), CI/CD (Phase 5) for re-indexing, security (Phase 6) for keys and HTTPS, WireGuard (Phase 1) for access, and monitoring (Phase 7).

1. **Gather sources:** sync notes into a repo and export ArchiMate models (Open Exchange XML or CSV).
2. **Clean before indexing:** no passwords, keys or real IPs. Anything indexed can appear in an answer.
3. **Split and embed:** a Python script chunks the notes, embeds them, and stores vectors in Qdrant or Postgres with pgvector on k3s.
4. **Choose the model:** a hosted model API to start (no GPU); a small local model through Ollama later.
5. **Build the app:** an `/ask` endpoint on the Phase 4 API that retrieves chunks, asks the model, and returns the answer with sources.
6. **Automate re-indexing** with GitHub Actions when `docs/`, `runbooks/` or `diagrams/` change.
7. **Test honestly:** 15 to 20 known-answer questions as a repeatable eval.

**Effort:** Large. **Evidence:** A live demo with cited sources and the eval score.

## Phase 9: Digital engineering (MBSE as code)

**Goal:** Model the lab's requirements and architecture in SysML v2 text, keep the model in Git, and have the pipeline trace every requirement to the code and evidence that satisfy it.

1. Requirements in SysML v2 under `model/` with IDs such as `REQ-NET-01` and verification methods.
2. Model the structure (host, VMs, nodes, NAS, router), interfaces (SSH, NFS, WireGuard) and `satisfy` links.
3. Use open-source SysML v2 tooling for syntax checks.
4. Tag implementing code with requirement IDs (`# satisfies: REQ-NET-01`).
5. A Python script builds a traceability matrix: requirement → model element → files → evidence.
6. A GitHub Actions job rebuilds the matrix, fails when a requirement loses its trace, and publishes `docs/traceability.md`.

**Effort:** Medium to Large; can start after Phase 2. **Evidence:** A pipeline-generated traceability matrix and a demo of a broken trace failing the build.

## All phases at a glance

| Phase | What it adds | Effort | Branch it fills |
|---|---|---|---|
| 0 Role focus | Platform 70 / Sys Eng 20 / DevSecOps 10 | None | All |
| 1 Foundation | VLANs, WireGuard, NAS leftovers | Large | Infra, Networking |
| 2 Git + Terraform | Repo, VMs as code, GCP | Medium | Platform Engineering |
| 3 Ansible | Automated VM and k3s setup | Medium | Platform Engineering |
| 4 Docker | Python API on k3s | Medium | Platform Engineering |
| 5 CI/CD | GitHub Actions pipeline | Medium | DevSecOps |
| 6 Security | Trivy, cert-manager, secrets, Falco | Medium to Large | Security, DevSecOps |
| 7 Monitoring | Prometheus, Grafana, Loki, Alertmanager | Medium | Security, DevSecOps |
| 8 RAG assistant | AI assistant over notes, models and runbooks | Large | AI and Automation |
| 9 Digital engineering | SysML v2 model, requirement traceability in CI | Medium to Large | Systems Engineering |

## Habits for every phase

At the end of each phase, add a write-up to [`docs/phases/`](phases/) using the [template](phases/_template.md):

- An **architecture decision**: what you chose and why (also in `docs/decisions/`).
- **Troubleshooting notes** and **lessons learned**.
- An updated **network or system diagram** in `diagrams/`.
- A **security decision** if the phase touched security.

MBSE mapping: requirements (secure remote admin, segmented network, centralized storage, Kubernetes hosting, VPN access) map to Phases 1, 1, 1, 4 and 1. **Verification** is the evidence each phase produces: the restore test, the timed rebuild, the green pipeline and the alerts.
