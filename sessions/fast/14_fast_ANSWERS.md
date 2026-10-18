# Answer key · Session 14 FAST · Monitoring, Observability & GitOps

⚠️ Only open after answering.

---

**1.** Your customer SLA promises 99.5% availability per 30 days, and your team's SLO is 99.9%. This month, outages have already added up to 50 minutes. Which statement is correct?
- A) The SLA is breached, so refunds are owed to customers
- B) Both are met, since 50 min is under 0.1% of 30 days
- C) The SLO is met, with about 7 minutes of budget left
- D) The error budget is gone, but no SLA penalty is due

**Correct: D.** The error budget comes from the SLO: 100% − 99.9% = 0.1% of 43,200 minutes = 43.2 minutes, and 50 minutes is more than that, so the budget is used up (by about 7 minutes). The SLA is looser: 100% − 99.5% = 0.5% of 43,200 = 216 minutes (3.6 hours) of allowed downtime, and 50 minutes is far below it, so the contract is still met and no penalty is owed. A mixes up the two levels: it is the internal SLO that was missed, not the customer SLA. B gets the maths wrong: 0.1% of 30 days is 43.2 minutes, which is less than 50. C is a sign mistake: 50 − 43.2 = 6.8 means about 7 minutes over budget, not 7 minutes left.

---

**2.** Customers are using the shop normally and its pages load fast. But in Prometheus, `up{job="shop"}` shows 0 and the target is marked DOWN. What is the most likely cause?
- A) Alertmanager has silenced this target's alerts
- B) The target has run for under two scrape intervals
- C) Prometheus can't reach the shop's /metrics page
- D) Grafana's data source still points at localhost

**Correct: C.** `up` only reports whether Prometheus's last scrape (its request to the target's `/metrics` page) worked, not whether customers can use the app. The shop clearly works, so the scrape itself is failing, for example because the scrape configuration has the wrong port or path, or the shop doesn't expose that page. A: silencing in Alertmanager only stops notifications from being delivered; it doesn't change what Prometheus scrapes or the value of `up`. B borrows the "two samples" rule from `rate()`; `up` is set after every single scrape, so one successful scrape would already give 1. D: Grafana only reads from Prometheus, so a wrong Grafana data source URL makes Grafana's panels fail but cannot change the `up` value that Prometheus records.

---

**3.** A team wants one Grafana dashboard with three panels: requests per second, the latest error log lines, and the slowest request traces. Which setup is correct?
- A) Prometheus, Loki and Jaeger as data sources, queried live
- B) Prometheus as the only data source, as it stores all three
- C) Grafana scrapes all three and keeps them in its own database
- D) Prometheus, Loki and Jaeger as data sources, copied every 15 s

**Correct: A.** Each pillar lives in its own tool: metrics in Prometheus, logs in Loki, traces in Jaeger. Grafana connects to each one as a data source and sends live queries whenever the dashboard loads or refreshes. B is wrong because Prometheus stores metrics only, not logs or traces. C is wrong because Grafana neither scrapes nor stores data; it keeps only its own settings and dashboards. D is the tempting one: the data sources are right, but Grafana doesn't copy anything on a timer (15 seconds is a typical Prometheus scrape interval); it asks the data sources each time, which is why its panels show "No data" when a data source is stopped.

---

**4.** An Argo CD Application has `automated` sync with `selfHeal: true`. It shows Synced + Degraded because the image tag in Git has a typo, so the Pods can't start. An engineer fixes the tag directly in the cluster with `kubectl` and doesn't touch Git. What happens next?
- A) The fix stays, and the app turns Synced + Healthy
- B) Self-heal puts the broken tag from Git back again
- C) Argo CD commits the fixed tag back to the Git repo
- D) The fix stays, and the app shows OutOfSync + Healthy

**Correct: B.** After the manual fix, the live state (correct tag) no longer matches Git (typo tag), which is drift. With `selfHeal: true`, Argo CD undoes manual cluster changes within seconds, so it re-applies the broken tag from Git and the app goes back to Degraded. Because Git is the single source of truth, the real fix is to correct the tag in Git (commit and push). A can't happen: the cluster no longer matches Git, so it can't be Synced, and self-heal won't let the change stay. C is wrong because Argo CD only reads Git; it never writes the cluster's state back into Git. D is what would happen without `selfHeal`: automated sync alone reacts only to Git changes, so the manual fix would stay and the app would just show OutOfSync.

---

**5.** A team replaces push-based CD (a CI job runs `kubectl apply` against the cluster) with pull-based GitOps using Argo CD. Which of these is NOT something the move gives them?
- A) CI no longer needs network access into the cluster
- B) Cluster credentials no longer live in CI secrets
- C) The agent now builds and pushes the image itself
- D) Manual kubectl edits are now detected as drift

**Correct: C.** Building, testing and pushing the image is still CI's job, done from the app repo; CI then commits the new image tag to the config repo, and Argo CD only applies the Kubernetes YAML it finds there. The other three are real gains of pull-based GitOps. A: the agent runs inside the cluster and only connects out to Git, so CI no longer has to reach the cluster's API server. B: since nothing outside the cluster runs `kubectl` any more, the cluster credentials never need to leave the cluster. D: the agent keeps comparing the live state with Git, so manual `kubectl` edits show up as drift (OutOfSync) instead of going unnoticed until the next deploy.

---
