# Demo Cheat Sheet (one page)

## 0. Setup (once)
```bash
cd ~/Developer/cloud-immersion
export AWS_PROFILE=cloud-basics-sso AWS_DEFAULT_REGION=ap-southeast-1
aws sso login
aws sts get-caller-identity          # account 000179588365, role CloudNewbies
MYIP=$(curl -s https://checkip.amazonaws.com); echo $MYIP
```
SSO = temporary credentials. SSH is limited to my IP /32, so check it fresh.

## Pattern (both demos)
Validate -> Change set (preview) -> Describe -> Execute -> Test -> Delete.
Say: IaC is repeatable. A change set is a safe dry run. Deleting the stack removes everything.

## DEMO 1: Act 1 (VPC + EC2 + S3 + IAM role)
```bash
A=Daga-ang-Stack-Act1; C=Daga-ang-ChangeSet-Act1
aws cloudformation validate-template --template-body file://cloud-basics-act1.yaml
aws cloudformation create-change-set --stack-name $A --change-set-name $C \
  --change-set-type CREATE --template-body file://cloud-basics-act1.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=MyIp,ParameterValue=$MYIP ParameterKey=KeyName,ParameterValue=Daga-ang-KeyPair-FinalAct1 \
  --tags Key=ProjectCode,Value=CloudBasics-Act1-CloudIntern
aws cloudformation wait change-set-create-complete --stack-name $A --change-set-name $C
aws cloudformation describe-change-set --stack-name $A --change-set-name $C \
  --query 'Changes[].ResourceChange.[Action,LogicalResourceId,ResourceType]' --output table   # 15, all Add
aws cloudformation execute-change-set --stack-name $A --change-set-name $C
aws cloudformation wait stack-create-complete --stack-name $A                                  # 2-3 min: explain architecture
aws cloudformation describe-stacks --stack-name $A --query 'Stacks[0].Outputs[].[OutputKey,OutputValue]' --output table
```
Console: stack + ProjectCode tag, VPC resource map.
Explain while waiting: public subnet -> IGW; private subnet has no internet route, no NAT (cost); SG = SSH from my /32; IAM role = bucket-only access, no keys on the box.

```bash
ssh -i ~/.ssh/Daga-ang-KeyPair-FinalAct1.pem ec2-user@<ip>
whoami
B=daga-ang-bucket-act1-000179588365-ap-southeast-1
aws s3 ls s3://$B
echo demo > t.txt; aws s3 cp t.txt s3://$B/t.txt
aws s3 cp s3://$B/t.txt back.txt && cat back.txt
aws s3 rm s3://$B/t.txt
aws s3 ls            # AccessDenied = least privilege
exit
```
Teardown (bucket must be empty):
```bash
aws s3api list-objects-v2 --bucket daga-ang-bucket-act1-000179588365-ap-southeast-1
aws cloudformation delete-stack --stack-name $A
aws cloudformation wait stack-delete-complete --stack-name $A
aws cloudformation describe-stacks --stack-name $A     # "does not exist"
```

## DEMO 2: Final Act (Bastion + private EC2)
```bash
F=Daga-ang-Stack-FinalAct1; G=Daga-ang-ChangeSet-FinalAct1
aws cloudformation validate-template --template-body file://final-act1.yaml
aws cloudformation create-change-set --stack-name $F --change-set-name $G \
  --change-set-type CREATE --template-body file://final-act1.yaml \
  --parameters ParameterKey=MyIp,ParameterValue=$MYIP \
  --tags Key=ProjectCode,Value=FinalAct1-CloudIntern
aws cloudformation wait change-set-create-complete --stack-name $F --change-set-name $G
aws cloudformation describe-change-set --stack-name $F --change-set-name $G \
  --query 'Changes[].ResourceChange.[Action,LogicalResourceId,ResourceType]' --output table   # 14, all Add
aws cloudformation execute-change-set --stack-name $F --change-set-name $G
aws cloudformation wait stack-create-complete --stack-name $F
aws cloudformation describe-stacks --stack-name $F --query 'Stacks[0].Outputs[].[OutputKey,OutputValue]' --output table
```
Console: resource map; bastion SG (22 from my /32); private SG (22 from the bastion's SG ID).
Concepts: bastion = only door in; private host has no public IP; SG referencing survives IP changes.

```bash
ssh-add ~/.ssh/Daga-ang-KeyPair-FinalAct1.pem      # key into local agent
ssh -A ec2-user@<bastion-ip>                       # -A forwards agent: .pem never copied
ssh ec2-user@<private-ip>                          # from the bastion
hostname -I                                        # 10.0.2.x = private subnet
exit; exit
ssh -o ConnectTimeout=5 ec2-user@<private-ip>      # from Mac: MUST time out
# one-liner alternative: ssh -J ec2-user@<bastion-ip> ec2-user@<private-ip>
```
Teardown:
```bash
aws cloudformation delete-stack --stack-name $F
aws cloudformation wait stack-delete-complete --stack-name $F
aws cloudformation describe-stacks --stack-name $F     # "does not exist"
```

## Q&A
- No NAT? Costs ~$30/month and isn't required. Tradeoff: no outbound internet from private.
- /24 subnets? Activity text says /16, which is impossible in a /16 VPC. /24 assumed, confirm with mentor.
- Why change set? Preview before changing anything.
- Secure? SSH only from my /32; private SG only from bastion SG; no keys on bastion; IAM scoped to one bucket.
- SSH timeout? My IP changed. Re-run MYIP and update the stack.

## Rules
One stack at a time. Delete Act 1 before starting Final. Delete the stack right after each demo.
