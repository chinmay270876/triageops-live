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

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 04](part-04-install-tools.md) | [Lab map](README.md) | Next: [Part 06: GitHub repo, token and clone](part-06-github.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
