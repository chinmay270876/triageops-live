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

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 13](part-13-break-it-and-diagnose.md) | [Lab map](README.md) | Next: [Part 15: Troubleshooting (every error we hit, and the fix)](part-15-troubleshooting.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
