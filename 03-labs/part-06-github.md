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

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 05](part-05-eks-cluster.md) | [Lab map](README.md) | Next: [Part 07: Gate 1, CONTRACT](part-07-gate-1-contract.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
