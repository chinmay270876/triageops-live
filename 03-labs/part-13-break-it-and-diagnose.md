# Part 13: Break it and diagnose

Production is not "it works". Production is "when it breaks, we know why in 2 minutes".
Run each break, look at the symptom, find the cause, fix it, and check the fix.

### Commands (EC2)

Helper for this part:

```bash
ask() { curl -s -X POST http://$LB/triage -H "Content-Type: application/json" \
  -d '{"complaint": "I was charged twice for my biryani"}'; echo; }
ask                                   # baseline: decided_by ai
```

**Break 1: the secret is missing from a new version**

```bash
kubectl set env deployment/triage AI_API_KEY-
kubectl get pods -w                   # a new pod goes Error, then CrashLoopBackOff. Ctrl+C
kubectl logs $(kubectl get pods -l app=triage --sort-by=.metadata.creationTimestamp -o name | tail -1) | tail -3
ask                                   # still works!
kubectl rollout undo deployment/triage && kubectl rollout status deployment/triage
```

**Break 2: wrong API key between our service and the AI**

```bash
kubectl patch secret triage-secrets -p '{"stringData":{"STUB_API_KEY":"wrong-key"}}'
kubectl rollout restart deployment/ai-stub && kubectl rollout status deployment/ai-stub
ask                                   # fallback: AI returned 401
read -s -p "Correct key: " AI_KEY; echo
kubectl patch secret triage-secrets -p "{\"stringData\":{\"STUB_API_KEY\":\"$AI_KEY\"}}"; unset AI_KEY
kubectl rollout restart deployment/ai-stub && kubectl rollout status deployment/ai-stub
ask
```

**Break 3: the AI answers with garbage**

```bash
kubectl set env deployment/ai-stub STUB_MODE=bad && kubectl rollout status deployment/ai-stub
ask                                   # fallback: invalid AI answer
kubectl set env deployment/ai-stub STUB_MODE=normal && kubectl rollout status deployment/ai-stub
```

**Break 4: the AI is slow**

```bash
kubectl set env deployment/ai-stub STUB_DELAY=5 && kubectl rollout status deployment/ai-stub
time ask                              # about 3 seconds, fallback: AI timed out
kubectl set env deployment/ai-stub STUB_DELAY=1.0 && kubectl rollout status deployment/ai-stub
```

**Break 5: the AI is down**

```bash
kubectl scale deployment/ai-stub --replicas=0
ask                                   # fallback: AI unavailable
kubectl scale deployment/ai-stub --replicas=1 && kubectl rollout status deployment/ai-stub
```

**Break 6: a bad health check in a new version**

```bash
kubectl patch deployment triage --type=json \
  -p '[{"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/port","value":9999}]'
kubectl get pods                      # new pod Running but 0/1 READY
kubectl describe pod $(kubectl get pods -l app=triage --sort-by=.metadata.creationTimestamp -o name | tail -1) | grep -i "readiness"
ask                                   # still works
kubectl rollout undo deployment/triage && kubectl rollout status deployment/triage
```

Finally, read the story of all of it from the logs:

```bash
kubectl logs deployment/triage --tail=20 | grep triage_completed
ask
```

### What we did and why

| Break | Symptom | Where you find the cause | Fix |
|-------|---------|--------------------------|-----|
| 1. Secret missing | New pod `CrashLoopBackOff` | `kubectl logs`: `ai_api_key Field required` | `rollout undo` |
| 2. Wrong key | `fallback: AI returned 401` | our JSON log, `decided_by` | correct the Secret, restart |
| 3. Garbage answer | `fallback: invalid AI answer` | our JSON log | fix the AI side |
| 4. Slow AI | answer after 3 s, `AI timed out` | `duration_ms` in the log | AI side, or tune `AI_TIMEOUT_SECONDS` |
| 5. AI down | `fallback: AI unavailable` | `kubectl get pods`, endpoints | scale back up |
| 6. Bad probe | new pod `0/1 READY`, rollout stuck | `kubectl describe pod`: Readiness probe failed | `rollout undo` |

- **Break 1 and 6, the big aha**: the broken version never received a single customer. `maxUnavailable: 0` plus the readiness probe kept the old pods serving. A bad deploy became a stuck deploy, not an outage.
- **Break 2 to 5**: the service never crashed. Every AI failure became a complaint in Human Review, and the log tells us exactly which failure it was. This is what the 4 gates bought us.
- **Break 2 also teaches**: changing a Secret does not change running pods. Environment variables are read at start, so we restart the Deployment.
- `kubectl logs`, `kubectl describe`, `kubectl get pods`, `rollout status/undo`: these four commands solve most problems you will see in a real job.

---

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 12](part-12-web-page-rolling-update.md) | [Lab map](README.md) | Next: [Part 14: Clean up everything](part-14-clean-up.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
