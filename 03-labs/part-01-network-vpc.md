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

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 00](part-00-before-you-start.md) | [Lab map](README.md) | Next: [Part 02: The workstation (IAM role, key pair, security group, EC2)](part-02-workstation-ec2.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
