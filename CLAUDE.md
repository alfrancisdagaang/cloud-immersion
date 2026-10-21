# Stratpoint Cloud Basics — Activity 1

Personal learning project: build and deploy a CloudFormation stack in a shared
Stratpoint sandbox AWS account. Keep costs minimal.

## Environment
- AWS profile: `cloud-basics-sso` (SSO; the older `cloud-basics` profile is expired, do not use it)
- AWS account: `000179588365`
- AWS role: `AWSReservedSSO_CloudNewbies` (`CloudNewbies`)
- Region: `ap-southeast-1` (Singapore). Always pass `--profile cloud-basics-sso --region ap-southeast-1`.
- Project directory: `C:\dev\cloud-basics` (not a git repository)
- Shell: Windows PowerShell 5.1
- Credentials are temporary SSO session credentials and may expire
  (`ExpiredToken`). If they do, stop and ask the user to refresh them through
  the Stratpoint SSO portal / `aws sso login --profile cloud-basics-sso`.
  Never ask for, print, or store credential values.
- The account is shared: many resources belong to other people. Only touch
  resources tagged `ProjectCode=CloudBasics-Act1-CloudIntern` or named
  `Daga-ang-*-Act1`.

## Architecture requirements
- VPC `10.0.0.0/16`
- Public subnet `10.0.1.0/24`, private subnet `10.0.2.0/24`
- Internet gateway + public route table (0.0.0.0/0 -> IGW) associated with the
  public subnet. The private subnet gets its own route table with no internet
  route. **No NAT Gateway** (cost) unless the activity explicitly requires one.
- EC2 `t2.micro` in the public subnet with a public IP, reachable via SSH
- Security group: SSH (22) only, from the user's current public IP as `/32`.
  Pass the IP as a template parameter (`MyIp`); check it fresh before each
  deploy because it changes (last seen: `175.158.207.4`).
- New S3 bucket with a globally unique name (include the account ID)
- IAM role + instance profile attached to the EC2 instance, allowing:
  `s3:ListBucket` on the bucket, and `s3:PutObject`, `s3:GetObject`,
  `s3:DeleteObject` on the bucket's objects. Scope to this bucket only.
- Amazon Linux 2023 AMI resolved at deploy time via the SSM parameter type
  `/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64`.
  Do not hardcode an AMI ID.

## Required tags
Apply both to every resource that supports tags:
- `Name`: `Daga-ang-{ResourceType}-Act1` (e.g. `Daga-ang-VPC-Act1`)
- `ProjectCode`: `CloudBasics-Act1-CloudIntern`

The CloudFormation stack itself must also be created with the `ProjectCode`
tag (`--tags Key=ProjectCode,Value=CloudBasics-Act1-CloudIntern`), because the
sandbox policy requires it.

## Progress so far
- [x] Project folder confirmed empty
- [x] Identity verified on `cloud-basics-sso`: account `000179588365`, role `CloudNewbies`
- [x] Read-only checks done: region `ap-southeast-1`; AL2023 SSM AMI lookup works;
      CloudFormation list and S3 list work; EC2 create-key-pair dry run succeeded
- [x] EC2 key pair created: `Daga-ang-KeyPair-Act1`
      (ID `key-0586613b6fb91ac41`, RSA, tagged with both required tags)
- [x] Private key saved locally at `C:\Users\user\.ssh\Daga-ang-KeyPair-Act1.pem`,
      with ACL restricted to the user. Never print, copy, commit, or move it.
      AWS cannot show it again; if lost, the key pair must be recreated.
- [x] CloudFormation template drafted, reviewed, and written to
      `C:\dev\cloud-basics\cloud-basics-act1.yaml`
- [x] Template validated with `aws cloudformation validate-template` (no errors;
      requires `CAPABILITY_IAM`; `MyIp` has no default). cfn-lint not installed.
- [x] Change set `Daga-ang-ChangeSet-Act1` previewed (15 resources, all Add), then
      executed with explicit approval. Stack `Daga-ang-Stack-Act1` reached
      `CREATE_COMPLETE` with the `ProjectCode` stack tag. (Stack later deleted; see Teardown.)
- [x] SSH verified: `ssh ... ec2-user@<ip> whoami` returned `ec2-user`.
- [x] S3 verified from the instance using its IAM role (no hardcoded credentials):
      list, upload, download, content match, delete, and 404 on the deleted
      object all succeeded. The bucket was left empty.
- [x] Teardown completed: stack `Daga-ang-Stack-Act1` deleted with explicit approval;
      `describe-stacks` confirmed it no longer exists. The key pair
      `Daga-ang-KeyPair-Act1` and the local `.pem` were intentionally kept.

Resolved: the role can create the CloudFormation stack, the IAM role and the
instance profile (`CAPABILITY_IAM`). The permissions required for this deployment
were confirmed by successful stack creation.

## Deployed resources (ap-southeast-1) — HISTORICAL, ALL DELETED
Status: the stack was deployed, tested (SSH and S3 both passed), and then deleted.
The IDs below are kept as a record only. These resources no longer exist, and the
public IP and bucket name are no longer in use. Redeploying means a new change set,
new IDs, and a fresh `MyIp` check.
- Stack: `Daga-ang-Stack-Act1` (was `CREATE_COMPLETE`; now deleted), tag `ProjectCode=CloudBasics-Act1-CloudIntern`
- Template: `cloud-basics-act1.yaml`; change set `Daga-ang-ChangeSet-Act1` (already executed)
- Parameters used: `MyIp=175.158.207.4`, `KeyName=Daga-ang-KeyPair-Act1`,
  `LatestAmiId` = SSM AL2023 path (resolved to `ami-08c9f7e48c17bba07` at deploy)
- VPC: `vpc-07319047c0e80f145`
- EC2 instance: `i-039960ad6dbe030b8` (t2.micro), public IP `18.136.211.118`
  (auto-assigned, not an Elastic IP; it changes if the instance is stopped/started)
- S3 bucket: `daga-ang-bucket-act1-000179588365-ap-southeast-1`
- Instance role: `Daga-ang-Stack-Act1-InstanceRole-u86TjeTKzRSB`
- SSH: `ssh -i C:\Users\user\.ssh\Daga-ang-KeyPair-Act1.pem ec2-user@18.136.211.118`
- If the user's public IP changes, SSH stops working. Fix with a stack update
  (new change set) passing the new `MyIp`; needs explicit approval.
- Other resource IDs (subnets, security group, etc.): read them with
  `aws cloudformation describe-stack-resources --stack-name Daga-ang-Stack-Act1`.
- Cost while running: about $0.50/day (t2.micro + public IPv4 + 8 GB gp3).

## Teardown (COMPLETED; procedure kept for reference, only run with explicit approval)
1. Confirm the bucket is empty:
   `aws s3api list-objects-v2 --bucket daga-ang-bucket-act1-000179588365-ap-southeast-1`
   If objects remain, deleting the stack fails on the bucket. Empty it first
   (with approval).
2. Delete the stack:
   `aws cloudformation delete-stack --stack-name Daga-ang-Stack-Act1 --profile cloud-basics-sso --region ap-southeast-1`
   then `aws cloudformation wait stack-delete-complete` with the same arguments.
3. The key pair `Daga-ang-KeyPair-Act1` (`key-0586613b6fb91ac41`) and the local
   `C:\Users\user\.ssh\Daga-ang-KeyPair-Act1.pem` are NOT part of the stack. Delete
   them only if the user asks (`aws ec2 delete-key-pair --key-name Daga-ang-KeyPair-Act1`,
   then remove the .pem). Never print the key contents.
4. The stack deletion also removes the VPC, instance, bucket and IAM role.
   Verify afterward that `describe-stacks` reports the stack does not exist.

**Teardown result:** `delete-stack` and `wait stack-delete-complete` both exited 0,
and `describe-stacks` returned "Stack with id Daga-ang-Stack-Act1 does not exist".

**Retained, NOT part of the stack:**
- EC2 key pair `Daga-ang-KeyPair-Act1` (`key-0586613b6fb91ac41`) still exists in AWS
  (free, but visible in the shared account).
- Local private key `C:\Users\user\.ssh\Daga-ang-KeyPair-Act1.pem` still exists.
  Never print, copy, or move it.
- Delete them only if the user explicitly asks.

## Workflow rules (follow exactly)
1. Work in small, reviewable stages. Explain the purpose of each major step
   before doing it.
2. Show proposed file contents in chat before writing any file. Write only
   after the user approves.
3. Never overwrite existing files without asking.
4. Validate the template before any deployment (`validate-template`, and
   cfn-lint if available). Show a change set or summary before deploying.
5. Never create, modify, or delete AWS resources without the user's explicit
   approval for that specific step. Read-only checks are fine.
6. Do not repeat completed steps.
7. Keep credentials and private-key contents secret. Never print them, paste
   them into chat, or write them into project files.
8. Avoid unnecessary cost: no NAT Gateway, no extra instances, stay on
   `t2.micro`, and remind the user to tear down when finished.

## Planned files
- `cloud-basics-act1.yaml`: the CloudFormation template
- `CLAUDE.md`: this file; update the progress checklist after each stage
- `act1-guide.md`, `final-act1-guide.md`: walkthrough and defense guides for each activity (written)

---

# FINAL ACTIVITY 1 — Bastion + Private EC2 (current work)

Activity 1 above is finished and kept for reference. Everything below applies
to the current task. Where they conflict, this section wins.

## Objective
CloudFormation stack with a bastion/jump host (EC2 in public subnet) and an EC2 in
a private subnet. SSH from local machine -> bastion -> private instance.
GitLab upload is OMITTED (supervisor confirmed). Keep code in this repo only; do not
change remotes without asking.

## Environment changes (now on macOS)
- Shell: zsh on macOS. Project dir: `/Users/alfrancis.dagaang/Developer/cloud-immersion`
- AWS profile `cloud-basics-sso` re-created on this Mac; same account and role.
- Region: `ap-southeast-1`. Always pass `--profile cloud-basics-sso --region ap-southeast-1`.
- cfn-lint: optional (not installed yet).

## Requirements
- VPC `10.0.0.0/16`; public subnet `10.0.1.0/24`; private subnet `10.0.2.0/24`
  (activity text says /16 for subnets, which is impossible; /24 assumed, confirm w/ mentor)
- IGW + public route table; private route table with no internet route; no NAT Gateway
- Bastion `t2.micro` in public subnet with public IP
- Private `t2.micro` in private subnet, no public IP
- Bastion SG: SSH 22 from my public IP `/32` only (`MyIp` parameter, check fresh before deploy)
- Private SG: SSH 22 only from the Bastion SG (source security group, not a CIDR)
- AL2023 AMI via SSM parameter; no hardcoded AMI ID
- Reach private host via agent forwarding (`ssh -A`) or ProxyJump (`-J`); never copy
  the .pem onto the bastion

## Tags
- Stack tag (required by sandbox policy): `ProjectCode=FinalAct1-CloudIntern`
- Every taggable resource: `Name=Daga-ang-{ResourceType}-FinalAct1`, `ProjectCode=FinalAct1-CloudIntern`
- Only touch resources with that ProjectCode or named `Daga-ang-*-FinalAct1`.

## Names
- Stack: `Daga-ang-Stack-FinalAct1`; change set: `Daga-ang-ChangeSet-FinalAct1`
- Template: `final-act1.yaml` (NEW file; never overwrite `cloud-basics-act1.yaml`)
- Key pair: `Daga-ang-KeyPair-FinalAct1` (`key-0a85406b463df87aa`)
- Private key: `~/.ssh/Daga-ang-KeyPair-FinalAct1.pem` (chmod 400). Never print, copy, commit, or move it.

## Deliverables
Code in this repo (GitLab omitted per supervisor); screenshots of AWS resources (taken by user); delete stack afterwards.

## Final Act progress
- [x] AWS CLI installed; SSO profile recreated; identity verified (CloudNewbies, 000179588365)
- [x] Key pair `Daga-ang-KeyPair-FinalAct1` created, private key saved locally
- [x] GitLab upload omitted (supervisor said it is not required)
- [x] `final-act1.yaml` drafted, reviewed, written
- [x] Template validated with validate-template (no errors, no IAM capability needed, MyIp has no default); cfn-lint not run
- [x] Change set previewed (14 resources, all Add) and executed with approval; stack CREATE_COMPLETE. VPC vpc-0c41b933730c5d805, bastion 18.141.225.230, private 10.0.2.240 (IPs change if stopped/started)
- [x] SSH tests passed: local -> bastion (10.0.1.160) -> private (10.0.2.240)
- [x] Screenshots taken by user (VPC resource map, SSH hop, console)
- [x] Stack deleted with approval; describe-stacks reports it does not exist; no VPC or instance tagged FinalAct1 remains. Retained: key pair Daga-ang-KeyPair-FinalAct1 and local .pem (not part of stack).

## After both activities: walkthrough and defense materials (done)
Both activities are complete and their stacks are deleted. Next came materials to
present and defend the work. Nothing below is deployed; all of it is local docs.
- `act1-guide.md`: pitch, concepts, live walkthrough script, likely Q&A and honest caveats for Activity 1
- `final-act1-guide.md`: same for Final Activity 1 (architecture sketch, SG referencing, bastion, ssh -A / -J)
- `deploy-commands.md`: copy-paste commands for both activities (validate, change set, execute,
  outputs, SSH/S3 tests, teardown). Uses `export AWS_PROFILE=cloud-basics-sso
  AWS_DEFAULT_REGION=ap-southeast-1` and a fresh `MYIP` each time.
- Activity 1 on this Mac: its original key pair's .pem is only on the old Windows
  machine, so the deploy commands pass `KeyName=Daga-ang-KeyPair-FinalAct1` instead.
- Redeploying for a live walkthrough is a new change set with a fresh `MyIp`, needs explicit
  approval per step, and the stack must be deleted right after (~$0.50-$1/day).

## Git state
- Remote `origin`: GitHub (`alfrancisdagaang/cloud-immersion`). GitLab was waived by the supervisor.
- `1358d57` (template + notes) was pushed by the user with `! git push origin main`.
- `c621a61` (demo docs) is committed locally and NOT pushed (user chose not to push yet).
- Claude's own `git push` was blocked by the auto-mode classifier even with chat approval;
  the user pushes with `!` or adds a `Bash(git push ...)` permission rule.
- The repo contains the AWS account ID and role name in this file; check GitHub
  visibility (private is preferable).

## Still retained in AWS / locally
- Key pairs `Daga-ang-KeyPair-Act1` and `Daga-ang-KeyPair-FinalAct1` (free, not in any stack)
- `~/.ssh/Daga-ang-KeyPair-FinalAct1.pem` (never print, copy, commit or move)
- Delete only if the user explicitly asks.
