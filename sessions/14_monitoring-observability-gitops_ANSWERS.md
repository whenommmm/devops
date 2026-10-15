# Answer key · Session 14 · Monitoring, Observability & GitOps

⚠️ Only open after answering.

---

## Warm-up

**W1.** A Trivy gate uses `--severity HIGH,CRITICAL --ignore-unfixed --exit-code 1`. The only finding is a HIGH CVE in `libc` with Status `affected` and no Fixed Version. What happens?
- A) Exit 1: HIGH findings always fail this gate
- B) Exit 0: unfixed findings are left out
- C) Exit 1: unfixed findings count as CRITICAL
- D) Exit 0: libc is skipped as an OS package

**Correct: B.** `--ignore-unfixed` drops every finding that has no fixed version yet (Status `affected`). After that filter nothing is left, so `--exit-code 1` has nothing to trigger on, and Trivy exits 0. (You saw this with `hello-app:2.0` in Session 13.) A ignores what `--ignore-unfixed` does. C: no flag "promotes" unfixed findings to CRITICAL. D: Trivy definitely scans OS packages like `libc` (that's where most of `nginx:1.27`'s 700 findings came from); the reason is the "unfixed" filter.

---

**W2.** A nightly rescan workflow has `on: schedule: - cron: "0 2 * * *"`. When does it run? (GitHub runs schedules in **UTC**, Coordinated Universal Time; **IST**, India Standard Time, is UTC + 5:30.)
- A) Every day at 02:00 UTC
- B) Every day at 2:00 PM UTC
- C) Every 2 hours, on the hour
- D) Every day at 02:00 IST

**Correct: A.** The cron fields are minute, hour, day of month, month, day of week: `0 2 * * *` = minute 0 of hour 2, every day. Cron uses **24-hour** time, so 2 means 02:00 at night (B would be `0 14 * * *`). C would be `0 */2 * * *`. D: GitHub Actions runs schedules in **UTC**, so 02:00 UTC is 07:30 in India (IST).

---

**W3.** Your PR's security gate failed. You fix `requirements.txt` **on the same branch** and push. What happens to the PR?
- A) You must open a new PR for the new commit
- B) Close and reopen it to rerun the checks
- C) Nothing until the fix is merged into main
- D) It updates, and its checks run again

**Correct: D.** A PR shows the **branch**, not a fixed set of commits. Pushing to that branch adds the commit to the PR and triggers the `pull_request` workflow again, so the gate re-runs on the fixed code. A and B are unnecessary. C is backwards: the whole point of the gate is that the fix gets checked **before** merging into main.

---

## Check 82

**82.1** Users report "checkout is slow". Your dashboard shows p95 latency for checkout jumped from 300 ms to 2 s, and nothing else looks wrong. Which signal best shows **which service** in the chain is slow?
- A) A CPU gauge for the whole cluster
- B) A trace of one slow checkout request
- C) The total request counter of the API
- D) An uptime check hitting the home page

**Correct: B.** A trace follows **one request** through every service, with a span (and a duration) for each, so it shows exactly where the 2 seconds go (e.g. the database span takes 1.7 s). A is a whole-cluster number: it can't point at one service, and the story says nothing else looks wrong. C is a counter of requests, so it shows traffic, not where time is spent. D only checks whether one page answers (blackbox monitoring); it can't see inside the checkout chain.

---

**82.2** Which of these is a **saturation** signal?
- A) Requests per second arriving at the API
- B) Share of requests answered with a 5xx code
- C) How full the CPU, memory or job queue is
- D) Time taken to serve each checkout request

**Correct: C.** Saturation = how "full" a resource is (CPU, memory, disk, queue length). The other three are the other golden signals: A is **traffic**, B is **errors**, D is **latency**.

---

**82.3** (one line) Write the full forms of SLI, SLO and SLA, and say which one carries penalties if it's missed.

**Model answer:** *"SLI = Service Level Indicator (the measured value, e.g. % of successful requests), SLO = Service Level Objective (the internal target for that SLI, e.g. 99.9% over 30 days), and SLA = Service Level Agreement (the contract with customers); the SLA carries penalties such as refunds or credits if it's missed, whereas missing an SLO only uses up the error budget."*

---

## Check 83

**83.1** Which of these should be exposed as a **counter**?
- A) Total HTTP requests served since start-up
- B) Number of jobs waiting in a queue right now
- C) Memory the process is using at this moment
- D) Temperature of the server room at this moment

**Correct: A.** A counter only goes up (resetting to 0 on restart), like a running total of requests; you read it with `rate()` to get requests per second. B, C and D can all go **down** as well as up, so they're **gauges**. B is the tempting one: a queue "counts" jobs, but its value falls as jobs are processed.

---

**83.2** A backup script runs for 10 seconds every night. Prometheus scrapes every 15 seconds, and the script's metrics never appear. What's the standard fix?
- A) Lower `scrape_interval` to 1s for every target
- B) Send the metrics straight to Alertmanager instead
- C) Add the script to Grafana as a new data source
- D) Push them to a Pushgateway, which gets scraped

**Correct: D.** Short-lived jobs can finish (and disappear) between scrapes. They **push** their metrics to the **Pushgateway**, which keeps them, and Prometheus scrapes the Pushgateway normally. A would multiply the load of every target by 15 and still miss a job that runs between scrapes. B: Alertmanager receives **alerts** from Prometheus; it doesn't store metrics. C: Grafana only visualises data stored somewhere else; a script isn't a data source.

---

**83.3** (one line) Why is Prometheus called "pull-based", and what's one advantage of pulling?

**Model answer:** *"Prometheus is pull-based because the Prometheus server itself fetches (scrapes) metrics from each target's /metrics HTTP endpoint on a schedule, whereas in a push model the targets send their data to the server; one advantage is that Prometheus knows immediately when a target is down, because the scrape fails and `up` becomes 0 (others: central control of what is collected and how often, and you can open any target's /metrics in a browser to debug)."*

---

## Check 84

**84.1** Which statement about Prometheus and Grafana is correct?
- A) Grafana stores the metrics; Prometheus draws charts
- B) Grafana scrapes /metrics endpoints on its own schedule
- C) Prometheus stores and queries; Grafana visualises
- D) Prometheus can't run unless Grafana is installed

**Correct: C.** Prometheus scrapes, stores (in its TSDB) and answers PromQL queries. Grafana sends queries to it and draws dashboards. A swaps the two roles. B describes Prometheus, not Grafana. D: Prometheus works alone, with its own basic UI at :9090 (you used it in Concept 83).

---

**84.2** Prometheus started 20 seconds ago with a 15-second scrape interval. In Grafana, `http_requests_total` shows values, but `rate(http_requests_total[5m])` shows **No data**. Why?
- A) `rate()` works only on gauges, not on counters
- B) `rate()` needs two samples, and there's only one
- C) Grafana needs a variable named `$range` first
- D) The `[5m]` window must be shorter than 15 seconds

**Correct: B.** `rate()` calculates the increase **between samples**, so it needs at least two inside the window. After 20 seconds at a 15-second interval there's only been one scrape (maybe two at the very edge). Wait a minute and it works. A is backwards: `rate()` is **for** counters. C: variables are optional. D is backwards too: the window should be **at least** a few scrape intervals long.

---

## Check 85

**85.1** Which is **NOT** one of the four GitOps principles?
- A) Declarative: the desired state is described
- B) Versioned and immutable: kept in Git history
- C) Pushed by CI: a pipeline runs kubectl apply
- D) Continuously reconciled: agents correct drift

**Correct: C** (it's not a principle). The four principles are **declarative**, **versioned and immutable**, **pulled automatically**, and **continuously reconciled**. "Pushed by CI" describes push-based CD, the opposite of principle 3: in GitOps, an agent **pulls** the desired state from Git. A, B and D are genuine principles.

---

**85.2** Argo CD shows an Application as **Synced** but **Degraded**. What does that mean?
- A) Git is ahead of the cluster, and a sync is due
- B) The repo can't be reached, so the status is stale
- C) Someone changed the cluster by hand with kubectl
- D) Cluster matches Git, but the app isn't working

**Correct: D.** **Sync** status compares the cluster with Git (Synced = they match). **Health** status says whether the resources actually work (Degraded = something failed, e.g. Pods in CrashLoopBackOff or a stuck rollout). So Argo CD applied exactly what Git says, and what Git says is broken; the fix goes into Git. A and C would both show **OutOfSync**. B would show an error or **Unknown** status, not a confident Synced.

---

**85.3** (one line) With GitOps, how do you roll back a bad release, and why is that better than running `kubectl rollout undo`?

**Model answer:** *"You `git revert` the bad commit in the config repo and push, and Argo CD syncs the cluster back to the previous desired state; this is better because the rollback is itself reviewed, recorded in Git history and repeatable, whereas `kubectl rollout undo` changes only the cluster, so Git and the cluster disagree (and self-heal or the next sync would put the bad version straight back)."*

---

## Session quiz

**Q1.** Prometheus and Grafana run as two services in one Compose file, with ports `9090:9090` and `3000:3000`. In Grafana's Prometheus data source you enter `http://localhost:9090`, and **Save & test** fails with `connection refused`. Opening `http://localhost:9090` in your Mac's browser works fine. What's the cause?
- A) Grafana needs Prometheus's IP, since names never resolve
- B) Port 9090 must also be published on the Grafana service
- C) Inside Grafana's container, localhost is Grafana itself
- D) Prometheus rejects queries that don't come from a browser

**Correct: C.** The data source query is sent **by the Grafana server**, from inside its container, where `localhost` means that container (loopback), and nothing listens on 9090 there. Your Mac's browser works because `-p 9090:9090` maps the **Mac's** port to Prometheus. The fix is `http://prometheus:9090`. A is false: on a Compose network, **service names do resolve** (Docker DNS, Session 04), which is why `prometheus:9090` works. B: publishing ports only affects reaching containers **from the host**; it doesn't make Prometheus appear inside Grafana's container. D: Prometheus's HTTP API answers any client (Grafana uses exactly that API).

---

**Q2.** Your SLO is 99.9% successful requests over 30 days. Outages have already used 30 minutes this month, and the team wants to ship a risky change today. What does the error budget say?
- A) About 13 minutes remain, so favour reliability
- B) The budget resets every day, so ship the change
- C) About 43 minutes remain; outages aren't counted
- D) It's used up, since 30 min is over 0.1% of 30 days

**Correct: A.** The budget is 0.1% of 30 days = 43,200 × 0.001 = **43.2 minutes**. 43.2 − 30 = about **13 minutes left**, so a risky release could easily use it all; the error-budget policy says slow down and prioritise reliability. B: the budget belongs to the SLO's window (30 days), not to each day. C: outages are exactly what the budget measures. D gets the maths wrong: 0.1% of 30 days is 43.2 minutes, which is **more** than 30.

---

**Q3.** An Argo CD Application has `automated: { prune: true }` but **no** `selfHeal`. Git says `replicas: 2`. Someone runs `kubectl scale deployment shop --replicas=5`. Nobody touches Git. What happens?
- A) Argo CD reverts the Deployment to 2 within seconds
- B) Argo CD commits `replicas: 5` back to the Git repo
- C) Prune deletes the Deployment as it differs from Git
- D) It stays at 5, and the app is shown as OutOfSync

**Correct: D.** Automated sync reacts to **changes in Git**. By default it does **not** react to changes made directly in the cluster; that's what `selfHeal: true` adds. So the drift stays (5 replicas), and Argo CD reports **OutOfSync** until someone syncs manually or a new Git commit triggers a sync. A is what would happen **with** `selfHeal: true`. B: Argo CD never writes the cluster's state back into Git; Git is the source of truth, not the copy. C: prune deletes resources that were **removed from Git**; the Deployment is still in Git, it just has a different replica count.

---

**Q4.** A Prometheus alerting rule has `expr: up == 0` and `for: 5m`. A target goes down for 3 minutes, then recovers. What happens?
- A) The alert fires at once and resolves after 3 minutes
- B) It goes pending, never fires, and no one is notified
- C) It fires after 5 minutes, although the target is back
- D) Alertmanager waits 5 minutes, then sends one message

**Correct: B.** When the condition becomes true, the alert enters **pending**. It only becomes **firing** (and is sent to Alertmanager) if the condition stays true for the **whole** `for:` duration. The target recovered after 3 minutes, so the alert went back to inactive and nobody was notified. That's exactly what `for:` is for: ignoring short blips. A ignores `for:`. C: the 5-minute clock only counts while the condition is continuously true. D: `for:` is evaluated by **Prometheus**, and Alertmanager never even received the alert.

---

**L1.** Explain monitoring vs observability in contrast form, with one example of each.

**Model answer:** *"Monitoring watches known, predefined signals and alerts when they cross a threshold, answering 'is something wrong?' (e.g. an alert when the error rate goes above 5%), whereas observability is the ability to understand the system's internal state from its outputs (metrics, logs and traces) well enough to answer new, unplanned questions about unknown problems, answering 'why is it wrong?' (e.g. using a trace to find that checkout is slow because one database query takes 1.7 s)."*

---

**L2.** Explain push-based CD vs pull-based GitOps in contrast form, including where the cluster credentials live.

**Model answer:** *"In push-based CD, the CI/CD pipeline pushes changes into the cluster by running `kubectl`/`helm` itself, so the cluster credentials (kubeconfig) are stored outside the cluster in CI secrets and CI must be able to reach the API server, whereas in pull-based GitOps an agent running inside the cluster (e.g. Argo CD) pulls the desired state from Git and continuously reconciles the cluster to it, so the credentials never leave the cluster, only outbound access to Git is needed, and drift from manual changes can be detected and reverted."*
