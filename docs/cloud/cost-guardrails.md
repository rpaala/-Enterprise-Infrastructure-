# Cloud Cost Guardrails

Every cloud lab here should cost close to $0. That only holds if you set alarms first and tear down every time. Budgets **alert**; they do not stop spending.

**Set up once per account:**

1. **Lock the front door.** Turn on MFA for the Azure sign-in and the AWS root user. Never create access keys for the AWS root user. Day to day, use IAM Identity Center (or an IAM user with MFA) and named CLI profiles. *Why:* a leaked key is the most expensive mistake in cloud.
2. **Azure budget.** In Cost Management, create a $10 monthly budget on the subscription with email alerts at 50%, 80% and 100%, plus a forecast alert.
3. **AWS budgets.** In Billing, Budgets, create the **zero spend** template budget and a $10 monthly cost budget with alerts at 50%, 80% and 100%. Turn on Free Tier usage alerts in Billing preferences.
4. **Know your free tier.** Both clouds have changed their free offers over time, so read the current terms for your account on the day you sign up and write them in your notes.

**Every lab session:**

- **Tag everything** `project=homelab` and `ttl=<TODAY>`, so leftovers are easy to find.
- **Build, check, destroy in the same sitting.** Prefer `terraform destroy` or deleting the whole Azure resource group.
- **Look for leftovers** before you log off: Azure resource group list, AWS Tag Editor (search by `project=homelab` in every region) and the Billing page.

**The usual surprise bills:**

| Resource | Why it costs | What to do |
|---|---|---|
| AWS NAT Gateway | Billed per hour plus per GB, even when idle | Create it last, delete it first, keep the lab under an hour |
| AWS Application Load Balancer | Billed per hour | Delete at the end of the session |
| Public IPv4 addresses (AWS) | Every public IPv4 is billed per hour, including unattached Elastic IPs | Release them; use Session Manager instead of public SSH |
| EBS volumes and snapshots, RDS snapshots | Survive after the instance is gone | Check the Volumes and Snapshots pages after teardown |
| AWS KMS customer managed keys | Monthly charge per key | Schedule deletion when the lab is done |
| Azure public IPs and managed disks | Can remain after a VM is deleted in the portal | Delete the resource group, not just the VM |
| Log Analytics and Sentinel | Billed per GB ingested | Daily cap on the workspace; collect only what you need |
