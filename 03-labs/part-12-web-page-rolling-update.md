# Part 12: The complaint web page, version 1.3.0, with a rolling update

Our support agents do not use curl. They want a page: type the complaint, see the team.
We ship it as a new version, with zero downtime.

### Commands (EC2)

```bash
cd ~/triageops
git checkout -b web-page
mkdir -p app/static

cat > app/static/index.html <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Complaint Triage</title>
<style>
  body { font-family: Arial, sans-serif; background: #f6f6f6; margin: 0; }
  header { background: #cb202d; color: #fff; padding: 16px 24px; font-size: 20px; font-weight: bold; }
  main { max-width: 640px; margin: 24px auto; background: #fff; padding: 24px; border-radius: 8px; }
  label { display: block; margin: 12px 0 4px; font-weight: bold; }
  textarea, input { width: 100%; box-sizing: border-box; padding: 8px; font-size: 15px; }
  textarea { height: 110px; }
  button { margin-top: 16px; padding: 10px 20px; font-size: 15px; background: #cb202d; color: #fff;
           border: 0; border-radius: 4px; cursor: pointer; }
  #result { margin-top: 20px; padding: 12px; border-radius: 6px; display: none; }
  .ok { background: #e8f5e9; }
  .review { background: #fff8e1; }
  .error { background: #ffebee; }
  table { border-collapse: collapse; }
  td { padding: 4px 12px 4px 0; }
  footer { text-align: center; color: #888; font-size: 13px; margin: 16px; }
</style>
</head>
<body>
<header>Complaint Triage</header>
<main>
  <label for="complaint">Customer complaint</label>
  <textarea id="complaint" placeholder="I was charged twice for my biryani"></textarea>
  <label for="order">Order ID (optional)</label>
  <input id="order" placeholder="ZMT-1001">
  <button onclick="sendComplaint()">Triage</button>
  <div id="result"></div>
</main>
<footer id="version">loading version...</footer>
<script>
async function sendComplaint() {
  const box = document.getElementById("result");
  const body = { complaint: document.getElementById("complaint").value };
  const order = document.getElementById("order").value.trim();
  if (order) { body.order_id = order; }
  box.style.display = "block";
  box.className = "";
  box.textContent = "Asking the AI...";
  try {
    const res = await fetch("/triage", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body)
    });
    const data = await res.json();
    if (!res.ok) {
      box.className = "error";
      box.textContent = "Rejected (HTTP " + res.status + "): " +
        data.detail.map(d => d.loc.join(".") + " - " + d.msg).join("; ");
      return;
    }
    box.className = data.decided_by === "ai" ? "ok" : "review";
    box.innerHTML = "<table>" +
      "<tr><td>Category</td><td><b>" + data.category + "</b></td></tr>" +
      "<tr><td>Priority</td><td><b>" + data.priority + "</b></td></tr>" +
      "<tr><td>Team</td><td><b>" + data.team + "</b></td></tr>" +
      "<tr><td>Decided by</td><td>" + data.decided_by + "</td></tr>" +
      "<tr><td>Request ID</td><td>" + res.headers.get("X-Request-ID") + "</td></tr>" +
      "</table>";
  } catch (err) {
    box.className = "error";
    box.textContent = "Service not reachable: " + err;
  }
}
fetch("/healthz").then(r => r.json()).then(h => {
  document.getElementById("version").textContent = "version " + h.version + " | " + h.environment;
});
</script>
</body>
</html>
EOF

cat > app/web.py <<'EOF'
"""The complaint web page for support agents."""
from pathlib import Path

from fastapi import APIRouter
from fastapi.responses import FileResponse

router = APIRouter()
PAGE = Path(__file__).parent / "static" / "index.html"


@router.get("/", include_in_schema=False)
async def home() -> FileResponse:
    return FileResponse(PAGE)
EOF

# Plug the page into the service and bump the version
sed -i '/^from app.triage import ai_triage$/a from app.web import router as web_router' app/main.py
sed -i 's/version="1.2.0"/version="1.3.0"/' app/main.py
echo 'app.include_router(web_router)' >> app/main.py
grep -n "web_router\|version=" app/main.py

# Build and push 1.3.0
docker build -t $REGISTRY/triageops:1.3.0 .
aws ecr get-login-password | docker login --username AWS --password-stdin $REGISTRY
docker push $REGISTRY/triageops:1.3.0
```

Now open a **second PuTTY window** (same saved session) and leave this running. It calls the service every second:

```bash
while true; do curl -s -m 2 http://$LB/healthz || echo "FAILED"; echo; sleep 1; done
```

Back in the first window, roll out 1.3.0:

```bash
cd ~/triageops
sed -i 's|triageops:1.2.0|triageops:1.3.0|' k8s/triage.yaml
sed "s|REGISTRY_PLACEHOLDER|$REGISTRY|" k8s/triage.yaml | kubectl apply -f -
kubectl rollout status deployment/triage
kubectl rollout history deployment/triage
echo "Open in your browser: http://$LB/"
```

Watch the second window: lines change from `"version":"1.2.0"` to `"version":"1.3.0"`, and there is no `FAILED` line.
Open `http://<LB>/` in your browser and send a few complaints, including one that is too short.

Roll back and forward again:

```bash
kubectl rollout undo deployment/triage
kubectl rollout status deployment/triage        # second window shows 1.2.0 again
sed "s|REGISTRY_PLACEHOLDER|$REGISTRY|" k8s/triage.yaml | kubectl apply -f -
kubectl rollout status deployment/triage        # back to 1.3.0
```

Commit:

```bash
git add app k8s/triage.yaml
git commit -m "Version 1.3.0: complaint web page"
git push -u origin web-page
```

On GitHub: pull request > merge > delete branch. Then `git checkout main && git pull && git branch -D web-page`.
Stop the loop in the second window with Ctrl+C.

### What we did and why

- **index.html**: a plain page. It sends the same JSON to the same `/triage` API. The browser is just another client of our contract. Green for `decided_by: ai`, yellow for Human Review, red for a 422 with the reason.
- **app/web.py**: a small FastAPI router that serves the page at `/`. `include_in_schema=False` keeps it out of the API docs.
- **Three edits to main.py**: import the router, bump the version to 1.3.0, include the router. The footer of the page shows the version from `/healthz`, so we can see which version served us.
- **New tag 1.3.0**: we never overwrite 1.2.0. That is what makes rollback possible.
- **The rolling update**: Kubernetes starts one new pod (`maxSurge: 1`), waits until its readiness probe passes, then removes one old pod, and repeats. `maxUnavailable: 0` means capacity never drops below 2. The loop in the second window is the proof: no `FAILED`.
- **rollout history / undo**: every change to the Deployment is a revision. `undo` goes back to the previous one in seconds.
- **We changed the YAML in Git**, not only the cluster. Git is the record of what should be running.

---

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 11](part-11-deploy-to-eks.md) | [Lab map](README.md) | Next: [Part 13: Break it and diagnose](part-13-break-it-and-diagnose.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
