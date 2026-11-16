# Part 08: Gate 2, ASYNC

The Support Team Lead says NO: "The AI takes 1 second per complaint. 1000 complaints one by one is 16 minutes. Customers are waiting."

Think first: while we wait 1 second for the AI, what is our service doing? Can it start the next complaint?

### Commands (EC2)

```bash
cd ~/triageops

cat > ai_stub/main.py <<'EOF'
"""A fake AI model. It answers like a real model would, but costs nothing.

STUB_DELAY    seconds to wait before answering (default 1.0), like a real model thinking
STUB_MODE     normal = sensible answers, bad = the wrong answer from our story
STUB_API_KEY  if set, callers must send "Authorization: Bearer <key>", like a real AI provider
"""
import asyncio
import os

from fastapi import FastAPI, Header, HTTPException

app = FastAPI(title="AI Stub")

DELAY = float(os.getenv("STUB_DELAY", "1.0"))
MODE = os.getenv("STUB_MODE", "normal")
API_KEY = os.getenv("STUB_API_KEY", "")


@app.get("/healthz")
async def healthz() -> dict:
    return {"status": "ok", "mode": MODE, "delay": DELAY}


@app.post("/classify")
async def classify(body: dict, authorization: str = Header(default="")) -> dict:
    if API_KEY and authorization != f"Bearer {API_KEY}":
        raise HTTPException(status_code=401, detail="invalid API key")

    await asyncio.sleep(DELAY)

    if MODE == "bad":
        return {"category": "payment", "priority": "High", "team": "payment team"}

    text = str(body.get("complaint", "")).lower()
    if "charged" in text or "payment" in text or "upi" in text:
        return {"category": "payments", "priority": 4, "team": "payments"}
    if "refund" in text:
        return {"category": "refund", "priority": 3, "team": "refunds"}
    if "late" in text or "delivery" in text or "rider" in text:
        return {"category": "delivery", "priority": 3, "team": "delivery_ops"}
    if "cold" in text or "missing" in text or "restaurant" in text:
        return {"category": "restaurant", "priority": 2, "team": "restaurant_support"}
    return {"category": "unknown", "priority": 2, "team": "human_review"}
EOF

cat > scripts/async_demo.py <<'EOF'
"""Gate 2 check: 10 complaints, and an AI that needs 1 second for each one.

python -m scripts.async_demo sync    one by one
python -m scripts.async_demo trap    async def, but with a blocking call inside
python -m scripts.async_demo async   truly async
"""
import asyncio
import sys
import time

import httpx

URL = "http://localhost:9000/classify"
HEADERS = {"Authorization": "Bearer lab-secret-123"}
BODY = {"complaint": "I was charged twice for my biryani"}
N = 10


def one_by_one() -> None:
    with httpx.Client(timeout=10) as client:
        for _ in range(N):
            client.post(URL, json=BODY, headers=HEADERS)


async def blocking_inside_async() -> None:
    async def ask() -> None:
        httpx.post(URL, json=BODY, headers=HEADERS, timeout=10)   # the trap: this call blocks
    await asyncio.gather(*(ask() for _ in range(N)))


async def truly_async() -> None:
    async with httpx.AsyncClient(timeout=10) as client:
        await asyncio.gather(*(client.post(URL, json=BODY, headers=HEADERS) for _ in range(N)))


mode = sys.argv[1] if len(sys.argv) > 1 else "async"
start = time.perf_counter()
if mode == "sync":
    one_by_one()
elif mode == "trap":
    asyncio.run(blocking_inside_async())
else:
    asyncio.run(truly_async())
print(f"{mode:5}: {N} complaints in {time.perf_counter() - start:.1f} seconds")
EOF

cat > app/ai_client.py <<'EOF'
"""Talks to the AI model over HTTP, asynchronously, with a timeout and an API key."""
import httpx

from app.config import settings


async def ask_ai(complaint: str) -> dict:
    headers = {"Authorization": f"Bearer {settings.ai_api_key.get_secret_value()}"}
    async with httpx.AsyncClient(timeout=settings.ai_timeout_seconds) as client:
        response = await client.post(settings.ai_api_url, json={"complaint": complaint}, headers=headers)
        response.raise_for_status()
        return response.json()
EOF

cat > app/triage.py <<'EOF'
"""The AI decides. The contract checks. Anything wrong goes to Human Review."""
import httpx
from pydantic import ValidationError

from app.ai_client import ask_ai
from app.models import Category, Team, TriageResponse, TriageResult

HUMAN_REVIEW = dict(category=Category.unknown, priority=2, team=Team.human_review)


async def ai_triage(complaint: str) -> TriageResponse:
    try:
        raw_answer = await ask_ai(complaint)
    except httpx.TimeoutException:
        return TriageResponse(**HUMAN_REVIEW, decided_by="fallback: AI timed out")
    except httpx.HTTPStatusError as error:
        status = error.response.status_code
        return TriageResponse(**HUMAN_REVIEW, decided_by=f"fallback: AI returned {status}")
    except httpx.HTTPError:
        return TriageResponse(**HUMAN_REVIEW, decided_by="fallback: AI unavailable")

    try:
        result = TriageResult.model_validate(raw_answer)
    except ValidationError:
        return TriageResponse(**HUMAN_REVIEW, decided_by="fallback: invalid AI answer")

    return TriageResponse(**result.model_dump(), decided_by="ai")
EOF

# Run the AI stub in the background and race the three versions
STUB_API_KEY=lab-secret-123 nohup uvicorn ai_stub.main:app --port 9000 > stub.log 2>&1 &
sleep 3
curl -s http://localhost:9000/healthz; echo
python -m scripts.async_demo sync
python -m scripts.async_demo trap
python -m scripts.async_demo async
```

Expected:

```
{"status":"ok","mode":"normal","delay":1.0}
sync : 10 complaints in 10.1 seconds
trap : 10 complaints in 10.1 seconds
async: 10 complaints in 1.1 seconds
```

Leave the stub running for Part 09.

### What we did and why

- **ai_stub/main.py**: our fake AI. `STUB_DELAY` makes it think for 1 second. `STUB_MODE=bad` returns the exact wrong answer from our story. `STUB_API_KEY` makes it reject callers without the right key (HTTP 401), like a real AI provider.
- **async_demo.py**, the aha moment:
  - `sync`: one complaint at a time, 10 x 1 second = 10 seconds.
  - `trap`: the code says `async def`, but inside it calls `httpx.post`, which blocks. Still 10 seconds. Writing `async` is not enough.
  - `async`: `httpx.AsyncClient` with `await`. While one complaint waits for the AI, the next one starts. 10 complaints in about 1 second.
- **app/ai_client.py**: how our service calls the AI. `async with httpx.AsyncClient(timeout=...)` and `await client.post(...)`. The key is sent as a Bearer header. `raise_for_status()` turns a 401 or 500 into an error we can catch.
- **app/triage.py**: every way the AI can fail ends in Human Review, never in a crash:
  - AI too slow: `fallback: AI timed out`
  - AI says 401 or 500: `fallback: AI returned 401`
  - AI not reachable: `fallback: AI unavailable`
  - AI answer breaks the contract (Check 2): `fallback: invalid AI answer`
  - All good: `decided_by: ai`

**Concept: async is about waiting, not about speed**
The AI is not faster. Our service just does not sit idle while it waits. And a timeout means a slow AI can never hold a customer forever.

(ai_client.py imports `app.config`, which we write in Part 09. That is why we do not run the service yet.)

---

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 07](part-07-gate-1-contract.md) | [Lab map](README.md) | Next: [Part 09: Gate 3, SERVICE](part-09-gate-3-service.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
