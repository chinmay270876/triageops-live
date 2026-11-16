# TriageOps Lab: From VS Code to EKS, End to End

Python and Production Engineering, hands-on lab
Written by Rahul Chaubey for you.

You will build one real service, step by step, and ship it to AWS EKS.
Every command is here. Run them in order and you will reach the end.

---

## The story we are solving

Friday night. It is raining, an IPL match is on, and everyone in the city is ordering food on our Zomato style app.
Payments are slow. Some customers are charged twice for their biryani.
Thousands of complaints arrive in a few minutes.

Someone has to read each complaint and decide three things:

- What is it about? (payments, delivery, restaurant, refund)
- How urgent is it? (priority 1 to 4)
- Which team should handle it?

That decision is called **triage**. We will build the **Complaint Triage Service**.
An AI model makes the decision. Our service makes sure the decision is safe to use.

The AI writes code fast. Production engineering decides whether it ships.
Our service passes 4 gates before it is allowed to run in production:

| Gate | Who says NO | What we add |
|------|-------------|-------------|
| 1. CONTRACT | Support Ops: "the AI sent a team name that does not exist" | Pydantic models, 422 on bad input, AI answer checked |
| 2. ASYNC | Support Team Lead: "1000 complaints, one by one, is too slow" | async/await, httpx.AsyncClient, timeouts, fallback |
| 3. SERVICE | Platform Engineer and Security Head: "no secrets in code, no phone numbers in logs" | config from environment, SecretStr, JSON logs, /healthz |
| 4. SHIP | Production | Docker image, Amazon ECR, Kubernetes on EKS, rolling update |

---

## What you will build

```
  Your laptop (VS Code + Git Bash)
        |
        |  AWS CLI: VPC, IAM role, key, security group, EC2
        v
  EC2 workstation (Amazon Linux 2023)  <---- PuTTY (SSH)
        |   Python, Docker, Git, kubectl, eksctl
        |
        |-- git push -------------> GitHub: discover-devops/triageops
        |-- docker push ----------> Amazon ECR: triageops:1.2.0, 1.3.0
        |-- eksctl / kubectl -----> Amazon EKS: triageops-eks
                                         |
                                         |  namespace triageops
                                         |   - Deployment triage (2 pods)  <-- Service LoadBalancer (port 80)
                                         |   - Deployment ai-stub (1 pod)  <-- Service ClusterIP (port 9000)
                                         |   - ConfigMap triage-config
                                         |   - Secret triage-secrets
```

The **AI stub** is a fake AI model we run ourselves. It answers like a real AI provider (it even needs an API key),
but it costs nothing and we can make it slow, wrong or broken on purpose.

## Lab map

| Part | Where | What |
|------|-------|------|
| 00 | Laptop | Before you start |
| 01 | Laptop (Git Bash) | Network: VPC, subnets, internet gateway |
| 02 | Laptop (Git Bash) | Workstation: IAM role, key pair, security group, EC2 |
| 03 | Laptop (PuTTY) | Connect to EC2 and check the IAM role |
| 04 | EC2 | Install tools |
| 05 | EC2 | Start the EKS cluster (runs in the background, about 20 minutes) |
| 06 | EC2 + GitHub | GitHub repo, token, clone |
| 07 | EC2 | Gate 1: CONTRACT |
| 08 | EC2 | Gate 2: ASYNC |
| 09 | EC2 | Gate 3: SERVICE, then the first pull request |
| 10 | EC2 | Gate 4: SHIP part 1: Docker image and Amazon ECR |
| 11 | EC2 | Gate 4: SHIP part 2: deploy to EKS |
| 12 | EC2 | Complaint web page, version 1.3.0, rolling update |
| 13 | EC2 | Break it and diagnose |
| 14 | EC2 + Laptop | Clean up everything (do not skip, EKS costs money) |
| 15 | - | Troubleshooting: every error we hit and the fix |

Golden rule for Git in this lab: **one branch per part, then pull request, merge, and come back to main before the next part.**

---

# Part 00: Before you start

You need:

- An AWS account with admin access (IAM user access keys for the AWS CLI on your laptop)
- A GitHub account (this runbook uses `discover-devops`; use your own username wherever you see it)
- VS Code with **Git Bash** as the terminal (Windows), or any bash terminal on Mac or Linux
- AWS CLI v2 on the laptop: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- PuTTY on Windows: https://www.putty.org/

### Commands (laptop, Git Bash in VS Code)

```bash
aws --version
aws configure            # paste your access key and secret key, region ap-south-1, output json
aws sts get-caller-identity

# Git Bash only: stop it from turning /aws/... and /dev/... into Windows paths
echo 'export MSYS_NO_PATHCONV=1' >> ~/.bashrc
echo 'export AWS_DEFAULT_REGION=ap-south-1' >> ~/.bashrc
source ~/.bashrc

# A small file that remembers every AWS ID we create, so we can clean up later
: > ~/triage-lab.env
save() {
  if [ -z "${!1}" ] || [ "${!1}" = "None" ]; then echo "ERROR: $1 is empty. Fix the command above before going on."; return 1; fi
  echo "export $1=${!1}" >> ~/triage-lab.env; echo "saved $1=${!1}"
}
```

### What we did and why

- `aws configure` stores your keys in `~/.aws/credentials` on the laptop. Never show this file on screen and never paste the keys anywhere.
- `aws sts get-caller-identity` answers one question: "who am I to AWS?" If it prints your account ID, the CLI works.
- `MSYS_NO_PATHCONV=1`: Git Bash rewrites anything that looks like a Linux path (for example `/aws/service/...` or `/dev/xvda`) into a Windows path. AWS then gets a wrong value. This switch turns that off.
- `~/triage-lab.env` and `save`: every ID we create (VPC, subnet, instance) is written here. If you close VS Code, run `source ~/triage-lab.env` and all IDs are back. The cleanup in Part 14 depends on it.
- `save` refuses an empty value. An empty variable is the most common reason a pasted AWS command fails later with "expected one argument". If you see `ERROR: ... is empty`, stop and fix that step first.
- If you open a new terminal later, run `source ~/triage-lab.env` and paste the `save() { ... }` lines again (functions are not saved in the file).

**Concept: why a workstation on EC2?**
Our laptops are all different: Windows, Mac, low RAM, office proxies. The EC2 workstation is the same for everyone and is already inside AWS. It is fast to push images to ECR and to talk to EKS. The laptop only creates the network and the workstation.

---

# Part 01: Network (VPC)

Think first: EKS needs subnets in at least two Availability Zones, and a public load balancer needs to know which subnets are public. How will EKS know that?

### Commands (laptop)

```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=triage-vpc}]' \
  --query Vpc.VpcId --output text); save VPC_ID
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames '{"Value":true}'

IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=triage-igw}]' \
  --query InternetGateway.InternetGatewayId --output text); save IGW_ID
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

SUBNET_A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=triage-public-a},{Key=kubernetes.io/role/elb,Value=1}]' \
  --query Subnet.SubnetId --output text); save SUBNET_A
SUBNET_B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=triage-public-b},{Key=kubernetes.io/role/elb,Value=1}]' \
  --query Subnet.SubnetId --output text); save SUBNET_B
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_A --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_B --map-public-ip-on-launch

RT_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=triage-public-rt}]' \
  --query RouteTable.RouteTableId --output text); save RT_ID
aws ec2 create-route --route-table-id $RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $RT_ID --subnet-id $SUBNET_A
aws ec2 associate-route-table --route-table-id $RT_ID --subnet-id $SUBNET_B

cat ~/triage-lab.env
```

### What we did and why

- **VPC 10.0.0.0/16** `triage-vpc`: our own private network. DNS hostnames are turned on because EKS nodes need them.
- **Internet gateway** `triage-igw`: the door from the VPC to the internet. Without it nothing can be reached from outside.
- **Two public subnets**, one in `ap-south-1a` and one in `ap-south-1b`. EKS needs two Availability Zones, so one AZ failure does not take the cluster down.
- **Tag `kubernetes.io/role/elb=1`**: this answers the question above. When we create a Kubernetes Service of type LoadBalancer, AWS looks for subnets with this tag to place the public load balancer.
- **map-public-ip-on-launch**: every EC2 and every EKS node in these subnets gets a public IP. We have no NAT gateway, so this is how the nodes reach ECR and the EKS API.
- **Route table** `triage-public-rt`: "everything to 0.0.0.0/0 goes to the internet gateway". This is what makes a subnet public.

**Concept: public subnet**
A subnet is public only because its route table sends 0.0.0.0/0 to an internet gateway. The name "public" does nothing by itself.

---

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

# Part 03: Connect with PuTTY and check the role

### Steps (laptop)

1. Open PuTTY.
2. **Session**: Host Name `ec2-user@<PUBLIC_IP>` (run `echo $PUBLIC_IP` in Git Bash), port 22.
3. **Connection > SSH > Auth > Credentials**: Private key file = `triage-key.ppk` from your `keys` folder (in Windows: `C:\Users\<you>\keys\triage-key.ppk`).
4. Back to **Session**, Saved Sessions name `triage-workstation`, click **Save**, then **Open**. Accept the host key.
5. To paste into PuTTY: right click.

### Commands (EC2)

```bash
aws configure set region ap-south-1
aws sts get-caller-identity
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/; echo
```

### What we did and why

- `aws configure set region`: the role gives credentials, not a default region. We set it once.
- `aws sts get-caller-identity` must show `assumed-role/triage-workstation-role/...`. That proves the EC2 is using the role and we never typed a key on it.
- The two `curl` lines ask the metadata service directly. It must print `triage-workstation-role`. If you get `404` or "Unable to locate credentials", see Part 15.

---

# Part 04: Install tools on EC2

### Commands (EC2)

```bash
sudo dnf install -y docker git python3.11 python3.11-pip
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user

# kubectl matching the cluster version (1.34)
curl -sLO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable-1.34.txt)/bin/linux/amd64/kubectl"
sudo install -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl

# eksctl
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz"
tar -xzf eksctl_Linux_amd64.tar.gz -C /tmp && rm eksctl_Linux_amd64.tar.gz
sudo install -m 0755 /tmp/eksctl /usr/local/bin/eksctl && rm /tmp/eksctl

exit
```

Now open the saved PuTTY session `triage-workstation` again, then:

```bash
docker run --rm hello-world | head -3
git --version; python3.11 --version; kubectl version --client; eksctl version; aws --version
```

### What we did and why

- **docker, git, python3.11** from the Amazon Linux repos. Python 3.11 is the version our Docker image also uses, so "works on EC2" means "works in the container".
- **usermod -aG docker**: lets `ec2-user` run Docker without sudo. Group changes apply only to a new login, so we `exit` and log in again. If you skip this you get "permission denied while trying to connect to the Docker daemon socket".
- **kubectl 1.34**: kubectl should be within one minor version of the cluster. We pin `stable-1.34` because the newest kubectl was too new for our 1.34 cluster.
- **eksctl**: one command that creates a full EKS cluster. Under the hood it writes CloudFormation stacks.

---

# Part 05: Start the EKS cluster (runs while we code)

The cluster takes about 15 to 20 minutes. We start it now, in the background, and build the service while it comes up.

### Commands (EC2)

```bash
mkdir -p ~/triage && cd ~/triage
CLUSTER=triageops-eks

# Pre-check 1: find the subnets of THIS workstation's VPC by their Name tag. Never reuse old IDs.
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
MAC=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/mac)
VPC_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/network/interfaces/macs/$MAC/vpc-id)
SUBNET_A=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID Name=tag:Name,Values=triage-public-a --query 'Subnets[0].SubnetId' --output text)
SUBNET_B=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID Name=tag:Name,Values=triage-public-b --query 'Subnets[0].SubnetId' --output text)
echo "VPC=$VPC_ID  SUBNET_A=$SUBNET_A  SUBNET_B=$SUBNET_B"      # all three filled in, none says None

# Pre-check 2: no cluster and no leftover eksctl stacks
eksctl get cluster --region ap-south-1
aws cloudformation list-stacks \
  --query "StackSummaries[?starts_with(StackName,'eksctl-') && StackStatus!='DELETE_COMPLETE'].[StackName,StackStatus]" \
  --output table
```

If the table shows any `eksctl-triageops-eks-...` stack, clean it first (see "Leftover stacks" below). If it is empty, continue:

```bash
cat > cluster.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: $CLUSTER
  region: ap-south-1
  version: "1.34"
vpc:
  subnets:
    public:
      ap-south-1a: { id: $SUBNET_A }
      ap-south-1b: { id: $SUBNET_B }
managedNodeGroups:
  - name: triage-nodes
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 2
    maxSize: 3
    volumeSize: 20
EOF
cat cluster.yaml

nohup eksctl create cluster -f cluster.yaml > eks-create.log 2>&1 &
sleep 20; tail -5 eks-create.log
```

Check on it any time (even after closing PuTTY):

```bash
tail -5 ~/triage/eks-create.log
```

### Leftover stacks (only if pre-check 2 found something)

```bash
# A stack still CREATE_IN_PROGRESS: wait for it, then delete the cluster properly
aws cloudformation wait stack-create-complete --stack-name eksctl-triageops-eks-cluster
eksctl delete cluster --name triageops-eks --region ap-south-1 --wait

# A stack in ROLLBACK_COMPLETE or DELETE_FAILED: delete the stack and wait
aws cloudformation delete-stack --stack-name <STACK_NAME>
aws cloudformation wait stack-delete-complete --stack-name <STACK_NAME>
```

Then run pre-check 2 again until the table is empty. If you are in a hurry, the simplest way out is a new cluster name (for example `triageops-eks2`) in `CLUSTER=`.

### If eksctl was stopped in the middle

If the eksctl process died (Ctrl+C, instance reboot) after the control plane stack started:

```bash
aws cloudformation wait stack-create-complete --stack-name eksctl-triageops-eks-cluster
eksctl create nodegroup -f cluster.yaml
eksctl utils write-kubeconfig --cluster triageops-eks --region ap-south-1
kubectl get nodes
```

### What we did and why

- **Subnets by Name tag, inside our own VPC**: the instance metadata service tells the workstation which VPC it lives in. We then look up the subnets by Name tag only inside that VPC. Copying old subnet IDs fails with `InvalidSubnetID.NotFound`, and an older VPC with the same tag names could be picked by mistake.
- **Pre-check for leftover stacks**: eksctl creates CloudFormation stacks named `eksctl-<cluster>-cluster` and `eksctl-<cluster>-nodegroup-<name>`. If a stack with that name already exists you get `AlreadyExistsException`.
- **cluster.yaml**: the cluster written as code. Version 1.34, our two public subnets, and a managed node group of 2 x t3.medium. `<<EOF` (without quotes) lets the shell fill in `$SUBNET_A`.
- **nohup ... &**: runs eksctl in the background and keeps it running even if PuTTY disconnects. All output goes to `eks-create.log`.
- **Concept: control plane and nodes.** AWS runs the Kubernetes control plane for us. The node group is our 2 EC2 machines where the pods run. eksctl creates the control plane first, then the nodes, then writes `~/.kube/config` so kubectl can talk to the cluster.

Leave it running. We come back to it in Part 11.

---

# Part 06: GitHub repo, token and clone

### Steps (browser)

1. On GitHub (logged in as `discover-devops`, or your own user): **New repository**, name `triageops`, Public, tick **Add a README file**, **Create repository**.
2. Create a token: profile picture > **Settings** > **Developer settings** > **Personal access tokens** > **Tokens (classic)** > **Generate new token (classic)**. Note `triageops-lab`, expiration 7 days, scope tick **repo**, **Generate token**. Copy it once. Do not paste it in chat, in a file, or on screen.

### Commands (EC2)

```bash
git config --global user.name  "discover-devops"
git config --global user.email "discover.devops@gmail.com"
git config --global init.defaultBranch main
git config --global credential.helper 'cache --timeout=36000'

cd ~
git clone https://github.com/discover-devops/triageops.git
cd ~/triageops && ls
```

Use your own GitHub username and email in the two config lines and in the clone URL.

### What we did and why

- **Classic token with `repo` scope**: GitHub does not accept your password for git push. A fine-grained token gave us `403` until it had the right repository permissions, so in this lab we use a classic token with `repo` scope and a short expiry.
- **credential cache**: the first `git push` asks for Username (your GitHub user) and Password (paste the token, nothing appears on screen, press Enter). Git keeps it **in memory only** for 10 hours. It is never written to disk.
- If a token ever shows on screen or in a chat: revoke it on GitHub right away and create a new one.

---

# Part 07: Gate 1, CONTRACT

Support Ops says NO: "Last night the AI sent `payment team`. We have no team called that. 500 complaints went nowhere."

Think first: who should catch a wrong team name, the AI, the support team, or our service?

### Commands (EC2)

```bash
cd ~/triageops
git checkout -b service

mkdir -p app ai_stub scripts
touch app/__init__.py ai_stub/__init__.py

cat > .gitignore <<'EOF'
.venv/
__pycache__/
*.pyc
.env
*.log
EOF

cat > requirements.txt <<'EOF'
fastapi==0.115.0
uvicorn[standard]==0.30.6
pydantic==2.9.2
pydantic-settings==2.5.2
httpx==0.27.2
EOF

cat > app/models.py <<'EOF'
"""The contract: what our service accepts and what it is allowed to return."""
from enum import Enum

from pydantic import BaseModel, ConfigDict, Field


class Category(str, Enum):
    payments = "payments"
    delivery = "delivery"
    restaurant = "restaurant"
    refund = "refund"
    unknown = "unknown"


class Team(str, Enum):
    payments = "payments"
    delivery_ops = "delivery_ops"
    restaurant_support = "restaurant_support"
    refunds = "refunds"
    human_review = "human_review"


class ComplaintIn(BaseModel):
    """What the food app sends us."""
    model_config = ConfigDict(extra="forbid")

    complaint: str = Field(min_length=5, max_length=2000)
    order_id: str | None = None


class TriageResult(BaseModel):
    """What we demand from the AI, and the core of what we promise back."""
    model_config = ConfigDict(extra="forbid")

    category: Category
    priority: int = Field(ge=1, le=4)
    team: Team


class TriageResponse(TriageResult):
    """What the food app receives: the result, plus who decided it."""
    decided_by: str
EOF

cat > scripts/contract_check.py <<'EOF'
"""Gate 1 check: the contract accepts good data and rejects bad data."""
from pydantic import ValidationError

from app.models import ComplaintIn, TriageResult

good_ai = {"category": "payments", "priority": 4, "team": "payments"}
bad_ai = {"category": "payment", "priority": "High", "team": "payment team"}

print("Good AI answer  ->", TriageResult.model_validate(good_ai))

for name, model, data in [
    ("Bad AI answer  ", TriageResult, bad_ai),
    ("Bad request    ", ComplaintIn, {"complaint": "hi", "coupon": "FREE100"}),
]:
    try:
        model.model_validate(data)
    except ValidationError as error:
        print(f"{name} -> REJECTED, {error.error_count()} problems:")
        for problem in error.errors():
            print("      ", problem["loc"], "-", problem["msg"])
EOF

python3.11 -m venv .venv
source .venv/bin/activate
pip install -q --upgrade pip
pip install -q -r requirements.txt
python -m scripts.contract_check
```

Expected output:

```
Good AI answer  -> category=<Category.payments: 'payments'> priority=4 team=<Team.payments: 'payments'>
Bad AI answer   -> REJECTED, 3 problems:
       ('category',) - Input should be 'payments', 'delivery', 'restaurant', 'refund' or 'unknown'
       ('priority',) - Input should be a valid integer, unable to parse string as an integer
       ('team',) - Input should be 'payments', 'delivery_ops', 'restaurant_support', 'refunds' or 'human_review'
Bad request     -> REJECTED, 2 problems:
       ('complaint',) - String should have at least 5 characters
       ('coupon',) - Extra inputs are not permitted
```

### What we did and why

- **Branch `service`**: Parts 07, 08 and 09 all go on this branch and become one pull request.
- **.gitignore first**: before any code, we tell Git what must never be committed: the virtual env, Python cache, log files and `.env` (our secrets file, coming in Part 09).
- **requirements.txt with exact versions**: everyone, and later the Docker image, installs exactly the same libraries.
- **Enums `Category` and `Team`**: the complete list of allowed values. `payment team` is not in the list, so it can never pass.
- **`ComplaintIn`**: what the food app must send. Complaint between 5 and 2000 characters. `extra="forbid"` rejects unknown fields like `coupon`. In the API this becomes HTTP **422**, the caller's mistake.
- **`TriageResult`**: what we demand from the AI. Priority must be a number from 1 to 4 (`ge=1, le=4`). `"High"` is rejected.
- **`TriageResponse`**: what we send back. Same fields plus `decided_by`, so everyone knows if the AI decided or our fallback did.
- **venv**: a private Python for this project, so its libraries do not mix with the system.

**Concept: two checks, one contract**
Check 1 is the request coming in: bad input gets 422, the caller must fix it.
Check 2 is the AI answer: a bad AI answer is not the caller's fault, so we do not fail. We send the complaint to Human Review (Part 08).
Answer to the question above: our service catches it. Never trust the AI blindly.

---

# Part 08: Gate 2, ASYNC

The Support Team Lead says NO: "The AI takes 1 second per complaint. 1000 complaints one by one is 16 minutes. Customers are waiting."

Think first: while we wait 1 second for the AI, what is our service doing? Can it start the next complaint?

### Commands (EC2)

```bash
cd ~/triageops

cat > ai_stub/main.py <<'EOF'
"""A fake AI model. It answers like a real model would, but costs nothing.

STUB_DELAY    seconds to wait before answering (default 1.0), like a real model thinking
STUB_MODE     normal = sensible answers, bad = the wrong answer from our story
STUB_API_KEY  if set, callers must send "Authorization: Bearer <key>", like a real AI provider
"""
import asyncio
import os

from fastapi import FastAPI, Header, HTTPException

app = FastAPI(title="AI Stub")

DELAY = float(os.getenv("STUB_DELAY", "1.0"))
MODE = os.getenv("STUB_MODE", "normal")
API_KEY = os.getenv("STUB_API_KEY", "")


@app.get("/healthz")
async def healthz() -> dict:
    return {"status": "ok", "mode": MODE, "delay": DELAY}


@app.post("/classify")
async def classify(body: dict, authorization: str = Header(default="")) -> dict:
    if API_KEY and authorization != f"Bearer {API_KEY}":
        raise HTTPException(status_code=401, detail="invalid API key")

    await asyncio.sleep(DELAY)

    if MODE == "bad":
        return {"category": "payment", "priority": "High", "team": "payment team"}

    text = str(body.get("complaint", "")).lower()
    if "charged" in text or "payment" in text or "upi" in text:
        return {"category": "payments", "priority": 4, "team": "payments"}
    if "refund" in text:
        return {"category": "refund", "priority": 3, "team": "refunds"}
    if "late" in text or "delivery" in text or "rider" in text:
        return {"category": "delivery", "priority": 3, "team": "delivery_ops"}
    if "cold" in text or "missing" in text or "restaurant" in text:
        return {"category": "restaurant", "priority": 2, "team": "restaurant_support"}
    return {"category": "unknown", "priority": 2, "team": "human_review"}
EOF

cat > scripts/async_demo.py <<'EOF'
"""Gate 2 check: 10 complaints, and an AI that needs 1 second for each one.

python -m scripts.async_demo sync    one by one
python -m scripts.async_demo trap    async def, but with a blocking call inside
python -m scripts.async_demo async   truly async
"""
import asyncio
import sys
import time

import httpx

URL = "http://localhost:9000/classify"
HEADERS = {"Authorization": "Bearer lab-secret-123"}
BODY = {"complaint": "I was charged twice for my biryani"}
N = 10


def one_by_one() -> None:
    with httpx.Client(timeout=10) as client:
        for _ in range(N):
            client.post(URL, json=BODY, headers=HEADERS)


async def blocking_inside_async() -> None:
    async def ask() -> None:
        httpx.post(URL, json=BODY, headers=HEADERS, timeout=10)   # the trap: this call blocks
    await asyncio.gather(*(ask() for _ in range(N)))


async def truly_async() -> None:
    async with httpx.AsyncClient(timeout=10) as client:
        await asyncio.gather(*(client.post(URL, json=BODY, headers=HEADERS) for _ in range(N)))


mode = sys.argv[1] if len(sys.argv) > 1 else "async"
start = time.perf_counter()
if mode == "sync":
    one_by_one()
elif mode == "trap":
    asyncio.run(blocking_inside_async())
else:
    asyncio.run(truly_async())
print(f"{mode:5}: {N} complaints in {time.perf_counter() - start:.1f} seconds")
EOF

cat > app/ai_client.py <<'EOF'
"""Talks to the AI model over HTTP, asynchronously, with a timeout and an API key."""
import httpx

from app.config import settings


async def ask_ai(complaint: str) -> dict:
    headers = {"Authorization": f"Bearer {settings.ai_api_key.get_secret_value()}"}
    async with httpx.AsyncClient(timeout=settings.ai_timeout_seconds) as client:
        response = await client.post(settings.ai_api_url, json={"complaint": complaint}, headers=headers)
        response.raise_for_status()
        return response.json()
EOF

cat > app/triage.py <<'EOF'
"""The AI decides. The contract checks. Anything wrong goes to Human Review."""
import httpx
from pydantic import ValidationError

from app.ai_client import ask_ai
from app.models import Category, Team, TriageResponse, TriageResult

HUMAN_REVIEW = dict(category=Category.unknown, priority=2, team=Team.human_review)


async def ai_triage(complaint: str) -> TriageResponse:
    try:
        raw_answer = await ask_ai(complaint)
    except httpx.TimeoutException:
        return TriageResponse(**HUMAN_REVIEW, decided_by="fallback: AI timed out")
    except httpx.HTTPStatusError as error:
        status = error.response.status_code
        return TriageResponse(**HUMAN_REVIEW, decided_by=f"fallback: AI returned {status}")
    except httpx.HTTPError:
        return TriageResponse(**HUMAN_REVIEW, decided_by="fallback: AI unavailable")

    try:
        result = TriageResult.model_validate(raw_answer)
    except ValidationError:
        return TriageResponse(**HUMAN_REVIEW, decided_by="fallback: invalid AI answer")

    return TriageResponse(**result.model_dump(), decided_by="ai")
EOF

# Run the AI stub in the background and race the three versions
STUB_API_KEY=lab-secret-123 nohup uvicorn ai_stub.main:app --port 9000 > stub.log 2>&1 &
sleep 3
curl -s http://localhost:9000/healthz; echo
python -m scripts.async_demo sync
python -m scripts.async_demo trap
python -m scripts.async_demo async
```

Expected:

```
{"status":"ok","mode":"normal","delay":1.0}
sync : 10 complaints in 10.1 seconds
trap : 10 complaints in 10.1 seconds
async: 10 complaints in 1.1 seconds
```

Leave the stub running for Part 09.

### What we did and why

- **ai_stub/main.py**: our fake AI. `STUB_DELAY` makes it think for 1 second. `STUB_MODE=bad` returns the exact wrong answer from our story. `STUB_API_KEY` makes it reject callers without the right key (HTTP 401), like a real AI provider.
- **async_demo.py**, the aha moment:
  - `sync`: one complaint at a time, 10 x 1 second = 10 seconds.
  - `trap`: the code says `async def`, but inside it calls `httpx.post`, which blocks. Still 10 seconds. Writing `async` is not enough.
  - `async`: `httpx.AsyncClient` with `await`. While one complaint waits for the AI, the next one starts. 10 complaints in about 1 second.
- **app/ai_client.py**: how our service calls the AI. `async with httpx.AsyncClient(timeout=...)` and `await client.post(...)`. The key is sent as a Bearer header. `raise_for_status()` turns a 401 or 500 into an error we can catch.
- **app/triage.py**: every way the AI can fail ends in Human Review, never in a crash:
  - AI too slow: `fallback: AI timed out`
  - AI says 401 or 500: `fallback: AI returned 401`
  - AI not reachable: `fallback: AI unavailable`
  - AI answer breaks the contract (Check 2): `fallback: invalid AI answer`
  - All good: `decided_by: ai`

**Concept: async is about waiting, not about speed**
The AI is not faster. Our service just does not sit idle while it waits. And a timeout means a slow AI can never hold a customer forever.

(ai_client.py imports `app.config`, which we write in Part 09. That is why we do not run the service yet.)

---

# Part 09: Gate 3, SERVICE

The Platform Engineer says NO: "The AI URL is hardcoded. How do I point DEV and PROD to different AIs?"
The Security Head says NO: "Your API key is in the code, and your logs contain the customer's complaint with their phone number."

### Commands (EC2)

```bash
cd ~/triageops

cat > .env.example <<'EOF'
# Copy this file to .env and set real values. Never commit .env.
ENVIRONMENT=dev
LOG_LEVEL=INFO
AI_API_URL=http://localhost:9000/classify
AI_TIMEOUT_SECONDS=3.0
AI_API_KEY=change-me
EOF

cp .env.example .env
sed -i 's/AI_API_KEY=change-me/AI_API_KEY=lab-secret-123/' .env

cat > app/config.py <<'EOF'
"""Configuration: every value comes from the environment. Nothing is hardcoded."""
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    environment: str = "dev"
    log_level: str = "INFO"
    ai_api_url: str = "http://localhost:9000/classify"
    ai_timeout_seconds: float = 3.0
    ai_api_key: SecretStr          # no default: the service refuses to start without it


settings = Settings()
EOF

cat > app/logging_config.py <<'EOF'
"""Structured logging: every log line is one JSON object."""
import json
import logging
import sys
from datetime import datetime, timezone


class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        entry = {
            "time": datetime.now(timezone.utc).isoformat(timespec="milliseconds"),
            "level": record.levelname,
            "event": record.getMessage(),
        }
        entry.update(getattr(record, "fields", {}))
        return json.dumps(entry)


def setup_logging(level: str) -> logging.Logger:
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JsonFormatter())
    logger = logging.getLogger("triage")
    logger.handlers = [handler]
    logger.setLevel(level.upper())
    logger.propagate = False
    return logger
EOF

cat > app/main.py <<'EOF'
"""Complaint Triage Service: contract, async AI call, configuration, secret, JSON logs, health check."""
import time
import uuid

from fastapi import FastAPI, Response

from app.config import settings
from app.logging_config import setup_logging
from app.models import ComplaintIn, TriageResponse
from app.triage import ai_triage

log = setup_logging(settings.log_level)
app = FastAPI(title="Complaint Triage Service", version="1.2.0")

log.info("service_starting", extra={"fields": {
    "version": app.version,
    "environment": settings.environment,
    "ai_api_url": settings.ai_api_url,
    "ai_timeout_seconds": settings.ai_timeout_seconds,
}})


@app.get("/healthz")
async def healthz() -> dict:
    return {"status": "ok", "version": app.version, "environment": settings.environment}


@app.post("/triage", response_model=TriageResponse)
async def triage(body: ComplaintIn, response: Response) -> TriageResponse:
    request_id = uuid.uuid4().hex[:12]
    start = time.perf_counter()

    result = await ai_triage(body.complaint)

    duration_ms = round((time.perf_counter() - start) * 1000)
    response.headers["X-Request-ID"] = request_id
    log.info("triage_completed", extra={"fields": {
        "request_id": request_id,
        "order_id": body.order_id,
        "complaint_length": len(body.complaint),
        "category": result.category.value,
        "priority": result.priority,
        "team": result.team.value,
        "decided_by": result.decided_by,
        "duration_ms": duration_ms,
    }})
    return result
EOF

# Start the service (the AI stub from Part 08 is still running on 9000)
nohup uvicorn app.main:app --host 0.0.0.0 --port 8000 --no-access-log > service.log 2>&1 &
sleep 3
```

Now test it:

```bash
# 1. Health
curl -s http://localhost:8000/healthz; echo

# 2. A real complaint
curl -s -i -X POST http://localhost:8000/triage -H "Content-Type: application/json" \
  -d '{"complaint": "I was charged twice for my biryani, call me on 9876543210", "order_id": "ZMT-1001"}'; echo

# 3. A bad request (Check 1)
curl -s -o /dev/null -w "HTTP %{http_code}\n" -X POST http://localhost:8000/triage \
  -H "Content-Type: application/json" -d '{"complaint": "hi", "coupon": "FREE100"}'

# 4. The logs: JSON, with request_id and duration, and no complaint text, phone number or key
cat service.log
grep -c "9876543210\|lab-secret-123" service.log

# 5. A slow AI: restart the stub with a 5 second delay (our timeout is 3)
pkill -f "uvicorn ai_stub" ; sleep 1
STUB_API_KEY=lab-secret-123 STUB_DELAY=5 nohup uvicorn ai_stub.main:app --port 9000 > stub.log 2>&1 &
sleep 3
curl -s -X POST http://localhost:8000/triage -H "Content-Type: application/json" \
  -d '{"complaint": "I was charged twice for my biryani"}'; echo

# 6. No secret, no start
mv .env /tmp/env.backup
python -c "import app.config" 2>&1 | tail -3
mv /tmp/env.backup .env

# Stop both
pkill -f uvicorn
```

Expected:

1. `{"status":"ok","version":"1.2.0","environment":"dev"}`
2. Headers include `x-request-id: ...` and the body is `{"category":"payments","priority":4,"team":"payments","decided_by":"ai"}`
3. `HTTP 422`
4. Our two JSON lines (`service_starting`, and `triage_completed` with `request_id`, `complaint_length` and `duration_ms`) between uvicorn's own startup lines, and the count is `0`
5. After about 3 seconds: `{"category":"unknown","priority":2,"team":"human_review","decided_by":"fallback: AI timed out"}`
6. `ai_api_key  Field required`

You can also open `http://<PUBLIC_IP>:8000/docs` in your browser (port 8000 is open to your IP) to see the API that FastAPI wrote for us. Do it before `pkill`.

### Commit, pull request, merge

```bash
git status --short                 # .env and .venv must NOT be in this list
git add .
git commit -m "Complaint Triage Service: contract, async AI call, config, secret, JSON logs, healthz"
git push -u origin service         # Username: your GitHub user. Password: paste the token.
```

On GitHub: **Compare and pull request** > **Create pull request** > **Merge pull request** > **Confirm** > **Delete branch**. Then:

```bash
git checkout main && git pull
ls                                  # app  ai_stub  scripts  requirements.txt  must be here
git branch -D service
```

### What we did and why

- **.env.example vs .env**: `.env.example` is committed and shows which settings exist. `.env` holds the real values and is in `.gitignore`, so it never reaches GitHub.
- **app/config.py**: `pydantic-settings` reads every value from environment variables (or from `.env` on our machine). The same code runs in DEV and PROD; only the environment changes. That answers the Platform Engineer.
- **`ai_api_key: SecretStr` with no default**: no key, no start (test 6). Better to fail now than at 2 AM. `SecretStr` prints as `**********`, so the key cannot slip into a log by accident. We read the real value only in one place: `get_secret_value()` in ai_client.py.
- **app/logging_config.py**: every log line is one JSON object. Tools like CloudWatch can search `decided_by` or `duration_ms` directly.
- **app/main.py**:
  - `GET /healthz`: "are you alive?" Kubernetes will ask this every few seconds.
  - `POST /triage`: Check 1 happens automatically because the body is `ComplaintIn`. `response_model=TriageResponse` makes sure we only ever return the contract.
  - `request_id`: a short ID in the log and in the `X-Request-ID` header. A customer support agent can give it to an engineer and the engineer finds the exact log line.
  - We log `complaint_length`, never the complaint text. The phone number in test 2 never reaches the log. That answers the Security Head.
- **`--no-access-log`**: uvicorn's own access log is turned off; our JSON log is the one source of truth.
- **Merge before the next branch**: we learned this live. If you create the next branch before the merge, `main` has no code, the new branch has no code, and Docker fails with `"/requirements.txt": not found`.

---

# Part 10: Gate 4, SHIP part 1: Docker image and Amazon ECR

"It works on my EC2" is not a release. We need one package that runs the same everywhere.

### Commands (EC2)

```bash
cd ~/triageops
git checkout -b docker

cat > .dockerignore <<'EOF'
.venv
__pycache__
*.pyc
*.log
.env
.git
EOF

cat > Dockerfile <<'EOF'
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Run as a normal user, never as root
RUN useradd --create-home --uid 10001 appuser

# Dependencies first: this layer is cached until requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Then the code
COPY app ./app
COPY ai_stub ./ai_stub

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://localhost:8000/healthz', timeout=2).status == 200 else 1)"

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--no-access-log"]
EOF

ls                                         # requirements.txt, app, ai_stub must be here
docker build -t triageops:1.2.0 .
docker images triageops
docker run --rm triageops:1.2.0 ls -la /app        # no .env, no .venv
```

Run both containers together:

```bash
docker network create triage-net 2>/dev/null || true

docker run -d --name ai-stub --network triage-net --no-healthcheck \
  -e STUB_API_KEY=lab-secret-123 \
  triageops:1.2.0 uvicorn ai_stub.main:app --host 0.0.0.0 --port 9000

docker run -d --name triage --network triage-net -p 8000:8000 \
  -e AI_API_URL=http://ai-stub:9000/classify \
  -e AI_API_KEY=lab-secret-123 \
  -e ENVIRONMENT=docker \
  triageops:1.2.0

sleep 5
curl -s http://localhost:8000/healthz; echo
curl -s -X POST http://localhost:8000/triage -H "Content-Type: application/json" \
  -d '{"complaint": "I was charged twice for my biryani"}'; echo
docker logs triage | tail -2
sleep 30; docker ps --format "table {{.Names}}\t{{.Status}}"

# No secret, no start
docker run --rm -e AI_API_URL=http://ai-stub:9000/classify triageops:1.2.0 2>&1 | tail -2
```

Push the image to Amazon ECR:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGISTRY=$ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com
echo "export REGISTRY=$REGISTRY" >> ~/.bashrc

aws ecr create-repository --repository-name triageops \
  --image-scanning-configuration scanOnPush=true --query repository.repositoryUri --output text
aws ecr get-login-password | docker login --username AWS --password-stdin $REGISTRY

docker tag triageops:1.2.0 $REGISTRY/triageops:1.2.0
docker push $REGISTRY/triageops:1.2.0
aws ecr describe-images --repository-name triageops \
  --query 'imageDetails[].[imageTags[0],imageSizeInBytes]' --output table
```

Clean up the local containers, then commit:

```bash
docker rm -f triage ai-stub
git add Dockerfile .dockerignore
git commit -m "Dockerfile: non-root user, health check, no secrets in the image"
git push -u origin docker
```

On GitHub: pull request > merge > delete branch. Then `git checkout main && git pull && git branch -D docker`.

### What we did and why

- **.dockerignore** lists `.env`. The secret cannot even reach the build. `ls -la /app` proves it.
- **python:3.11-slim**: the same Python as our EC2, in a small image.
- **COPY requirements.txt, pip install, then COPY the code**: Docker caches each step. Change one line of Python and the slow pip step is reused. Rebuilds take seconds.
- **USER appuser (uid 10001)**: if someone breaks into the container, they are not root.
- **HEALTHCHECK**: Docker itself knows if the service is healthy (`docker ps` shows `(healthy)` after about 30 seconds). On EKS the liveness and readiness probes do this job instead.
- **One image, two commands**: the same image runs the service (default CMD) and the AI stub (we pass a different command). For the stub we add `--no-healthcheck`, because the HEALTHCHECK checks port 8000 and the stub listens on 9000.
- **`-e` at runtime**: configuration and secret come in from outside. The same image goes to DEV and PROD.
- **triage-net**: containers on the same Docker network find each other by name, so the AI URL is `http://ai-stub:9000`. Kubernetes gives us the same thing with Services.
- **No secret, no start**: `ai_api_key Field required`. On EKS this shows up as `CrashLoopBackOff` (Part 13).
- **ECR with scanOnPush**: our private image registry in AWS. Every pushed image is scanned for known vulnerabilities for free. EKS nodes can pull from it without passwords.
- **Tag 1.2.0, never latest**: you must always know exactly which version is running and be able to roll back to it.
- **REGISTRY in ~/.bashrc**: Parts 11 and 12 need it, even after you reconnect PuTTY.

---

# Part 11: Gate 4, SHIP part 2: deploy to EKS

### Commands (EC2)

```bash
tail -3 ~/triage/eks-create.log      # wait for: EKS cluster "triageops-eks" in "ap-south-1" region is ready
kubectl get nodes                    # 2 nodes, STATUS Ready, VERSION v1.34.x
```

If `kubectl get nodes` says it cannot connect but the log says ready, run
`eksctl utils write-kubeconfig --cluster triageops-eks --region ap-south-1`.

```bash
cd ~/triageops
git checkout -b k8s
mkdir -p k8s

cat > k8s/namespace.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: triageops
EOF

cat > k8s/configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: triage-config
  namespace: triageops
data:
  ENVIRONMENT: "eks"
  LOG_LEVEL: "INFO"
  AI_API_URL: "http://ai-stub:9000/classify"
  AI_TIMEOUT_SECONDS: "3.0"
EOF

cat > k8s/ai-stub.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-stub
  namespace: triageops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-stub
  template:
    metadata:
      labels:
        app: ai-stub
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
      containers:
        - name: ai-stub
          image: REGISTRY_PLACEHOLDER/triageops:1.2.0
          command: ["uvicorn", "ai_stub.main:app", "--host", "0.0.0.0", "--port", "9000"]
          ports:
            - containerPort: 9000
          env:
            - name: STUB_DELAY
              value: "1.0"
            - name: STUB_API_KEY
              valueFrom:
                secretKeyRef:
                  name: triage-secrets
                  key: STUB_API_KEY
          readinessProbe:
            httpGet:
              path: /healthz
              port: 9000
            periodSeconds: 5
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: ai-stub
  namespace: triageops
spec:
  type: ClusterIP
  selector:
    app: ai-stub
  ports:
    - port: 9000
      targetPort: 9000
EOF

cat > k8s/triage.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triage
  namespace: triageops
spec:
  replicas: 2
  minReadySeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: triage
  template:
    metadata:
      labels:
        app: triage
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
      containers:
        - name: triage
          image: REGISTRY_PLACEHOLDER/triageops:1.2.0
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: triage-config
          env:
            - name: AI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: triage-secrets
                  key: AI_API_KEY
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8000
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: triage
  namespace: triageops
spec:
  type: LoadBalancer
  selector:
    app: triage
  ports:
    - port: 80
      targetPort: 8000
EOF

# 1. Namespace and config
kubectl apply -f k8s/namespace.yaml
kubectl config set-context --current --namespace=triageops
kubectl apply -f k8s/configmap.yaml

# 2. Secret: typed in, never written to a file, never committed
read -s -p "AI API key for the lab (type lab-secret-123): " AI_KEY; echo
kubectl create secret generic triage-secrets \
  --from-literal=AI_API_KEY="$AI_KEY" --from-literal=STUB_API_KEY="$AI_KEY"
unset AI_KEY

# 3. The AI stub and the service, with our registry filled in
sed "s|REGISTRY_PLACEHOLDER|$REGISTRY|" k8s/ai-stub.yaml | kubectl apply -f -
sed "s|REGISTRY_PLACEHOLDER|$REGISTRY|" k8s/triage.yaml  | kubectl apply -f -
kubectl rollout status deployment/ai-stub
kubectl rollout status deployment/triage
kubectl get all
```

Wait for the load balancer (2 to 3 minutes), then call it from the internet:

```bash
kubectl get svc triage -w          # wait until EXTERNAL-IP shows a name, then Ctrl+C
LB=$(kubectl get svc triage -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'); echo $LB
echo "export LB=$LB" >> ~/.bashrc

curl -s http://$LB/healthz; echo
curl -s -X POST http://$LB/triage -H "Content-Type: application/json" \
  -d '{"complaint": "My delivery is 40 minutes late and the rider is not picking up", "order_id": "ZMT-2002"}'; echo
kubectl logs deployment/triage --tail=3
```

If `curl` says "Could not resolve host", the load balancer DNS is not ready yet. Wait one more minute.

Expected: `"environment":"eks"`, then `{"category":"delivery","priority":3,"team":"delivery_ops","decided_by":"ai"}`, and JSON log lines.

Commit (the YAML has no account ID and no secret in it):

```bash
grep -c "lab-secret\|dkr.ecr" k8s/*.yaml        # every file must show 0
git add k8s
git commit -m "Kubernetes: namespace, config, secret by reference, ai-stub, triage with probes and LoadBalancer"
git push -u origin k8s
```

On GitHub: pull request > merge > delete branch. Then `git checkout main && git pull && git branch -D k8s`.

### What we did and why

- **Namespace `triageops`**: our own space in the cluster. `set-context --namespace` means we do not type `-n triageops` every time. Deleting the namespace later deletes everything in it.
- **ConfigMap `triage-config`**: the non-secret settings. This is our `.env.example` for Kubernetes. `envFrom` turns every key into an environment variable, and `pydantic-settings` reads them. Not one line of Python changed.
- **Secret `triage-secrets`**: created from the keyboard with `read -s` (nothing on screen, nothing in shell history, nothing in Git). The YAML only references it by name with `secretKeyRef`.
- **ai-stub Deployment**: the same image with a different `command`. **Service type ClusterIP**: reachable only inside the cluster, at `http://ai-stub:9000`. That is the URL in our ConfigMap.
- **triage Deployment**:
  - `replicas: 2`: two pods, on two nodes. One can die and customers do not notice.
  - `readinessProbe`: "can you take traffic?" Until it passes, the Service sends nothing to the pod.
  - `livenessProbe`: "are you still alive?" If it fails 3 times, Kubernetes restarts the container.
  - `runAsNonRoot` and `runAsUser: 10001`: Kubernetes double checks that we are not root.
  - `resources`: a small reservation so the scheduler places the pods sensibly, and a memory limit so one pod cannot eat a node.
  - `RollingUpdate` with `maxUnavailable: 0`: during an update, old pods keep serving until new ones are ready. We see this in Part 12 and Part 13.
- **Service type LoadBalancer**: AWS creates a public load balancer in our subnets tagged `kubernetes.io/role/elb=1` (Part 01) and sends port 80 to port 8000 on the pods.
- **REGISTRY_PLACEHOLDER and sed**: the account ID stays out of Git. In a real company a CI pipeline fills it in.

**Concept: liveness vs readiness**
Readiness: "not ready yet, do not send customers to me." Liveness: "I am stuck, restart me." Same `/healthz` endpoint here, two different decisions.

---

# Part 12: The complaint web page, version 1.3.0, with a rolling update

Our support agents do not use curl. They want a page: type the complaint, see the team.
We ship it as a new version, with zero downtime.

### Commands (EC2)

```bash
cd ~/triageops
git checkout -b web-page
mkdir -p app/static

cat > app/static/index.html <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Complaint Triage</title>
<style>
  body { font-family: Arial, sans-serif; background: #f6f6f6; margin: 0; }
  header { background: #cb202d; color: #fff; padding: 16px 24px; font-size: 20px; font-weight: bold; }
  main { max-width: 640px; margin: 24px auto; background: #fff; padding: 24px; border-radius: 8px; }
  label { display: block; margin: 12px 0 4px; font-weight: bold; }
  textarea, input { width: 100%; box-sizing: border-box; padding: 8px; font-size: 15px; }
  textarea { height: 110px; }
  button { margin-top: 16px; padding: 10px 20px; font-size: 15px; background: #cb202d; color: #fff;
           border: 0; border-radius: 4px; cursor: pointer; }
  #result { margin-top: 20px; padding: 12px; border-radius: 6px; display: none; }
  .ok { background: #e8f5e9; }
  .review { background: #fff8e1; }
  .error { background: #ffebee; }
  table { border-collapse: collapse; }
  td { padding: 4px 12px 4px 0; }
  footer { text-align: center; color: #888; font-size: 13px; margin: 16px; }
</style>
</head>
<body>
<header>Complaint Triage</header>
<main>
  <label for="complaint">Customer complaint</label>
  <textarea id="complaint" placeholder="I was charged twice for my biryani"></textarea>
  <label for="order">Order ID (optional)</label>
  <input id="order" placeholder="ZMT-1001">
  <button onclick="sendComplaint()">Triage</button>
  <div id="result"></div>
</main>
<footer id="version">loading version...</footer>
<script>
async function sendComplaint() {
  const box = document.getElementById("result");
  const body = { complaint: document.getElementById("complaint").value };
  const order = document.getElementById("order").value.trim();
  if (order) { body.order_id = order; }
  box.style.display = "block";
  box.className = "";
  box.textContent = "Asking the AI...";
  try {
    const res = await fetch("/triage", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body)
    });
    const data = await res.json();
    if (!res.ok) {
      box.className = "error";
      box.textContent = "Rejected (HTTP " + res.status + "): " +
        data.detail.map(d => d.loc.join(".") + " - " + d.msg).join("; ");
      return;
    }
    box.className = data.decided_by === "ai" ? "ok" : "review";
    box.innerHTML = "<table>" +
      "<tr><td>Category</td><td><b>" + data.category + "</b></td></tr>" +
      "<tr><td>Priority</td><td><b>" + data.priority + "</b></td></tr>" +
      "<tr><td>Team</td><td><b>" + data.team + "</b></td></tr>" +
      "<tr><td>Decided by</td><td>" + data.decided_by + "</td></tr>" +
      "<tr><td>Request ID</td><td>" + res.headers.get("X-Request-ID") + "</td></tr>" +
      "</table>";
  } catch (err) {
    box.className = "error";
    box.textContent = "Service not reachable: " + err;
  }
}
fetch("/healthz").then(r => r.json()).then(h => {
  document.getElementById("version").textContent = "version " + h.version + " | " + h.environment;
});
</script>
</body>
</html>
EOF

cat > app/web.py <<'EOF'
"""The complaint web page for support agents."""
from pathlib import Path

from fastapi import APIRouter
from fastapi.responses import FileResponse

router = APIRouter()
PAGE = Path(__file__).parent / "static" / "index.html"


@router.get("/", include_in_schema=False)
async def home() -> FileResponse:
    return FileResponse(PAGE)
EOF

# Plug the page into the service and bump the version
sed -i '/^from app.triage import ai_triage$/a from app.web import router as web_router' app/main.py
sed -i 's/version="1.2.0"/version="1.3.0"/' app/main.py
echo 'app.include_router(web_router)' >> app/main.py
grep -n "web_router\|version=" app/main.py

# Build and push 1.3.0
docker build -t $REGISTRY/triageops:1.3.0 .
aws ecr get-login-password | docker login --username AWS --password-stdin $REGISTRY
docker push $REGISTRY/triageops:1.3.0
```

Now open a **second PuTTY window** (same saved session) and leave this running. It calls the service every second:

```bash
while true; do curl -s -m 2 http://$LB/healthz || echo "FAILED"; echo; sleep 1; done
```

Back in the first window, roll out 1.3.0:

```bash
cd ~/triageops
sed -i 's|triageops:1.2.0|triageops:1.3.0|' k8s/triage.yaml
sed "s|REGISTRY_PLACEHOLDER|$REGISTRY|" k8s/triage.yaml | kubectl apply -f -
kubectl rollout status deployment/triage
kubectl rollout history deployment/triage
echo "Open in your browser: http://$LB/"
```

Watch the second window: lines change from `"version":"1.2.0"` to `"version":"1.3.0"`, and there is no `FAILED` line.
Open `http://<LB>/` in your browser and send a few complaints, including one that is too short.

Roll back and forward again:

```bash
kubectl rollout undo deployment/triage
kubectl rollout status deployment/triage        # second window shows 1.2.0 again
sed "s|REGISTRY_PLACEHOLDER|$REGISTRY|" k8s/triage.yaml | kubectl apply -f -
kubectl rollout status deployment/triage        # back to 1.3.0
```

Commit:

```bash
git add app k8s/triage.yaml
git commit -m "Version 1.3.0: complaint web page"
git push -u origin web-page
```

On GitHub: pull request > merge > delete branch. Then `git checkout main && git pull && git branch -D web-page`.
Stop the loop in the second window with Ctrl+C.

### What we did and why

- **index.html**: a plain page. It sends the same JSON to the same `/triage` API. The browser is just another client of our contract. Green for `decided_by: ai`, yellow for Human Review, red for a 422 with the reason.
- **app/web.py**: a small FastAPI router that serves the page at `/`. `include_in_schema=False` keeps it out of the API docs.
- **Three edits to main.py**: import the router, bump the version to 1.3.0, include the router. The footer of the page shows the version from `/healthz`, so we can see which version served us.
- **New tag 1.3.0**: we never overwrite 1.2.0. That is what makes rollback possible.
- **The rolling update**: Kubernetes starts one new pod (`maxSurge: 1`), waits until its readiness probe passes, then removes one old pod, and repeats. `maxUnavailable: 0` means capacity never drops below 2. The loop in the second window is the proof: no `FAILED`.
- **rollout history / undo**: every change to the Deployment is a revision. `undo` goes back to the previous one in seconds.
- **We changed the YAML in Git**, not only the cluster. Git is the record of what should be running.

---

# Part 13: Break it and diagnose

Production is not "it works". Production is "when it breaks, we know why in 2 minutes".
Run each break, look at the symptom, find the cause, fix it, and check the fix.

### Commands (EC2)

Helper for this part:

```bash
ask() { curl -s -X POST http://$LB/triage -H "Content-Type: application/json" \
  -d '{"complaint": "I was charged twice for my biryani"}'; echo; }
ask                                   # baseline: decided_by ai
```

**Break 1: the secret is missing from a new version**

```bash
kubectl set env deployment/triage AI_API_KEY-
kubectl get pods -w                   # a new pod goes Error, then CrashLoopBackOff. Ctrl+C
kubectl logs $(kubectl get pods -l app=triage --sort-by=.metadata.creationTimestamp -o name | tail -1) | tail -3
ask                                   # still works!
kubectl rollout undo deployment/triage && kubectl rollout status deployment/triage
```

**Break 2: wrong API key between our service and the AI**

```bash
kubectl patch secret triage-secrets -p '{"stringData":{"STUB_API_KEY":"wrong-key"}}'
kubectl rollout restart deployment/ai-stub && kubectl rollout status deployment/ai-stub
ask                                   # fallback: AI returned 401
read -s -p "Correct key: " AI_KEY; echo
kubectl patch secret triage-secrets -p "{\"stringData\":{\"STUB_API_KEY\":\"$AI_KEY\"}}"; unset AI_KEY
kubectl rollout restart deployment/ai-stub && kubectl rollout status deployment/ai-stub
ask
```

**Break 3: the AI answers with garbage**

```bash
kubectl set env deployment/ai-stub STUB_MODE=bad && kubectl rollout status deployment/ai-stub
ask                                   # fallback: invalid AI answer
kubectl set env deployment/ai-stub STUB_MODE=normal && kubectl rollout status deployment/ai-stub
```

**Break 4: the AI is slow**

```bash
kubectl set env deployment/ai-stub STUB_DELAY=5 && kubectl rollout status deployment/ai-stub
time ask                              # about 3 seconds, fallback: AI timed out
kubectl set env deployment/ai-stub STUB_DELAY=1.0 && kubectl rollout status deployment/ai-stub
```

**Break 5: the AI is down**

```bash
kubectl scale deployment/ai-stub --replicas=0
ask                                   # fallback: AI unavailable
kubectl scale deployment/ai-stub --replicas=1 && kubectl rollout status deployment/ai-stub
```

**Break 6: a bad health check in a new version**

```bash
kubectl patch deployment triage --type=json \
  -p '[{"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/port","value":9999}]'
kubectl get pods                      # new pod Running but 0/1 READY
kubectl describe pod $(kubectl get pods -l app=triage --sort-by=.metadata.creationTimestamp -o name | tail -1) | grep -i "readiness"
ask                                   # still works
kubectl rollout undo deployment/triage && kubectl rollout status deployment/triage
```

Finally, read the story of all of it from the logs:

```bash
kubectl logs deployment/triage --tail=20 | grep triage_completed
ask
```

### What we did and why

| Break | Symptom | Where you find the cause | Fix |
|-------|---------|--------------------------|-----|
| 1. Secret missing | New pod `CrashLoopBackOff` | `kubectl logs`: `ai_api_key Field required` | `rollout undo` |
| 2. Wrong key | `fallback: AI returned 401` | our JSON log, `decided_by` | correct the Secret, restart |
| 3. Garbage answer | `fallback: invalid AI answer` | our JSON log | fix the AI side |
| 4. Slow AI | answer after 3 s, `AI timed out` | `duration_ms` in the log | AI side, or tune `AI_TIMEOUT_SECONDS` |
| 5. AI down | `fallback: AI unavailable` | `kubectl get pods`, endpoints | scale back up |
| 6. Bad probe | new pod `0/1 READY`, rollout stuck | `kubectl describe pod`: Readiness probe failed | `rollout undo` |

- **Break 1 and 6, the big aha**: the broken version never received a single customer. `maxUnavailable: 0` plus the readiness probe kept the old pods serving. A bad deploy became a stuck deploy, not an outage.
- **Break 2 to 5**: the service never crashed. Every AI failure became a complaint in Human Review, and the log tells us exactly which failure it was. This is what the 4 gates bought us.
- **Break 2 also teaches**: changing a Secret does not change running pods. Environment variables are read at start, so we restart the Deployment.
- `kubectl logs`, `kubectl describe`, `kubectl get pods`, `rollout status/undo`: these four commands solve most problems you will see in a real job.

---

# Part 14: Clean up everything

EKS, the load balancer and the nodes cost money every hour. Clean up the same day.

### Commands (EC2)

```bash
kubectl delete namespace triageops           # deletes the load balancer too. Wait for it to finish
aws elb describe-load-balancers --query 'LoadBalancerDescriptions[].LoadBalancerName' --output text   # must be empty
eksctl delete cluster --name triageops-eks --region ap-south-1 --wait     # about 10 to 15 minutes
aws ecr delete-repository --repository-name triageops --force
```

### Commands (laptop, Git Bash)

```bash
source ~/triage-lab.env

# EC2 workstation, key pair, security group
aws ec2 terminate-instances --instance-ids $INSTANCE_ID --query 'TerminatingInstances[0].CurrentState.Name'
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID
aws ec2 delete-key-pair --key-name triage-key && rm -f ~/keys/triage-key.ppk
[ "$(aws ec2 describe-security-groups --group-ids $SG_ID --query 'SecurityGroups[0].GroupName' --output text)" != "default" ] \
  && aws ec2 delete-security-group --group-id $SG_ID     # the default group goes away with the VPC

# IAM
aws iam remove-role-from-instance-profile --instance-profile-name triage-workstation-role --role-name triage-workstation-role
aws iam delete-instance-profile --instance-profile-name triage-workstation-role
aws iam detach-role-policy --role-name triage-workstation-role --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam delete-role --role-name triage-workstation-role

# Network
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-subnet --subnet-id $SUBNET_A
aws ec2 delete-subnet --subnet-id $SUBNET_B
aws ec2 delete-route-table --route-table-id $RT_ID
aws ec2 delete-vpc --vpc-id $VPC_ID && echo "VPC deleted"
rm ~/triage-lab.env
```

### Steps (browser)

- GitHub: Settings > Developer settings > Personal access tokens > Tokens (classic) > delete `triageops-lab`.
- Keep the `triageops` repo. It is your proof of work for interviews.
- AWS console: CloudFormation (no `eksctl-` stacks), EC2 > Load Balancers (empty), EC2 > Instances (all terminated).

### What we did and why

- **Namespace first**: deleting the namespace deletes the Service of type LoadBalancer, and Kubernetes asks AWS to delete the load balancer. If we delete the cluster first, the load balancer can be left behind, keep costing money, and block the VPC delete.
- **eksctl delete cluster --wait**: removes the node group and control plane stacks in the right order.
- **Order on the laptop**: an instance uses the key, the security group and the role, so it goes first. The VPC goes last because everything lives inside it.
- If `delete-security-group` or `delete-vpc` says `DependencyViolation`, something is still inside the VPC. See Part 15.

---

# Part 15: Troubleshooting (every error we hit, and the fix)

| Error | Cause | Fix |
|-------|-------|-----|
| AMI lookup or `/dev/xvda` gives a strange `C:/Program Files/Git/...` path | Git Bash path conversion | `export MSYS_NO_PATHCONV=1` (Part 00) |
| `Invalid IAM Instance Profile name` on run-instances | IAM not visible to EC2 yet | wait 10 seconds and run again |
| On EC2: `Unable to locate credentials`, or 404 from 169.254.169.254 | role not delivered to the instance | Console: EC2 > Actions > Security > Modify IAM role: remove, Save, then select `triage-workstation-role`, Save. Prevent it by attaching the role at launch |
| PuTTY: `No supported authentication methods` | wrong key file or wrong user | user is `ec2-user`, key is `triage-key.ppk` in Connection > SSH > Auth > Credentials |
| `save: command not found` | a new terminal does not have the `save` function | `source ~/triage-lab.env`, then paste the `save() { ... }` lines from Part 00 again |
| PuTTY times out | your IP changed (new Wi-Fi), or the instance has the VPC `default` security group | check the group name (Part 02 check); add your current IP on port 22 |
| Console Connect: `Failed to connect to your instance. Error establishing SSH connection` | the security group allows 22 only from your IP; the console connects from AWS IPs | add the `ec2-instance-connect` prefix list rule (Part 02, optional lines) |
| `argument --group-id: expected one argument` (or any "expected one argument") | the variable is empty, for example `SG_ID` not in `~/triage-lab.env` | `echo $SG_ID`; look the ID up again and `save` it |
| Public IP changed | the instance was stopped and started | `describe-instances` for the new `PublicIpAddress`, update PuTTY |
| `permission denied ... docker.sock` | group change needs a new login | `exit`, open PuTTY again |
| kubectl warns about version skew, or odd errors | kubectl newer than the cluster | install kubectl from `stable-1.34` (Part 04) |
| eksctl `InvalidSubnetID.NotFound` | old subnet IDs from a deleted VPC | look up subnets by Name tag (Part 05) |
| eksctl `AlreadyExistsException` | leftover CloudFormation stack | Part 05 "Leftover stacks", or use a new cluster name |
| eksctl stopped half way | process killed | Part 05 "If eksctl was stopped in the middle" |
| `git push` 403 | fine-grained token without permission, or wrong username | classic token with `repo` scope; username must be your GitHub user |
| `remote: Invalid username or password` | pasted the GitHub password | paste the token, not the password |
| `index file corrupt` (Git on a Windows shared drive) | broken index | `rm .git/index && git reset` |
| `[Errno 98] address already in use` on 8000 or 9000 | an old uvicorn still running | `ss -ltnp \| grep -E '8000\|9000'` then `pkill -f uvicorn` |
| `ai_api_key Field required` | no `.env`, or no `AI_API_KEY` env var | create `.env` from `.env.example`; in Docker pass `-e AI_API_KEY=...` |
| `decided_by: fallback: AI unavailable` locally | the stub is not running | start the AI stub on 9000 (Part 08) |
| `docker build`: `"/requirements.txt": not found` | branch created before the previous PR was merged; no code in the folder | merge the PR, `git checkout main && git pull`, then `git checkout <branch> && git merge main`; always `ls` before `docker build` |
| `docker push`: `no basic auth credentials` | ECR login expired (12 hours) | run the `aws ecr get-login-password ... docker login` line again |
| Pod `ImagePullBackOff` | wrong image name or tag, or REGISTRY empty | `echo $REGISTRY`; `kubectl describe pod`; check the tag exists in ECR |
| Pod `CreateContainerConfigError` | the Secret or ConfigMap does not exist | `kubectl get secret,cm`; create the Secret (Part 11) |
| On EKS every answer is `fallback: AI unavailable` | the triage pods cannot reach the ai-stub Service | `kubectl get pods -o wide` and `kubectl get endpoints ai-stub` (must show a pod IP); test from each triage pod: `kubectl exec <pod> -- python -c "import urllib.request; print(urllib.request.urlopen('http://ai-stub:9000/healthz', timeout=3).read())"`. If only the pod on the other node fails, check that both nodes have the EKS cluster security group |
| Pod `CrashLoopBackOff` | the app exits at start | `kubectl logs <pod>` (add `--previous` if it restarted) |
| Service EXTERNAL-IP stays `<pending>` | subnets missing the `kubernetes.io/role/elb=1` tag | tag both subnets (Part 01) and recreate the Service |
| `curl: Could not resolve host` for the LB | DNS for the new load balancer not ready | wait 1 to 3 minutes |
| `DependencyViolation` when deleting the VPC | a load balancer, ENI or eksctl security group is left | delete leftover load balancers and `eksctl-` stacks, then retry |

Never do these, in any lab or job:

- Never paste a token, key or `.env` content into a chat, a ticket or a screen share. If it happens, revoke it right away.
- Never commit `.env`, `.ppk` files or real keys. `git status` before every `git add .`.
- Never log the complaint text. Log its length and the request ID.

---

## What you built

- A FastAPI service with a strict Pydantic contract (Check 1 for requests, Check 2 for AI answers)
- Async AI calls with a timeout and a Human Review fallback for every failure
- Configuration from the environment, a secret that never touches code, image or logs, JSON logs with request IDs, and a health endpoint
- A small, non-root Docker image in Amazon ECR, scanned on push
- A Kubernetes deployment on EKS with 2 replicas, probes, a ConfigMap, a Secret and a public load balancer
- A zero-downtime rolling update to 1.3.0 with a web page, a rollback, and six real failures diagnosed

AI writes code fast. Production engineering decides whether it ships. Today, you did the production engineering.

---

## Keep learning with me

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/

Rahul Chaubey
