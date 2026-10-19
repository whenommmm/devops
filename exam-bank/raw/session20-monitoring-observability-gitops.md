# Labs · Session 20 · Monitoring, Observability and GitOps

Acronyms: PromQL = Prometheus Query Language. RED = Rate, Errors, Duration (the three standard panels for any service). p95 = 95th percentile. CRD = Custom Resource Definition (a new object type added to Kubernetes, such as Argo CD's `Application`). UI = User Interface. CLI = Command-Line Interface.

## Lab 1 · Expose app metrics, make Prometheus scrape them, check the target is UP, name two metrics
**Answer (what to write in the exam):**
```bash
python3 -m venv venv && . venv/bin/activate && pip install prometheus-client
python3 app.py &                                     # metrics at :8000/metrics
curl -s localhost:8000/metrics | grep http_requests_total
docker run -d --name prometheus -p 9090:9090 \
  --add-host=host.docker.internal:host-gateway \
  -v "$(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml" \
  prom/prometheus
# Browser: http://localhost:9090 → Status → Targets  (both jobs must show UP)
curl -s 'localhost:9090/api/v1/query?query=up'
```
`app.py`
```python
from prometheus_client import Counter, Histogram, start_http_server
import random, time

REQUESTS = Counter("http_requests_total", "Total HTTP requests", ["status"])
LATENCY = Histogram("http_request_duration_seconds", "Request latency in seconds")

start_http_server(8000)          # serves http://localhost:8000/metrics
while True:                      # fake traffic: one request at a time
    with LATENCY.time():
        time.sleep(random.random() / 5)
    REQUESTS.labels(status=random.choice(["200", "200", "200", "500"])).inc()
```
`prometheus.yml`
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: myapp
    scrape_interval: 5s
    metrics_path: /metrics
    static_configs:
      - targets: ["host.docker.internal:8000"]
```
**What each line does:**
- `Counter` / `Histogram`: the app records a request count (labelled by HTTP status) and the time each request took. `start_http_server(8000)` publishes these numbers as text at `/metrics`.
- `scrape_configs`: the list of things Prometheus pulls metrics from. `job_name` is the label it gives them. `scrape_interval` sets how often it pulls (5s here; otherwise the global 15s). `static_configs.targets` lists the `host:port` to scrape.
- `--add-host=host.docker.internal:host-gateway`: lets the Prometheus container reach the app running on the host machine (needed on Linux; Docker Desktop on Mac has it built in).
- `-v …prometheus.yml:/etc/prometheus/prometheus.yml`: mounts your config file over the default one.

**Verify:** Status → Targets shows `myapp` and `prometheus` with State **UP**. In the Graph tab, `up{job="myapp"}` → `1` (0 would mean the scrape failed).
**Two or more metrics:**
- `http_requests_total{status="200"}`: a counter of requests served (it only goes up).
- `http_request_duration_seconds_bucket`: a latency histogram (used for p95).
- `process_cpu_seconds_total{job="prometheus"}`: CPU (Central Processing Unit) seconds the process has used. (Mac: the Python app shows `process_*` metrics only on Linux, so read this one from the `prometheus` job.)

**🧠 In simple words:** The app shows its numbers on a `/metrics` web page, and Prometheus visits that page every few seconds and stores each value with a timestamp. The Targets page and the `up` metric prove the scraping works. In short: expose `/metrics`, list it under `scrape_configs`, and check that Targets says UP.

## Lab 2 · Grafana dashboard with request rate, error and latency panels; explain each panel
**Answer (what to write in the exam):**
```bash
docker network create monitoring
docker network connect monitoring prometheus
docker run -d --name grafana --network monitoring -p 3000:3000 grafana/grafana
# Browser: http://localhost:3000 (first login admin/admin, then set a new password)
# Connections → Data sources → Add → Prometheus → URL http://prometheus:9090 → Save & test
# Dashboards → New → Add visualization → Prometheus → paste PromQL → set unit → Apply
# Repeat for the 3 panels below → Save dashboard ("App RED")
```
| Panel (Time series) | PromQL | Unit | What it indicates |
|---|---|---|---|
| Request rate | `sum(rate(http_requests_total[5m]))` | req/s | Traffic: how many requests per second the app is serving. |
| Error rate | `sum(rate(http_requests_total{status=~"5.."}[5m]))` | req/s | Server errors (5xx) per second. It should stay near 0. |
| Error % (optional) | `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))` | percent (0.0-1.0) | The fraction of requests that fail. |
| Latency p95 | `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))` | seconds | 95% of requests finished faster than this value. Shows slowness that users feel. |

**What each part means:**
- `rate(x[5m])`: the per-second increase of a counter, averaged over the last 5 minutes (it handles counter resets).
- `sum(...)`: adds all label series (all statuses or instances) into one line.
- `{status=~"5.."}`: a regular-expression match, meaning any status from 500 to 599.
- `sum by (le)`: keeps the histogram's `le` ("less than or equal") bucket label, which `histogram_quantile` needs to work out p95.
- Data source URL `http://prometheus:9090`: both containers are on the `monitoring` network, so Grafana reaches Prometheus by its container name.
- If the supplied environment has no app, use Prometheus's own metrics: `prometheus_http_requests_total` (its status label is `code`) and `prometheus_http_request_duration_seconds_bucket`.

**Verify:** "Save & test" → `Successfully queried the Prometheus API.` With the app from Lab 1, the panels show about 10 req/s, about 25% errors and a p95 of about 0.2 s.

**🧠 In simple words:** Grafana does not store data. It asks Prometheus with PromQL and draws the answers as graphs. Rate, errors and latency (RED) answer "how busy is it, how often does it fail, how slow is it", which is enough to spot most problems. In short: add Prometheus as a data source and build three panels: `rate()` for traffic, a `5..` filter for errors, and `histogram_quantile(0.95, …)` for latency.

## Lab 3 · GitOps with Argo CD: change the replica count in Git and see the cluster reconcile
**Answer (what to write in the exam):**
```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
kubectl -n argocd port-forward svc/argocd-server 8080:443 &        # UI: https://localhost:8080
kubectl apply -f argocd-app.yaml                   # register the app with Argo CD
kubectl get deploy web -n web                      # READY 2/2
# In the Git repo, edit k8s/deployment.yaml: replicas: 2 -> 4
git add k8s/deployment.yaml && git commit -m "Scale web to 4 replicas" && git push origin main
argocd app get web                                 # or watch the UI
kubectl get deploy web -n web                      # READY 4/4
```
`argocd-app.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-user>/gitops-demo.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: web
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```
`k8s/deployment.yaml` (in the Git repo)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```
**What each line does:**
- `apply --server-side`: installs Argo CD (its CRDs are too big for a normal client-side apply). The `argocd-initial-admin-secret` holds the first `admin` password.
- `repoURL` / `targetRevision` / `path`: which Git repo, branch and folder hold the desired state. For a private repo, add a token under Settings → Repositories in the UI, never in this YAML.
- `destination`: which cluster (`https://kubernetes.default.svc` = the cluster Argo CD runs in) and which namespace to deploy into. `CreateNamespace=true` creates `web` if it is missing.
- `automated`: sync automatically when Git changes. `prune: true` deletes objects that were removed from Git. `selfHeal: true` undoes manual changes made in the cluster (drift).
- `git commit` + `push`: the only way to change the app. Argo CD polls Git about every 3 minutes (run `argocd app sync web` or click Refresh to sync sooner).

**Verify:** `argocd app get web` → `Sync Status: Synced to main (<commit>)`, `Health Status: Healthy`, and the UI shows the app green with **Synced + Healthy**. `kubectl get deploy web -n web` → `READY 4/4`. Drift test: `kubectl scale deploy web -n web --replicas=1` → selfHeal sets it back to 4 within seconds.

**🧠 In simple words:** In GitOps, Git holds the desired state and Argo CD keeps comparing the cluster with Git, fixing any difference. You never run `kubectl apply` for the app yourself: you change a YAML file, commit and push, and Argo CD rolls it out. In short: Application YAML points Argo CD at a repo folder; edit `replicas` → commit → push → Argo CD syncs → `kubectl get deploy` shows the new count.
