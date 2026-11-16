# Part 02: The workstation (IAM role, key pair, security group, EC2)

Think first: our EC2 will run `aws`, `eksctl` and `docker push`. Where should it get AWS permissions from? Copying our access keys onto it is the wrong answer. Why?

### Commands (laptop)

```bash
# 1. IAM role for the EC2 workstation (lab only: AdministratorAccess)
aws iam create-role --role-name triage-workstation-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  --query Role.Arn --output text
aws iam attach-role-policy --role-name triage-workstation-role \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam create-instance-profile --instance-profile-name triage-workstation-role
aws iam add-role-to-instance-profile --instance-profile-name triage-workstation-role \
  --role-name triage-workstation-role
sleep 15     # IAM needs a few seconds before EC2 can use the new profile

# 2. Key pair in PuTTY format (.ppk)
mkdir -p ~/keys
aws ec2 create-key-pair --key-name triage-key --key-type rsa --key-format ppk \
  --query KeyMaterial --output text > ~/keys/triage-key.ppk
ls -l ~/keys/triage-key.ppk

# 3. Security group: SSH and port 8000 only from your own IP
MY_IP=$(curl -s https://checkip.amazonaws.com); echo "My IP: $MY_IP"
SG_ID=$(aws ec2 create-security-group --group-name triage-workstation-sg \
  --description "TriageOps workstation" --vpc-id $VPC_ID --query GroupId --output text); save SG_ID
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22   --cidr $MY_IP/32
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 8000 --cidr $MY_IP/32
# Optional: also allow the console "Connect" button (EC2 Instance Connect uses AWS IPs, not yours)
PL_ID=$(aws ec2 describe-managed-prefix-lists \
  --filters Name=prefix-list-name,Values=com.amazonaws.ap-south-1.ec2-instance-connect \
  --query 'PrefixLists[0].PrefixListId' --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --ip-permissions "IpProtocol=tcp,FromPort=22,ToPort=22,PrefixListIds=[{PrefixListId=$PL_ID}]"

# 4. Latest Amazon Linux 2023 image, then launch the workstation WITH the role attached
AMI_ID=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text); echo "AMI: $AMI_ID"
INSTANCE_ID=$(aws ec2 run-instances --image-id $AMI_ID --instance-type t3.medium \
  --key-name triage-key --subnet-id $SUBNET_A --security-group-ids $SG_ID \
  --iam-instance-profile Name=triage-workstation-role \
  --block-device-mappings 'DeviceName=/dev/xvda,Ebs={VolumeSize=20,VolumeType=gp3}' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=triage-workstation}]' \
  --query 'Instances[0].InstanceId' --output text); save INSTANCE_ID
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].[State.Name,SecurityGroups[0].GroupName,IamInstanceProfile.Arn]' --output text
PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text); save PUBLIC_IP
```

### What we did and why

- **IAM role + instance profile**: this answers the question above. Access keys copied onto a server can leak and never expire. A role gives the EC2 temporary credentials that rotate on their own. The instance profile is the "holder" that attaches the role to EC2. AdministratorAccess is for the lab only. In a company you give only what is needed.
- **Attach the role at launch** (`--iam-instance-profile`). We learned this the hard way: attaching a role to an already running instance sometimes leaves it with no credentials until you detach and attach again.
- **sleep 15**: IAM is global and takes a few seconds to be visible to EC2. If `run-instances` says "Invalid IAM Instance Profile name", wait 10 seconds and run it again.
- **Key pair with `--key-format ppk`**: AWS gives us the private key directly in PuTTY format. No PuTTYgen conversion needed. AWS keeps only the public half. Lose this file and you cannot log in. Never commit it and never share it.
- **The check after launch** must show `running`, `triage-workstation-sg` and the instance profile ARN. If it shows `default` as the group name, the instance has the VPC default security group and SSH will not work.
- **Security group**: port 22 (SSH) and 8000 (to test our service from the browser) open only to your IP (`/32`), not to the whole internet.
- **AMI from SSM**: AWS publishes the latest Amazon Linux 2023 image ID under a fixed SSM path. No hunting for AMI IDs. This is the command that needs `MSYS_NO_PATHCONV=1` in Git Bash.
- **t3.medium, 20 GiB gp3**: 2 vCPU and 4 GB RAM is enough for Docker builds. 20 GiB because Docker images take space.

**Launching from the AWS console instead?** Same choices, on the Launch instance page:
Name `triage-workstation`, Amazon Linux 2023, t3.medium, key pair `triage-key`,
Network settings > **Edit**: VPC `triage-vpc`, subnet `triage-public-a`, Auto-assign public IP **Enable**,
**Select existing security group** `triage-workstation-sg` (never leave it on `default`),
Storage 20 GiB gp3, Advanced details > IAM instance profile `triage-workstation-role`.
Then save the IDs on the laptop:

```bash
INSTANCE_ID=$(aws ec2 describe-instances --filters Name=tag:Name,Values=triage-workstation Name=instance-state-name,Values=running \
  --query 'Reservations[0].Instances[0].InstanceId' --output text); save INSTANCE_ID
PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text); save PUBLIC_IP
SG_ID=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' --output text); save SG_ID
```

**Concept: the role is the identity of the machine**
Humans log in with users. Machines get roles. When code on EC2 calls AWS, it asks the metadata service at 169.254.169.254 for temporary credentials of its role.

---

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 01](part-01-network-vpc.md) | [Lab map](README.md) | Next: [Part 03: Connect with PuTTY and check the role](part-03-connect-with-putty.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
