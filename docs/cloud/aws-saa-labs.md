# AWS SAA-C03 Side Labs

These five labs turn the SAA-C03 exam guide into things you build. They are separate from the home lab phases, so you can do one a week while you work through Phases 2 to 9. The exam's four domains are design secure architectures, design resilient architectures, design high-performing architectures, and design cost-optimized architectures.

Optional, run alongside any phase. Read the [cost guardrails](cost-guardrails.md) first.

**Every lab uses the same four passes:**

1. **Console first.** Click through it slowly and read every option. *Why:* the exam's answer choices are often console settings, and the console shows you what defaults exist.
2. **Terraform second.** Rebuild the same thing in `terraform/aws/<lab-name>/` with the `aws` provider. *Why:* writing it as code forces you to name every piece you clicked past, and it gives you portfolio code.
3. **Check and break it.** Run the checks, then break one thing on purpose and fix it. *Why:* troubleshooting questions are a large part of the exam.
4. **Tear it down and recap.** `terraform destroy`, confirm nothing is left, and answer the recap questions out loud or in your notes.

| Lab | Main exam domains | Time | Cost risk |
|---|---|---|---|
| A. VPC from scratch | Secure, Resilient | 1 evening | Low; NAT Gateway is the one to watch |
| B. EC2, ALB and Auto Scaling across two AZs | Resilient, High-performing | 1 evening | Medium; ALB and instances bill hourly |
| C. S3 lifecycle and encryption, RDS read replica | Secure, Cost-optimized | 1 to 2 evenings | Medium; two DB instances and snapshots |
| D. API Gateway, Lambda and DynamoDB | High-performing, Cost-optimized | 1 evening | Very low |
| E. IAM, KMS and CloudTrail | Secure | 1 evening | Low; KMS key per month |

## Lab A: VPC from scratch

**Build it (console, then Terraform):**

1. A VPC with a /16 range (in the repo, write `<VPC_CIDR>`), with DNS hostnames turned on.
2. Four subnets across two Availability Zones: one public and one private in each.
3. An internet gateway, a public route table with `0.0.0.0/0` to it, and a private route table with no internet route yet.
4. A **gateway endpoint for S3** on the private route table. *Why:* private instances can reach S3 with no NAT and no internet, and gateway endpoints are free.
5. One small EC2 instance in a private subnet with an instance role for **Systems Manager Session Manager**, so you can connect without SSH or a public IP.
6. Turn on **VPC Flow Logs** to CloudWatch Logs for a short while.
7. Last and briefly: a NAT Gateway in one public subnet, and a route to it from the private route table. Delete it once you have tested it.

**Check it:**

- [ ] Session Manager opens a shell on the private instance, and the instance has no public IP.
- [ ] From that shell, `aws s3 ls` works before the NAT exists, and `curl -I https://example.com` times out.
- [ ] After adding the NAT route, `curl -I https://example.com` returns headers.
- [ ] Flow Logs show REJECT records when you try a blocked port.

**Break it:** remove the S3 endpoint from the private route table and run `aws s3 ls` again. Explain what changed, then fix it.

**Recap questions:**

1. Security groups are stateful and network ACLs are stateless. What does that mean for return traffic?
2. Why put the NAT Gateway in a public subnet, and what happens to private subnets in AZ b if AZ a fails while there is only one NAT?
3. Gateway endpoint versus interface endpoint: which services use which, and which one costs money?

## Lab B: EC2 with an Application Load Balancer and Auto Scaling across two AZs

**Build it:**

1. A **launch template** for a small free-tier instance with user data that serves a page showing the instance ID and Availability Zone (Amazon Linux ships with Python, so `python3 -m http.server` works without internet access). Require **IMDSv2**.
2. A **target group** with a health check on `/`.
3. An **Application Load Balancer** in the two public subnets from Lab A, whose security group allows HTTP from your IP only.
4. An **Auto Scaling group** in the two private subnets: minimum 2, desired 2, maximum 4, with a target tracking policy at 50% CPU. The instances' security group allows HTTP only from the ALB's security group.

**Check it:**

- [ ] Running `curl http://<ALB_DNS_NAME>/` ten times shows answers from both AZs.
- [ ] Both targets show **healthy** in the target group.
- [ ] Terminate one instance: the group launches a replacement, and the ALB keeps answering the whole time.
- [ ] Load one instance's CPU (for example `yes > /dev/null &` twice in Session Manager): a scale-out activity appears in the group's Activity tab within a few minutes.

**Break it:** change the health check path to `/missing`. Watch the targets go unhealthy and the group start replacing them. Explain why, then fix it.

**Recap questions:**

1. When would you choose a Network Load Balancer over an Application Load Balancer?
2. What is the health check grace period for, and what happens if it is too short?
3. Compare target tracking, step and scheduled scaling. When would you use each?
4. Why reference the ALB's security group instead of an IP range in the instances' rule?

## Lab C: S3 lifecycle and encryption, plus an RDS read replica

**Build it, S3 part:**

1. A bucket with **Block Public Access** on and **versioning** on.
2. Default encryption with **SSE-KMS** (a customer managed key from Lab E, or the AWS managed key for now).
3. A bucket policy that denies any request where `aws:SecureTransport` is false. *Why:* that forces HTTPS.
4. A lifecycle rule: move objects to Standard-IA after 30 days, to Glacier Flexible Retrieval after 90 days, and expire noncurrent versions after 30 days.

**Build it, RDS part:**

1. A DB subnet group on the two private subnets from Lab A.
2. A small free-tier MySQL or PostgreSQL instance, Single-AZ, not publicly accessible, with automated backups **on**. *Why:* read replicas need automated backups.
3. A **read replica** of it.

**Check it:**

- [ ] `aws s3api head-object --bucket <BUCKET> --key <KEY>` shows `ServerSideEncryption: aws:kms`.
- [ ] Uploading the same key twice leaves two versions in `aws s3api list-object-versions`.
- [ ] An upload forced over plain HTTP is denied: `aws s3 cp <FILE> s3://<BUCKET>/ --endpoint-url http://s3.<REGION>.amazonaws.com`.
- [ ] From the Lab A private instance: a row written on the primary shows up on the replica, and an `INSERT` on the replica fails because it is read-only.

**Teardown order:** delete the replica, then the primary (decide whether to keep a final snapshot, and delete it later if you do), then the DB subnet group. Empty the bucket, including all versions, before deleting it.

**Recap questions:**

1. Read replica versus Multi-AZ: which one is for scaling reads and which is for failover? Which is synchronous?
2. SSE-S3, SSE-KMS and SSE-C: who manages the key in each, and which one gives you a CloudTrail record of key use?
3. Which S3 storage class fits data you rarely read but need back within milliseconds?

## Lab D: Serverless API with API Gateway, Lambda and DynamoDB

**Build it:**

1. A DynamoDB table `homelab-notes` with partition key `id`, in **on-demand** capacity mode.
2. A Python Lambda function with an execution role allowing only `dynamodb:PutItem` and `dynamodb:GetItem` on that table's ARN, plus CloudWatch Logs.
3. An **HTTP API** in API Gateway with `POST /notes` and `GET /notes/{id}` routed to the function.

**Check it:**

- [ ] `curl -X POST <API_URL>/notes -d '{"text":"hello"}'` returns an ID, and `curl <API_URL>/notes/<ID>` returns the note.
- [ ] The function's CloudWatch log group shows one log stream per cold start.
- [ ] The item is visible in the DynamoDB console's Explore items page.

**Break it:** remove `dynamodb:PutItem` from the role. Call `POST` again, find the `AccessDeniedException` in the logs, and fix the policy.

**Recap questions:**

1. HTTP API versus REST API in API Gateway: what do you give up for the lower price?
2. What is a Lambda cold start, and what are two ways to reduce its effect?
3. On-demand versus provisioned capacity in DynamoDB: which fits a spiky, unpredictable workload?
4. Why is a good partition key one with many distinct values?

## Lab E: IAM, KMS and CloudTrail

**Build it:**

1. **CloudTrail:** create a trail for management events in all regions, delivered to a dedicated S3 bucket with log file validation turned on.
2. **IAM:** create a role `homelab-reader` with the `ReadOnlyAccess` managed policy and a trust policy that only your admin identity can assume. Assume it with `aws sts assume-role` and try a write.
3. **Explicit deny:** add an inline policy to the role that denies `s3:GetObject` on one bucket. *Why:* an explicit deny beats any allow; this is on the exam.
4. **KMS:** create a customer managed key with a key policy that lets your admin identity administer it and `homelab-reader` use it only to decrypt. Encrypt a file with `aws kms encrypt` and decrypt it as the role. Never remove your own admin access from a key policy, or you can lock yourself out of the key.
5. Turn on **IAM Access Analyzer** for the account and read its findings.

**Check it:**

- [ ] As `homelab-reader`, `aws s3 ls` works and `aws s3 mb s3://<TEST_BUCKET>` fails with AccessDenied.
- [ ] As `homelab-reader`, `aws s3 cp s3://<DENIED_BUCKET>/<KEY> .` fails even though ReadOnlyAccess allows reads.
- [ ] `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=Decrypt` shows your decrypt call and who made it.
- [ ] `aws cloudtrail validate-logs --trail-arn <TRAIL_ARN> --start-time <START_TIME>` reports the log files as valid.

**Teardown:** delete the trail and its bucket, delete the role, and schedule deletion of the KMS key (the minimum waiting period is 7 days).

**Recap questions:**

1. Identity-based versus resource-based policies: give one example of each from this lab.
2. Walk through how AWS evaluates a request that has an allow in one policy and a deny in another.
3. Why does every KMS key need a key policy, even when IAM policies grant access?
4. What is the difference between CloudTrail management events and data events?

**After all five labs:** take one full-length practice exam. For every question you miss, write which lab it maps to and repeat that lab's break-it step.
