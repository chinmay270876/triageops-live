# Part 11: Gate 4, SHIP part 2: deploy to EKS

### Commands (EC2)

```bash
tail -3 ~/triage/eks-create.log      # wait for: EKS cluster "triageops-eks" in "ap-south-1" region is ready
kubectl get nodes                    # 2 nodes, STATUS Ready, VERSION v1.34.x
```

If `kubectl get nodes` says it cannot connect but the log says ready, run
`eksctl utils write-kubeconfig --cluster triageops-eks --region ap-south-1`.

```bash
cd ~/triageops
git checkout -b k8s
mkdir -p k8s

cat > k8s/namespace.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: triageops
EOF

cat > k8s/configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: triage-config
  namespace: triageops
data:
  ENVIRONMENT: "eks"
  LOG_LEVEL: "INFO"
  AI_API_URL: "http://ai-stub:9000/classify"
  AI_TIMEOUT_SECONDS: "3.0"
EOF

cat > k8s/ai-stub.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-stub
  namespace: triageops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-stub
  template:
    metadata:
      labels:
        app: ai-stub
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
      containers:
        - name: ai-stub
          image: REGISTRY_PLACEHOLDER/triageops:1.2.0
          command: ["uvicorn", "ai_stub.main:app", "--host", "0.0.0.0", "--port", "9000"]
          ports:
            - containerPort: 9000
          env:
            - name: STUB_DELAY
              value: "1.0"
            - name: STUB_API_KEY
              valueFrom:
                secretKeyRef:
                  name: triage-secrets
                  key: STUB_API_KEY
          readinessProbe:
            httpGet:
              path: /healthz
              port: 9000
            periodSeconds: 5
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: ai-stub
  namespace: triageops
spec:
  type: ClusterIP
  selector:
    app: ai-stub
  ports:
    - port: 9000
      targetPort: 9000
EOF

cat > k8s/triage.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triage
  namespace: triageops
spec:
  replicas: 2
  minReadySeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: triage
  template:
    metadata:
      labels:
        app: triage
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
      containers:
        - name: triage
          image: REGISTRY_PLACEHOLDER/triageops:1.2.0
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: triage-config
          env:
            - name: AI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: triage-secrets
                  key: AI_API_KEY
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8000
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: triage
  namespace: triageops
spec:
  type: LoadBalancer
  selector:
    app: triage
  ports:
    - port: 80
      targetPort: 8000
EOF

# 1. Namespace and config
kubectl apply -f k8s/namespace.yaml
kubectl config set-context --current --namespace=triageops
kubectl apply -f k8s/configmap.yaml

# 2. Secret: typed in, never written to a file, never committed
read -s -p "AI API key for the lab (type lab-secret-123): " AI_KEY; echo
kubectl create secret generic triage-secrets \
  --from-literal=AI_API_KEY="$AI_KEY" --from-literal=STUB_API_KEY="$AI_KEY"
unset AI_KEY

# 3. The AI stub and the service, with our registry filled in
sed "s|REGISTRY_PLACEHOLDER|$REGISTRY|" k8s/ai-stub.yaml | kubectl apply -f -
sed "s|REGISTRY_PLACEHOLDER|$REGISTRY|" k8s/triage.yaml  | kubectl apply -f -
kubectl rollout status deployment/ai-stub
kubectl rollout status deployment/triage
kubectl get all
```

Wait for the load balancer (2 to 3 minutes), then call it from the internet:

```bash
kubectl get svc triage -w          # wait until EXTERNAL-IP shows a name, then Ctrl+C
LB=$(kubectl get svc triage -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'); echo $LB
echo "export LB=$LB" >> ~/.bashrc

curl -s http://$LB/healthz; echo
curl -s -X POST http://$LB/triage -H "Content-Type: application/json" \
  -d '{"complaint": "My delivery is 40 minutes late and the rider is not picking up", "order_id": "ZMT-2002"}'; echo
kubectl logs deployment/triage --tail=3
```

If `curl` says "Could not resolve host", the load balancer DNS is not ready yet. Wait one more minute.

Expected: `"environment":"eks"`, then `{"category":"delivery","priority":3,"team":"delivery_ops","decided_by":"ai"}`, and JSON log lines.

Commit (the YAML has no account ID and no secret in it):

```bash
grep -c "lab-secret\|dkr.ecr" k8s/*.yaml        # every file must show 0
git add k8s
git commit -m "Kubernetes: namespace, config, secret by reference, ai-stub, triage with probes and LoadBalancer"
git push -u origin k8s
```

On GitHub: pull request > merge > delete branch. Then `git checkout main && git pull && git branch -D k8s`.

### What we did and why

- **Namespace `triageops`**: our own space in the cluster. `set-context --namespace` means we do not type `-n triageops` every time. Deleting the namespace later deletes everything in it.
- **ConfigMap `triage-config`**: the non-secret settings. This is our `.env.example` for Kubernetes. `envFrom` turns every key into an environment variable, and `pydantic-settings` reads them. Not one line of Python changed.
- **Secret `triage-secrets`**: created from the keyboard with `read -s` (nothing on screen, nothing in shell history, nothing in Git). The YAML only references it by name with `secretKeyRef`.
- **ai-stub Deployment**: the same image with a different `command`. **Service type ClusterIP**: reachable only inside the cluster, at `http://ai-stub:9000`. That is the URL in our ConfigMap.
- **triage Deployment**:
  - `replicas: 2`: two pods, on two nodes. One can die and customers do not notice.
  - `readinessProbe`: "can you take traffic?" Until it passes, the Service sends nothing to the pod.
  - `livenessProbe`: "are you still alive?" If it fails 3 times, Kubernetes restarts the container.
  - `runAsNonRoot` and `runAsUser: 10001`: Kubernetes double checks that we are not root.
  - `resources`: a small reservation so the scheduler places the pods sensibly, and a memory limit so one pod cannot eat a node.
  - `RollingUpdate` with `maxUnavailable: 0`: during an update, old pods keep serving until new ones are ready. We see this in Part 12 and Part 13.
- **Service type LoadBalancer**: AWS creates a public load balancer in our subnets tagged `kubernetes.io/role/elb=1` (Part 01) and sends port 80 to port 8000 on the pods.
- **REGISTRY_PLACEHOLDER and sed**: the account ID stays out of Git. In a real company a CI pipeline fills it in.

**Concept: liveness vs readiness**
Readiness: "not ready yet, do not send customers to me." Liveness: "I am stuck, restart me." Same `/healthz` endpoint here, two different decisions.

---

Lab sheet: `02-diagrams/drawio/triageops-lab.drawio`

Previous: [Part 10](part-10-docker-and-ecr.md) | [Lab map](README.md) | Next: [Part 12: The complaint web page, version 1.3.0, with a rolling update](part-12-web-page-rolling-update.md)

## Keep learning

- YouTube: https://www.youtube.com/@BuildAutomateArchitect
- System Design playlist: https://www.youtube.com/playlist?list=PLeAHTLMwFYQ0
- Telegram: https://t.me/BuildAutomateArchitect
- Website: https://rahulchaubeyconsulting.com/
