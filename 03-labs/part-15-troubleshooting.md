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

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 14](part-14-clean-up.md) | [Lab map](README.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
