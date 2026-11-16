# Python and Production Engineering: TriageOps

Take a Python API from "it works on my laptop" to a production service running on Amazon EKS.

Friday night, rain, an IPL match, and thousands of complaints on a Zomato style food app: "charged twice for my biryani".
You build the **Complaint Triage Service**: an AI decides the category, priority and team for each complaint, and your service makes sure that decision is safe to use.
The service must pass four gates before it ships:

| Gate | The question it answers | What you build |
|------|-------------------------|----------------|
| 1. CONTRACT | What exactly can this service return? | Pydantic models, 422 for bad requests, Human Review for bad AI answers |
| 2. ASYNC | Thousands of complaints, and we wait on another system? | httpx.AsyncClient, timeouts, fallbacks |
| 3. SERVICE | Where are the secrets? Why is customer data in the logs? | config from the environment, SecretStr, JSON logs, /healthz |
| 4. SHIP | How do we run this? | Docker, Amazon ECR, Amazon EKS, rolling update |

AI writes code fast. Production engineering decides whether it ships.

## How to use this repository

1. **Theory**: read [01-theory](01-theory/README.md) in order. Each page has a diagram, the concept, where it shows up in the lab, and interview questions.
2. **Diagrams**: open [02-diagrams](02-diagrams/README.md) in draw.io or Excalidraw to see the whole picture.
3. **Lab**: build everything yourself, step by step, with [03-labs](03-labs/README.md). Commands first, then what we did and why.
4. **Code**: compare with the finished service in [04-code](04-code/README.md) if something does not work.
5. **Interview**: practise with [05-interview-questions](05-interview-questions/README.md).

## Repository structure

```
.
|-- README.md                       you are here
|-- 01-theory/                      10 concept pages, each with its Excalidraw picture
|-- 02-diagrams/
|   |-- drawio/
|   |   |-- python-production-engineering.drawio   theory pages 01 to 10
|   |   `-- triageops-lab.drawio                   lab sheets L01 to L16
|   `-- excalidraw/                 .svg pictures and .excalidraw sources
|-- 03-labs/                        Part 00 to Part 15, one file each, plus the full lab in one file
|-- 04-code/triageops/              the finished service (version 1.3.0), Dockerfile, Kubernetes files
`-- 05-interview-questions/         questions by topic, by lab part, and scenarios
```

## What you need

- An AWS account (region ap-south-1, Mumbai). The lab creates paid resources: run the clean up in Part 14 the same day.
- AWS CLI on your laptop, Git Bash (Windows) or any bash terminal, PuTTY (Windows) or ssh.
- A GitHub account.
- No Python, Docker or Kubernetes on your laptop. All the work happens on an EC2 workstation in AWS.

## The system you build

```
Your laptop --SSH--> EC2 workstation --git push--> GitHub
                          |---------docker push--> Amazon ECR
                          `---------kubectl------> Amazon EKS
                                                     Load Balancer -> triage pods (x2) -> ai-stub pod
                                                     ConfigMap + Secret, probes on /healthz
```

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
