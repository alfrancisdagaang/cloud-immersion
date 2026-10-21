# Deployment Commands — step by step (Activity 1 and Final Activity 1)

Run these from the project folder in the Mac terminal. Both activities deploy real
resources (about $0.50-$1/day), so deploy only when ready, and
delete the stack right after.

Companion docs: `act1-guide.md`, `final-act1-guide.md` (concepts and Q&A).

## Common setup (both activities)

```bash
cd ~/Developer/cloud-immersion
export AWS_PROFILE=cloud-basics-sso AWS_DEFAULT_REGION=ap-southeast-1
aws sso login                                  # skip if the session is still valid
aws sts get-caller-identity                    # should show account 000179588365, role CloudNewbies
MYIP=$(curl -s https://checkip.amazonaws.com); echo $MYIP   # check this fresh each time
```

## Activity 1

**Key pair warning.** The Act 1 key pair's private key was saved on the old
Windows machine, so it is not on this Mac. The simplest fix is to pass the Final
Act key pair (`KeyName` is a parameter), as the commands below do.

```bash
# 1. Validate
aws cloudformation validate-template --template-body file://cloud-basics-act1.yaml

# 2. Create the change set (preview only)
aws cloudformation create-change-set \
  --stack-name Daga-ang-Stack-Act1 --change-set-name Daga-ang-ChangeSet-Act1 \
  --change-set-type CREATE --template-body file://cloud-basics-act1.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=MyIp,ParameterValue=$MYIP ParameterKey=KeyName,ParameterValue=Daga-ang-KeyPair-FinalAct1 \
  --tags Key=ProjectCode,Value=CloudBasics-Act1-CloudIntern
aws cloudformation wait change-set-create-complete --stack-name Daga-ang-Stack-Act1 --change-set-name Daga-ang-ChangeSet-Act1
aws cloudformation describe-change-set --stack-name Daga-ang-Stack-Act1 --change-set-name Daga-ang-ChangeSet-Act1 \
  --query 'Changes[].ResourceChange.[Action,LogicalResourceId,ResourceType]' --output table

# 3. Execute and wait (about 2-3 min)
aws cloudformation execute-change-set --stack-name Daga-ang-Stack-Act1 --change-set-name Daga-ang-ChangeSet-Act1
aws cloudformation wait stack-create-complete --stack-name Daga-ang-Stack-Act1

# 4. Get the instance IP and bucket name
aws cloudformation describe-stacks --stack-name Daga-ang-Stack-Act1 --query 'Stacks[0].Outputs[].[OutputKey,OutputValue]' --output table
```

Show in the console: the stack and its `ProjectCode` tag, then the VPC resource map.

```bash
# 5. SSH in (replace <ip>)
ssh -i ~/.ssh/Daga-ang-KeyPair-FinalAct1.pem ec2-user@<ip>
whoami
```

On the instance, proving the IAM role with no stored credentials:

```bash
B=daga-ang-bucket-act1-000179588365-ap-southeast-1
aws s3 ls s3://$B                              # list
echo "test" > t.txt
aws s3 cp t.txt s3://$B/t.txt                  # upload
aws s3 cp s3://$B/t.txt back.txt && cat back.txt   # download and compare
aws s3 rm s3://$B/t.txt                        # delete
aws s3 ls                                      # least privilege: should be AccessDenied
exit
```

Teardown (the bucket must be empty first):

```bash
aws s3api list-objects-v2 --bucket daga-ang-bucket-act1-000179588365-ap-southeast-1
aws cloudformation delete-stack --stack-name Daga-ang-Stack-Act1
aws cloudformation wait stack-delete-complete --stack-name Daga-ang-Stack-Act1
aws cloudformation describe-stacks --stack-name Daga-ang-Stack-Act1   # expect "does not exist"
```

## Final Activity 1

```bash
# 1. Validate
aws cloudformation validate-template --template-body file://final-act1.yaml

# 2. Create the change set (preview only; no IAM capability needed)
aws cloudformation create-change-set \
  --stack-name Daga-ang-Stack-FinalAct1 --change-set-name Daga-ang-ChangeSet-FinalAct1 \
  --change-set-type CREATE --template-body file://final-act1.yaml \
  --parameters ParameterKey=MyIp,ParameterValue=$MYIP \
  --tags Key=ProjectCode,Value=FinalAct1-CloudIntern
aws cloudformation wait change-set-create-complete --stack-name Daga-ang-Stack-FinalAct1 --change-set-name Daga-ang-ChangeSet-FinalAct1
aws cloudformation describe-change-set --stack-name Daga-ang-Stack-FinalAct1 --change-set-name Daga-ang-ChangeSet-FinalAct1 \
  --query 'Changes[].ResourceChange.[Action,LogicalResourceId,ResourceType]' --output table

# 3. Execute and wait
aws cloudformation execute-change-set --stack-name Daga-ang-Stack-FinalAct1 --change-set-name Daga-ang-ChangeSet-FinalAct1
aws cloudformation wait stack-create-complete --stack-name Daga-ang-Stack-FinalAct1

# 4. Get the IPs and ready-made SSH commands
aws cloudformation describe-stacks --stack-name Daga-ang-Stack-FinalAct1 --query 'Stacks[0].Outputs[].[OutputKey,OutputValue]' --output table
```

Show in the console: the VPC resource map, the bastion security group (22 from
my /32), and the private security group (22 from the bastion's SG ID).

```bash
# 5. Load the key and hop (replace the IPs from the outputs)
ssh-add ~/.ssh/Daga-ang-KeyPair-FinalAct1.pem
ssh -A ec2-user@<bastion-ip>
# on the bastion:
ssh ec2-user@<private-ip>
hostname -I          # should show 10.0.2.x
whoami
exit; exit

# 6. Negative test from the Mac: this should time out
ssh -o ConnectTimeout=5 ec2-user@<private-ip>
```

Teardown:

```bash
aws cloudformation delete-stack --stack-name Daga-ang-Stack-FinalAct1
aws cloudformation wait stack-delete-complete --stack-name Daga-ang-Stack-FinalAct1
aws cloudformation describe-stacks --stack-name Daga-ang-Stack-FinalAct1   # expect "does not exist"
```

## Tips
- **Reusing change set names:** `create-change-set` fails if that name already exists
  on a live stack. If a previous attempt failed, delete that stack first.
- **If SSH times out:** the public IP changed. Re-run `MYIP=...`, then create an
  UPDATE change set (`--change-set-type UPDATE`) with the new IP, or redeploy.
- **Known-hosts warnings:** new instances get new host keys. `ssh-keygen -R <ip>`
  clears an old entry if an address is reused.
- **Delete when done.** Run the teardown right after you finish.
