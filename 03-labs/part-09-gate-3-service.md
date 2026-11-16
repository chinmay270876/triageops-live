# Part 09: Gate 3, SERVICE

The Platform Engineer says NO: "The AI URL is hardcoded. How do I point DEV and PROD to different AIs?"
The Security Head says NO: "Your API key is in the code, and your logs contain the customer's complaint with their phone number."

### Commands (EC2)

```bash
cd ~/triageops

cat > .env.example <<'EOF'
# Copy this file to .env and set real values. Never commit .env.
ENVIRONMENT=dev
LOG_LEVEL=INFO
AI_API_URL=http://localhost:9000/classify
AI_TIMEOUT_SECONDS=3.0
AI_API_KEY=change-me
EOF

cp .env.example .env
sed -i 's/AI_API_KEY=change-me/AI_API_KEY=lab-secret-123/' .env

cat > app/config.py <<'EOF'
"""Configuration: every value comes from the environment. Nothing is hardcoded."""
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    environment: str = "dev"
    log_level: str = "INFO"
    ai_api_url: str = "http://localhost:9000/classify"
    ai_timeout_seconds: float = 3.0
    ai_api_key: SecretStr          # no default: the service refuses to start without it


settings = Settings()
EOF

cat > app/logging_config.py <<'EOF'
"""Structured logging: every log line is one JSON object."""
import json
import logging
import sys
from datetime import datetime, timezone


class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        entry = {
            "time": datetime.now(timezone.utc).isoformat(timespec="milliseconds"),
            "level": record.levelname,
            "event": record.getMessage(),
        }
        entry.update(getattr(record, "fields", {}))
        return json.dumps(entry)


def setup_logging(level: str) -> logging.Logger:
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JsonFormatter())
    logger = logging.getLogger("triage")
    logger.handlers = [handler]
    logger.setLevel(level.upper())
    logger.propagate = False
    return logger
EOF

cat > app/main.py <<'EOF'
"""Complaint Triage Service: contract, async AI call, configuration, secret, JSON logs, health check."""
import time
import uuid

from fastapi import FastAPI, Response

from app.config import settings
from app.logging_config import setup_logging
from app.models import ComplaintIn, TriageResponse
from app.triage import ai_triage

log = setup_logging(settings.log_level)
app = FastAPI(title="Complaint Triage Service", version="1.2.0")

log.info("service_starting", extra={"fields": {
    "version": app.version,
    "environment": settings.environment,
    "ai_api_url": settings.ai_api_url,
    "ai_timeout_seconds": settings.ai_timeout_seconds,
}})


@app.get("/healthz")
async def healthz() -> dict:
    return {"status": "ok", "version": app.version, "environment": settings.environment}


@app.post("/triage", response_model=TriageResponse)
async def triage(body: ComplaintIn, response: Response) -> TriageResponse:
    request_id = uuid.uuid4().hex[:12]
    start = time.perf_counter()

    result = await ai_triage(body.complaint)

    duration_ms = round((time.perf_counter() - start) * 1000)
    response.headers["X-Request-ID"] = request_id
    log.info("triage_completed", extra={"fields": {
        "request_id": request_id,
        "order_id": body.order_id,
        "complaint_length": len(body.complaint),
        "category": result.category.value,
        "priority": result.priority,
        "team": result.team.value,
        "decided_by": result.decided_by,
        "duration_ms": duration_ms,
    }})
    return result
EOF

# Start the service (the AI stub from Part 08 is still running on 9000)
nohup uvicorn app.main:app --host 0.0.0.0 --port 8000 --no-access-log > service.log 2>&1 &
sleep 3
```

Now test it:

```bash
# 1. Health
curl -s http://localhost:8000/healthz; echo

# 2. A real complaint
curl -s -i -X POST http://localhost:8000/triage -H "Content-Type: application/json" \
  -d '{"complaint": "I was charged twice for my biryani, call me on 9876543210", "order_id": "ZMT-1001"}'; echo

# 3. A bad request (Check 1)
curl -s -o /dev/null -w "HTTP %{http_code}\n" -X POST http://localhost:8000/triage \
  -H "Content-Type: application/json" -d '{"complaint": "hi", "coupon": "FREE100"}'

# 4. The logs: JSON, with request_id and duration, and no complaint text, phone number or key
cat service.log
grep -c "9876543210\|lab-secret-123" service.log

# 5. A slow AI: restart the stub with a 5 second delay (our timeout is 3)
pkill -f "uvicorn ai_stub" ; sleep 1
STUB_API_KEY=lab-secret-123 STUB_DELAY=5 nohup uvicorn ai_stub.main:app --port 9000 > stub.log 2>&1 &
sleep 3
curl -s -X POST http://localhost:8000/triage -H "Content-Type: application/json" \
  -d '{"complaint": "I was charged twice for my biryani"}'; echo

# 6. No secret, no start
mv .env /tmp/env.backup
python -c "import app.config" 2>&1 | tail -3
mv /tmp/env.backup .env

# Stop both
pkill -f uvicorn
```

Expected:

1. `{"status":"ok","version":"1.2.0","environment":"dev"}`
2. Headers include `x-request-id: ...` and the body is `{"category":"payments","priority":4,"team":"payments","decided_by":"ai"}`
3. `HTTP 422`
4. Our two JSON lines (`service_starting`, and `triage_completed` with `request_id`, `complaint_length` and `duration_ms`) between uvicorn's own startup lines, and the count is `0`
5. After about 3 seconds: `{"category":"unknown","priority":2,"team":"human_review","decided_by":"fallback: AI timed out"}`
6. `ai_api_key  Field required`

You can also open `http://<PUBLIC_IP>:8000/docs` in your browser (port 8000 is open to your IP) to see the API that FastAPI wrote for us. Do it before `pkill`.

### Commit, pull request, merge

```bash
git status --short                 # .env and .venv must NOT be in this list
git add .
git commit -m "Complaint Triage Service: contract, async AI call, config, secret, JSON logs, healthz"
git push -u origin service         # Username: your GitHub user. Password: paste the token.
```

On GitHub: **Compare and pull request** > **Create pull request** > **Merge pull request** > **Confirm** > **Delete branch**. Then:

```bash
git checkout main && git pull
ls                                  # app  ai_stub  scripts  requirements.txt  must be here
git branch -D service
```

### What we did and why

- **.env.example vs .env**: `.env.example` is committed and shows which settings exist. `.env` holds the real values and is in `.gitignore`, so it never reaches GitHub.
- **app/config.py**: `pydantic-settings` reads every value from environment variables (or from `.env` on our machine). The same code runs in DEV and PROD; only the environment changes. That answers the Platform Engineer.
- **`ai_api_key: SecretStr` with no default**: no key, no start (test 6). Better to fail now than at 2 AM. `SecretStr` prints as `**********`, so the key cannot slip into a log by accident. We read the real value only in one place: `get_secret_value()` in ai_client.py.
- **app/logging_config.py**: every log line is one JSON object. Tools like CloudWatch can search `decided_by` or `duration_ms` directly.
- **app/main.py**:
  - `GET /healthz`: "are you alive?" Kubernetes will ask this every few seconds.
  - `POST /triage`: Check 1 happens automatically because the body is `ComplaintIn`. `response_model=TriageResponse` makes sure we only ever return the contract.
  - `request_id`: a short ID in the log and in the `X-Request-ID` header. A customer support agent can give it to an engineer and the engineer finds the exact log line.
  - We log `complaint_length`, never the complaint text. The phone number in test 2 never reaches the log. That answers the Security Head.
- **`--no-access-log`**: uvicorn's own access log is turned off; our JSON log is the one source of truth.
- **Merge before the next branch**: we learned this live. If you create the next branch before the merge, `main` has no code, the new branch has no code, and Docker fails with `"/requirements.txt": not found`.

---

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 08](part-08-gate-2-async.md) | [Lab map](README.md) | Next: [Part 10: Gate 4, SHIP part 1: Docker image and Amazon ECR](part-10-docker-and-ecr.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
