# Part 10: Gate 4, SHIP part 1: Docker image and Amazon ECR

"It works on my EC2" is not a release. We need one package that runs the same everywhere.

### Commands (EC2)

```bash
cd ~/triageops
git checkout -b docker

cat > .dockerignore <<'EOF'
.venv
__pycache__
*.pyc
*.log
.env
.git
EOF

cat > Dockerfile <<'EOF'
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Run as a normal user, never as root
RUN useradd --create-home --uid 10001 appuser

# Dependencies first: this layer is cached until requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Then the code
COPY app ./app
COPY ai_stub ./ai_stub

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://localhost:8000/healthz', timeout=2).status == 200 else 1)"

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--no-access-log"]
EOF

ls                                         # requirements.txt, app, ai_stub must be here
docker build -t triageops:1.2.0 .
docker images triageops
docker run --rm triageops:1.2.0 ls -la /app        # no .env, no .venv
```

Run both containers together:

```bash
docker network create triage-net 2>/dev/null || true

docker run -d --name ai-stub --network triage-net --no-healthcheck \
  -e STUB_API_KEY=lab-secret-123 \
  triageops:1.2.0 uvicorn ai_stub.main:app --host 0.0.0.0 --port 9000

docker run -d --name triage --network triage-net -p 8000:8000 \
  -e AI_API_URL=http://ai-stub:9000/classify \
  -e AI_API_KEY=lab-secret-123 \
  -e ENVIRONMENT=docker \
  triageops:1.2.0

sleep 5
curl -s http://localhost:8000/healthz; echo
curl -s -X POST http://localhost:8000/triage -H "Content-Type: application/json" \
  -d '{"complaint": "I was charged twice for my biryani"}'; echo
docker logs triage | tail -2
sleep 30; docker ps --format "table {{.Names}}\t{{.Status}}"

# No secret, no start
docker run --rm -e AI_API_URL=http://ai-stub:9000/classify triageops:1.2.0 2>&1 | tail -2
```

Push the image to Amazon ECR:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGISTRY=$ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com
echo "export REGISTRY=$REGISTRY" >> ~/.bashrc

aws ecr create-repository --repository-name triageops \
  --image-scanning-configuration scanOnPush=true --query repository.repositoryUri --output text
aws ecr get-login-password | docker login --username AWS --password-stdin $REGISTRY

docker tag triageops:1.2.0 $REGISTRY/triageops:1.2.0
docker push $REGISTRY/triageops:1.2.0
aws ecr describe-images --repository-name triageops \
  --query 'imageDetails[].[imageTags[0],imageSizeInBytes]' --output table
```

Clean up the local containers, then commit:

```bash
docker rm -f triage ai-stub
git add Dockerfile .dockerignore
git commit -m "Dockerfile: non-root user, health check, no secrets in the image"
git push -u origin docker
```

On GitHub: pull request > merge > delete branch. Then `git checkout main && git pull && git branch -D docker`.

### What we did and why

- **.dockerignore** lists `.env`. The secret cannot even reach the build. `ls -la /app` proves it.
- **python:3.11-slim**: the same Python as our EC2, in a small image.
- **COPY requirements.txt, pip install, then COPY the code**: Docker caches each step. Change one line of Python and the slow pip step is reused. Rebuilds take seconds.
- **USER appuser (uid 10001)**: if someone breaks into the container, they are not root.
- **HEALTHCHECK**: Docker itself knows if the service is healthy (`docker ps` shows `(healthy)` after about 30 seconds). On EKS the liveness and readiness probes do this job instead.
- **One image, two commands**: the same image runs the service (default CMD) and the AI stub (we pass a different command). For the stub we add `--no-healthcheck`, because the HEALTHCHECK checks port 8000 and the stub listens on 9000.
- **`-e` at runtime**: configuration and secret come in from outside. The same image goes to DEV and PROD.
- **triage-net**: containers on the same Docker network find each other by name, so the AI URL is `http://ai-stub:9000`. Kubernetes gives us the same thing with Services.
- **No secret, no start**: `ai_api_key Field required`. On EKS this shows up as `CrashLoopBackOff` (Part 13).
- **ECR with scanOnPush**: our private image registry in AWS. Every pushed image is scanned for known vulnerabilities for free. EKS nodes can pull from it without passwords.
- **Tag 1.2.0, never latest**: you must always know exactly which version is running and be able to roll back to it.
- **REGISTRY in ~/.bashrc**: Parts 11 and 12 need it, even after you reconnect PuTTY.

---

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 09](part-09-gate-3-service.md) | [Lab map](README.md) | Next: [Part 11: Gate 4, SHIP part 2: deploy to EKS](part-11-deploy-to-eks.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
