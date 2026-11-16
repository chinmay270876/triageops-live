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

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

[Lab map](README.md) | Next: [Part 01: Network (VPC)](part-01-network-vpc.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
