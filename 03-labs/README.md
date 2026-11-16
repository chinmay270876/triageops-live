# Lab: From VS Code to EKS, End to End

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

One file per part. Start with Part 00 and go in order. Prefer one file? Use [full-lab-single-file.md](full-lab-single-file.md), the same content in one page.
The matching drawing for each part is in `02-diagrams/drawio/triageops-lab.drawio` (sheets L01 to L16).

| Part | Where | What |
|------|-------|------|
| [00](part-00-before-you-start.md) | Laptop | Before you start |
| [01](part-01-network-vpc.md) | Laptop (Git Bash) | Network: VPC, subnets, internet gateway |
| [02](part-02-workstation-ec2.md) | Laptop (Git Bash) | Workstation: IAM role, key pair, security group, EC2 |
| [03](part-03-connect-with-putty.md) | Laptop (PuTTY) | Connect to EC2 and check the IAM role |
| [04](part-04-install-tools.md) | EC2 | Install tools |
| [05](part-05-eks-cluster.md) | EC2 | Start the EKS cluster (runs in the background, about 20 minutes) |
| [06](part-06-github.md) | EC2 + GitHub | GitHub repo, token, clone |
| [07](part-07-gate-1-contract.md) | EC2 | Gate 1: CONTRACT |
| [08](part-08-gate-2-async.md) | EC2 | Gate 2: ASYNC |
| [09](part-09-gate-3-service.md) | EC2 | Gate 3: SERVICE, then the first pull request |
| [10](part-10-docker-and-ecr.md) | EC2 | Gate 4: SHIP part 1: Docker image and Amazon ECR |
| [11](part-11-deploy-to-eks.md) | EC2 | Gate 4: SHIP part 2: deploy to EKS |
| [12](part-12-web-page-rolling-update.md) | EC2 | Complaint web page, version 1.3.0, rolling update |
| [13](part-13-break-it-and-diagnose.md) | EC2 | Break it and diagnose |
| [14](part-14-clean-up.md) | EC2 + Laptop | Clean up everything (do not skip, EKS costs money) |
| [15](part-15-troubleshooting.md) | - | Troubleshooting: every error we hit and the fix |

Golden rule for Git in this lab: **one branch per part, then pull request, merge, and come back to main before the next part.**

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

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/

Rahul Chaubey
