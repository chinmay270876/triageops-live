# Interview Questions

Answer from what you built in the lab, not from memorised definitions. A good answer names the problem, what you did, and how you proved it worked.

## The service

1. What is the difference between a script and a service?
2. What happens when the food app calls an API? Walk through the request and the response.
3. What is the difference between GET and POST? Why is /triage a POST?
4. What is the difference between 4xx and 5xx status codes? When does an API return 422?
5. What is the difference between FastAPI and Uvicorn?

## Contract

6. What problem does Pydantic solve?
7. Pydantic checks data in two places in our service. Where, and what happens when each check fails?
8. Why should a model's output be treated as untrusted input?

## Async

9. Why does async def alone not make code asynchronous?
10. requests versus httpx.AsyncClient: what is the difference?
11. Why does every external call need a timeout?
12. Why can retries be dangerous?

## Service

13. Configuration versus secrets: what is the difference?
14. Why should a secret never be inside a Docker image?
15. What is structured logging, and what must never appear in logs?
16. Why do we need /healthz? Running versus healthy?
17. Liveness probe versus readiness probe?

## Ship

18. Walk me through your deployment flow, from code to a running service.
19. Why attach an IAM role to EC2 instead of storing access keys on it?
20. A pod starts and immediately crashes. How do you debug it?

## From the lab, part by part

| Lab sheet | Question |
|-----------|----------|
| L01 What We Are Going to Build | Why do we build on an EC2 workstation and not on our laptop? |
| L02 Lab Roadmap | Why do we start the EKS cluster in Part 05, before writing a single line of Python? |
| L03 Part 01 Network VPC | Why two subnets in two Availability Zones? What breaks if the elb tag is missing? |
| L04 Part 02-03 Workstation | Why not copy your access keys to the EC2? What happened when we attached the role after launch? |
| L05 Part 04-05 Tools and EKS | kubectl 1.37 and cluster 1.34: what is the rule? What does nohup protect you from? |
| L06 Part 06 GitHub Workflow | Why one branch per part? Why run git status before git add . ? |
| L07 Part 07 Gate 1 CONTRACT | Why does extra=forbid matter? Why not return the AI answer as it is? |
| L08 Part 08 Gate 2 ASYNC | Your code says async def but it is still slow. Why? What happens to a complaint when the AI times out? |
| L09 Part 09 Gate 3 SERVICE | Why SecretStr? Why log complaint_length and not the complaint? |
| L10 How the Files Connect | The AI is down. Which file decides what the customer gets back? |
| L11 Part 10 Docker and ECR | Why copy requirements.txt before the code? How do you prove .env is not inside the image? |
| L12 Part 11 Kubernetes Objects | Why is ai-stub ClusterIP and triage LoadBalancer? What happens if the Secret does not exist? |
| L13 One Complaint on EKS | A complaint returns decided_by: fallback: AI returned 401. Where do you look first? |
| L14 Part 12 Rolling Update | Why must 1.3.0 be a new tag and not overwrite 1.2.0? How do you prove there was no downtime? |
| L15 Part 13 Break It | A pod is in CrashLoopBackOff. What are your first two commands? |
| L16 Part 14 Clean Up | Why delete the namespace before the cluster? Why does the VPC go last? |

## Scenario questions

1. Users report that every complaint goes to Human Review with `fallback: AI timed out`. What do you check, in order?
2. You deployed version 1.3.0 and the rollout is stuck with one pod at 0/1 READY. Are customers affected? What do you do?
3. Someone committed an API key to GitHub an hour ago. What are your next three actions?
4. The load balancer EXTERNAL-IP stays `<pending>`. What is the most likely cause in this setup?
5. You need different AI URLs for DEV and PROD. What changes in the code? (Trick question.)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
