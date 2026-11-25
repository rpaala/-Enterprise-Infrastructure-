# Phase Plan

The roadmap for the lab, from what's already built through an AI assistant and MBSE-as-code. Each phase lists its goal, what it builds on, the main steps, effort, and the **interview evidence** it should produce.

Move to the next phase only when every box in its **Exit test** is ticked. Names in angle brackets, such as `<PROXMOX_HOST>`, are placeholders for your real values, which stay out of this repo.

Effort: **Small** = 1 to 2 evenings, **Medium** = 1 to 2 weekends, **Large** = 3 or more weekends.

**Cloud track.** Azure is the lab's primary cloud, aimed at AZ-104 first and AZ-500 later. It is woven into Phases 2, 3, 5, 6 and 8 as an **Azure add-on**, done after that phase's own exit test ([Azure add-ons](cloud/azure-addons.md)). AWS stays as five optional [SAA-C03 side labs](cloud/aws-saa-labs.md) you can run alongside any phase. Every cloud step follows the same loop: do it, check it, answer the recap questions, tear it down. Read the [cost guardrails](cloud/cost-guardrails.md) before your first cloud step.

## Where the lab is today

| Piece | Status | Notes |
|---|---|---|
| Proxmox host (iMac) | Built | VMs start at boot in a set order |
| Ubuntu VM template | Built | Base for every new VM |
| 3-node k3s cluster | Built | Ready to host workloads |
| NAS storage | Built | NFS shares for backups and k3s volumes; nightly Proxmox backups |
| NAS leftovers | Done | Restore test, SMART alerts, offsite copy, workstation backups |
| Router VLANs | In progress | Segmentation; Lab VLAN done, the rest (Management, Trusted, IoT, Guest) to do |
| WireGuard VPN | Not started | Secure remote admin |
| Terraform, Ansible, Docker, CI/CD, security tools, monitoring | Not started | Phases 2 to 7 |

Known limit: NAS writes are slow over powerline (about 2 MB/s), which matters for monitoring data and app volumes later.

## Phase 0: Role focus

**Goal:** Decide what kind of engineer the lab shows off. Split: Platform Engineering 70%, Systems Engineering 20%, DevSecOps 10%.

- Platform work (k3s, Terraform, Docker, CI/CD) gets the most depth.
- Systems Engineering shows through MBSE notes: requirements, architecture diagrams, interfaces and verification.
- DevSecOps shows up in Phases 5 and 6.

**Effort:** None. **Evidence:** A one-paragraph "why I built this" statement for the README.

**Exit test (pass before moving on):**

- [ ] The "why I built this" paragraph is written and saved in the README.
- [ ] The 70/20/10 split is written down where you will see it when choosing what to polish.

## Phase 1: Finish the foundation (VLANs, WireGuard, NAS)

**Goal:** Finish VLAN-segmented networking, NAS-backed storage and secure remote administration.

1. Plan the VLANs on paper first (Management, Lab, Trusted, IoT, Guest), each with a placeholder range such as `<MGMT_SUBNET>`.
2. Create the VLANs on the router, then move one device at a time. Keep a cable or console way back in.
3. Add firewall rules between VLANs. Start simple: Trusted reaches Lab and Management; IoT reaches only the internet.
4. Set up WireGuard with one peer for the laptop and one for the phone. Test that only what you intend is reachable from outside.
5. Finish the NAS leftovers: restore a template VM under a new ID to prove backups work, turn on SMART alerts, set up the offsite copy, add workstation backups.

**Effort:** Large (VLANs can lock you out; go slowly and log each change). **Evidence:** A sanitized network diagram, a WireGuard runbook, and a restore-test write-up ("restored a VM from backup in X minutes").

**Exit test (pass before moving on):**

- [ ] From a Trusted VLAN device, `ping <LAB_VM>` works. From an IoT device, `ping <PROXMOX_HOST>` fails.
- [ ] From the IoT VLAN, `nmap -sn <MGMT_SUBNET>` finds no hosts.
- [ ] On a phone with Wi-Fi off and WireGuard on, `https://<PROXMOX_HOST>:8006` loads. With WireGuard off, it does not.
- [ ] `wg show` lists a recent handshake for each peer.
- [ ] The restored template VM boots and you can log in. Record how long the restore took.
- [ ] A test SMART alert reaches your email or phone.
- [ ] The offsite bucket shows last night's backup with today's date.
- [ ] One file restored from each workstation backup (Time Machine and Windows).
- [ ] After a full Proxmox reboot, every VM comes back in order and `kubectl get nodes` shows 3 nodes Ready.

## Phase 2: Git repo and Terraform

**Goal:** Stop building VMs by hand. Describe them with Terraform and keep everything in Git from day one.

1. Set up the repo layout: `terraform/`, `ansible/`, `kubernetes/`, `helm/`, `docs/`, `diagrams/`, `runbooks/`, with a `.gitignore` that blocks secrets and state.
2. Move existing notes into `docs/` and `runbooks/` (sanitized).
3. Install Terraform and connect it to Proxmox with a dedicated API token stored in a password manager, never in the repo.
4. Write Terraform for one test VM cloned from the template. `plan`, `apply`, `destroy` until it's boring.
5. Describe the k3s VMs in Terraform; import or recreate the existing nodes once confident.
6. Add Azure: a free account with a budget alert, then Terraform for a resource group, a VNet and one tiny VM; destroy it after.
7. Commit after each working step, with clear messages.

**Effort:** Medium. **Evidence:** The repo's commit history, a "one command builds a VM, one removes it" demo, and Azure code for cloud provisioning.

**Exit test (pass before moving on):**

- [ ] `git clone` of the repo works on a second machine, and `git log` shows your commits.
- [ ] `git ls-files | grep tfstate` prints nothing, and the gitleaks check passes (no passwords, tokens or real IPs).
- [ ] `terraform plan` on unchanged code reports "No changes".
- [ ] `terraform apply` creates the test VM, you can SSH into it, and `terraform destroy` removes it from Proxmox.
- [ ] In Azure, `apply` creates the resource, `destroy` removes it, the budget alert is set, and billing shows close to $0.

**Azure add-on (AZ-104):** Entra ID users, groups and RBAC scope, then a VNet and VM in Terraform. Steps, checks and recap questions: [Azure add-ons, Phase 2](cloud/azure-addons.md#phase-2-az-104-identity-first-then-a-vnet-and-vm-in-terraform).

## Phase 3: Ansible

**Goal:** Terraform creates machines; Ansible configures them, so a rebuilt VM comes back the same every time.

1. Write an inventory listing the k3s VMs by name with placeholder addresses.
2. A base playbook: updates, a standard user, key-only SSH, firewall, time sync.
3. A playbook that installs k3s (one server, agents). Test it on fresh Terraform VMs.
4. Add the Raspberry Pis.
5. Save playbooks under `ansible/`.

**Effort:** Medium. **Evidence:** "I can tear down and rebuild my whole cluster from Git in under an hour", timed.

**Exit test (pass before moving on):**

- [ ] `ansible all -m ping` returns "pong" for every host.
- [ ] Running the baseline playbook a second time reports `changed=0` (idempotent, safe to re-run).
- [ ] `ssh -o PubkeyAuthentication=no <USER>@<K3S_NODE_1>` is refused, because password login is off.
- [ ] Timed rebuild: destroy a test cluster, rebuild it with Terraform and Ansible, and `kubectl get nodes` shows 3 nodes Ready. Record the time.

**Azure add-on (AZ-104):** onboard the lab VMs to Azure Arc with an Ansible role and audit them with Azure Policy. Steps, checks and recap questions: [Azure add-ons, Phase 3](cloud/azure-addons.md#phase-3-az-104-manage-the-home-lab-vms-with-azure-arc).

## Phase 4: Docker and a small app

**Goal:** Build a small app, containerize it, and run it on k3s.

1. A small Python API (FastAPI or Flask) with `/health` and a lab-info endpoint.
2. A Dockerfile; test with Docker Compose.
3. Kubernetes manifests (Deployment, Service) under `kubernetes/`, deployed with `kubectl apply`.
4. Optional: a Helm chart under `helm/`.

**Effort:** Medium. **Evidence:** The running app, its Dockerfile and manifests, and a screenshot or short video.

**Exit test (pass before moving on):**

- [ ] After `docker compose up`, `curl http://localhost:<PORT>/health` returns OK.
- [ ] `kubectl get pods` shows the app Running with 0 restarts after 10 minutes.
- [ ] `curl http://<K3S_NODE_1>:<NODEPORT>/health` from the workstation returns OK.
- [ ] After `kubectl delete pod <POD>`, a new pod starts on its own and `/health` still answers.

## Phase 5: CI/CD with GitHub Actions

**Goal:** Every push tests, scans, builds and deploys automatically.

1. Add pytest tests to the API.
2. A workflow: test, security scan, build the image, push to GitHub Container Registry, deploy to k3s.
3. Deploy through a **self-hosted runner** inside the Lab VLAN, so the cluster is never exposed to the internet.
4. Store tokens as GitHub Secrets, never in files.

**Effort:** Medium. **Evidence:** A green pipeline badge in the README, and an explanation of why the runner sits inside the network.

**Exit test (pass before moving on):**

- [ ] Push a small change: every stage goes green and the new version is live on k3s with no manual steps (e.g. `/health` shows the new version number).
- [ ] Push a deliberately broken test: the pipeline stops at the test stage and nothing is deployed.
- [ ] The self-hosted runner shows as Idle in the repo settings, and the router has no port forwarded to the cluster.
- [ ] Tokens exist only in GitHub Secrets; the gitleaks check finds none in the repo.

**Azure add-on (AZ-104/AZ-500):** GitHub Actions deploys to Azure with OIDC, so no cloud secret is stored. Steps, checks and recap questions: [Azure add-ons, Phase 5](cloud/azure-addons.md#phase-5-az-104-az-500-github-actions-deploys-to-azure-with-oidc-no-stored-secret).

## Phase 6: Security tools

**Goal:** Image scanning, runtime alerts, TLS and secrets handling.

1. **Trivy** as the pipeline's scan stage, failing builds on critical vulnerabilities.
2. **cert-manager** for internal HTTPS certificates.
3. **Secrets:** Kubernetes Secrets first; Sealed Secrets or SOPS later.
4. **Falco** for runtime alerts (test by opening a shell in a container).
5. A short security decision note in `docs/decisions/` for each tool.

**Effort:** Medium to Large. **Evidence:** A build failed by a planted vulnerable image, a Falco alert screenshot, and the decision notes.

**Exit test (pass before moving on):**

- [ ] A build using an old, known-vulnerable base image fails at the Trivy stage.
- [ ] `kubectl get certificate` shows READY True, and `curl -v https://<APP_HOSTNAME>` shows your certificate.
- [ ] `kubectl exec -it <POD> -- sh` triggers a Falco alert within a minute.
- [ ] The app reads its secret from a Kubernetes Secret, and no secret value appears in any file in the repo.

**Azure add-on (AZ-500):** Arc VM auth logs into Azure Monitor and a Microsoft Sentinel detection you trigger yourself. Steps, checks and recap questions: [Azure add-ons, Phase 6](cloud/azure-addons.md#phase-6-az-500-send-lab-logs-to-azure-monitor-and-catch-an-attack-with-microsoft-sentinel).

## Phase 7: Monitoring

**Goal:** See what the lab is doing and get told when something breaks.

1. kube-prometheus-stack (Prometheus, Grafana, Alertmanager).
2. Loki for logs.
3. One dashboard for the cluster, one for the API.
4. Two or three real alerts (node down, disk nearly full, backup missing) to email or phone.
5. Keep monitoring data on local disks until NAS write speed is fixed.

**Effort:** Medium. **Evidence:** Dashboard screenshots and an incident note in `runbooks/` for an alert you received and fixed.

**Exit test (pass before moving on):**

- [ ] Grafana loads over WireGuard, and the cluster dashboard shows all 3 nodes.
- [ ] Prometheus's Targets page shows every target as UP.
- [ ] In Grafana Explore, Loki shows the API's log lines from the last 5 minutes.
- [ ] Shut down one k3s agent VM: the "node down" alert arrives within about 5 minutes, and clears after it starts again.
- [ ] After 7 days, monitoring data still fits within the VM disk space set aside.

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

**Exit test (pass before moving on):**

- [ ] The index script reports how many chunks it stored, and searching a known phrase returns the right note.
- [ ] At least 15 of the 20 test questions get a correct answer with the correct source link.
- [ ] A question about something not in the notes gets "I don't know", not a made-up answer.
- [ ] A scan of the indexed content finds no passwords, keys or real IPs.
- [ ] Edit a runbook and push: after the pipeline runs, the assistant's answer reflects the change.
- [ ] From outside the home network with WireGuard off, the assistant can't be reached.

**Azure add-on (optional):** Azure OpenAI as one swappable model, compared in your eval. Steps, checks and recap questions: [Azure add-ons, Phase 8](cloud/azure-addons.md#phase-8-optional-azure-openai-as-one-model-choice-for-the-assistant).

## Phase 9: Digital engineering (MBSE as code)

**Goal:** Model the lab's requirements and architecture in SysML v2 text, keep the model in Git, and have the pipeline trace every requirement to the code and evidence that satisfy it.

1. Requirements in SysML v2 under `model/` with IDs such as `REQ-NET-01` and verification methods.
2. Model the structure (host, VMs, nodes, NAS, router), interfaces (SSH, NFS, WireGuard) and `satisfy` links.
3. Use open-source SysML v2 tooling for syntax checks.
4. Tag implementing code with requirement IDs (`# satisfies: REQ-NET-01`).
5. A Python script builds a traceability matrix: requirement → model element → files → evidence.
6. A GitHub Actions job rebuilds the matrix, fails when a requirement loses its trace, and publishes `docs/traceability.md`.

**Effort:** Medium to Large; can start after Phase 2. **Evidence:** A pipeline-generated traceability matrix and a demo of a broken trace failing the build.

**Exit test (pass before moving on):**

- [ ] The pipeline parses the model with no syntax errors.
- [ ] `docs/traceability.md` lists every requirement, each with at least one file that satisfies it and one piece of evidence.
- [ ] On a test branch, remove one `# satisfies:` tag: the build fails and names that requirement. Put it back and the build passes.

## AWS SAA-C03 side labs (optional)

Five labs, each built in the console first, then in Terraform, then broken on purpose, then torn down: A. VPC from scratch · B. EC2, ALB and Auto Scaling across two AZs · C. S3 lifecycle and encryption plus an RDS read replica · D. API Gateway, Lambda and DynamoDB · E. IAM, KMS and CloudTrail. Full steps: [AWS SAA-C03 side labs](cloud/aws-saa-labs.md).

## All phases at a glance

| Phase | What it adds | Effort | Branch it fills |
|---|---|---|---|
| 0 Role focus | Platform 70 / Sys Eng 20 / DevSecOps 10 | None | All |
| 1 Foundation | VLANs, WireGuard, NAS leftovers | Large | Infra, Networking |
| 2 Git + Terraform | Repo, VMs as code, Azure | Medium | Platform Engineering |
| 3 Ansible | Automated VM and k3s setup | Medium | Platform Engineering |
| 4 Docker | Python API on k3s | Medium | Platform Engineering |
| 5 CI/CD | GitHub Actions pipeline | Medium | DevSecOps |
| 6 Security | Trivy, cert-manager, secrets, Falco | Medium to Large | Security, DevSecOps |
| 7 Monitoring | Prometheus, Grafana, Loki, Alertmanager | Medium | Security, DevSecOps |
| 8 RAG assistant | AI assistant over notes, models and runbooks | Large | AI and Automation |
| 9 Digital engineering | SysML v2 model, requirement traceability in CI | Medium to Large | Systems Engineering |

## Habits for every phase

At the end of each phase, run its exit test and record the results in the write-up's **Test / exit criteria** table, then add a write-up to [`docs/phases/`](phases/) using the [template](phases/_template.md):

- An **architecture decision**: what you chose and why (also in `docs/decisions/`).
- **Troubleshooting notes** and **lessons learned**.
- An updated **network or system diagram** in `diagrams/`.
- A **security decision** if the phase touched security.

MBSE mapping: requirements (secure remote admin, segmented network, centralized storage, Kubernetes hosting, VPN access) map to Phases 1, 1, 1, 4 and 1. **Verification** is the evidence each phase produces: the restore test, the timed rebuild, the green pipeline and the alerts.
