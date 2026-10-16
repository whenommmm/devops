# Session 14 · Monitoring, Observability & GitOps (Oct 17, Session 2)

**Time:** ~110 min · **Concepts:** 82 → 85 · **Answers:** `MY_ANSWERS.md` or say "Monitoring questions" · **Answer key:** `14_monitoring-observability-gitops_ANSWERS.md`

**Before you start:**
```bash
colima status                          # Docker must be running
minikube status                        # Kubernetes must be running (minikube start if not)
kubectl get nodes                      # minikube   Ready
lsof -iTCP:9090 -iTCP:3000 -sTCP:LISTEN   # nothing printed = ports 9090 and 3000 are free
```
> ⚠️ **Never remove the `minikube` or `localstack` containers.** Leave the `argocd`, `ingress-nginx` and `final-app` namespaces and the existing Argo CD Applications alone (your course assignment uses them).
>
> Text after `#` in commands is a note for you: don't type it (your zsh would pass it on as arguments), or run `setopt interactivecomments` first.

## 🔁 Warm-up
**W1.** A Trivy gate uses `--severity HIGH,CRITICAL --ignore-unfixed --exit-code 1`. The only finding is a HIGH CVE in `libc` with Status `affected` and no Fixed Version. What happens?
- A) Exit 1: HIGH findings always fail this gate
- B) Exit 0: unfixed findings are left out
- C) Exit 1: unfixed findings count as CRITICAL
- D) Exit 0: libc is skipped as an OS package

**W2.** A nightly rescan workflow has `on: schedule: - cron: "0 2 * * *"`. When does it run? (GitHub runs schedules in **UTC**, Coordinated Universal Time; **IST**, India Standard Time, is UTC + 5:30.)
- A) Every day at 02:00 UTC
- B) Every day at 2:00 PM UTC
- C) Every 2 hours, on the hour
- D) Every day at 02:00 IST

**W3.** Your PR's security gate failed. You fix `requirements.txt` **on the same branch** and push. What happens to the PR?
- A) You must open a new PR for the new commit
- B) Close and reopen it to rerun the checks
- C) Nothing until the fix is merged into main
- D) It updates, and its checks run again

## Concept 82 · Monitoring vs observability, and the three pillars
**Words you'll learn:** telemetry, monitoring, alert, observability, known unknowns, unknown unknowns, metric, time series, log, structured log, trace, span, trace ID, context propagation, golden signals, latency, percentile (p95), traffic, errors, saturation, SLI, SLO, SLA, error budget, ELK, EFK, Loki, Jaeger, OpenTelemetry.

**Problem:** Saturday night, customers say checkout is slow, yet the dashboard shows CPU 40%, memory 60%, all servers up. One checkout crosses an API gateway, the orders and payments services, and the database (microservices); which one is slow? The team guesses for two hours.
- **Telemetry** = data a system sends out about itself: numbers, messages, timings.
- **Monitoring** = watching **known** signals and **alerting** (an automatic notification when a rule breaks, e.g. "error rate > 5% for 5 minutes → message the on-call engineer"). It answers "**is** it healthy?" and covers **known unknowns** (problems you predicted).
- **Observability** = how well you can understand what's happening inside a system **just from the data it sends out**, including problems nobody predicted (**unknown unknowns**), by asking new questions without shipping new code. It answers "**why**?".
- **Analogy (the course's): a car.** Monitoring = the "engine temperature HIGH" warning light; observability = the mechanic's tools (coolant level, engine logs, temperature history) that show **why**. ⚠️ Not competitors: monitoring is **part** of observability; monitoring tells you **when** to look, observability helps you find **why**.

| | **Monitoring** | **Observability** |
|---|---|---|
| Question | Is something wrong? | **Why** is it wrong? |
| Problems | known failure signals, planned in advance | unknown problems you explore |
| Output / view | dashboards and alerts; health overview | metrics + logs + traces you query freely; deep investigation |

| Pillar | What it is | Answers | Tools |
|---|---|---|---|
| **Metrics** | named numbers (`http_requests_total`); measured repeatedly = a **time series** of (time, value) pairs; small and cheap | how much? how often? | Prometheus |
| **Logs** | timestamped records of events; a **structured log** uses fields (JSON) so tools can filter it | what exactly happened? | ELK/EFK, Loki |
| **Traces** | one request's path across services, made of **spans** (one piece of work in one service), all sharing a **trace ID** that is passed along in HTTP headers (**context propagation**); e.g. checkout: gateway 20 ms → orders 80 → payments 120 → **database 600 ms** | where did the time go? | Jaeger, OpenTelemetry |

- **Four golden signals** (Google's **SRE**, Site Reliability Engineering, book):
  - **Latency** (how long requests take, e.g. p95 = 300 ms) · **Traffic** (demand, e.g. 500 requests/s)
  - **Errors** (failures, e.g. 2% HTTP 5xx) · **Saturation** (how full a resource is: CPU 90%, disk 95%, a 10,000-job queue)
- A **percentile**: p95 = 300 ms means 95% of requests were faster. Averages hide slow ones (99 × 10 ms + one 10 s ≈ 110 ms average).
- **SLI** (**Service Level Indicator**) = the **measurement** ("% of requests OK in under 300 ms").
- **SLO** (**Service Level Objective**) = the team's internal **target** ("99.9% over 30 days").
- **SLA** (**Service Level Agreement**) = a **contract** with customers, with **penalties** (refunds, credits), usually looser (SLA 99.5%, SLO 99.9%).
- **Error budget** = 100% − SLO = the failure allowed: 30 × 24 × 60 × 0.001 = **43.2 minutes** per 30 days at 99.9% (99% → 7.2 h; 99.99% → 4.32 min). Budget left → ship features; nearly used → slow down and fix reliability.
- **Tools:** logs: **ELK** = **Elasticsearch** (stores and searches logs) + **Logstash** (collects and reshapes them) + **Kibana** (the web UI); **EFK** swaps in **Fluentd**/**Fluent Bit**; **Loki** (Grafana Labs) indexes only labels, so it's cheaper.
  - Traces: **Jaeger** stores and shows them; **OpenTelemetry** (**OTel**) is the vendor-neutral standard for **producing** metrics, logs and traces (SDKs + a collector).
  - On Kubernetes, apps log to **stdout/stderr** and an agent such as Fluent Bit on each node ships the logs.

### 🛠 Do (~8 min): metrics vs logs, on your own containers
```bash
docker run -d --name s14-web -p 8085:80 nginx:alpine                    # a small web server
for i in 1 2 3; do curl -s -o /dev/null http://localhost:8085/; done    # 3 normal requests (page thrown away)
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8085/missing-page   # print only the status code: 404
docker logs s14-web 2>&1 | tail -5                   # LOGS: one EVENT per line, e.g. "GET /missing-page HTTP/1.1" 404 (2>&1 adds the error log)
docker logs s14-web 2>/dev/null | grep -c '" 404 '   # count the 404 lines: 1 (a metric would give this instantly)
docker stats --no-stream s14-web                     # METRICS: NUMBERS (CPU %, memory, I/O, PIDs); stored every 15 s = time series
kubectl top nodes                                    # node CPU (millicores) and memory, from metrics-server
echo "30*24*60*0.001" | bc                           # the 99.9% error budget: 43.200 minutes
docker rm -f s14-web                                 # clean up
```

### ✅ Check 82
**82.1** Users report "checkout is slow". Your dashboard shows p95 latency for checkout jumped from 300 ms to 2 s, and nothing else looks wrong. Which signal best shows **which service** in the chain is slow?
- A) A CPU gauge for the whole cluster
- B) A trace of one slow checkout request
- C) The total request counter of the API
- D) An uptime check hitting the home page

**82.2** Which of these is a **saturation** signal?
- A) Requests per second arriving at the API
- B) Share of requests answered with a 5xx code
- C) How full the CPU, memory or job queue is
- D) Time taken to serve each checkout request

**82.3** (one line) Write the full forms of SLI, SLO and SLA, and say which one carries penalties if it's missed.

## Concept 83 · Prometheus: collecting and querying metrics
**Words you'll learn:** Prometheus, CNCF, target, `/metrics` endpoint, exporter, node-exporter, scrape, scrape interval, job, pull model, push model, time-series database (TSDB), retention, service discovery, Pushgateway, alerting rule, pending, firing, Alertmanager, counter, gauge, histogram, summary, label, cardinality, PromQL, `up`, `rate()`, `sum by`.

**Problem:** 50 microservices each know useful numbers, but nothing **collects** them, keeps their **history**, lets you **query** across them, or **alerts**.
- **Prometheus** = a free, open-source **monitoring system with a built-in time-series database**, the second **CNCF** (**Cloud Native Computing Foundation**, which looks after Kubernetes) project to graduate. Metrics only, with a web **UI** (User Interface) and an HTTP **API** (Application Programming Interface).
  - The course: *"Prometheus goes around asking applications: 'Give me your metrics.'"*
  - **Analogy: a meter reader** visits every house on a schedule and writes each reading in a ledger. Prometheus is the reader, each app's `/metrics` page is the meter, the time-series database is the ledger.
- **How it fits:** Prometheus (:9090) **scrapes** (`GET /metrics`, a **pull**) each **target** (anything it collects from) every **scrape interval** (`scrape_interval: 15s`); a **job** is a named group of similar targets.
  - It stores the data in its **TSDB** (**time-series database**; **retention** 15 days by default), serves a UI + API (Grafana queries it with PromQL), and checks **alerting rules** (`up == 0`).
  - Firing alerts go to **Alertmanager** (:9093), which groups, de-duplicates, **silences** and **routes** them (email, Slack, PagerDuty).
- An **exporter** **translates** a system's numbers into Prometheus format (**node-exporter**: a Linux machine's CPU, memory, disk, network, port 9100).
- **Service discovery** asks a source for targets (on Kubernetes, the API server), so new Pods are scraped automatically.
- The **Pushgateway** (:9091) is for **short-lived jobs**: they **push** metrics there, and Prometheus scrapes it.
- A `/metrics` page: `# HELP` (description), `# TYPE` (metric type), then `name{label="value",…} number`, e.g. `prometheus_http_requests_total{code="200",handler="/metrics"} 42`.

| Type | Behaviour | Example | Use |
|---|---|---|---|
| **Counter** | **only goes up** (resets to 0 on restart); names end in `_total` | total requests served | **`rate()`** for "per second" |
| **Gauge** | goes up **and** down | memory in use, queue length | the value directly |
| **Histogram** | counts observations into **buckets**; exposes `_bucket`, `_sum`, `_count` | request duration | p95 via `histogram_quantile()` in PromQL; combines across Pods |
| **Summary** | the app computes percentiles itself | request duration | can't be combined across Pods |

> ⚠️ "Jobs waiting in the queue **right now**" = **gauge**; "jobs processed **since start**" = **counter**. A counter dropping to 0 means a restart (`rate()` handles it).
- A **label** (`key="value"`) splits a metric: **each unique label combination is its own time series** (like Pod labels). **Cardinality** = the number of series: never label by user ID or email, or Prometheus runs out of memory.
- **PromQL** (**Prometheus Query Language**): `up` (1 = last scrape worked, 0 = failed) · `up == 0` (only down targets) · `metric{code="200"}` (label filter) · `metric[5m]` (a range of samples)
  - **`rate(http_requests_total[5m])`** (average per-second increase) · `sum by (job) (rate(…[5m]))` (one result per job) · `sum(up)`.
  - ⚠️ `rate()` needs **at least two samples** in its window, so it shows **no data** right after start-up.

| | **Pull** (Prometheus's default) | **Push** |
|---|---|---|
| Who connects | the server fetches from each target | each target sends to the server |
| Target down? | noticed **automatically** (scrape fails, `up` = 0) | only when data **stops** arriving |
| Short-lived jobs | may end before a scrape → **Pushgateway** | natural fit |
| Examples | Prometheus | Pushgateway, Graphite, InfluxDB, cloud agents |

- **Alerting rule** (awareness): `alert: TargetDown`, `expr: up == 0`, **`for: 5m`**, `labels: severity: critical`. States: **inactive** → **pending** (true, but not yet for the whole `for:`) → **firing** (sent to Alertmanager). If it turns false while pending, **nobody is notified**: `for:` stops a 10-second blip from waking anyone.

### 🛠 Do (~15 min): run Prometheus (and Grafana, for Concept 84) with Docker Compose
The two `cat > file <<'EOF'` commands write the lines up to `EOF` into `prometheus.yml` and `compose.yaml`. ⚠️ The course's file says `prom/prometheus:v3.5.0` (Docker Hub); the **quay.io** copy is the same image, already on your Mac, and avoids Docker Hub's `429 Too Many Requests`.
```bash
mkdir -p ~/devops-practice/monitoring && cd ~/devops-practice/monitoring
cat > prometheus.yml <<'EOF'
global:
  scrape_interval: 15s               # scrape every target every 15 seconds
scrape_configs:
  - job_name: prometheus             # Prometheus scrapes its OWN /metrics page
    static_configs:
      - targets: ["prometheus:9090"]  # "prometheus" = the Compose service name (DNS)
  - job_name: grafana                # Grafana also publishes a /metrics page
    static_configs:
      - targets: ["grafana:3000"]
EOF
cat > compose.yaml <<'EOF'
services:
  prometheus:
    image: quay.io/prometheus/prometheus:v3.5.0   # same image as prom/prometheus:v3.5.0
    ports:
      - "9090:9090"                               # Prometheus web UI + API
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro   # bind mount: our config, read-only
  grafana:
    image: grafana/grafana:12.1.1
    ports:
      - "3000:3000"                               # Grafana web UI
    volumes:
      - grafana-data:/var/lib/grafana             # named volume: dashboards survive "down"
    depends_on:
      - prometheus
volumes:
  grafana-data:
EOF
docker compose config --quiet && echo "compose file OK"    # validate the YAML: compose file OK
docker compose up -d && docker compose ps                   # monitoring-prometheus-1 and monitoring-grafana-1, both Up
curl -s http://localhost:9090/metrics | grep "^prometheus_http_requests_total" | head -3   # raw counter lines
curl -s 'http://localhost:9090/api/v1/query?query=up'      # the HTTP API Grafana uses: JSON, "status":"success"
```
- Open **http://localhost:9090** → **Status → Target health**: `prometheus` and `grafana` both **UP**. On the **Query** page run `up`: `up{instance="grafana:3000", job="grafana"} 1` and the same for prometheus (`instance` and `job` are added automatically).
- After a minute (so `rate()` has two samples) try: `prometheus_http_requests_total` (many series: labels!) · `prometheus_http_requests_total{handler="/metrics"}` (only grows: a counter)
  - `rate(prometheus_http_requests_total{handler="/metrics"}[1m])` ≈ **0.067** (one scrape per 15 s; see the **Graph** tab)
  - `process_resident_memory_bytes / 1024 / 1024` (MB, a gauge) · `sum(up)` = 2 · `up == 0` = empty.
- **Pull notices a dead target:** run `docker compose stop grafana`, wait ~30 s, run `up` again: `up{job="grafana"}` = **0** and Target health shows **DOWN**. Then `docker compose start grafana`. **Leave the stack running: the Grafana part below uses it.**

### ✅ Check 83
**83.1** Which of these should be exposed as a **counter**?
- A) Total HTTP requests served since start-up
- B) Number of jobs waiting in a queue right now
- C) Memory the process is using at this moment
- D) Temperature of the server room at this moment

**83.2** A backup script runs for 10 seconds every night. Prometheus scrapes every 15 seconds, and the script's metrics never appear. What's the standard fix?
- A) Lower `scrape_interval` to 1s for every target
- B) Send the metrics straight to Alertmanager instead
- C) Add the script to Grafana as a new data source
- D) Push them to a Pushgateway, which gets scraped

**83.3** (one line) Why is Prometheus called "pull-based", and what's one advantage of pulling?

## Concept 84 · Grafana: dashboards and visualisation
**Words you'll learn:** Grafana, data source, dashboard, panel, visualization, query editor, time range, refresh interval, variable (templating), dashboard import, provisioning, Grafana alerting, contact point.

**Problem:** at 3 a.m. the on-call engineer doesn't want to type PromQL: she wants **one screen** with traffic, errors, latency and saturation side by side, red where things are bad.
- **Grafana** (Grafana Labs, free, open-source) is a **visualisation** tool that turns data into **dashboards**.
  - **It doesn't collect or store your metrics**: it sends live queries to a **data source** (a connection it can query: Prometheus for metrics, Loki/Elasticsearch for logs, Jaeger/Tempo for traces, MySQL/PostgreSQL, CloudWatch…) whenever a dashboard loads or refreshes.
  - Course: **Prometheus → collects/stores metrics; Grafana → visualises them.**
  - **Analogy: the TV weather presenter.** Weather stations measure and record (Prometheus); the presenter measures nothing and only shows their data as charts (Grafana).
- A **dashboard** = one page of related charts; a **panel** = one box: queries + a **visualization** (**Time series** line, **Stat** big number that can turn red past a threshold, **Gauge** dial, **Table**, bar chart, heatmap, logs).
  - The **query editor** has **Builder** (click) and **Code** (type PromQL) modes. The **time range** ("Last 6 hours") and **refresh interval** apply to every panel. A **variable** (**templating**) is a dropdown (`$job`) that updates every panel using it.
  - **Import** shared dashboards by ID (**1860** = "Node Exporter Full"); **provisioning** loads data sources and dashboards from files at start-up.
  - **Grafana alerting** (awareness): an alert rule on a query + a **contact point** (email, Slack).

| | **Prometheus** | **Grafana** |
|---|---|---|
| Main job | **collect** (scrape), **store**, **query**; evaluate alert rules | **visualise** data in dashboards |
| Stores your metrics? | ✅ yes, in its TSDB | ❌ no (only its settings and dashboards) |
| Data sources / language | its targets / PromQL | many / each source's own language |
| Default port / UI | 9090 / basic | 3000 / rich |

> ⚠️ **Data-source URL trap:** inside Grafana's container, `localhost` is **Grafana itself**, so `http://localhost:9090` gives **connection refused**. Use the Compose **service name**: **`http://prometheus:9090`**. (`localhost:9090` works only from your Mac's browser, through `-p 9090:9090`.)

### 🛠 Do (~15 min): connect Grafana to Prometheus and build a dashboard
1. `curl -s http://localhost:3000/api/health` → `{"database": "ok", "version": "12.1.1", …}` (the Prometheus + Grafana Compose stack you started above is still up). Open **http://localhost:3000**, log in as **admin / admin**, and click **Skip** on the new-password screen (lab only).
2. **Connections → Data sources → Add data source → Prometheus.** Try URL `http://localhost:9090` → **Save & test** fails (`connection refused`); change it to **`http://prometheus:9090`** → "Successfully queried the Prometheus API."
3. **Dashboards → New → New dashboard → + Add visualization**, pick Prometheus, switch the editor to **Code**, query `up`, **Run queries**, change the visualization to **Stat** (two **1**s), title `Targets up`; then query `sum(up)` → **2**. **Save dashboard** as `My first dashboard`.
4. **Add → Visualization**: `rate(prometheus_http_requests_total{handler="/metrics"}[1m])`, keep **Time series**, title `Scrapes per second`; time range **Last 15 minutes**, refresh **10s**: lines near 0.067. Save.
5. (Optional) **Dashboard settings → Variables → Add variable**: name `job`, type **Query**, query `label_values(up, job)`; change the Stat query to `up{job="$job"}`: a **job** dropdown appears.
6. **Prove Grafana stores nothing:** `docker compose stop prometheus`, refresh: **No data**. Then `docker compose start prometheus`. When finished, `docker compose down` removes the containers and network (the dashboard survives in the volume; `down -v` deletes it).

### ✅ Check 84
**84.1** Which statement about Prometheus and Grafana is correct?
- A) Grafana stores the metrics; Prometheus draws charts
- B) Grafana scrapes /metrics endpoints on its own schedule
- C) Prometheus stores and queries; Grafana visualises
- D) Prometheus can't run unless Grafana is installed

**84.2** Prometheus started 20 seconds ago with a 15-second scrape interval. In Grafana, `http_requests_total` shows values, but `rate(http_requests_total[5m])` shows **No data**. Why?
- A) `rate()` works only on gauges, not on counters
- B) `rate()` needs two samples, and there's only one
- C) Grafana needs a variable named `$range` first
- D) The `[5m]` window must be shorter than 15 seconds

## Concept 85 · GitOps and Argo CD
**Words you'll learn:** GitOps, desired state (recap), live state, reconciliation loop (recap), drift, single source of truth, the four GitOps principles, push-based CD (recap), pull-based GitOps, app repo, config repo, Argo CD, Application, project, source (`repoURL`, `targetRevision`, `path`), destination, sync, Synced / OutOfSync, health (Healthy / Progressing / Degraded / Missing), manual vs automated sync, self-heal, prune, `CreateNamespace`, finalizer, Flux.

**Problem:** Friday 6 p.m., Arjun runs `kubectl scale … --replicas=10` and `kubectl edit` to "quickly" fix production, and tells nobody. Monday's deploy re-applies the old YAML and wipes his fix, and nobody knows who changed what. The cluster has **drifted**: its **live state** (what actually runs) matches nothing written down.
- **GitOps** = **Git holds the desired state** (declarative files) and **an agent inside the cluster continuously makes the cluster match Git**.
  - **Desired state** = what you want; **live state** = what runs; **drift** = any difference; **single source of truth** = Git, so if they disagree, **Git wins**.
  - It's the K8s **reconciliation loop** one level up: read the desired state from Git → compare with the live cluster → apply the difference, every few minutes.
  - **Analogy: a building site.** The blueprint (Git) is the only official plan; a site supervisor (the agent) keeps walking the site and moves back any wall a worker moved; to change the building, you change (and review) the blueprint.

| GitOps principle (OpenGitOps) | Meaning |
|---|---|
| **1. Declarative** | the desired state is **described** (YAML), not step-by-step commands |
| **2. Versioned and immutable** | stored in **Git**: every change is a commit, history is kept |
| **3. Pulled automatically** | agents **pull** it from Git; no person or CI pushes it in |
| **4. Continuously reconciled** | agents keep comparing live vs desired, and correct drift |

Why Git: history, review (PRs), diffs, a rollback point, an audit trail (`git diff` shows `-  replicas: 2` / `+  replicas: 3`; `git log` shows who and when). Push vs pull: `CI ──kubectl/helm──► cluster` vs `CI ──commit──► Git ◄──pull── Argo CD (inside the cluster)`.

| | **Push-based CD** (CI → cluster) | **Pull-based GitOps** (cluster ← Git) |
|---|---|---|
| Who applies changes | the CI runner | an agent **inside** the cluster |
| Cluster credentials stored | in CI secrets (outside the cluster) | nowhere outside the cluster |
| Network | CI must reach **into** the API server | the agent only connects **out** to Git (works for local minikube) |
| Drift (manual `kubectl` edits) | unnoticed until the next deploy | **detected**; can be **reverted automatically** |
| Rollback / audit trail | rerun an old pipeline or `rollout undo` / pipeline logs | **`git revert`** / Git history |

- **App repo** (source + Dockerfile; CI tests, scans, builds and pushes `shop:3f9c2a1`) vs **config repo** (Kubernetes YAML or Helm values; CI, a bot or a PR commits the new tag there, and Argo CD watches it, polling about **every 3 minutes** or instantly via a webhook).
  - A bad release → `git revert` in the config repo.
  - Separate repos: config changes don't rebuild code, approvers can differ, and the history is a clean deployment record.
- **Argo CD** = a free, open-source **GitOps agent for Kubernetes** (CNCF graduated) that runs **as Pods inside the cluster**, with a web UI, a CLI (`argocd`) and an API.
  - Course: **Git = desired state, cluster = actual state, Argo CD = keeps them synchronised.**
  - Pods in `argocd`: `argocd-server` (API + UI), `argocd-repo-server` (clones Git, renders YAML/Helm/Kustomize), `argocd-application-controller` (the reconciliation loop), `argocd-redis` (cache), `argocd-dex-server` (single sign-on), plus ApplicationSet and notifications controllers.
- You declare what to watch with an **Application**, a **CRD** (**Custom Resource Definition**: a new object type that Argo CD adds to Kubernetes). Keep its file **outside** the folder it points at, and apply it once yourself:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd                     # Applications live in Argo CD's own namespace
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # deleting the Application also deletes its resources
spec:
  project: default                      # a project groups apps and limits what they may do
  source:                               # WHERE the desired state is
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD                # branch, tag or commit (HEAD = the default branch)
    path: guestbook                     # folder inside the repo
  destination:                          # WHERE to deploy it
    server: https://kubernetes.default.svc   # this same cluster
    namespace: guestbook
  syncPolicy:
    automated:                          # sync automatically when Git changes
      prune: true                       # delete resources removed from Git
      selfHeal: true                    # undo manual changes made in the cluster
    syncOptions:
      - CreateNamespace=true            # create the "guestbook" namespace if missing
```
- **Sync status** (does the cluster match Git?): **Synced** (live = Git) or **OutOfSync** (a new commit, or drift).
- **Health status** (is the app working?): **Healthy** (running and ready), **Progressing** (still rolling out), **Degraded** (failed: CrashLoopBackOff, a stuck rollout) or **Missing** (not created yet).
- ⚠️ **Synced ≠ Healthy:** "Synced + Degraded" = Argo CD applied exactly what Git says, but that doesn't work: the fix goes into **Git**.
- **Manual sync** (default): shows OutOfSync and waits for **Sync** (or `argocd app sync guestbook`). **Automated** (`automated:`): syncs when **Git** changes.
  - **Self-heal** (`selfHeal: true`): also reverts **cluster** drift within seconds; ⚠️ without it, automated sync ignores manual cluster changes and just shows **OutOfSync**.
  - **Prune** (`prune: true`): deletes resources removed from Git; ⚠️ **off** by default.
- Roll back with **`git revert`** (`argocd app rollback` is refused while automated sync is on).
- **Flux** (awareness): the other CNCF-graduated GitOps tool: small controllers (Git source, Kustomize, Helm), set up with `flux bootstrap`, **no built-in web UI**.

### 🛠 Do (optional, heavy: ~20 min, uses ~1 GB of RAM in minikube): Argo CD on your minikube
Argo CD is **already installed** on your Mac, so skip the install.
- Only if the first command shows nothing: `kubectl create namespace argocd`, then `kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml` (`--server-side` because some CRDs are too big for a normal apply).
- ⚠️ Leave the existing Applications alone.
- ⚠️ **Apple Silicon:** the guestbook image is **amd64 only**; Colima usually emulates it, but if its Pod shows `CrashLoopBackOff` / `exec format error`, the app is **Synced but Degraded**: a real "Synced ≠ Healthy" example.
```bash
kubectl get pods -n argocd      # ~7 Pods Running (argocd-server, repo-server, application-controller…)
kubectl port-forward svc/argocd-server -n argocd 8080:443   # in a SECOND tab; open https://localhost:8080 (self-signed cert: Advanced → Proceed)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo   # admin's password
mkdir -p ~/devops-practice/gitops && cd ~/devops-practice/gitops
code guestbook-app.yaml                                     # paste the Application YAML above, save
kubectl apply -f guestbook-app.yaml                         # "application.argoproj.io/guestbook created" (a finalizer Warning is harmless)
kubectl get applications -n argocd                          # guestbook   Synced   Healthy (others may be listed too)
kubectl get all -n guestbook                                # Deployment + Service guestbook-ui, created by Argo CD, not by you
kubectl scale deployment guestbook-ui --replicas=3 -n guestbook   # make drift…
kubectl get deployment guestbook-ui -n guestbook -w         # …3, then back to 1 within seconds (self-heal); Ctrl + C
kubectl patch application guestbook -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":{"selfHeal":false}}}}'   # (optional) self-heal off
kubectl scale deployment guestbook-ui --replicas=3 -n guestbook   # 3 stays
kubectl get applications -n argocd guestbook                # OutOfSync
kubectl delete -f guestbook-app.yaml                        # clean up (finalizer removes its resources)
kubectl delete namespace guestbook --ignore-not-found       # then Ctrl + C the port-forward
```

### ✅ Check 85
**85.1** Which is **NOT** one of the four GitOps principles?
- A) Declarative: the desired state is described
- B) Versioned and immutable: kept in Git history
- C) Pushed by CI: a pipeline runs kubectl apply
- D) Continuously reconciled: agents correct drift

**85.2** Argo CD shows an Application as **Synced** but **Degraded**. What does that mean?
- A) Git is ahead of the cluster, and a sync is due
- B) The repo can't be reached, so the status is stale
- C) Someone changed the cluster by hand with kubectl
- D) Cluster matches Git, but the app isn't working

**85.3** (one line) With GitOps, how do you roll back a bad release, and why is that better than running `kubectl rollout undo`?

## ➕ Small but important (monitoring, observability, GitOps): read before the quiz
| Item | What it is |
|---|---|
| **Blackbox vs whitebox monitoring** | **blackbox** = test from **outside** like a user ("does / answer 200 within 1 s?", **blackbox-exporter**); **whitebox** = the app's **internal** metrics and logs. You need both |
| **Alert fatigue** | so many (unimportant, flapping, duplicate) alerts that people **ignore** them. Fix: alert on **symptoms users feel**, use `for:`, group in Alertmanager, delete alerts nobody acts on |
| **RED** / **USE** methods | services (APIs, websites): **R**ate, **E**rrors, **D**uration / resources (CPU, memory, disk, network): **U**tilization, **S**aturation, **E**rrors |
| **kube-state-metrics** | the **state of Kubernetes objects** from the API server: desired vs available replicas, Pod phase, restarts |
| **metrics-server** | **current** CPU/memory per node from the kubelet, for **`kubectl top`** and the **HPA**; no history, not for Prometheus |
| **node-exporter** | a DaemonSet exposing each **machine's** CPU, memory, disk, network on port **9100** |
| **kube-prometheus-stack** (Helm) | Prometheus (via the **Prometheus Operator**), Alertmanager, Grafana, node-exporter, kube-state-metrics + dashboards: `helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace`; targets via a **ServiceMonitor** |
| **Log levels** / **structured logs** | **TRACE < DEBUG < INFO < WARN < ERROR < FATAL** (production logs INFO+) / JSON fields, with a `trace_id` linking a log line to its trace |
| **MTTD / MTTR** | **M**ean **T**ime **T**o **D**etect / **M**ean **T**ime **T**o **R**ecover (Restore/Repair); MTTR relates to the **DORA** (DevOps Research and Assessment) metric "time to restore service" |
| **Default ports** | Prometheus **9090**, Pushgateway **9091**, Alertmanager **9093**, node-exporter **9100**, Grafana **3000**, Loki **3100** |
| **Retention** / **recording rules** | 15 days by default (`--storage.tsdb.retention.time`; long-term: Thanos, Mimir) / pre-calculate an expensive query into a new metric |
| **`argocd` CLI** / extras | `argocd app list` · `get` · `sync` · `diff` (Git vs live) · `history` · `delete guestbook` (`brew install argocd`) / **Image Updater** commits new tags, **app-of-apps**, **ApplicationSet** templates |
| **HPA + GitOps** | Git says `replicas: 2`, the HPA scales to 5, and self-heal fights it: leave `replicas` out of Git (or ignore that field) |

**Quick checks:** (1) Which component answers "how many replicas of my Deployment are available right now?" as a Prometheus metric: kube-state-metrics, metrics-server or node-exporter? (2) RED or USE for a disk? For a REST API?
*Answers:* (1) **kube-state-metrics** (it reports Kubernetes object state; metrics-server only gives current CPU/memory for `kubectl top`/HPA, and node-exporter gives machine stats). (2) **USE** for a disk (a resource: utilization, saturation, errors); **RED** for a REST API (a request-driven service: rate, errors, duration).

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory
**Q1.** Prometheus and Grafana run as two services in one Compose file, with ports `9090:9090` and `3000:3000`. In Grafana's Prometheus data source you enter `http://localhost:9090`, and **Save & test** fails with `connection refused`. Opening `http://localhost:9090` in your Mac's browser works fine. What's the cause?
- A) Grafana needs Prometheus's IP, since names never resolve
- B) Port 9090 must also be published on the Grafana service
- C) Inside Grafana's container, localhost is Grafana itself
- D) Prometheus rejects queries that don't come from a browser

**Q2.** Your SLO is 99.9% successful requests over 30 days. Outages have already used 30 minutes this month, and the team wants to ship a risky change today. What does the error budget say?
- A) About 13 minutes remain, so favour reliability
- B) The budget resets every day, so ship the change
- C) About 43 minutes remain; outages aren't counted
- D) It's used up, since 30 min is over 0.1% of 30 days

**Q3.** An Argo CD Application has `automated: { prune: true }` but **no** `selfHeal`. Git says `replicas: 2`. Someone runs `kubectl scale deployment shop --replicas=5`. Nobody touches Git. What happens?
- A) Argo CD reverts the Deployment to 2 within seconds
- B) Argo CD commits `replicas: 5` back to the Git repo
- C) Prune deletes the Deployment as it differs from Git
- D) It stays at 5, and the app is shown as OutOfSync

**Q4.** A Prometheus alerting rule has `expr: up == 0` and `for: 5m`. A target goes down for 3 minutes, then recovers. What happens?
- A) The alert fires at once and resolves after 3 minutes
- B) It goes pending, never fires, and no one is notified
- C) It fires after 5 minutes, although the target is back
- D) Alertmanager waits 5 minutes, then sends one message

**L1.** Explain monitoring vs observability in contrast form, with one example of each.

**L2.** Explain push-based CD vs pull-based GitOps in contrast form, including where the cluster credentials live.

### ✍️ Write the command
**C1.** In `~/devops-practice/monitoring`, start completely fresh: remove the Prometheus + Grafana stack **including** its `grafana-data` volume, start it again in the background, then follow Grafana's logs live. Each step should run only if the one before it worked.

**C2.** From your Mac's terminal, fetch Prometheus's own metrics page (port 9090, path `/metrics`) without curl's progress output, and show only the **first 3** lines that **start with** `prometheus_http_requests_total`.

**C3.** Open the Argo CD web UI at `https://localhost:8080` by forwarding port 8080 on your Mac to port 443 of the `argocd-server` Service in the `argocd` namespace.

**C4.** Write the PromQL query that gives the **per-second** rate of `prometheus_http_requests_total`, only for the series whose `handler` label is `/metrics`, averaged over the last **5 minutes**.
