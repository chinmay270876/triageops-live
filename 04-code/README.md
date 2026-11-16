# Code: the finished service (version 1.3.0)

This is what your own repository looks like at the end of Part 12. Build it yourself in the lab first.
Use this folder to compare when something does not work, or to check a file that did not paste fully.

```
triageops/
  .gitignore            what Git must never commit (.env, .venv, logs)
  .dockerignore         what Docker must never copy into the image (.env, .venv, .git)
  .env.example          the settings that exist. Copy to .env and add the real key. Never commit .env
  requirements.txt      exact library versions
  Dockerfile            python:3.11-slim, non-root user, health check
  app/
    models.py           Gate 1 CONTRACT: ComplaintIn, TriageResult, TriageResponse
    ai_client.py        Gate 2 ASYNC: calls the AI with httpx.AsyncClient, timeout and API key
    triage.py           Gate 2 ASYNC: the decision, and every fallback to Human Review
    config.py           Gate 3 SERVICE: settings from the environment, SecretStr key
    logging_config.py   Gate 3 SERVICE: one JSON object per log line
    main.py             the front door: POST /triage, GET /healthz, X-Request-ID
    web.py              version 1.3.0: serves the complaint web page at /
    static/index.html   the complaint web page
  ai_stub/
    main.py             a fake AI model: STUB_DELAY, STUB_MODE, STUB_API_KEY
  scripts/
    contract_check.py   Part 07: proves the contract
    async_demo.py       Part 08: sync vs trap vs async race
  k8s/
    namespace.yaml      namespace triageops
    configmap.yaml      ENVIRONMENT, LOG_LEVEL, AI_API_URL, AI_TIMEOUT_SECONDS
    ai-stub.yaml        Deployment + ClusterIP Service for the fake AI
    triage.yaml         Deployment (2 replicas, probes, rolling update) + LoadBalancer Service
  eks/
    cluster.yaml        the EKS cluster as code (Part 05 fills in the subnet IDs)
```

## Concept: how one complaint flows through the files

    curl or web page -> main.py (check 1 via models.py) -> triage.py -> ai_client.py (url, timeout, key from config.py)
      -> ai_stub -> back in triage.py (check 2 via models.py, or Human Review) -> main.py logs and answers

## Run it on your workstation

```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # then set AI_API_KEY in .env
STUB_API_KEY=<same key> uvicorn ai_stub.main:app --port 9000 &
uvicorn app.main:app --port 8000
```

The Kubernetes files contain `REGISTRY_PLACEHOLDER` instead of your AWS account. Part 11 shows how to fill it in with `sed` when you apply them.
The Secret is never in a file. Part 11 creates it from the keyboard with `read -s`.

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
