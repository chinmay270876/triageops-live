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

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 02](part-02-workstation-ec2.md) | [Lab map](README.md) | Next: [Part 04: Install tools on EC2](part-04-install-tools.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
