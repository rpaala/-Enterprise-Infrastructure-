# Azure Add-ons

Azure is the lab's primary cloud, aimed at AZ-104 (Azure Administrator) first and AZ-500 (Azure Security) later. Each add-on below sits under a phase in the [phase plan](../phase-plan.md) and is done after that phase's own exit test. Read the [cost guardrails](cost-guardrails.md) before your first step.

Every add-on follows the same loop: you do the steps, run the checks, answer the recap questions, and tear down what you don't need.

| Phase | Add-on | Exam |
|---|---|---|
| 2 Git + Terraform | Entra ID, RBAC, and a VNet and VM in Terraform | AZ-104 |
| 3 Ansible | Azure Arc for the lab VMs, Azure Policy | AZ-104 |
| 5 CI/CD | GitHub Actions to Azure with OIDC | AZ-104, AZ-500 |
| 6 Security | Azure Monitor and Microsoft Sentinel | AZ-500 |
| 8 RAG assistant | Azure OpenAI as one model choice (optional) | None |

## Phase 2 (AZ-104): identity first, then a VNet and VM in Terraform

**Why this fits here:** Phase 2 is where you learn "infrastructure as code." Doing the same thing against Azure proves the skill is not tied to Proxmox, and it covers two AZ-104 domains: identities and governance, and virtual networking.

**Your steps:**

1. **Create the account safely.** Sign up for an Azure free account, turn on MFA for your sign-in, and create the $10 budget from the cost guardrails section. *Why:* the budget is your smoke alarm; set it before you build anything.
2. **Install the Azure CLI and log in.** Run `az login`, then `az account show -o table`. *Why:* Terraform can borrow this login for local runs, so no secret is ever written to disk.
3. **Entra ID basics in the portal.** Create one test user (`<LAB_TEST_USER>`) and one group (`lab-readers`), and add the user to the group. *Why:* Entra ID is Azure's identity service. Groups, not individual users, should hold permissions.
4. **Azure RBAC at the right scope.** Create a resource group `rg-homelab-dev` with `az group create`, then give `lab-readers` the **Reader** role on that resource group only. *Why:* RBAC scope (management group, subscription, resource group, resource) is a core AZ-104 topic. Least privilege means the smallest scope that works.
5. **Write the Terraform.** In `terraform/azure/`, use the `azurerm` provider (it needs an empty `features {}` block) and describe: a VNet with one subnet (use a placeholder such as `<AZURE_VNET_CIDR>` in the repo), a network security group that allows SSH only from `<MY_PUBLIC_IP>/32`, a public IP, a NIC, and one small Linux VM (a free-tier size such as B1s) that uses your SSH public key. Tag every resource `project=homelab` and `ttl=<TODAY>`.
6. **Run the loop.** `terraform init`, `terraform plan` (read it out loud: what will be created and why), `terraform apply`, SSH in, then `terraform destroy`. Repeat until it is boring.

**Check it (paste the results into your Phase 2 write-up):**

- [ ] `az vm list -d -o table` shows the VM running with a public IP after `apply`.
- [ ] `ssh <AZURE_ADMIN_USER>@<AZURE_VM_PUBLIC_IP>` works from your home connection, and fails from your phone on mobile data (the NSG only allows your IP).
- [ ] `az role assignment list --assignee <LAB_TEST_USER> --all -o table` shows Reader scoped to `rg-homelab-dev` only.
- [ ] Signed in as the test user, creating a storage account in the resource group fails with `AuthorizationFailed`.
- [ ] After `terraform destroy`, `az resource list -g rg-homelab-dev -o table` is empty and Cost Management shows close to $0.

**Try it yourself:** change the VM size in your code and run `terraform plan`. Before reading the output, predict: update in place, or destroy and recreate? Then check.

**Recap questions:**

1. What is the difference between an Entra ID role (like User Administrator) and an Azure RBAC role (like Reader)?
2. Why assign Reader at the resource group scope instead of the subscription?
3. An NSG can attach to a subnet or a NIC. If both have rules, which ones apply to inbound traffic, and in what order?
4. What does `terraform plan` compare your code against?

## Phase 3 (AZ-104): manage the home lab VMs with Azure Arc

**Why this fits here:** Phase 3 is about configuring machines the same way every time. Azure Arc lets Azure see and govern your Proxmox VMs as if they lived in Azure. Hybrid management is one of the most common things Azure job posts ask about.

**Your steps:**

1. **Register the resource providers.** Run `az provider register --namespace Microsoft.HybridCompute`, then the same for `Microsoft.GuestConfiguration` and `Microsoft.HybridConnectivity`. *Why:* a subscription only uses a service after its provider is registered. This is a common "why won't this deploy?" fix.
2. **Onboard one VM by hand.** In the portal, go to Azure Arc, Machines, Add, and generate the Linux script for a single server. Read the script before you run it on one lab VM (for example `ops`). *Why:* you should know what any script you run as root does.
3. **Automate it with Ansible.** Create a service principal with only the **Azure Connected Machine Onboarding** role on `rg-homelab-dev`. Store its secret with Ansible Vault, and write an `arc_onboard` role that installs the agent and runs `azcmagent connect`. Onboard the k3s VMs with it. *Why:* this joins Phase 3 (Ansible) and Azure. The onboarding identity can do nothing except onboard.
4. **Apply one policy.** Assign a built-in Azure Policy in **audit** mode to the resource group, such as one that requires a `project` tag. Then look at the compliance results. *Why:* Azure Policy is how organizations enforce standards. Audit mode reports without blocking anything.

**Check it:**

- [ ] `sudo azcmagent show` on each VM shows `Agent Status: Connected`.
- [ ] `az connectedmachine list -g rg-homelab-dev -o table` lists every onboarded VM.
- [ ] Running the `arc_onboard` playbook a second time reports `changed=0`.
- [ ] The policy's compliance page shows at least one compliant and one non-compliant machine. Add the missing tag and watch the non-compliant one flip (this can take up to 30 minutes).
- [ ] The onboarding secret appears only in the Ansible Vault file, and gitleaks passes.

**Cost note:** Arc's inventory and Azure Policy on Arc servers are free. Paid add-ons (Defender for Servers, Update Manager for Arc machines) are not needed for this lab. Leave them off.

**Recap questions:**

1. What kind of identity does the Arc agent use to talk to Azure after it is onboarded?
2. Why give the onboarding service principal a dedicated role instead of Contributor?
3. What is the difference between Azure Policy and Azure RBAC?

## Phase 5 (AZ-104, AZ-500): GitHub Actions deploys to Azure with OIDC, no stored secret

**Why this fits here:** pipelines that hold long-lived cloud keys are a top cause of breaches. OpenID Connect (OIDC) lets GitHub ask Azure for a short-lived token for each run, so there is no password to leak. Interviewers love this one.

**Your steps:**

1. **Bootstrap remote state.** With the CLI, create a long-lived resource group `rg-homelab-ci` and a storage account with a `tfstate` container. *Why:* each pipeline run starts on a clean machine, so Terraform state has to live somewhere shared and locked.
2. **Create the pipeline identity.** In Entra ID, add an app registration `gh-actions-homelab`. Under Certificates and secrets, add a **federated credential** for GitHub Actions with entity type **Environment**, environment `azure-lab`. *Why:* the subject `repo:<GITHUB_USER>/<REPO>:environment:azure-lab` means only jobs in that environment of that repo can get a token.
3. **Give it the least it needs.** Contributor on `rg-homelab-ci` only, plus **Storage Blob Data Contributor** on the state storage account.
4. **Set up GitHub.** Create the `azure-lab` environment with yourself as a required reviewer. Add `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` and `AZURE_SUBSCRIPTION_ID` as variables. They are identifiers, not secrets.
5. **Write the workflow.** Give it `permissions: id-token: write, contents: read`. Use `azure/login@v2` with those three values, and set `ARM_USE_OIDC=true` for Terraform. Run `terraform plan` on pull requests and `terraform apply` on `main` behind the environment approval. Start with something tiny, such as a storage account that hosts a static status page for the lab.

**Check it:**

- [ ] Repo Settings, Secrets shows **no** Azure client secret anywhere.
- [ ] A run on `main` logs "Login successful", pauses for your approval, then applies.
- [ ] A pull request run shows a plan and changes nothing.
- [ ] Break it on purpose: run the job without `environment: azure-lab`. Login fails with an AADSTS error about no matching federated identity. Write down the error code, then put it back.
- [ ] `az role assignment list --assignee <AZURE_CLIENT_ID> --all -o table` shows exactly the two scoped roles.

**Recap questions:**

1. In your own words, why is OIDC safer than storing a client secret in GitHub?
2. What does the federated credential's subject claim pin down, and what would happen if you used a wildcard branch instead?
3. Why does Terraform in CI need remote state, and what stops two runs from writing it at once?

## Phase 6 (AZ-500): send lab logs to Azure Monitor and catch an attack with Microsoft Sentinel

**Why this fits here:** Phase 6 adds security tools inside the cluster. This add-on adds a cloud SIEM (a tool that collects logs and raises incidents) on top, using the Arc VMs from Phase 3. Prometheus and Grafana stay your main monitoring in Phase 7; Sentinel is the security view.

**Your steps:**

1. **Create a Log Analytics workspace** in `rg-homelab-dev`, and set a **daily cap** (for example 0.1 GB). *Why:* log ingestion is billed per GB. The cap is your guardrail.
2. **Collect the right logs.** Create a data collection rule that sends Syslog `auth` and `authpriv` from your Arc VMs to the workspace. This installs the Azure Monitor Agent on them. *Why:* failed logins live in the auth logs, and a data collection rule decides exactly what is sent, so you do not pay for noise.
3. **Turn on Microsoft Sentinel** for that workspace. New workspaces get a free trial; check the current trial terms on the pricing page before you enable it.
4. **Write a detection.** Create a scheduled analytics rule in KQL that raises an incident when one source has more than 5 failed SSH logins in 5 minutes.
5. **Attack yourself.** From another lab VM, fail SSH to an Arc VM six times. Then work the incident in Sentinel: assign it to yourself, add a comment, and close it with a classification.
6. **Read Defender for Cloud's free recommendations** for the resource group, fix one (for example an NSG with SSH open to the internet), and watch the secure score change.

**Check it:**

- [ ] In the workspace, this KQL returns rows: `Syslog | where Facility in ("auth","authpriv") | where SyslogMessage has "Failed password" | summarize count() by HostName`
- [ ] Your attack creates a Sentinel incident within about 10 minutes.
- [ ] The incident is closed with a classification and a comment, and you saved a sanitized screenshot.
- [ ] The workspace's daily cap is set, and Cost Management shows under $1 for the week.

**Teardown:** when the trial ends, remove Sentinel from the workspace and delete the data collection rule association, or delete the workspace.

**Recap questions:**

1. What is the difference between a SIEM and a SOAR? Which part of Sentinel is which?
2. What does a data collection rule control, and why does that matter for cost?
3. Why does a scheduled rule look back over a time window instead of at single events?

## Phase 8 (optional): Azure OpenAI as one model choice for the assistant

**Why this fits here:** many companies that use Azure reach language models through Azure OpenAI, because it sits inside their existing identity, network and billing. Adding it as one option shows you can build an AI app that does not depend on a single provider.

**Your steps:**

1. **Create an Azure OpenAI resource** in `rg-homelab-dev` (through Azure AI Foundry), and deploy a small, low-cost chat model with a low tokens-per-minute quota. *Why:* a low quota caps both spend and runaway loops.
2. **Keep the key out of code.** Put the endpoint and key in a Kubernetes Secret, as you set up in Phase 6. Stretch goal: use Entra ID auth instead of a key.
3. **Make the model swappable.** Add a `MODEL_PROVIDER` setting to the `/ask` endpoint (for example `hosted`, `azure` or `ollama`) so the same app can call any of them.
4. **Compare honestly.** Run your 20-question eval against each provider. Record accuracy, average answer time and cost per 100 questions in a small table in your Phase 8 write-up.

**Check it:**

- [ ] `/ask` answers correctly with `MODEL_PROVIDER=azure`.
- [ ] The eval table has one row per provider.
- [ ] The resource's metrics show the token usage you expect, and the budget has not alerted.
- [ ] No key or endpoint value appears in the repo, and gitleaks passes.

**Teardown:** delete the model deployment when you are not demoing it.

**Recap questions:**

1. What does Azure OpenAI's content filtering do, and how would you notice it in your app?
2. Why build the app so the model provider can change without code edits?
