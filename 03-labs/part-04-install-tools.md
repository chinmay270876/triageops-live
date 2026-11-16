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

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 03](part-03-connect-with-putty.md) | [Lab map](README.md) | Next: [Part 05: Start the EKS cluster (runs while we code)](part-05-eks-cluster.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
