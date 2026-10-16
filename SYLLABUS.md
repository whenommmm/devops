# Official Syllabus: Subtopics per Lecture

Source: shared by the user on "Oct 6" (2026). This list defines exam scope; the study plan follows it.
Legend: ✅ covered · 🟡 in progress / partly covered · ⬜ not yet · 📅 scheduled session

| # | Lecture | Official subtopics | Status | 📅 |
|---|---------|--------------------|--------|----|
| 1 | Introduction to DevOps & Microservices | Microservices vs Monolith · Microservices Architecture · What is DevOps? · DevOps Lifecycle · CI/CD Overview · DevOps Tools & Roles | ✅ except **Tools & Roles 🟡** (tools per phase done; roles pending) | Roles: Oct 8 S2 |
| 2 | Linux Fundamentals | Linux File System · Essential Linux Commands · Users & Permissions · Processes · Services · Logs & Basic Troubleshooting | ✅ except **Logs & Basic Troubleshooting 🟡** (/var/log, tail -f, journalctl, grep done; df/du/free/top pending) | Oct 8 S2 |
| 3 | Shell Scripting | Variables · Conditions · Loops · Functions · Arguments · Pipes & Redirection · Basic Automation Script | ✅ | – |
| 4 | Networking Fundamentals | IP Address · Public vs Private IP · Ports · TCP vs UDP · DNS · HTTP/HTTPS · Basic Networking Commands | ✅ covered (quiz weak: re-test in warm-ups) | Oct 6 S1 ✓ |
| 5 | Git & GitHub | Git vs GitHub · Repository & Commits · Branching · Merge & Rebase · Pull Requests · Merge Conflicts · Basic Git Workflow | ✅ covered (Oct 11); practical pending | Oct 11 S1 ✓ |
| 6 | Docker Fundamentals | Containers vs Virtual Machines · Docker Architecture · Docker Images · Docker Containers · Docker Hub · Container Lifecycle | ✅ covered (Oct 11) | Oct 7 S1 |
| 7 | Docker Images & Dockerfile | Dockerfile · FROM, RUN, COPY · WORKDIR, ENV, EXPOSE · CMD vs ENTRYPOINT · .dockerignore · Multi-stage Builds · Image Optimization | ✅ covered (Oct 12) | Oct 7 S2 |
| 8 | Docker Networking, Volumes & Compose | Docker Networking · Container-to-Container Communication · Port Mapping · Docker Volumes · Bind Mounts · Docker Compose · Multi-container Application | ✅ covered (Oct 14) | Oct 8 S1 |
| 9 | Kubernetes Fundamentals | Why Kubernetes? · Kubernetes Architecture · Control Plane & Worker Nodes · API Server · Scheduler · Controller Manager · etcd · kubelet · kubectl | ✅ covered (Oct 15) | Oct 9 S1 |
| 10 | Pods, ReplicaSets & Deployments | Pods · Pod Lifecycle · Labels & Selectors · ReplicaSets · Deployments · Scaling · Rolling Updates · Rollbacks | ✅ covered (Oct 16) | Oct 9 S2 |
| 11 | Kubernetes Networking & Services | Pod Networking · Kubernetes DNS · Service Discovery · ClusterIP · NodePort · LoadBalancer · Service Selectors · port vs targetPort | ✅ covered (Oct 16) | Oct 10 S1 |
| 12 | Ingress, ConfigMaps & Secrets | Ingress · Ingress Controller · Host & Path-based Routing · TLS · ConfigMaps · Secrets · Environment Variables | ⬜ | Oct 10 S2 |
| 13 | Storage, HPA & Probes | Volumes · PersistentVolume · PersistentVolumeClaim · StorageClass · HPA · Metrics Server · Liveness Probe · Readiness Probe · Startup Probe | ⬜ | Oct 11 S1 + S2 |
| 14 | Kubernetes troubleshooting (K8s 6–8) | kubectl get · kubectl describe · kubectl logs · kubectl exec · Events · CrashLoopBackOff · ImagePullBackOff · Pending Pods · Service & DNS Troubleshooting | ⬜ | Oct 11 S2 + Oct 12 S1 |
| 15 | Helm | What is Helm? · Helm Charts · Chart Structure · Chart.yaml · values.yaml · Templates · Install & Upgrade · Rollback · Deploying an Application with Helm | ⬜ | Oct 13 S1 |
| 16 | CI/CD & GitHub Actions | CI vs CD · CI/CD Pipeline · GitHub Actions · Workflows · Jobs & Steps · Runners · Secrets · Artifacts · Build & Test Pipeline | ⬜ | Oct 13 S2 |
| 17 | Complete CI/CD & DevSecOps | Docker Build in Pipeline · Container Registry · Kubernetes Deployment · SAST · SCA · Secret Scanning · Container Image Scanning · Security Gates | ⬜ | Oct 14 S1 |
| 18 | Terraform & IaC | What is IaC? · Terraform Architecture · Providers · Resources · Variables · Outputs · init · plan · apply · destroy · Terraform State | ⬜ | Oct 15 S1 |
| 19 | Cloud & Terraform in Action | Cloud Fundamentals · IaaS, PaaS, SaaS · Regions & Availability Zones · VPC/VNet · Subnets · Route Tables · Security Groups · Provisioning Infrastructure using Terraform | ⬜ | Oct 15 S2 |
| 20 | Monitoring, Observability & GitOps | Monitoring vs Observability · Metrics, Logs & Traces · Prometheus · Grafana · Introduction to GitOps · Git as Source of Truth · Argo CD · Continuous Deployment with GitOps | ⬜ | Oct 14 S2 |
| 21 | Final DevOps Project & Troubleshooting | Application → GitHub · Dockerize Application · CI/CD Pipeline · Security Scan · Deploy to Kubernetes · Helm · Ingress · HPA · Monitoring · Troubleshooting · Final Project Demo | ⬜ | Oct 16 S1 |

**Not in the official list (awareness only):** OSI model, NAT, scheduling (affinity/taints), RBAC, StatefulSets, DaemonSets, Jobs/CronJobs, Jenkins, Kustomize, PromQL, Terraform modules. These came from an earlier guessed "Kubernetes 6–8" list. The official K8s 6–8 content is lecture 14 (troubleshooting).

## Topics revealed by course assignments (beyond the subtopic titles)
The course's assignment questions go deeper than the subtopic list and mix topics. Add each one here as it's discovered, and make sure it's taught.
| Found in | Topic | Covered where |
|---|---|---|
| Git assignment Q2 | **Git hooks** (pre-commit / commit-msg / pre-push / server-side pre-receive), exit code decides; secret scanning as a pre-commit hook | Explained Oct 12; added to `sessions/01_git-github.md`; revisit in DevSecOps (Oct 17) |
| Small-topic audit (Oct 12) | Lectures 1–5 extras: DevOps roles/SRE/SLO, symlinks, SSH & keys, df/du/free/top/uptime/dmesg/OOM, find/sort/uniq/cut/awk/sed/tar, stdin/stdout/stderr, 2>&1, /dev/null, case, arrays, bash -x, OSI layers, subnet masks, MAC/DHCP/NAT/firewall, forward vs reverse proxy, cherry-pick/restore/reflog/blame/detached HEAD/squash/branch protection | `notes/extras_foundations.md` |
| Small-topic audit (Oct 12) | Docker extras: HEALTHCHECK, LABEL, VOLUME, digests, save/load, prune, distroless/scratch, restart policies, resource limits, docker cp, anonymous volumes, 127.0.0.1 port binding, env_file, depends_on service_healthy, compose config, --scale | "➕ Small but important" sections in sessions 03 and 04 |
| Git assignment Q3 (a Linux networking question inside the Git assignment) | **`ss` flags**: -t -u -l -a -n -p (-K); `ss -tlnp`, `ss -tulpn`, `netstat` equivalent, Mac `lsof -iTCP -sTCP:LISTEN -P -n` | Explained Oct 12; `notes/extras_foundations.md` §4d |
