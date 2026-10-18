# Session 14 · Monitoring, Observability & GitOps · FAST (10-min read)
*Watching running systems (Prometheus collects numbers, Grafana draws them), and keeping a cluster matching Git with GitOps and Argo CD.*

## 1 · Monitoring vs observability, and the three pillars
Every server can look "up" while checkout is still slow. **Monitoring** watches signals you chose in advance and sends an **alert** (an automatic notification) when a rule breaks: it answers "is something wrong?". **Observability** is how well you can understand a system from the data it sends out, even for problems nobody predicted: it answers "why?". Analogy: monitoring is a car's "engine hot" warning light, observability is the mechanic's tools that show why; monitoring is part of observability, not a rival.
- **Metrics** are named numbers measured again and again (`http_requests_total`); each becomes a **time series**, a list of (time, value) pairs, answering "how much, how often?" (tool: Prometheus).
- **Logs** are timestamped text records of single events, answering "what exactly happened?" (tools: Loki, Elasticsearch).
- **Traces** follow one request across services as **spans** (one step in one service) sharing a **trace ID**, answering "where did the time go?" (tool: Jaeger).
- An **SLI (Service Level Indicator)** is the measurement, an **SLO (Service Level Objective)** is the team's internal target, and an **SLA (Service Level Agreement)** is a looser contract with customers (say 99.5% against an SLO of 99.9%) that carries penalties such as refunds.
- The **error budget** is 100% minus the SLO: 99.9% over 30 days (43,200 minutes) allows 0.1% × 43,200 = 43.2 minutes of failure; once it is used up, risky releases stop.

⚠️ **Exam trap:** a trace shows which service was slow in one request, whereas a metric shows totals across all requests. Missing the SLO only uses up the error budget, whereas missing the SLA costs money.

## 2 · Prometheus: pulling and querying metrics
Many services know useful numbers, but nothing collects them, keeps history or alerts. **Prometheus** is a free monitoring system with a built-in **TSDB (time-series database)**: every **scrape interval** (say 15 seconds) it **scrapes** each **target**, meaning it requests the app's `/metrics` web page and stores the numbers it gets back. Analogy: a meter reader who visits every house on schedule and logs each reading.
- This is the **pull model** (the server fetches from targets), so a dead target is noticed at once: its scrape fails.
- An **exporter** translates another system's numbers into Prometheus format; **node-exporter**, for example, exposes a Linux machine's CPU (Central Processing Unit), memory and disk.
- A **counter** only goes up (requests served since start), whereas a **gauge** goes up and down (memory in use right now).
- In **PromQL (Prometheus Query Language)**, `up` is 1 if a target's last scrape worked and 0 if it failed.
- `rate(http_requests_total[5m])` gives a counter's average per-second increase over the last 5 minutes; it needs two or more samples, so it shows no data right after start-up.
- An **alerting rule** (such as `up == 0` for 5 minutes) is checked by Prometheus; firing alerts go to **Alertmanager**, which groups, silences and routes them to people.

⚠️ **Exam trap:** Prometheus decides when an alert fires, whereas Alertmanager only delivers it.

## 3 · Grafana: dashboards on top of data sources
On-call engineers want one screen of charts, not typed queries. **Grafana** is a free visualisation tool that collects and stores **no** metrics itself: it sends live queries to a **data source** (a saved connection it can query) each time a dashboard loads or refreshes. Analogy: Prometheus is the weather station that measures; Grafana is the TV presenter who only shows its data.
- A data source can be Prometheus for metrics, Loki or Elasticsearch for logs, and Jaeger or Tempo for traces.
- A **dashboard** is one page of related charts, and a **panel** is one box on it: a query plus a visualisation (line chart, big number, gauge dial, table).
- Grafana saves only its own settings and dashboards, so if Prometheus stops, every panel shows "No data".

⚠️ **Exam trap:** Prometheus scrapes, stores and queries metrics, whereas Grafana only visualises data stored elsewhere.

## 4 · GitOps: Git as the single source of truth
Someone fixes production by hand with `kubectl` (the Kubernetes command-line tool) and tells nobody; the next deploy silently wipes the fix. **GitOps** means Git holds the **desired state** (what you want, as YAML (YAML Ain't Markup Language) files) and an agent inside the cluster keeps making the **live state** (what actually runs) match it; any difference is **drift**, and because Git is the **single source of truth**, Git wins. Analogy: the blueprint (Git) is the only official plan, and a site supervisor (the agent) moves back any wall a worker moved.
- The four GitOps principles: declarative (described, not step-by-step), versioned and immutable (every change is a reviewable Git commit), pulled automatically (by an agent), and continuously reconciled (drift keeps getting corrected).
- In **push-based CD (Continuous Deployment)**, the CI (Continuous Integration) runner runs `kubectl` or `helm` (the Kubernetes package installer) itself, so the cluster credentials sit in CI secrets (its stored keys) and CI must reach into the cluster's API (Application Programming Interface) server.
- In **pull-based GitOps**, the agent runs inside the cluster and only connects out to Git, so credentials never leave the cluster and manual edits show up as drift.
- CI still builds and pushes the image from the **app repo** (source code), then commits the new image tag to the **config repo** (Kubernetes YAML) that the agent watches.

⚠️ **Exam trap:** a GitOps rollback is `git revert`, whereas `kubectl rollout undo` changes only the cluster and leaves it disagreeing with Git.

## 5 · Argo CD: the GitOps agent for Kubernetes
**Argo CD** is a free GitOps tool that runs as Pods (groups of running containers) inside the cluster, pulls from Git and keeps the cluster synchronised with it, with a web UI (User Interface). You tell it what to watch with an **Application**, a new Kubernetes object type (a CRD, Custom Resource Definition) that names a Git **source** and a cluster **destination**.
```yaml
spec:
  source:
    repoURL: https://github.com/me/shop-config.git
    path: k8s
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
`source` is the repo and folder holding the desired state; `automated` syncs whenever Git changes; `selfHeal` also undoes manual cluster changes; `prune` deletes resources whose files left Git (off by default).

- **Sync status** compares the cluster with Git: **Synced** means they match, and **OutOfSync** means they differ (a new commit, or drift).
- **Health status** says whether the app works: **Healthy** (running and ready), **Progressing** (still rolling out) or **Degraded** (failed, such as a container that keeps crashing).
- With `automated` but no `selfHeal`, a manual `kubectl` change stays and the app just shows OutOfSync; with `selfHeal: true`, Argo CD reverts it within seconds.
- Argo CD never writes the cluster's state back into Git; it only reads Git.

⚠️ **Exam trap:** Synced is not Healthy: "Synced + Degraded" means Argo CD applied exactly what Git says, but that configuration is broken, so the fix must go into Git.

## 🎯 Exam checklist (everything that could be asked)
*MCQ = know the fact · CMD = be able to write it from memory.*

| Asked as | Item | What you must know / the exact command |
|---|---|---|
| MCQ | **Observability** · monitoring vs observability | Monitoring watches signals chosen in advance and sends alerts ("is something wrong?"), whereas observability lets you find "why?" from the system's data, even for unpredicted problems. Monitoring is part of observability, not a rival. |
| MCQ | **Three pillars** · metrics | Named numbers measured again and again (`http_requests_total`); each becomes a time series of (time, value) pairs. Answers "how much, how often?" across all requests. Tool: Prometheus. |
| MCQ | Logs | Timestamped text records of single events; answer "what exactly happened?". Tools: ELK = Elasticsearch + Logstash + Kibana; EFK swaps in Fluentd; Loki is cheaper (indexes only labels). |
| MCQ | Traces | One request's path across services, made of spans (one step in one service) sharing a trace ID; shows which service was slow in that one request, whereas a metric shows totals. Tools: Jaeger, OpenTelemetry. |
| MCQ | Four golden signals | Latency (how long requests take; p95 = 300 ms means 95% were faster), traffic (requests per second), errors (e.g. share of HTTP 5xx), saturation (how full CPU, memory, disk or a queue is). |
| MCQ | **SLI / SLO / SLA** | SLI = Service Level Indicator (the measurement); SLO = Service Level Objective (team's internal target); SLA = Service Level Agreement (looser customer contract, e.g. 99.5% vs SLO 99.9%, with penalties such as refunds). |
| MCQ | Error budget | 100% − SLO. At 99.9% over 30 days (43,200 min): 0.1% × 43,200 = 43.2 minutes allowed; when used up, risky releases stop. Missing the SLO only uses budget, whereas missing the SLA costs money. |
| MCQ | **Prometheus** · what it does | A free monitoring system with a built-in TSDB (time-series database). Every scrape interval (e.g. 15 s) it scrapes each target's `/metrics` page and stores the numbers; default retention 15 days. |
| MCQ | Pull model vs push model | Prometheus pulls (fetches from each target), so a dead target is noticed at once (`up` = 0), whereas with push it is noticed only when data stops. Short-lived jobs push to the Pushgateway, which Prometheus scrapes. |
| MCQ | Exporters · node-exporter vs kube-state-metrics vs metrics-server | An exporter translates another system's numbers into Prometheus format. node-exporter = a machine's CPU, memory, disk; kube-state-metrics = Kubernetes object state (available replicas, restarts); metrics-server = current CPU/memory for `kubectl top` and the HPA (Horizontal Pod Autoscaler), no history. |
| MCQ | Counter vs gauge | A counter only goes up (resets to 0 on restart; names end in `_total`, e.g. total requests served), whereas a gauge goes up and down (memory in use, jobs waiting in a queue right now). |
| CMD | PromQL (Prometheus Query Language) `up` | `up` → 1 if the target's last scrape worked, 0 if it failed · `up == 0` → only the down targets · `sum(up)` → how many targets are up. |
| CMD | PromQL `rate()` | `rate(http_requests_total[5m])` → a counter's average per-second increase over the last 5 minutes. With a label filter: `rate(prometheus_http_requests_total{handler="/metrics"}[5m])`. Needs 2+ samples, so "No data" right after start-up. |
| CMD | PromQL `sum by` | `sum by (job) (rate(http_requests_total[5m]))` → adds the series together and returns one result per value of the `job` label (each label combination is its own time series). |
| CMD | `prometheus.yml` scrape config | `global.scrape_interval: 15s` · `scrape_configs[].job_name: prometheus` · `scrape_configs[].static_configs[].targets: ["prometheus:9090"]` → a job is a named group of targets to scrape. |
| CMD | Read raw metrics from the terminal | `curl -s http://localhost:9090/metrics \| grep "^prometheus_http_requests_total" \| head -3` → silent download, keep only lines starting with that name, show the first 3. |
| MCQ | Alerting rule and `for:` | Rule `expr: up == 0` with `for: 5m`: inactive → pending (true, but not yet for 5 minutes) → firing. If the target recovers while pending, the alert never fires and nobody is notified. |
| MCQ | Prometheus vs Alertmanager | Prometheus evaluates alerting rules and decides when an alert fires, whereas Alertmanager only groups, de-duplicates, silences and routes fired alerts to people (email, Slack, PagerDuty). |
| MCQ | Default ports | Prometheus 9090, Pushgateway 9091, Alertmanager 9093, node-exporter 9100, Grafana 3000 (first login admin / admin), Loki 3100. |
| MCQ | **Grafana** · Prometheus vs Grafana | Prometheus scrapes, stores and queries metrics, whereas Grafana stores none and only visualises them: a dashboard is one page, a panel is one query plus a chart. If Prometheus stops, panels show "No data". |
| MCQ | Data source and the `localhost` trap | A data source is a saved connection Grafana queries live (Prometheus: metrics; Loki/Elasticsearch: logs; Jaeger/Tempo: traces). In Docker Compose use `http://prometheus:9090`; `localhost` inside Grafana's container is Grafana itself ("connection refused"). |
| CMD | Restart the Compose stack from scratch | `docker compose down -v && docker compose up -d && docker compose logs -f grafana` → remove containers, network and the `grafana-data` volume, start again in the background, follow Grafana's logs. |
| MCQ | **GitOps** · Git as the single source of truth | Git holds the desired state (declarative YAML); an agent inside the cluster keeps the live state matching it. Any difference is drift, and because Git is the single source of truth, Git wins. |
| MCQ | The four GitOps principles | Declarative, versioned and immutable (every change is a Git commit), pulled automatically (by an agent), continuously reconciled (drift keeps being corrected). "Pushed by CI" is NOT a principle. |
| MCQ | Push-based CD (Continuous Deployment) vs pull-based GitOps | In push-based CD the CI (Continuous Integration) runner runs `kubectl`/`helm`, so cluster credentials sit in CI secrets, whereas in GitOps the in-cluster agent only connects out to Git, so credentials never leave the cluster. |
| MCQ | Continuous Deployment with GitOps · app repo vs config repo | CI still tests, builds and pushes the image from the app repo (source code), then commits the new image tag to the config repo (Kubernetes YAML), which Argo CD watches (polls about every 3 minutes). |
| CMD | GitOps rollback | `git revert HEAD && git push` in the config repo → Argo CD syncs the previous version back. `kubectl rollout undo` changes only the cluster, which then disagrees with Git. |
| MCQ | **Argo CD** · what it is | A free GitOps tool that runs as Pods inside the cluster (namespace `argocd`), pulls from Git and keeps the cluster synchronised, with a web UI and the `argocd` command-line tool. It never writes cluster state back to Git. |
| CMD | Application YAML: identity | `apiVersion: argoproj.io/v1alpha1` · `kind: Application` · `metadata.namespace: argocd` · `spec.project: default` → an Application is a CRD (Custom Resource Definition) object living in Argo CD's namespace. |
| CMD | Application YAML: source and destination | `spec.source.repoURL: https://github.com/me/shop-config.git` · `spec.source.targetRevision: HEAD` · `spec.source.path: k8s` (where the desired state is) · `spec.destination.server: https://kubernetes.default.svc` · `spec.destination.namespace: shop` (where to deploy). |
| CMD | Application YAML: sync policy | `spec.syncPolicy.automated.selfHeal: true` (undo manual cluster changes) · `spec.syncPolicy.automated.prune: true` (delete what was removed from Git; off by default) · `spec.syncPolicy.syncOptions: [CreateNamespace=true]` (create the namespace if missing). |
| MCQ | Sync status vs health status | Sync: Synced = cluster matches Git, OutOfSync = differs (new commit or drift). Health: Healthy, Progressing (rolling out), Degraded (failed, e.g. CrashLoopBackOff), Missing (not created yet). Synced + Degraded = Git's config is broken: fix it in Git. |
| MCQ | Manual vs automated sync, and `selfHeal` | Manual sync (default) shows OutOfSync and waits for Sync (`argocd app sync guestbook`). `automated` syncs when Git changes; without `selfHeal` a manual `kubectl` change stays (OutOfSync), with it Argo CD reverts it within seconds. |
| CMD | Create and check an Application | `kubectl apply -f guestbook-app.yaml` → creates the Application · `kubectl get applications -n argocd` → lists each with its sync and health status, e.g. `Synced   Healthy`. |
| CMD | Open the Argo CD web UI | `kubectl port-forward svc/argocd-server -n argocd 8080:443` → Mac port 8080 to the Service's HTTPS port 443; open `https://localhost:8080` (on your own Mac use 9080, because 8080 is taken by your other cluster). |

## 📝 Quiz (5 MCQs; answer in chat as "1 B, 2 D, …")
**1.** Your customer SLA promises 99.5% availability per 30 days, and your team's SLO is 99.9%. This month, outages have already added up to 50 minutes. Which statement is correct?
- A) The SLA is breached, so refunds are owed to customers
- B) Both are met, since 50 min is under 0.1% of 30 days
- C) The SLO is met, with about 7 minutes of budget left
- D) The error budget is gone, but no SLA penalty is due

**2.** Customers are using the shop normally and its pages load fast. But in Prometheus, `up{job="shop"}` shows 0 and the target is marked DOWN. What is the most likely cause?
- A) Alertmanager has silenced this target's alerts
- B) The target has run for under two scrape intervals
- C) Prometheus can't reach the shop's /metrics page
- D) Grafana's data source still points at localhost

**3.** A team wants one Grafana dashboard with three panels: requests per second, the latest error log lines, and the slowest request traces. Which setup is correct?
- A) Prometheus, Loki and Jaeger as data sources, queried live
- B) Prometheus as the only data source, as it stores all three
- C) Grafana scrapes all three and keeps them in its own database
- D) Prometheus, Loki and Jaeger as data sources, copied every 15 s

**4.** An Argo CD Application has `automated` sync with `selfHeal: true`. It shows Synced + Degraded because the image tag in Git has a typo, so the Pods can't start. An engineer fixes the tag directly in the cluster with `kubectl` and doesn't touch Git. What happens next?
- A) The fix stays, and the app turns Synced + Healthy
- B) Self-heal puts the broken tag from Git back again
- C) Argo CD commits the fixed tag back to the Git repo
- D) The fix stays, and the app shows OutOfSync + Healthy

**5.** A team replaces push-based CD (a CI job runs `kubectl apply` against the cluster) with pull-based GitOps using Argo CD. Which of these is NOT something the move gives them?
- A) CI no longer needs network access into the cluster
- B) Cluster credentials no longer live in CI secrets
- C) The agent now builds and pushes the image itself
- D) Manual kubectl edits are now detected as drift
