# Demo Guide — Activity 1 (VPC + EC2 + S3 + IAM role)

## 30-second pitch
"I used one CloudFormation template to build a VPC with a public and a private
subnet, a t2.micro EC2 instance I can SSH into only from my own IP, a private S3
bucket, and an IAM role that lets the instance reach only that bucket. I deployed
it with a change set, tested it, then deleted the stack."

## What I built
| Piece | Value | Why |
|---|---|---|
| VPC | 10.0.0.0/16 | My own isolated network |
| Public subnet | 10.0.1.0/24 | Route to the internet gateway |
| Private subnet | 10.0.2.0/24 | Own route table, no internet route |
| Internet gateway + public route table | 0.0.0.0/0 -> IGW | This route is what makes a subnet "public" |
| Security group | SSH 22 from my IP /32 | Least exposure |
| EC2 | t2.micro, AL2023, public IP | Cheapest, reachable by SSH |
| S3 bucket | name includes account ID, public access blocked, AES256 | Names are global, so the account ID keeps it unique |
| IAM role + instance profile | ListBucket + Put/Get/Delete on this bucket only | Least privilege, no access keys on the box |
| Tags | Name + ProjectCode on everything | Cost tracking and sandbox policy |

## Concepts and how to explain them
- **CloudFormation / IaC:** The template is the source of truth. The stack is the
  deployed instance of it. Same template gives the same result every time, and
  deleting the stack removes everything it created.
- **What makes a subnet public?** Not its name. A subnet is public if its route
  table sends 0.0.0.0/0 to an internet gateway (plus the instance has a public IP).
  My private subnet has only the local route.
- **Security group:** A stateful firewall on the instance. Inbound is deny by
  default. I allow only TCP 22 from my IP as a /32 (exactly one address).
- **IAM role vs access keys:** The instance gets temporary credentials from its role
  through the instance profile. There are no secrets on the box to leak or rotate.
- **Least privilege:** The policy names one bucket. ListBucket applies to the bucket
  ARN. Object actions apply to the ARN with /* on the end. They are different resources.
- **SSM AMI parameter:** The AMI ID is resolved at deploy time, so I never hardcode
  an ID that goes stale.
- **Change set:** A preview of what CloudFormation will add, change or delete. I
  reviewed it before executing.
- **IMDSv2 required:** Blocks a class of SSRF attacks that steal role credentials.

## Live demo script (5 min)
1. Show the template: parameters, then network, then security group, then IAM policy.
2. Show the stack in the console, with its resources and the ProjectCode tag.
3. Show the VPC resource map: public subnet to IGW, private subnet with no route out.
4. SSH in: `ssh -i <key> ec2-user@<ip>` then `whoami`.
5. Prove the role works with no credentials configured:
   `aws s3 ls s3://<bucket>`, then upload, download and delete a test file.
6. Prove least privilege: access to any other bucket should be denied.
7. Teardown: delete the stack, then confirm "stack does not exist".

## Likely questions and answers
- **Why is the private subnet unused here?** The activity needed the architecture, not
  a workload there. Final Act 1 puts an instance in it.
- **Why no NAT gateway?** It costs about $30+/month and the activity didn't need
  outbound internet from the private subnet.
- **What if your IP changes?** SSH stops working. I'd update the stack with the new
  MyIp parameter through a new change set.
- **Is the public IP permanent?** No. It's auto-assigned and changes on stop/start.
  An Elastic IP would keep it, at extra cost.
- **Why can't you SSH to the private subnet?** There is no route to it from the internet.
- **Why did stack deletion need an empty bucket?** CloudFormation can't delete a
  non-empty bucket. I left it empty after testing.
- **Why t2.micro?** Cheapest option that meets the requirement, and no extra instances.
- **Security weaknesses?** SSH on the internet with a key (even locked to one IP),
  and a single AZ. A production setup would use SSM Session Manager and multiple AZs.

## Honest caveats
- Single AZ, no high availability (fine for a sandbox).
- Shared account: I only touched resources tagged ProjectCode or named Daga-ang-*.
- Key pair and local .pem are not part of the stack and were kept.
