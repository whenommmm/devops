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

## ⚡ Must-remember
| Term | Meaning |
|---|---|
| `up` | 1 = last scrape worked, 0 = it failed |
| `rate(x[5m])` | per-second increase of counter `x` over 5 minutes |
| Alertmanager | groups, silences and routes alerts that Prometheus fires |
| Default ports | Prometheus 9090, Grafana 3000, Alertmanager 9093 |
| Error budget | 100% − SLO; 99.9% over 30 days = 43.2 minutes |
| Synced / OutOfSync | live state matches Git / differs from Git |
| `selfHeal` / `prune` | revert manual cluster changes / delete what was removed from Git |

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
