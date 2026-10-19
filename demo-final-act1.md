# Demo Guide — Final Activity 1 (Bastion host + private EC2)

## 30-second pitch
"I built a VPC with CloudFormation where the only door in is a bastion host in the
public subnet, locked to my IP. The second server sits in a private subnet with no
public IP, and only the bastion can SSH to it. I hopped from my Mac through the
bastion to the private server without ever copying my key onto the bastion."

## Architecture
```
Internet --(SSH 22, my IP /32 only)--> [Bastion  10.0.1.x, public subnet]
                                              |
                                  (SSH 22, from bastion SG only)
                                              v
                                       [Private  10.0.2.x, private subnet, no public IP]
```
| Piece | Value |
|---|---|
| VPC | 10.0.0.0/16 |
| Public subnet | 10.0.1.0/24 (route 0.0.0.0/0 to IGW) |
| Private subnet | 10.0.2.0/24 (local route only, no NAT) |
| Bastion SG | Inbound 22 from MyIp/32 |
| Private SG | Inbound 22 from the **bastion security group** (not a CIDR) |
| Instances | 2 x t2.micro, AL2023 via SSM parameter, IMDSv2 required |
| Tags | Name + ProjectCode=FinalAct1-CloudIntern; the stack is tagged too |

## Concepts and how to explain them
- **Bastion / jump host:** One hardened, exposed entry point. Private servers are
  never exposed directly, so there is one place to monitor and lock down.
- **Security group referencing:** The private SG's source is the bastion's SG ID, not
  an IP. So "only the bastion" stays true even if the bastion's IP changes, and
  anything else in the VPC is still blocked.
- **Public vs private subnet:** Decided by the route table. The private route table
  has no path to the IGW, and the instance has no public IP.
- **Why a private subnet at all:** Defense in depth. Databases and app servers don't
  need to be reachable from the internet.
- **Agent forwarding (`ssh -A`) / ProxyJump (`-J`):** My private key stays on my Mac.
  The bastion only relays the authentication. Copying a .pem onto a shared bastion
  would be a real security risk.
- **Stateful security groups:** Reply traffic is allowed automatically, so I only
  write inbound rules.
- **Change set:** Previewed 14 resources, all Add, before executing.
- **Parameters:** MyIp has no default on purpose, so I must check it fresh at each deploy.

## Live demo script (5 min)
1. Show the template: SGs first, because they are the heart of the design.
2. VPC resource map: two subnets, two route tables, one IGW, only the public side routed.
3. Show the bastion SG (22 from my /32) and the private SG (22 from the bastion SG).
4. `ssh-add ~/.ssh/Daga-ang-KeyPair-FinalAct1.pem`, then `ssh -A ec2-user@<bastion-ip>`.
5. From the bastion: `ssh ec2-user@<private-ip>`, then `hostname -I` (shows 10.0.2.x).
6. Negative test: from my Mac, `ssh ec2-user@<private-ip>` times out, because the
   private instance is unreachable directly.
7. Teardown: delete the stack, then verify "stack does not exist".

## Likely questions and answers
- **Subnets are /16 in the spec. Why /24?** Two /16 subnets can't fit inside a /16 VPC
  (the VPC only has one /16 of space). /24 is the sensible reading. I flagged it.
- **Why can't the private instance reach the internet?** No NAT gateway. It would add
  roughly $30+/month and the activity doesn't need outbound access. The trade-off is
  no package updates on that server.
- **How do you fix it if your IP changes?** Update the stack with the new MyIp through
  a change set.
- **Why reference an SG instead of 10.0.1.0/24?** A CIDR would allow every host in the
  public subnet. The SG reference allows only members of the bastion SG.
- **Single point of failure?** Yes, one bastion in one AZ. Production could use
  multiple AZs, or SSM Session Manager to drop the bastion and open ports entirely.
- **Why the same key for both instances?** Simpler for the sandbox. Better practice is
  separate keys, or SSM.
- **Isn't SSH open to the internet risky?** It's limited to one /32 and needs the
  private key. Still, SSM Session Manager would be better.
- **Why IMDSv2?** It blocks credential theft through SSRF.
- **How do you know it works?** Screenshot of the hop with prompts ip-10-0-1-x then
  ip-10-0-2-240, plus the direct-access timeout.

## Honest caveats
- Single AZ, one bastion, public IP not elastic (changes on stop/start).
- Both instances share one key pair.
- Subnets are /24 by my interpretation of the spec.
- cfn-lint was not run; only `validate-template`.
