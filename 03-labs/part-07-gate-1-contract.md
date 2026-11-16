# Part 07: Gate 1, CONTRACT

Support Ops says NO: "Last night the AI sent `payment team`. We have no team called that. 500 complaints went nowhere."

Think first: who should catch a wrong team name, the AI, the support team, or our service?

### Commands (EC2)

```bash
cd ~/triageops
git checkout -b service

mkdir -p app ai_stub scripts
touch app/__init__.py ai_stub/__init__.py

cat > .gitignore <<'EOF'
.venv/
__pycache__/
*.pyc
.env
*.log
EOF

cat > requirements.txt <<'EOF'
fastapi==0.115.0
uvicorn[standard]==0.30.6
pydantic==2.9.2
pydantic-settings==2.5.2
httpx==0.27.2
EOF

cat > app/models.py <<'EOF'
"""The contract: what our service accepts and what it is allowed to return."""
from enum import Enum

from pydantic import BaseModel, ConfigDict, Field


class Category(str, Enum):
    payments = "payments"
    delivery = "delivery"
    restaurant = "restaurant"
    refund = "refund"
    unknown = "unknown"


class Team(str, Enum):
    payments = "payments"
    delivery_ops = "delivery_ops"
    restaurant_support = "restaurant_support"
    refunds = "refunds"
    human_review = "human_review"


class ComplaintIn(BaseModel):
    """What the food app sends us."""
    model_config = ConfigDict(extra="forbid")

    complaint: str = Field(min_length=5, max_length=2000)
    order_id: str | None = None


class TriageResult(BaseModel):
    """What we demand from the AI, and the core of what we promise back."""
    model_config = ConfigDict(extra="forbid")

    category: Category
    priority: int = Field(ge=1, le=4)
    team: Team


class TriageResponse(TriageResult):
    """What the food app receives: the result, plus who decided it."""
    decided_by: str
EOF

cat > scripts/contract_check.py <<'EOF'
"""Gate 1 check: the contract accepts good data and rejects bad data."""
from pydantic import ValidationError

from app.models import ComplaintIn, TriageResult

good_ai = {"category": "payments", "priority": 4, "team": "payments"}
bad_ai = {"category": "payment", "priority": "High", "team": "payment team"}

print("Good AI answer  ->", TriageResult.model_validate(good_ai))

for name, model, data in [
    ("Bad AI answer  ", TriageResult, bad_ai),
    ("Bad request    ", ComplaintIn, {"complaint": "hi", "coupon": "FREE100"}),
]:
    try:
        model.model_validate(data)
    except ValidationError as error:
        print(f"{name} -> REJECTED, {error.error_count()} problems:")
        for problem in error.errors():
            print("      ", problem["loc"], "-", problem["msg"])
EOF

python3.11 -m venv .venv
source .venv/bin/activate
pip install -q --upgrade pip
pip install -q -r requirements.txt
python -m scripts.contract_check
```

Expected output:

```
Good AI answer  -> category=<Category.payments: 'payments'> priority=4 team=<Team.payments: 'payments'>
Bad AI answer   -> REJECTED, 3 problems:
       ('category',) - Input should be 'payments', 'delivery', 'restaurant', 'refund' or 'unknown'
       ('priority',) - Input should be a valid integer, unable to parse string as an integer
       ('team',) - Input should be 'payments', 'delivery_ops', 'restaurant_support', 'refunds' or 'human_review'
Bad request     -> REJECTED, 2 problems:
       ('complaint',) - String should have at least 5 characters
       ('coupon',) - Extra inputs are not permitted
```

### What we did and why

- **Branch `service`**: Parts 07, 08 and 09 all go on this branch and become one pull request.
- **.gitignore first**: before any code, we tell Git what must never be committed: the virtual env, Python cache, log files and `.env` (our secrets file, coming in Part 09).
- **requirements.txt with exact versions**: everyone, and later the Docker image, installs exactly the same libraries.
- **Enums `Category` and `Team`**: the complete list of allowed values. `payment team` is not in the list, so it can never pass.
- **`ComplaintIn`**: what the food app must send. Complaint between 5 and 2000 characters. `extra="forbid"` rejects unknown fields like `coupon`. In the API this becomes HTTP **422**, the caller's mistake.
- **`TriageResult`**: what we demand from the AI. Priority must be a number from 1 to 4 (`ge=1, le=4`). `"High"` is rejected.
- **`TriageResponse`**: what we send back. Same fields plus `decided_by`, so everyone knows if the AI decided or our fallback did.
- **venv**: a private Python for this project, so its libraries do not mix with the system.

**Concept: two checks, one contract**
Check 1 is the request coming in: bad input gets 422, the caller must fix it.
Check 2 is the AI answer: a bad AI answer is not the caller's fault, so we do not fail. We send the complaint to Human Review (Part 08).
Answer to the question above: our service catches it. Never trust the AI blindly.

---

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 06](part-06-github.md) | [Lab map](README.md) | Next: [Part 08: Gate 2, ASYNC](part-08-gate-2-async.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
