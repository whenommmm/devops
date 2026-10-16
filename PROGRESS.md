# DevOps Exam — Progress Tracker

**Exam:** Mon 19 Oct 2026 · **Plan:** v3, 2 sessions/day (see STUDY_PLAN.md). **Current position:** Shell I + Shell II done (finished early on calendar Oct 7). **Plan v4 (16 sessions, "Oct 11"–18).** Networking done. Oct 7–10 lost (usage limit; no sessions). **Oct 11 done (Git + Docker I). All 16 session files are written (`sessions/01–16`, each with an answer key; MCQ length rule checked by script). Next: "Oct 12" S1 Docker II (03), S2 Docker III (04).** Answers go in `sessions/MY_ANSWERS.md`. Block 1 test: 11 of 12 MCQs still pending (warm-ups).
**Status rules:** NOT STARTED → LEARNING (taught, untested) → WEAK (<60% / recurring misconception / failed re-test) → UNDERSTOOD (≥70% incl. an application question) → STRONG (≥85% on two separate days, incl. scenario/practical, no hints). Never upgraded just because it was explained.

## Topic status

| Block | Concept | Status | Best / latest score | Last tested | Next review |
|-------|---------|--------|--------------------|-------------|-------------|
| B1 | How software runs (OS, process, client/server, deploy) | UNDERSTOOD | S1 checks ~85%, quiz Q1–Q4 4/4 | Oct 3 | Oct 5 test |
| B1 | What DevOps is + lifecycle + CALMS | UNDERSTOOD (sub-items: lifecycle order, CI, now 1 correct each) | S2 checks ~75%, quiz 7/9; S3 W1 ✓ W2 ✓ | Oct 4 | Oct 5 test |
| B1 | Microservices vs monolith | UNDERSTOOD | 9.1 ✓ 9.2 ✓ Q6 ✓; 9.3/9.4 partial (vague 'differences') | Oct 3 | Oct 5 test |
| B1 | Linux intro, kernel vs shell | UNDERSTOOD | 10.1 ✗ → S3 quiz Q1 ✓; 10.2–10.4 ✓ | Oct 4 | Oct 5 test |
| B1 | Terminal, files & paths (tree, abs/rel paths, file cmds, >, >>, \|, grep) | UNDERSTOOD (sub-item WEAK: leading `/`) | S3 quiz 5/5 MCQ + bug ½ ≈ 79%; drill 4/5 | Oct 4 | Oct 5 test |
| B1 | Permissions (rwx, chmod, chown, sudo, umask) | WEAK | 14.1 ✗ (740), 14.2 ✓, 14.3 ✓, 14.4 ½ (745); exercise ✓ | Oct 4 | Oct 5 test |
| B1 | Processes, signals, daemons, env vars, PATH, package managers, systemd | UNDERSTOOD | 15.1–15.3 ✓, 15.4 ½; 16.1–16.3 ✓; 17.1–17.3 ✓ (~90%) | Oct 4 | Oct 5 test |
| B2 | Shell scripting: shebang, run modes, variables, `$(...)`, args, quotes | WEAK (args ✓; `$`-syntax and quoting weak) | Checks: 18.x ✓, 19.1 ✓ 19.2 ✗ 19.3 ½, 20.x ✓; quiz 4.5/9 (50%) | Oct 5 | Oct 6 warm-up |
| B2 | Shell scripting: if/tests, loops, exit codes & set -e, functions, cron | UNDERSTOOD (traps weak) | Checks 11.5/12 (22.3 ½); quiz 3.5/8 (44%) → ~75% overall | Oct 7 | Oct 8 warm-up |
| B2 | Networking: IP, public/private, CIDR, ports, TCP/UDP, DNS, HTTP/HTTPS/TLS, networking commands, LB/reverse proxy | WEAK under mixed questions (concept checks strong) | Checks ~96% (26.1 ✗, 26.3 ½); quiz 1.7/6 (28%) | "Oct 6" | Oct 7 warm-up |

| B2 | Git & GitHub: Git vs GitHub, repo, commits, staging, branching, merge vs rebase, conflicts, remotes, fetch/pull/push, fork, PR, revert/reset | UNDERSTOOD (practical not done yet) | Checks 11/11; quiz 4.5/6 (75%) | Oct 11 | Oct 12 warm-up |
| B3 | Docker I: containers vs VMs, architecture (client/daemon/API/registry), images vs containers, lifecycle & commands | UNDERSTOOD (confirmed by unbiased re-test) | Checks 10/10; quiz 4.5/6; re-test 3/4 | Oct 11 | Oct 12 warm-up |
| B3 | Docker II: Dockerfile instructions, layers/cache, CMD vs ENTRYPOINT, RUN vs CMD, ARG vs ENV, .dockerignore, multi-stage, optimisation, USER | UNDERSTOOD (cache order + quiz traps weak) | Checks 10/10; quiz 3/6 | Oct 12 | Oct 13 warm-up |
| B4 | Docker III: volumes vs bind mounts, networks & container-to-container, YAML, Compose | UNDERSTOOD (sub-items WEAK: container addressing; where volumes live) | Checks 5.5/8; quiz 4.5/6 (~71%) | Oct 14 | Oct 15 warm-up |
| B5 | K8s I: why K8s, architecture (API server, etcd, scheduler, controller manager, kubelet, kube-proxy, runtime), apply flow, kubectl/manifests/namespaces | UNDERSTOOD (concepts) · WEAK: apiVersion groups, kubelet-local restarts, namespaces, writing commands | Set 1: checks 9/12, quiz 3/6 · Set 2: warm-up 3/3, checks 8.5/12, quiz 2.5/6 | Oct 15 | Oct 16 warm-up |
| B5 | K8s II: Pods, phases, restartPolicy, labels/selectors, ReplicaSets, Deployments, rolling updates, rollback | UNDERSTOOD (concepts) · WEAK: selector logic (!=, matchLabels AND), rollout maths in scenarios | Checks 8.5/11; quiz MCQs 1/4 (L1/L2/C-tasks pending) | Oct 16 | Oct 17 warm-up |
| B5 | K8s III: Pod networking (flat, CNI), Services (ClusterIP/NodePort/LoadBalancer/headless), selectors & endpoints, port vs targetPort, DNS & service discovery, kube-proxy | UNDERSTOOD (checks strong; quiz scenarios weak) | Warm-up 3/3 · checks 9.5/12 · quiz MCQs 1/4 · L 1.5/2 · C 2/4 | Oct 16 | Oct 17 warm-up |
| B5 | Ingress: controller, host/path routing, TLS | NOT STARTED | – | – | – |
| B6 | ConfigMaps (env + volume) | NOT STARTED | – | – | – |
| B6 | Secrets (env + volume, base64) | NOT STARTED | – | – | – |
| B6 | Volumes, PV, PVC, StorageClass, dynamic provisioning | NOT STARTED | – | – | – |
| B6 | StatefulSets (stateful vs stateless) | NOT STARTED | – | – | – |
| B6 | Probes: liveness / readiness / startup | NOT STARTED | – | – | – |
| B6 | Resource requests & limits | NOT STARTED | – | – | – |
| B6 | HPA | NOT STARTED | – | – | – |
| B6 | Scheduling: nodeSelector, affinity, taints/tolerations | NOT STARTED | – | – | – |
| B6 | Security: ServiceAccounts, RBAC | NOT STARTED | – | – | – |
| B6 | K8s troubleshooting (Pending, CrashLoopBackOff, ImagePullBackOff, selectors, ports, probes) | NOT STARTED | – | – | – |
| B7 | Helm | NOT STARTED | – | – | – |
| B7 | CI/CD concepts & deployment strategies | NOT STARTED | – | – | – |
| B7 | GitHub Actions workflow YAML | NOT STARTED | – | – | – |
| B7 | Complete CI/CD pipeline | NOT STARTED | – | – | – |
| B7 | DevSecOps (SAST/DAST/SCA, scanning) | NOT STARTED | – | – | – |
| B7 | GitOps (ArgoCD, push vs pull) | NOT STARTED | – | – | – |
| B8 | IaC concepts | NOT STARTED | – | – | – |
| B8 | Terraform (HCL, init/plan/apply, state) | NOT STARTED | – | – | – |
| B8 | Cloud basics + Terraform on cloud | NOT STARTED | – | – | – |
| B8 | Monitoring (Prometheus, Grafana) | NOT STARTED | – | – | – |
| B8 | Observability (metrics/logs/traces, SLI/SLO/SLA) | NOT STARTED | – | – | – |
| B8 | Final DevOps project (full lifecycle) | NOT STARTED | – | – | – |
| B8 | Cross-syllabus troubleshooting | NOT STARTED | – | – | – |

## Prerequisite diagnostic (Day 1)

| Area | Self-rating (1–5) | Measured | Notes |
|------|-------------------|----------|-------|
| Using a terminal | 2–3 | Improving fast | S3 quiz 5/5 MCQ; main slip = leading `/` on absolute paths |
| Networking | 2 | Low | Couldn't describe what happens when you open a URL → teach from zero in B2 |
| Git | 4 | Not yet measured | Self-rated strongest area → verify in B2, may go faster |
| Programming | 3 | – | Unity game dev (likely C#); makes WebGL builds, uploads to itch.io |
| OS concepts (process, kernel) | – | Good | OS = manages everything ✓; program = blueprint, process = execution ✓ |
| HTTP / web | – | Low → improving | Now knows server = program answering requests; curl/browser = clients |
| YAML / JSON | – | – | – |

## Weak-topic queue (re-test at +1, +3, +7 days; leaves after 2 correct on separate days)

| Concept | Added | Times missed | Next re-test | Correct streak |
|---------|-------|--------------|--------------|----------------|
| **Leading `/` = absolute path (`/var/log`, not `var/log`)** | Oct 4 | **3** (11.3, 12.3, S3 bug) | Oct 5 test | 0 |
| **Permission numbers: `r-x` = 5 (4+0+1)** | Oct 4 | 2 (14.1 → 740, 14.4 → 745) | Oct 5 test | 0 |
| `cd ..` / `../..`: count the `..`, cross out that many names | Oct 3 | 3 (S1 Q5, S2 W1 misread, drill step 2 overshoot) | Oct 5 test | 1 (S3 Q2 ✓) |
| `>` sends output into a FILE; `\|` sends it into a COMMAND | Oct 4 | 1 (bug: reason given was "erases content") | Oct 5 test | 0 |
| Kernel = CPU/memory/disk/devices; shell = reads commands | Oct 4 | 1 (10.1) | Oct 5 test | 1 (S3 Q1 ✓, same day) |
| `!` inside double quotes → zsh history expansion; use single quotes | Oct 3 | 1 | Oct 5 shell sessions | 1 (S2 W3 ✓) |
| CI DETECTS and reports failures; humans fix them | Oct 3 | 2 | Oct 6, Oct 15 | 1 (S3 W2 ✓, Oct 4) |
| DevOps lifecycle order: Plan→Code→Build→Test→Release→Deploy→Operate→Monitor | Oct 3 | 2 | Oct 5 test | 1 (S3 W1 ✓, Oct 4) |
| 'Differences' written as contrasts ("X does…, whereas Y…") | Oct 3 | 1 | Oct 5 test | 1 (15.3 ✓) |
| Server = the program (not just the machine) | Oct 3 | 0 (partial) | Oct 5 test | 0 |
| Server-side libraries live on the server, not the user's machine | Oct 3 | 1 | Oct 7 (Docker) | 1 (S2 W2 ✓) |
| Daemon definition (background service process; the "d" is only a naming habit) | Oct 4 | 1 (15.4 ½) | Oct 6 warm-up | 0 |
| **Quotes: the OUTER quotes decide. `'...'` = all literal; `"..."` expands `$` and keeps spaces together; no quotes = expand + split at spaces** | Oct 5 | **7** (env exercise, Shell I Q4 + Q6, Shell II W1, Shell II quiz Q4, Git W2, K8s I W1) | Oct 17 warm-up | 1 (K8s I set-2 W1 ✓) |
| `>` inside `[ ]` is REDIRECTION (creates a file, test becomes true); use `-gt` | Oct 7 | 1 (Shell II Q1) | Oct 8 warm-up | 0 |
| `a && b \|\| c` is not if/else: c also runs if b fails | Oct 7 | 1 (Shell II Q5) | Oct 8 warm-up | 0 |
| Cron uses the 24-hour clock (11 PM = 23); minute comes first | Oct 7 | 1 (Shell II L1) | Oct 8 warm-up | 0 |
| **301 = moved permanently, 302 = moved temporarily** | Oct 6 | 2 (curl exercise, Net quiz Q3) | Oct 13 warm-up | 1 (Git W1 ✓, Oct 11) |
| **Protocols: HTTP/HTTPS/SSH run over TCP; DNS mostly UDP (port 53)** | Oct 6 | 1 (Net quiz Q4, hedged B/D) | Oct 13 warm-up | 1 (Git W3 ✓, Oct 11) |
| A PR tracks a branch: push new commits to the same branch to address review comments | Oct 11 | 1 (Git Q3, hedged A/D) | Oct 12 warm-up | 0 |
| Resolving a conflict: edit + remove markers → `git add` (marks resolved) → `git commit` | Oct 11 | 1 (L2 ½, missed add/commit) | Oct 12 warm-up | 0 |
| **127.0.0.1 = same machine only; listen on 0.0.0.0 to accept outside connections** (in K8s: an app on 127.0.0.1 = only its own Pod, even with endpoints listed) | Oct 6 | **3** (26.3 ½, Net quiz Q2, K8s III Q1) | Oct 17 warm-up | 0 |
| Short DNS names resolve in the Pod's OWN namespace; other namespaces need `svc.namespace` | Oct 16 | 1 (57.1 guess) | Oct 17 warm-up | 0 |
| minikube (docker driver) node IP isn't reachable from the Mac → use port-forward / `minikube service --url`; tunnel is for LoadBalancer/Ingress | Oct 16 | 1 (55.2) | Oct 18 | 0 |
| Headless Service (clusterIP: None): no virtual IP, kube-proxy writes no rules, DNS returns Pod IPs | Oct 16 | 1 (Q4) | Oct 18 | 0 |
| FQDN = Fully Qualified Domain Name; format `<svc>.<ns>.svc.cluster.local` | Oct 16 | 1 (L2 ½) | Oct 17 warm-up | 0 |
| Commands: `kubectl port-forward svc/<name> <mac>:<svc>` and `kubectl exec <pod> -- <cmd>` | Oct 16 | 2 (C1, C4 idk) | Oct 17 warm-up | 0 |
| Private ranges: 10.x, 172.16–172.31.x, 192.168.x (193.168 is public!) | Oct 6 | 1 (26.1) | Oct 8 warm-up | 0 |
| Full forms: TCP = Transmission **Control** Protocol, TLS = Transport Layer Security | Oct 6 | 1 (Net quiz L1) | Oct 7 warm-up | 0 |
| Certificate check: signed by a trusted CA, right domain name, not expired → else "connection not private" | Oct 6 | 1 (L2 skipped) | Oct 7 warm-up | 0 |
| A function's exit code = its `return N` (or its last command); `$?` after the call shows it | Oct 7 | **2** (func.sh prediction, Docker W3) | Oct 14 warm-up | 1 (Docker II W1 ✓, Oct 12) |
| `git status` lists WHICH files changed; `git diff` shows WHAT changed (unstaged); `--staged` for staged | Oct 11 | 1 (Docker W1) | Oct 14 warm-up | 1 (Docker II W2 ✓) |
| ~~A stopped container keeps its NAME~~ ✅ cleared (R2 ✓ Oct 11, Docker II W3 ✓ Oct 12, and seen for real) | Oct 11 | 1 | – | 2 ✓ |
| **Layer cache: the first changed step and EVERY step after it rebuild; copy requirements → install → copy code** | Oct 12 | 1 (Q1 not understood; 38.2 right but unsure) | Oct 15 warm-up | 1 (Docker III W2 ✓; also saw it live in the build output) |
| **Compose/Docker DNS uses SERVICE/CONTAINER names, never image names** (DB_HOST=db, not postgres) | Oct 15 | 1 (K8s I W2) | Oct 16 warm-up | 0 |
| **Control plane down: running Pods keep running, but NOTHING new happens (no scheduling, no rescheduling of lost Pods)** | Oct 15 | 1 (K8s I Q1: chose "lost ones move") | Oct 16 warm-up | 0 |
| Pod journey: API server → etcd → scheduler → kubelet → runtime (stored BEFORE scheduled) | Oct 15 | 1 (48.2) | Oct 17 | 1 (set-2 L2 ✓, 48.1 ✓) |
| A running Pod's spec is mostly immutable (only the image etc. can change); Deployments handle changes | Oct 15 | 1 (K8s I Q4) | Oct 17 | 1 (set-2 Q2 ✓) |
| **apiVersion groups: Pod/Service/ConfigMap/Secret/Namespace = `v1`; Deployment/ReplicaSet/StatefulSet/DaemonSet = `apps/v1`; kind is case-sensitive (`Pod`)** | Oct 15 | 2 (set-2 49.2, Q4) | Oct 16 warm-up | 0 |
| **The kubelet restarts crashed containers LOCALLY (no control plane needed); replacing/moving Pods needs the control plane** | Oct 15 | 2 (set-1 Q1, set-2 Q1) | Oct 16 warm-up | 0 |
| Object names are unique per namespace (per kind) | Oct 15 | 1 (set-2 Q3 guess) | Oct 16 warm-up | 0 |
| etcd quorum: needs a majority (3 → tolerates 1 failure, 5 → 2) | Oct 15 | 1 (set-2 47.3) | Oct 17 | 0 |
| **Writing commands from a plain-language goal** (user-flagged weak area) | Oct 15 | 2 (49.3 idk; set-1 49.3 no commands); K8s II C4 skipped | every session (✍️ section) | 1 (K8s II C1–C3 all ✓, Oct 16) |
| Answer EVERY part of a question (commands asked for, "who writes each", full forms) | Oct 15 | 3 (46.3 CNCF, 49.3 no commands, L2 who writes) | every session | 0 |
| **Container addressing: inside a container, localhost = ITSELF; reach others by container/service NAME (Docker DNS on user-defined/Compose networks); a network name isn't an address** | Oct 14 | **3** (43.1 hedged C/A, 45.1, Q3 idk) | Oct 15 warm-up | 0 |
| **Volumes live OUTSIDE the container** (Docker's storage area, /var/lib/docker/volumes), attached at a folder; bind mount = a host folder you choose | Oct 14 | 1 (L1: said a volume is storage inside the container) | Oct 15 warm-up | 0 |
| YAML debugging answers must name the exact problem ("image has 3 spaces, ports 4; they must align"), not just "the indentation" | Oct 14 | 1 (44.1 ½) | Oct 16 | 0 |
| Shell-form CMD → a shell is PID 1 and doesn't forward SIGTERM → `docker stop` waits 10s → SIGKILL; use exec form `["…"]` | Oct 12 | 1 (Q2) | Oct 13 warm-up | 0 |
| `latest` is just a tag; no tag = `:latest`, which may be OLD; it doesn't mean newest | Oct 12 | 1 (Q4) | Oct 13 warm-up | 0 |
| `docker rmi` refuses while containers (even stopped) use the image; remove the containers first | Oct 11 | 1 (re-test R4) | Oct 13 warm-up | 0 |
| Unquoted `$1` in `[ ]` → 'unary operator expected' when empty; always `"$1"` | Oct 7 | 1 (22.3 ½) | Oct 8 warm-up | 0 |
| **`$VAR`/`${VAR}` = variable · `$(cmd)` = run command · `$((math))` = arithmetic** | Oct 5 | 2 (19.3 used `$( )` for a variable; L1 said `$(date)` is arithmetic) | Oct 6 warm-up | 0 |
| **Variable name boundary: `$A_old` looks for a variable named A_old → use `${A}_old`** | Oct 5 | 2 (19.2/19.3, Shell I Q2) | Oct 6 warm-up | 0 |
| `bash script.sh` ignores the shebang and needs no `x` (bash reads the file); `./script.sh` needs `x` and uses the shebang | Oct 5 | 1 (L2 ½) | Oct 7 | 0 |
| **umask: files start from 666, folders from 777** | Oct 5 | **2** (S4 quiz Q2, K8s II W3) | Oct 17 warm-up | 0 |
| **Label selectors: `!=` also matches objects WITHOUT that label; matchLabels = AND (every pair needed; extra labels fine); template labels must match the selector, or it's rejected** | Oct 16 | 3 (51.1, 51.2, Q2) | Oct 17 warm-up | 0 |
| Rollout maths in practice: replicas 4 + 25%/25% + broken image → 3 old + 2 new failing (max 5 total, min 3 available) | Oct 16 | 1 (Q3) | Oct 17 warm-up | 0 |
| `kubectl apply` makes the live object match the WHOLE file (image AND replicas) | Oct 16 | 1 (Q4) | Oct 17 warm-up | 0 |
| `export PATH=X` replaces PATH; use `"$PATH:X"` to add | Oct 5 | 1 (S4 quiz L3) | Oct 6 warm-up | 0 |

## Study habits to fix (flagged in sessions)

| Habit | Seen | Fix |
|-------|------|-----|
| Skips the last 1–2 questions in a set | S1, S2, S3, S4 (most rounds) | Write "idk" or a rough guess; never leave blanks |
| Answers "what" but not "why" | Log exercise Q3, 12.4, 15.4 | When a question has two parts, answer both |
| Misreads key details ("twice", "NOT") | S2 W1; Net quiz Q2 (answered a possible cause to a NOT question) | Underline numbers and keywords (NOT, twice, most) first |
| Hedging between two options ("A or B", "B or D") | Net quiz Q2, Q4; Git Q3; Docker Q2; K8s I 46.1, W2 | Commit to one; a hedge scores zero in the exam |
| One-word "difference" answers ("scalability") | 9.3 | Use the "X does…, whereas Y…" form |

## Misconception log

| Misconception | First seen | Corrected to | Re-test result |
|---------------|-----------|--------------|----------------|
| "Server = the machine" (half right) | Oct 3 | Server = a program that waits for requests and responds; the machine is called a server by extension | – |
| Missing library on server → thought "user's system" | Oct 3 | Code runs on the server; users only send requests | W2 ✓ |
| `~` = /Users | Oct 3 | `~` = your home folder, /Users/vanshsrivastava | – |
| zsh "event not found" error blamed on a missing `<` bracket | Oct 3 | It's `!` inside double quotes → use single quotes | W3 ✓ |
| CI fixes errors / rewrites code automatically | Oct 3 (twice) | CI builds + tests every merge automatically and REPORTS failures; developers fix | S3 W2 ✓ |
| DevOps = practices for 'post-coding' only | Oct 3 | DevOps covers the whole loop, including Plan and Code | – |
| Drawing the desktop is the kernel's job | Oct 4 | That's the graphical interface (a program on top of the kernel) | S3 Q1 ✓ |
| `../..` takes you all the way home | Oct 4 | Each `..` = exactly one level up | S3 Q2 ✓ |
| `r-x` = 4 | Oct 4 | r(4) + -(0) + x(1) = 5 | – |
| Daemon = process whose name ends in "d" | Oct 4 | Background process that continuously provides a service | – |
| Env var = "value that changes per device" | Oct 4 | A value set outside the code that programs read at runtime; someone sets it | – |
| PATH contains every command | Oct 4 | PATH = folders the shell searches; built-ins (cd, echo, export) aren't files | – |

## Repeatedly missed questions

| Question | Times missed | Last attempt |
|----------|--------------|--------------|
| Writing a system path without the leading `/` | 3 | S3 bug (`var/log/server.log`) |
| Repeated `cd ..` / `../..`: what does pwd print? | 3 | Drill step 2 ✗ → S3 Q2 ✓ |
| What does CI do when a test fails? | 2 | S3 W2 ✓ (recovering) |
| DevOps lifecycle order | 2 | S3 W1 ✓ (recovering) |
| Converting `r-x` style permissions to numbers | 2 | 14.4 (745) |

## Practical skills checklist

- [x] Mac Terminal: navigate (pwd/ls/cd/mkdir -p), cd drill
- [x] Ran a local web server (python3 -m http.server) and connected as a client (browser + curl)
- [?] Files: create/copy/move/rename/delete; tail; redirection; fake log + grep. **Unverified:** no `files`/`drill` folders existed on disk on Oct 5, so these exercises were likely not run
- [?] Permissions exercise (deploy.sh in `files`): unverified, same reason
- [x] Processes: background server with &, ps aux | grep, kill
- [ ] Env vars: export vs plain variable exercise (python os.environ). **Not run yet; do it in the Oct 5 warm-up**
- [x] Shell scripts: hello.sh, backup.sh (debugged missing folder, added set -e, verified exit code 1), greet/new.sh (args), check.sh (if/tests), loop.sh, func.sh
- [ ] Networking tools on Mac: curl, dig, ping, lsof
- [ ] Git: branch, merge, resolve conflict, PR on GitHub
- [ ] Docker: run/exec/logs/inspect a container
- [ ] Write a Dockerfile (incl. multi-stage, non-root) and push an image
- [ ] Docker volumes + user-defined network
- [ ] Docker Compose multi-container app
- [ ] Enable Kubernetes in Docker Desktop and use kubectl
- [ ] Pod / ReplicaSet / Deployment YAML; rolling update + rollback
- [ ] Service + Ingress; debug "no endpoints"
- [ ] ConfigMap + Secret consumed by a Pod
- [ ] PVC, probes, HPA
- [ ] Helm chart install/upgrade/rollback
- [ ] GitHub Actions CI workflow (real, on GitHub)
- [ ] Full CI/CD pipeline (build → scan → push → deploy)
- [ ] Scan an image with Trivy
- [ ] Terraform init/plan/apply/destroy (local providers)
- [ ] Draw the full DevOps lifecycle architecture from memory

## Test scores

| Date | Test | Score | Application-question score | Notes |
|------|------|-------|---------------------------|-------|
| Oct 3 | B1 S1 session quiz | 5/7 (71%) | Bug section 1/2 | MCQs 4/4 · Q5 cd .. ✗ · bug: quotes ✗ |
| Oct 3 | B1 S2 session quiz | 7/9 (78%) | Q6 scenario ✓, spot-errors 2/3 | Q3 ✗ (CI fixes code, again) · missed Deploy/Release swap |
| Oct 4 | B1 S3 session quiz | ~79% (5/5 MCQ + bug ½) | Bug: found `>`→`\|` (wrong reason), missed leading `/` | Re-tests passed: kernel vs shell, `cd ../../..` |
| Oct 4 | B1 S4 | Checks: 14.x ~60%, 15.x ~90%, 16.x 100%, 17.x 100% | – | Permission numbers weak |
| Oct 5 | S4 quiz (hard, done as warm-up) | 5.5/8 (~69%) | L2 711 ✓ (permission numbers recovering) | Q2 ✗ umask on a file (used 777 instead of 666) · Q4 hedged · L3 ✗ (`export PATH=` replaces the list) |
| Oct 5 | Env-var exercise | ½ | – | export ✓; single vs double quotes reported backwards |
| Oct 5 | Shell I quiz (hard) | 4.5/9 (50%) | Q3 args-with-quotes ✓ | Q2 ✗ `$A_old` · Q4 idk (unquoted var splits) · Q6 ✗ outer quotes · L1 ✗ `$(...)` vs `${...}` · L2 ½ |
| Oct 7 (calendar) | Shell II warm-up | 2/3 | – | W1 ✗ outer quotes · W2 ✓ · W3 ✓ (Block 1 test Q4 done) |
| Oct 7 (calendar) | Shell II checks | 11.5/12 | check.sh predictions 4/4 · set -e fix verified on disk | 22.3 ½ (unquoted $1) |
| "Oct 16" | K8s III (answered in chat) | Warm-up 3/3 · checks 9.5/12 · quiz 1/4 · L1 ✓ L2 ½ · C 2/4 | 55.3, 57.3, L1 full marks; C2, C3 ✓ | 55.2 ✗ hedged · 57.1 ✗ guess · Q1 ✗ 127.0.0.1 · Q3 unanswered · Q4 ✗ headless · C1/C4 idk |
| "Oct 16" | K8s II (answered in chat) | Warm-up 2/3 · checks 8.5/11 · quiz MCQs 1/4 | 53.1–53.3 rollout ✓, 50.x phases ✓ | W3 ✗ umask · 51.1 ✗ != · 51.2 ✗ selector mismatch · 51.3 ½ · Q2 ✗ AND · Q3 ✗ rollout maths · Q4 ✗ apply · L1 ✓ (add why) · L2 ½ (results, not settings) · C1–C3 ✓, C4 skipped |
| "Oct 15" | K8s I set 2 (new questions) | Warm-up 3/3 · checks 8.5/12 · quiz 2.5/6 | Q2 (Pod image mutable) ✓, L2 journey ✓ | 47.3 ✗ quorum · 49.2 ✗ + Q4 ✗ apiVersion · 49.3 idk (command) · Q1 ✗ kubelet · Q3 ✗ namespaces · L1 ½ |
| "Oct 15" | K8s I (answered in chat) | Warm-up 1/3 · checks 9/12 · quiz 3/6 | Q3 (scheduler, hard) ✓, 48.1 ✓ | W1 ✗ quotes · W2 ✗ image vs service name · 46.1 ✗ hedged · 48.2 ✗ order · 47.3 skipped · Q1 ✗ · Q4 ✗ immutability |
| "Oct 14" | Docker III (answered in chat) | Warm-up 3/3 · checks 5.5/8 · quiz 4.5/6 | Real port-conflict debugged live (hello vs counter-web) | 43.1 ✗ hedged · 45.1 ✗ · Q3 idk (all container addressing) · L1 ½ (volume location) · 44.1 ½ |
| "Oct 12" | Docker II (answered in chat) | Warm-up 3/3 · checks 10/10 · quiz 3/6 | L1, L2 full marks | Q1 not understood (cache order) · Q2 ✗ shell-form PID 1 · Q3 counted ✓ (question was ambiguous; reworded) · Q4 ✗ latest |
| "Oct 11" | Docker I (answered in chat) | Warm-up 1/3 · checks 10/10 · quiz 4.5/6 | (MCQ options length-biased; user flagged it) | W1 ✗ git status/diff · W3 ✗ function return · Q2 ✗ hedged · L2 ½ |
| "Oct 11" | Git session (answered in chat) | Warm-up 2/3 · checks 11/11 · quiz 4.5/6 | Q1 staging trap ✓ | W2 ✗ outer quotes · Q3 ✗ hedged (PR updates) · L2 ½ |
| "Oct 6" | Networking checks | 11.5/12 | curl/lsof/dig exercises done | 26.1 ✗ (193.168), 26.3 ½ |
| "Oct 6" | Networking quiz | 1.7/6 (28%) | Q1 CIDR ✓ | Q2 ✗ (NOT question), Q3 ✗ 302, Q4 ✗ hedged, L1 ⅔, L2 skipped |
| Oct 7 (calendar) | Shell II quiz (hard) | 3.5/8 (44%) | Q2 function exit code ✓, Q3 set -e ✓, Q6 cron PATH ✓ | Q1 ✗ `>` in [ ] · Q4 ✗ quoted loop list · Q5 ✗ && \|\| chain · L1 ✗ 11 vs 23 · L2 ½ (no fix given) |

## Daily log

| Day | Date | Planned | Actually covered | Overflow to next day |
|-----|------|---------|------------------|---------------------|
| D1 | Sat 3 Oct | B1 D1 (S1–S3) | Diagnostic · S1 (OS/process, client/server, deploying + environments, terminal basics, local web server) · S2 (Dev vs Ops, Waterfall/Agile/DevOps, definition, lifecycle, CALMS, CI/CD, IaC, DORA, monolith vs microservices) | S3 |
| D2 | Sun 4 Oct (ran to ~6:30 AM Oct 5) | B1 D2 (S4–S6) | S3 (Linux intro, kernel/shell, folder tree, abs/rel paths, cd drill, file commands, >, >>, \|, grep) · S4 (permissions + umask, processes/signals/daemons, env vars, PATH, package managers, systemd) | B1 practice + test (merged) → Oct 5 |
| D3 | "Oct 5" (calendar Oct 6 → early Oct 7) | v3: Shell I + Shell II | Warm-up (S4 quiz + env exercise) · switched to v3 · Shell I done (shebang, run modes, variables, `$(...)`, `$(( ))`, args, read, quotes); debugged backup.sh (missing folder; script continues after errors); unsaved-file lesson (Cmd+S) · Shell II done (if/tests, loops, exit codes, set -e, functions, cron) | – |
| D4 | "Oct 6" (S1) | Networking + Git | Networking done (IP, CIDR, ports, TCP/UDP, DNS, HTTP/HTTPS/TLS, commands, LB/proxy); Git not started (usage limit hit) | Git → Oct 11 |
| – | Oct 7–10 | – | **Lost days** (usage limit). Tools were installed meanwhile: Colima, docker, compose, minikube, kubectl, helm, terraform | Plan rebuilt as v4 |
| D5 | "Oct 11" | Git + Docker I | Git (answered in chat, practical pending) · Docker I rewritten at zero-knowledge level, answered + unbiased re-test 3/4 · Docker II/III session files written | – |
