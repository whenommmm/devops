# DevOps Exam — Study Plan (v3: 2 sessions/day)

**Exam:** Mon 19 Oct 2026 · **Plan window:** Oct 5 → Oct 18 (14 days × 2 sessions = 28 sessions)
**Status:** IN PROGRESS. Intro, Linux, Shell, Networking done. Rebuilt as v4 on "Oct 11" (16 sessions left).
**Revision due:** when the exam handbook arrives (~Oct 6–7), re-weight topics and switch tests to its format.

### Why v3
v2 assumed 3 sessions/day (~5 h). Real capacity is **at most 2 sessions/day**. v3:
- 2 sessions/day, ~90 min each, **~4 concepts per session** (no 10-concept marathons)
- Less important concepts are **shortened**, never dropped (see §3)
- **All new material finishes Oct 16 (S23).** Oct 16 S24 is a buffer, Oct 17 is the mock exam, Oct 18 is revision. The subject is fully done before the 19th.

---

## 0. Assumptions (unchanged from v2 unless noted)

- 3rd-year student, new to DevOps. Explain from zero: plain words → everyday analogy → technical term → tiny example. Minimal game-dev analogies.
- **Goal: full marks.**
- Mac only (Apple Silicon, 16 GB). Practicals: Terminal, git + GitHub (account ready, `gh` logged in), Docker Desktop (+ built-in Kubernetes), `brew install` helm/terraform/trivy. No Linux VM, no cloud account.
- Exam (expected): MCQs + debugging/coding questions. Handbook pending.

## 1. Session format (~90 min)

| Part | Time |
|------|------|
| Warm-up: 2–3 MCQs (spaced recall + pending Block 1 test questions + weak queue) | ~5–10 min |
| **~4 concepts**, each: explain → check (you answer first) → small Mac task | ~15–20 min each |
| **Session quiz: ≥5 hard MCQs** (+ 2–3 one-liners) | ~10–15 min |
| Close-out: tracker updated | 2 min |

Rules: answer before reveal · "idk" beats a blank · mark guesses with `?`.

## 2. Roadmap (v4: 16 sessions, rebuilt on "Oct 11" after a 3-day break)

Oct 8–10 were lost (usage limit; no sessions done). From "Oct 11": **8 days × 2 sessions = 16**. To fit everything before the exam:
- the separate checkpoint tests, buffer, and two revision days are removed
- Terraform + Cloud are merged into one session
- every session quiz mixes in earlier topics, and the last session holds a mini mock + revision

**Tools (installed via Homebrew):** Docker CLI + **Colima** (runs Docker inside a small Ubuntu virtual machine; no Docker Desktop) · Docker Compose · **minikube** (local Kubernetes cluster) · kubectl · Helm · Terraform.

| Day | Session 1 | Session 2 |
|------|-----------|-----------|
| **Oct 11** | **Git & GitHub** (file: `sessions/01_git-github.md`) | **Docker I:** containers vs VMs · architecture · images, containers, Docker Hub · lifecycle & commands (`sessions/02_…`) |
| **Oct 12** | **Docker II:** Dockerfile instructions · CMD vs ENTRYPOINT · .dockerignore, multi-stage · image optimisation | **Docker III:** volumes & bind mounts · networking, container-to-container, port mapping · YAML · Compose multi-container app |
| **Oct 13** | **K8s I:** why Kubernetes · architecture (API server, scheduler, controller manager, etcd, kubelet) · kubectl, manifests, declarative | **K8s II:** Pods & lifecycle · labels & selectors · ReplicaSets · Deployments (scaling, rolling updates, rollbacks) |
| **Oct 14** | **K8s III:** Pod networking · Services (ClusterIP, NodePort, LoadBalancer) · selectors, port vs targetPort · DNS & service discovery | **K8s IV:** Ingress & controller · host/path routing, TLS · ConfigMaps · Secrets & env vars |
| **Oct 15** | **K8s V:** volumes, PV, PVC, StorageClass · liveness/readiness/startup probes · HPA & Metrics Server | **K8s VI (troubleshooting):** kubectl get/describe/logs/exec, events · CrashLoopBackOff, ImagePullBackOff, Pending · Service & DNS issues |
| **Oct 16** | **Helm:** charts, Chart.yaml, values.yaml, templates · install/upgrade/rollback · deploy an app | **CI/CD + GitHub Actions:** CI vs CD · workflows, jobs, steps, runners · secrets, artifacts · build & test pipeline |
| **Oct 17** | **Complete CI/CD + DevSecOps:** Docker build & registry in pipeline · K8s deploy · SAST, SCA, secret & image scanning · security gates | **Monitoring, Observability & GitOps:** monitoring vs observability, metrics/logs/traces · Prometheus & Grafana · GitOps, Argo CD |
| **Oct 18** | **IaC + Terraform + Cloud:** IaC concepts · Terraform providers/resources/variables/outputs, init/plan/apply/destroy, state · cloud fundamentals (IaaS/PaaS/SaaS, regions/AZs, VPC, subnets, route tables, security groups) | **Final project + mini mock + revision:** full lifecycle walkthrough · mixed mock quiz · weak-queue sweep |
| **Oct 19** | **Exam.** 20–30 min with `notes/` + the glossary only. | |

**Folded into reading (no dedicated session):** DevOps roles, and Linux logs & basic troubleshooting. These go in a short notes file and appear in quizzes.
**Session-end quiz:** 4 hard MCQs + 2 one-liners, mixing in earlier topics.

## 3. Scope decisions after the official list (Oct 6)

**Pending items added:** DevOps tools & roles, Linux logs & basic troubleshooting (both Oct 8 S2) · basic networking commands (Oct 6 S1) · Git rebase at full depth · image optimization · Metrics Server · security gates · route tables & VNet · Argo CD · deploying an app with Helm.

**Not in the official list → awareness only (one or two sentences when relevant, no dedicated time):** OSI model · NAT/firewall internals · scheduling (nodeSelector, affinity, taints) · RBAC & ServiceAccounts · StatefulSets · DaemonSets · Jobs/CronJobs · Jenkins · Kustomize · PromQL · Terraform modules. (Cron and DORA were already covered.)

## 4. Testing

- **Every session:** 4 hard MCQs + 2 one-liners (multi-step, output prediction, scenario, near-miss options).
- **Warm-ups:** spaced recall + weak queue (re-test at +1, +3, +7 days; out after 2 correct on separate days).
- **Checkpoints:** Oct 8 (foundations + Docker) · Oct 12 (Kubernetes) · Oct 17 full mock · Oct 18 mini-mock.
- Question types: MCQ · find-the-bug · predict-the-output · write-it · explain (contrast form for "differences").

## 5. Progress tracking

`PROGRESS.md`, updated at the end of each session (statuses: NOT STARTED → LEARNING → WEAK / UNDERSTOOD → STRONG, evidence-based only). Revision notes go in `notes/` for Oct 18.

## 6. How the concepts depend on each other

```
How software runs ─► DevOps lifecycle ─► every tool sits on the loop
Terminal/files ─► shell scripts ─► Dockerfile RUN, CI `run:` steps
Processes ─► containers (isolated processes)
Networking (IP, port, DNS, LB, proxy) ─► Docker -p & networks ─► K8s Services/DNS ─► Ingress ─► cloud VPC/SG
Git ─► GitHub Actions ─► GitOps
YAML ─► Compose ─► K8s manifests ─► GHA workflows ─► Helm values
Image ─► registry ─► K8s pulls it (ImagePullBackOff)
Labels/selectors ─► ReplicaSets, Deployments, Services
Docker volumes ─► PV/PVC · requests/limits ─► HPA · desired vs actual ─► K8s ─► ArgoCD ─► Terraform
```

## 7. Commonly confused pairs (MCQ drills)

- **Basics/Linux:** program vs process · client vs server · `>` vs `>>` vs `|` · absolute vs relative · chmod vs chown · SIGTERM vs SIGKILL · start vs enable · apt update vs upgrade · single vs double quotes
- **Networking/Git:** IP vs port · public vs private IP · 127.0.0.1 vs 0.0.0.0 · TCP vs UDP · LB vs reverse proxy · Git vs GitHub · fetch vs pull · merge vs rebase · reset vs revert
- **Docker:** image vs container · container vs VM · CMD vs ENTRYPOINT · COPY vs ADD · ARG vs ENV · EXPOSE vs `-p` · volume vs bind mount · Dockerfile vs Compose
- **Kubernetes:** Pod vs container · ReplicaSet vs Deployment · Deployment vs StatefulSet vs DaemonSet · Service vs Ingress vs Ingress controller · ClusterIP vs NodePort vs LoadBalancer · port vs targetPort vs nodePort · ConfigMap vs Secret · PV vs PVC vs StorageClass · liveness vs readiness vs startup · requests vs limits · Role vs ClusterRole · Pending vs CrashLoopBackOff vs ImagePullBackOff
- **Automation/infra:** chart vs release · CI vs Delivery vs Deployment · workflow vs job vs step · `uses` vs `run` · SAST vs DAST vs SCA · GitOps vs CI/CD · declarative vs imperative · plan vs apply · Terraform vs Ansible · IaaS vs PaaS vs SaaS · monitoring vs observability · metrics vs logs vs traces · SLI vs SLO vs SLA

## 8. Open items

1. **Exam handbook** (~Oct 6–7): share it; I'll re-weight topics and use its format for the mock.
2. **Docker Desktop:** install before Oct 7 S1 (normal Mac app, ~2 GB, sign-in can be skipped).
3. **Pending practicals:** hello.sh run results; backup.sh output.

## Schedule adjustments log
- **Oct 4:** study day ran to ~6:30 AM Oct 5; Block 1 S3–S4 done, S4 quiz done in the Oct 5 warm-up (~69%).
- **Oct 5:** switched to **v3 (2 sessions/day)**. Block 1 test is spread across warm-ups. Practice sessions are folded into session quizzes and checkpoints.
- **"Oct 11":** 3 days lost to the usage limit. Rebuilt as **v4: 16 sessions**. Checkpoints, buffer, and revision days removed; Terraform + Cloud merged; mini mock in the final session. Tools were found installed via Homebrew (Colima, minikube, helm, terraform).
- **Oct 6:** brief compression to 24 sessions was reverted. The user counts late night + early hours as the same study day, so "Oct 6" (Networking + Git) runs now and the original 28-session v3 stays. Quiz size changed to 4 MCQs + 2 one-liners.
