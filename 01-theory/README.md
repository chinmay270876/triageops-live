# Theory

Read these in order. Each page starts with the Zomato story, explains the concept, shows where you use it in the lab, and ends with interview questions.
The pictures come from `02-diagrams/excalidraw`. The same topics are on the pages of `02-diagrams/drawio/python-production-engineering.drawio`.

| # | Page | The one idea to remember |
|---|------|--------------------------|
| 0 | [Before the Lab: What You Need](00-prerequisites-for-the-lab.md) | All the work happens on an EC2 workstation in AWS |
| 1 | [From Python Code to a Production Service](01-python-code-to-production-service.md) | Writing the code is only the beginning. Four gates decide if it ships |
| 2 | [API and HTTP: How Two Systems Talk](02-api-and-http.md) | An API is the boundary. HTTP is how systems talk across it |
| 3 | [FastAPI and Uvicorn: Turning Python Code into a Service](03-fastapi-and-uvicorn.md) | FastAPI defines the API. Uvicorn is the server that runs it |
| 4 | [Pydantic: The Contract](04-pydantic-the-contract.md) | Never trust data that arrives. Check the request and check the AI answer |
| 5 | [Async: What Does Our Service Do While It Waits?](05-async.md) | The AI does not get faster. We stop sitting idle while we wait |
| 6 | [Configuration and Secrets: Outside the Code](06-07-configuration-and-secrets.md) | Build once, configure at runtime. Secrets arrive at runtime |
| 7 | [Structured Logging and Health Checks](08-09-logging-and-health-checks.md) | Enough information to diagnose, nothing sensitive. Running is not healthy |
| 8 | [Liveness vs Readiness](09b-liveness-vs-readiness.md) | Liveness restarts. Readiness stops traffic |
| 9 | [What We Are Going to Build](10-what-we-are-going-to-build.md) | The full system on Amazon EKS |

Next: [the lab](../03-labs/README.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
