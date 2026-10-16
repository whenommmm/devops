# Session 16 · Final DevOps project walkthrough + mini mock exam (Oct 18, Session 2)

**Time:** ~2 h 45 min (walkthrough ~75 min · break 10 min · **mock exam 60 min** · marking ~20 min) · **Concepts:** 90 → 92 · **Answers:** `MY_ANSWERS.md` (section 16) · **Answer key:** `16_final-project-mock_ANSWERS.md`

**How this session works:** this is the **last session before the exam**, so it teaches almost nothing new.
- Part 1 (Concepts 90–92) **joins everything together**: the whole pipeline in one picture, your final project step by step, and troubleshooting across every layer.
- Part 2 is a **60-minute mini mock exam** in the exam's format (MCQs + "fix the code"). It ends with a **last-look sheet** for the morning of the exam.

**Before you start:**
```bash
colima status          # "colima is running"
minikube status        # host, kubelet, apiserver: Running
kubectl get nodes      # minikube   Ready
```
> ⚠️ **Never delete the `minikube` or `localstack` containers.** The 🛠 tasks in this session are optional re-runs of things you've already done; skip them if you're short of time. The drawing exercise and the mock are **not** optional.
> Leave existing namespaces and Argo CD apps alone too: only delete what you create here (the `counter` release and namespace).

---

## 🔁 Warm-up (from Session 15 and the weak list)

**W1.** A teammate runs `terraform state rm aws_s3_bucket.logs` and then `terraform plan`. The `aws_s3_bucket.logs` block is still in the code. What happens?
- A) The real bucket is deleted at once in the cloud
- B) Plan wants to create it, as Terraform forgot it
- C) Plan shows no changes; the bucket is still in code
- D) Plan wants to destroy the bucket on the next apply

**W2.** A provider is constrained with `version = "~> 6.4.0"`. Which versions are allowed?
- A) Any 6.x from 6.4.0 up, but not 7.0
- B) Exactly 6.4.0, and no other version
- C) 6.4.0 and newer 6.4.x, not 6.5.0
- D) Any version at all from 6.4.0 upwards

**W3.** What does this print?
```bash
deploy() { echo "deploying"; return 1; }
deploy && echo "success" || echo "failed"
echo "code: $?"
```
- A) `deploying` / `failed` / `code: 0`
- B) `deploying` / `failed` / `code: 1`
- C) `deploying` / `success` / `code: 1`
- D) `deploying` / `success` / `code: 0`

---

## Concept 90 · The whole DevOps lifecycle, end to end (one picture)

**Words you'll learn (mostly recap):** end-to-end, source of truth, artifact, commit SHA, security gate, push-based vs pull-based deployment, feedback loop, shift left.

**The story.** Riya changes the counter app's greeting from "Hello!" to "Welcome!".
- She opens a PR; after review and merge, GitHub Actions tests and scans the code, builds an image tagged with the commit SHA, scans it, and pushes it to a registry.
- Helm or Argo CD deploys it to Kubernetes, which rolls it out safely; Prometheus and Grafana watch it, and what the team learns becomes the next work item.
- Underneath, Terraform built the network and the cluster.
- **Analogy: a car factory.** Parts go down an assembly line (CI) past inspection points (gates); finished cars wait in the car park (registry), are delivered to showrooms (CD to the cluster), and customer feedback (monitoring) shapes the next model.

**Recap words:**
- the **source of truth** = the one place everyone agrees is correct (the Git repo); an **artifact** = something a pipeline produces for later use (the image);
- a **commit SHA** = a commit's unique ID (**SHA** = Secure Hash Algorithm), `a1b2c3d` short or 40 characters full, so every image traces back to its exact code;
- a **gate** = a step whose non-zero exit code stops everything; **end-to-end** = from the first commit to the user and back.

### The big diagram (learn to draw this)
```
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │ DevSecOps runs through EVERY stage: secret scan · SAST · SCA · image scan · IaC scan     │
 └──────────────────────────────────────────────────────────────────────────────────────────┘

 ① PLAN + CODE      issue → feature branch → commits (pre-commit hook: gitleaks secret scan)
        │           → git push → Pull Request → review (+ branch protection) → merge to main
        ▼
 ② SOURCE OF TRUTH  GitHub repo: app.py · tests · Dockerfile · Helm chart · workflow · Terraform
        │           (the push/merge EVENT triggers the workflow)
        ▼
 ③ CI  (GitHub Actions, on a fresh runner)
        │   job "test":  checkout → install → pytest → SAST (Bandit) → SCA (pip-audit)
        │   job "build": needs: test → docker build -t counter:<sha> → Trivy scan ══ GATE ══╳ stop
        │                → docker push (only on main)
        ▼
 ④ REGISTRY         ghcr.io/vansh/counter:<sha>      ← an immutable, traceable artifact
        │
        ▼
 ⑤ CD               PUSH: a CI job runs  helm upgrade --install … --set image.tag=<sha>
        │           PULL (GitOps): new tag committed to Git → Argo CD (in the cluster) syncs it
        ▼
 ⑥ KUBERNETES CLUSTER  (minikube on your Mac · EKS / AKS / GKE in the cloud)
   ┌──────────────────────────────────── namespace: counter ────────────────────────────────────┐
   │                                                                                            │
   │ Ingress ◄──HTTPS── USERS   (host counter.local · TLS Secret · ingress-nginx controller)    │
   │    │ path /                                                                                │
   │    ▼                                                                                       │
   │ Service counter-web  (ClusterIP: port 80 → targetPort 5000; selector app=counter-web)      │
   │    │                                                                                       │
   │    ▼                                          HPA: CPU 50%, 2–5 replicas                   │
   │ Deployment → ReplicaSet → Pods [web] [web] … ◄── (reads Metrics Server)                    │
   │    │  probes: readiness + liveness on /health                                              │
   │    │  config: env from ConfigMap / Secret · requests + limits                              │
   │    ▼                                                                                       │
   │ Service counter-redis :6379 → Pod [redis] → PVC → PV (StorageClass "standard")             │
   └──────────────────────────────────────────────┬─────────────────────────────────────────────┘
                                                  │ /metrics scraped (pulled) every few seconds
                                                  ▼
 ⑦ OPERATE + MONITOR   Prometheus (metrics) → Grafana (dashboards) → Alertmanager (alerts)
        │              logs: kubectl logs / Loki · traces: OpenTelemetry → Jaeger/Tempo
        ▼
 ⑧ FEEDBACK            alerts · incidents · postmortems → new issues ───────► back to ① PLAN

 ⓪ UNDERNEATH          Terraform (IaC): VPC · public/private subnets · route tables ·
                       internet gateway · security groups · the managed cluster (EKS) ·
                       registry · IAM roles      (state in S3, with locking)
```
*(**Loki** = a log store Grafana can search; **OpenTelemetry** = a standard way for apps to send traces; **Jaeger** / **Tempo** = tools that store and show traces. Awareness only.)*

| # | Stage → lifecycle phase | Tool(s) | What goes in → what comes out |
|---|---|---|---|
| ⓪ | Infrastructure | **Terraform** (IaC) | `.tf` files → VPC, subnets, cluster, registry |
| ①② | Plan + code, source of truth → **Plan, Code** | Git, GitHub issues, PRs, branch protection | an idea → reviewed commits on `main` |
| ③ | **CI** (Continuous Integration) → **Build, Test** | **GitHub Actions** | a commit → a tested, scanned image |
| ④ | Registry → **Release** | GHCR (GitHub Container Registry), ECR (Elastic Container Registry), Docker Hub | image → stored, versioned artifact |
| ⑤ | **CD** (Continuous Delivery / Deployment) → **Deploy** | **Helm** (push) or **Argo CD** (pull) | new tag → cluster updated |
| ⑥ | Run → **Operate** | **Kubernetes** (self-healing, scaling) | desired state (YAML) → running Pods |
| ⑦⑧ | Monitor + feedback → **Monitor**, back to **Plan** | **Prometheus**, **Grafana**, logs, traces, postmortems | running app → alerts → next plan |

Every lecture has a place: Linux, shell, and networking run under everything (runners, containers, nodes, ports, VPCs); Docker is ③ + ⑥; Kubernetes, Ingress, storage, probes, HPA, and troubleshooting are ⑥; Helm and GitOps are ⑤; monitoring is ⑦; Terraform and cloud are ⓪.

| | **Push** (classic CI/CD: Helm from CI) | **Pull** (**GitOps**: Argo CD) |
|---|---|---|
| Who changes the cluster | The CI runner (`helm upgrade` / `kubectl apply`) | Argo CD, **inside** the cluster, watching Git; CI only updates Git (e.g. the tag in `values.yaml`) |
| Cluster credentials stored in CI? | **Yes** (a kubeconfig secret) | **No**: safer |
| Drift (someone runs `kubectl edit`) | Stays until the next deploy | **Detected**, and reverted if self-heal is on |
| Works with minikube on your laptop? | **No**: GitHub's runners can't reach your Mac | **Yes**: Argo CD reaches out to GitHub |

**Shift left** = move security checks **earlier**, where problems are cheaper to fix:
- **secret scanning** (gitleaks: pre-commit hook + CI) · **SAST** (**Static Application Security Testing**: reads source code without running it; Bandit, CodeQL, Semgrep)
- **SCA** (**Software Composition Analysis**: third-party libraries with known **CVEs**, Common Vulnerabilities and Exposures; pip-audit, Dependabot, `trivy fs`) · **image scanning** (Trivy) · **IaC scanning** (Checkov, `trivy config`)
- **DAST** (**Dynamic Application Security Testing**: attacks the running app from outside; OWASP ZAP, OWASP = Open Worldwide Application Security Project) · in the cluster: non-root containers, least-privilege RBAC, no Secrets in Git.
- A scan is only a **security gate** if its non-zero exit fails the job, so nothing ships.

**Exam recipe (~8 minutes for 10 marks):**
- (1) five boxes left to right: Code (Git) → CI (GitHub Actions) → Registry → CD (Helm/Argo CD) → Kubernetes;
- (2) users arriving through an **Ingress**, Prometheus/Grafana watching;
- (3) the **feedback arrow** from Monitoring back to Plan (that loop **is** DevOps);
- (4) Terraform as a strip underneath ("provisions VPC + cluster");
- (5) a 🔒 on CI listing secret, SAST, SCA, image (Trivy) = **gate**;
- (6) one sentence per arrow, ending with **rollback** (`helm rollback` / `kubectl rollout undo` / `git revert` with Argo CD).

**Model answer, in short sentences:** *"The developer pushes code to GitHub, which is the single source of truth. The push triggers a GitHub Actions workflow that runs unit tests, SAST, and SCA, builds a Docker image tagged with the commit SHA, scans it with Trivy, and fails the pipeline if critical vulnerabilities are found (the security gate). Only a clean image is pushed to the registry. Deployment uses Helm (`helm upgrade --install`) or, GitOps-style, Argo CD syncs the new tag from Git. In Kubernetes, a Deployment manages a ReplicaSet of Pods, a Service load-balances them, an Ingress routes external traffic, probes check health, and an HPA scales on CPU. Prometheus scrapes metrics, Grafana visualises them, and alerts feed back into planning. Terraform provisions the underlying cloud network and cluster as code."*

**Mistakes that lose marks:** no feedback arrow (then it's a pipeline, not a lifecycle) · Prometheus **before** deployment, or the registry "running" the app · "CI fixes bugs" (CI **detects and reports**; developers fix) · scans that don't fail the pipeline (no gate) · mixing up the **Ingress** (routing rules) with the **Ingress controller** (the program that applies them).

### 🛠 Do (8 min)
Close this file and draw the big diagram from memory in **5 minutes**; compare and add what you missed in another colour. Then see real pieces (read-only):
- `kubectl get nodes` (`minikube   Ready`)
- `kubectl get pods -n kube-system` (`etcd-minikube`, `kube-apiserver-minikube`, `kube-scheduler-minikube`, `kube-controller-manager-minikube`, `coredns-…`)
- `kubectl get ingressclass` (`nginx`), and `kubectl top nodes` (numbers prove Metrics Server works).

### ✅ Check 90

**90.1** In the **pull-based (GitOps)** version of the pipeline, which component actually changes the cluster?
- A) The GitHub Actions runner, using kubectl apply
- B) Argo CD, running inside the cluster itself
- C) Terraform, after it reads the Helm chart
- D) The container registry, when the tag is pushed

**90.2** Which pairing of pipeline step → security category is **WRONG**?
- A) Trivy on the built image → image scanning
- B) pip-audit on requirements.txt → SCA scanning
- C) Bandit on app.py → DAST of the running app
- D) gitleaks on the commits → secret scanning

---

## Concept 91 · The final project, step by step (as you'd demo it)

**Words you'll learn (mostly recap):** project layout, health endpoint, unit test, `minikube image load`, `# nosec`, smoke test, demo script, blameless postmortem, runbook.

The counter app (a Flask web page that counts visits, keeping the count in Redis), taken all the way to Kubernetes, in demo order. **Project layout:**
- `app.py` (plus a **`/health`** route: a **health endpoint** that answers `ok` without touching Redis, for probes and tests)
- `test_app.py` (one **unit test**, a test of one small piece: Flask's test client checks `/health` returns 200)
- `requirements.txt` (`flask==3.1.3`, `redis==5.0.8`), `Dockerfile`, `.dockerignore`, `compose.yaml`, `.github/workflows/ci-cd.yaml`
- the Helm chart `charts/counter/` (`Chart.yaml`, `values.yaml`, `templates/`: deployment, service, ingress, hpa, redis), and `infra/` (Terraform, on paper).
- **Analogy: a cooking show.** Every ingredient is prepped in order before the cameras roll, so the live demo is just the steps, plus one planned "burnt dish" you rescue on camera (the rollback).

| # | Step (lectures) | What you build / do | Key commands |
|---|---|---|---|
| 1 | App → GitHub (Git) | Commit, branch, PR; protect `main` (Settings → Branches: merges need a PR + a green workflow) | `gh repo create counter --private --source . --push` · `git switch -c feature/welcome` · `gh pr create --fill` |
| 2 | Dockerize (Docker, Dockerfile, Compose) | `FROM python:3.12-slim` → `COPY requirements.txt .` → `RUN pip install …` → `COPY app.py .` → `USER appuser` (non-root) → `CMD ["python", "app.py"]` | `TAG=$(git rev-parse --short HEAD)` (short SHA) · `docker build -t counter:$TAG .` · `docker compose up -d` |
| 3 | CI/CD pipeline (Actions, DevSecOps) | Jobs `test` → `build-scan-push` (`needs: test`) → `deploy`; each job gets a fresh runner; push/deploy steps run only `if: github.ref == 'refs/heads/main'`, so PRs are tested but never deployed | `pytest -q` · `bandit -r . -ll` · `pip-audit -r requirements.txt` · gitleaks · `docker push "$IMAGE:${{ github.sha }}"` |
| 4 | Security gate | Trivy fails the step on fixable HIGH/CRITICAL findings, so push and deploy never run | `trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 "$IMAGE:${{ github.sha }}"` |
| 5 | Into minikube (K8s) | minikube has its own container runtime, so it can't see Colima's images; the chart's `imagePullPolicy: IfNotPresent` then uses the loaded copy | `minikube image load counter:$TAG` · `minikube image ls \| grep counter` |
| 6 | Helm | Deployment (readiness + liveness on `/health`, requests/limits, `REDIS_HOST: {{ .Release.Name }}-redis`), Service 80 → 5000, Redis + PVC (`Bound` via StorageClass `standard`) | `helm lint ./charts/counter` · `helm upgrade --install counter ./charts/counter -n counter --create-namespace --set image.tag=$TAG --wait` · `kubectl get pods,svc,ingress,hpa,pvc -n counter` |
| 7 | Ingress | Host `counter.local`, path `/` → `counter-web:80` (the docker driver needs a port-forward to the controller) | `kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80` · `curl -H "Host: counter.local" http://localhost:8080/` (visited 1, then 2 times) |
| 8 | HPA | CPU 50% of the 100m request, 2–5 replicas | `kubectl run load -n counter --rm -it --image=busybox --restart=Never -- sh -c 'while true; do wget -qO- http://counter-web >/dev/null; done'` · `kubectl get hpa -n counter -w` |
| 9 | Monitoring | kube-prometheus-stack: one chart with Prometheus, Grafana, and Alertmanager (~1.5 GB RAM; `helm uninstall monitoring -n monitoring` afterwards) | `helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace` · `kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80` |
| 10 | Break it, fix it | Deploy a tag that doesn't exist, then roll back | `helm upgrade counter ./charts/counter -n counter --reuse-values --set image.tag=2.0` · `helm history counter -n counter` · `helm rollback counter 1 -n counter` |

- **The scan findings (great exam stories).** Bandit reported **`B104: Possible binding to all interfaces`** on `app.run(host="0.0.0.0")`.
  - In a container the app **must** listen on `0.0.0.0` (or `-p` and Services can't reach it), so "fixing" it to 127.0.0.1 breaks the app; the professional fix is an explicit, reviewed exception on that one line: `# nosec B104 (containers must listen on all interfaces)`.
  - pip-audit reported `flask 3.0.3 PYSEC-2026-2151, Fix Versions 3.1.3`: the fix for a vulnerable dependency is to **upgrade** to the fixed version, re-run the tests, and commit the new pin.
- ⚠️ **Never "fix" a gate by deleting the scan or adding `|| true`** (`true` always exits 0, so the gate is gone).
- **Image tags:** skip `minikube image load` and Pods show **`ErrImagePull` / `ImagePullBackOff`** (Kubernetes looks for `docker.io/library/counter:…` on Docker Hub).
  - A `:latest` tag makes `imagePullPolicy` default to **`Always`**, so use a real tag. In CI, **`github.sha` is the full 40-character SHA**: deploy exactly the tag you pushed (deploying the short SHA instead finds no image: `ImagePullBackOff`).
- **Helm details:** with autoscaling on, the template **leaves out `replicas`** so the HPA controls the count.
  - Without `--wait`, `helm upgrade` reports `STATUS: deployed` as soon as Kubernetes accepts the YAML, even while the new Pod sits in `ImagePullBackOff` (the old Pods keep serving).
  - After `helm rollback`, `helm history` shows **revision 3, "Rollback to 1"**: a rollback is a **new** revision.
- **Checks along the way:** a request without the `Host: counter.local` header gets **`404 Not Found`** from nginx.
  - The HPA shows `cpu: <unknown>/50%` for the first 1–2 minutes, then grows by desired = ceil(current × actual ÷ target) and shrinks ~5 minutes after the load stops.
  - In Grafana (user `admin`, password in the `monitoring-grafana` Secret), Prometheus **pulls** (scrapes) `/metrics`; watch the **four golden signals**: latency, traffic, errors, saturation.
- **The troubleshooting story.** The chart copied `REDIS_HOST: cache` from `compose.yaml`. Pods were `1/1 Running` (the probes hit `/health`, which skips Redis), but `/` returned **500**.
  - `kubectl logs deploy/counter-web -n counter` showed `Error -2 connecting to cache:6379. Name or service not known` (DNS found no `cache`); `kubectl get svc -n counter` listed `counter-redis`.
  - Fix the template, `helm upgrade counter ./charts/counter -n counter --reuse-values`, and prevent a repeat with a **smoke test** (a tiny post-deploy check: `curl` must return 200).
  - Then write a **blameless postmortem** (what happened, impact, root cause, fix, prevention; no blaming people) and add a **runbook** entry (the team's step-by-step guide for known problems).
- **Pull-style alternative:** point an Argo CD `Application` at `charts/counter`; changing `image.tag` in `values.yaml` and pushing to Git is then the whole deployment.
- **Demo script (5–7 min):** the picture → repo, PR, branch protection → pipeline going green (and the gate failing once: the B104 story) → the image tagged with the SHA
  - → `kubectl get pods,svc,ingress,hpa,pvc -n counter` + curl twice → load test + HPA → Grafana → bad tag → `ImagePullBackOff` → `helm rollback`
  - → "In the cloud, Terraform builds the VPC and the EKS cluster underneath."

### 🛠 Do (optional, ~25 min)
Run steps 2 and 5–8 plus the rollback on your minikube, using the files in `~/devops-practice/counter/`. Clean up only what you made: `helm uninstall counter -n counter && kubectl delete namespace counter`.

### ✅ Check 91

**91.1** You built `counter:1.0` with Colima's Docker and deployed it to minikube with `imagePullPolicy: IfNotPresent`. The Pods show `ErrImagePull: pull access denied, repository does not exist`. Which command was missed?
- A) `minikube image load counter:1.0`
- B) `docker image push counter:1.0`
- C) `kubectl rollout restart deploy/web`
- D) `minikube addons enable registry`

**91.2** Bandit fails the pipeline with `B104: Possible binding to all interfaces` on `app.run(host="0.0.0.0")` in the containerised app. What's the best fix?
- A) Change it to 127.0.0.1 so Bandit is satisfied
- B) Delete the SAST step so the pipeline goes green
- C) Run Bandit with `|| true` so failures are ignored
- D) Keep 0.0.0.0; add `# nosec B104` with a reason

---

## Concept 92 · Cross-syllabus troubleshooting: 10 scenarios

**Words you'll learn (mostly recap):** symptom, root cause, first command, Events, EndpointSlice, stale lock, drift.

**The method (every layer):** read the **exact** error → run the **first command** that shows more → form **one** hypothesis → fix **one** thing → **verify**.
- **Analogy: a doctor.** Symptoms first, then one test that narrows it down, a diagnosis, one treatment, and a check-up.
- A **symptom** = what you see; the **root cause** = why it happens; an **EndpointSlice** = the list of Ready Pod addresses behind a Service; a **stale lock** = a lock left behind by a crashed run.
- Cover the last two columns and try to fill them in first.

| # · layer | Symptom | First command | Likely cause | Fix |
|---|---|---|---|---|
| 1 · Git | `! [rejected] main -> main (fetch first)` … `the remote contains work that you do not have locally` | `git fetch origin` then `git log --oneline main..origin/main` | A teammate pushed first; your `main` is **behind**. (`GH006: Protected branch update failed` = `main` is protected: use a PR) | `git pull --rebase origin main`, resolve (edit → `git add` → `git rebase --continue`), `git push`. **Never `--force` a shared branch** |
| 2 · Actions | "Log in to GHCR" fails with `password is empty`; or `Error: Process completed with exit code 1.` | The failed step's log, or `gh run view --log-failed` | The secret is missing or misspelled (`secrets.GHCR_TOKN`), so an empty password was piped (secrets also aren't passed to fork PRs). Exit 1 = a command failed (test, scanner, `set -e`) | Create the secret with the **exact** name; use `${{ secrets.NAME }}`. Read the line above "exit code 1" and fix it; **don't** add `\|\| true` |
| 3 · Dockerfile | One `app.py` change re-runs `pip install`; or `COPY failed: file not found in build context` | `docker build .` (which steps say **`CACHED`**?) · `cat .dockerignore` | `COPY . .` before `RUN pip install` invalidates that layer **and every one after it**. (COPY error: file outside the build context or in `.dockerignore`) | `COPY requirements.txt .` → `RUN pip install …` → `COPY . .`; build from the right folder |
| 4 · Docker | `docker ps -a`: `Exited (1) 3 seconds ago` | `docker logs <name>` | PID 1 crashed: `ModuleNotFoundError: No module named 'flask'`, a missing env var, a `CMD` typo. (`Exited (137)` = 128 + 9 = **SIGKILL**, often OOM, Out Of Memory) | Fix what the log says, rebuild, rerun; look inside with `docker run -it --entrypoint sh <image>` |
| 5 · K8s + CI | New Pod `ImagePullBackOff`; old Pods still `Running` | `kubectl describe pod <pod> -n counter` → **Events** | `Failed to pull image "ghcr.io/vansh/counter:a1b2c3d": … not found`: CI pushed the **full 40-char** `github.sha`, the deploy used the **short** SHA. (Also: a name typo; a private registry with no `imagePullSecrets` → `401 Unauthorized`) | One tag variable for push **and** deploy (`--set image.tag=<full sha>`), or `helm rollback`; private registry: a `docker-registry` Secret under `imagePullSecrets` |
| 6 · Scheduling | `Pending` for minutes, no restarts, no IP | `kubectl describe pod <pod>` → Events | `0/1 nodes are available: 1 Insufficient cpu.`: the Pod **requests** more than any node has free. (Also: `unbound immediate PersistentVolumeClaims`; a taint/nodeSelector no node matches) | Lower the **requests**, free capacity, or add nodes. **Requests** decide scheduling; **limits** cap runtime use |
| 7 · Service | `curl http://counter-web` from a Pod: `Connection refused` / timeout; Pods `Running` | `kubectl get endpointslices -l kubernetes.io/service-name=counter-web` (or `kubectl get endpoints counter-web`) → none | The Service **`selector`** (`app: counter`) ≠ the Pods' **labels** (`app: counter-web`), or the Pods aren't **Ready**. (Endpoints exist but fail → **`targetPort`** ≠ the app's port) | `kubectl get pods --show-labels`; make selector and labels match; fix the readiness probe; `targetPort: 5000` |
| 8 · Ingress | `curl http://localhost:8080/` → `404 Not Found` with an `nginx` footer | `kubectl describe ingress counter-web -n counter` · `kubectl get ingressclass` | The **Host header** ≠ the rule's host (`counter.local`), so the default backend answers; or a path mismatch; or no `ingressClassName`. (**503** = no ready endpoints: the Service's selector doesn't match the Pods, or they aren't Ready; **connection refused** = no controller) | `curl -H "Host: counter.local" …` (or an `/etc/hosts` entry); fix the path / `pathType: Prefix`; add `ingressClassName: nginx` |
| 9 · HPA | `TARGETS cpu: <unknown>/50%`, never scales | `kubectl describe hpa counter-web` → Conditions/Events | `missing request for cpu` (utilisation is a % **of the request**) or `no metrics returned from resource metrics API` (**Metrics Server** missing). Normal for the first 1–2 minutes | Add `resources.requests.cpu` (e.g. `100m`); `minikube addons enable metrics-server`; check `kubectl top pods`; wait a minute |
| 10 · Terraform | (a) `Error acquiring the state lock … Who: runner@fv-az123 … OperationTypeApply`; (b) `plan` changes an SG **nobody changed in Git** | (a) Is that run (teammate or CI) **still going**? (b) Read which attributes change, from what to what | (a) Another apply holds the lock, or a crash left a **stale** lock. (b) **Drift**: a hand edit in the console | (a) **Wait**; only if nothing is running: `terraform force-unlock <LOCK_ID>`; never delete the state or use `-lock=false`. (b) `apply` to restore the code's version, or update the code; then stop console edits (read-only access, PRs only) |

**First command per layer (memorise):**
| Layer | First command |
|---|---|
| Git · shell | `git status` · `git fetch` + `git log --oneline main..origin/main` · `bash -x script.sh` (prints each line as it runs) · `echo $?` |
| Pipeline · Docker | Actions tab → failed step log · `gh run view --log-failed` · `docker ps -a` → `docker logs <name>` → `docker inspect` |
| Kubernetes Pod | `kubectl get pods` → `kubectl describe pod` (Events) → `kubectl logs [--previous]` → `kubectl exec -it … -- sh` |
| K8s networking · Ingress | `kubectl get svc,endpointslices` · `kubectl get pods --show-labels` · `kubectl run tmp --rm -it --image=busybox -- nslookup <svc>` · `kubectl describe ingress` · `curl -H "Host: …"` |
| HPA · Helm | `kubectl describe hpa` · `kubectl top pods` · `helm status` · `helm history` · `helm get values` · `helm template` |
| Terraform · cloud network | `terraform validate` · `plan` · `state list` · public IP? → route `0.0.0.0/0 → IGW`? → SG port? → app on 0.0.0.0? |

### ✅ Check 92

**92.1** `kubectl get pods` shows three Pods, each `1/1 Running`. But `kubectl get endpointslices -l kubernetes.io/service-name=web` shows **no endpoints** for the `web` Service. What's the most likely cause?
- A) The Service's port differs from the targetPort
- B) CoreDNS has no record yet for the Service name
- C) The Service's selector doesn't match Pod labels
- D) The Pods' readiness probes are failing right now

**92.2** `terraform apply` fails with `Error acquiring the state lock`, and the lock info says `Who: runner@fv-az123` (a CI runner). What should you do **first**?
- A) Run `terraform force-unlock` right away and apply
- B) Check whether that CI run is still in progress
- C) Delete terraform.tfstate so the lock disappears
- D) Re-run with `-lock=false` to skip the lock

---

## ➕ Small but important (project & cross-topic): read before the mock

| Item | What it is |
|---|---|
| `kubectl rollout status deploy/X` | Waits until a rollout finishes, and **exits non-zero if it fails**. Good as a pipeline step after deploying |
| `helm upgrade --install` | Install if the release doesn't exist, otherwise upgrade: **safe to re-run** (idempotent). `--wait` = wait until the Pods are Ready before reporting success. Helm 4's **`--rollback-on-failure`** (called `--atomic` in Helm 3) rolls back automatically if the upgrade fails |
| `helm get values <release>` / `helm status <release>` | The values a release is actually using / its current state and notes |
| **Image tags** | Tag with the **commit SHA** (or a version like `1.4.2`): unique and traceable. Avoid deploying `:latest`: you can't tell which code it is, rollbacks are unclear, and it makes `imagePullPolicy` default to **`Always`** |
| `imagePullPolicy` defaults | Tag `:latest` or no tag → **`Always`**; any other tag → **`IfNotPresent`** |
| **Build once, deploy many** | Build **one** image per commit and **promote the same image** through dev → staging → prod, changing only config (values, ConfigMaps, Secrets). Never rebuild for each environment |
| **Deployment strategies** | **Rolling update** (Kubernetes default: replace Pods gradually) · **Recreate** (stop all, then start new: brief downtime) · **Blue-green** (run the new version alongside, then switch all traffic at once) · **Canary** (send a small % of traffic to the new version first) |
| **Manual approval for prod** | GitHub **Environments** with required reviewers pause a job until someone approves. That's **Continuous Delivery** (prod needs a human click); with no approval it's **Continuous Deployment** |
| `kubectl port-forward` / `minikube tunnel` / `minikube service --url` | Three ways to reach a Service from your Mac when minikube uses the docker driver |
| `kubectl top pods` / `kubectl top nodes` | Live CPU/memory, from **Metrics Server** (same data the HPA uses) |
| **DORA metrics** (**DORA** = DevOps Research and Assessment) | **Deployment frequency**, **lead time for changes**, **change failure rate**, **time to restore service** (MTTR, Mean Time To Restore). The automated commit → CI → registry → CD → monitoring pipeline improves all four |
| **Blameless postmortem** / **runbook** | A write-up after an incident focused on causes and fixes, **not** on blaming people / a step-by-step guide for handling known problems |
| **Smoke test** | A tiny check right after deploying ("does `/` return 200?") that catches broken releases fast |
| **Pin actions** | `uses: actions/checkout@v4` trusts whatever the `v4` tag points to. Security-strict teams pin to a full commit SHA (`@8f4b7f8…`), because tags can be moved (the same tag-vs-digest idea as images) |

**Quick checks:** (1) Why does `helm upgrade` show `STATUS: deployed` even when the new Pods are in `ImagePullBackOff`? (2) Which strategy sends 5% of users to the new version first?
*Answers:* (1) Without `--wait`, Helm reports success as soon as Kubernetes **accepts** the YAML; it doesn't wait for the Pods to become Ready. (2) Canary.

---

### ✍️ Write the command (whole syllabus)
**C1.** Your `git push` to `main` was rejected with `! [rejected] main -> main (fetch first)`. Bring in your teammate's new commits by **replaying your commits on top of** the remote's `main` (no merge commit); then, only if that worked, push again.

**C2.** Save the current commit's **short** SHA in a shell variable `TAG`; then, only if that worked, build an image named `counter`, tagged with that SHA, from the current folder.

**C3.** You changed `app.py` in the counter project. Rebuild the images and start the Compose stack in the background; then, only if that worked, list the stack's containers and their status.

**C4.** The `counter-web` Service has no endpoints. List all Pods in the `counter` namespace **with their labels shown**, so you can compare them with the Service's selector.

**C5.** Deploy the local chart `./charts/counter` as the release `counter` in the namespace `counter`: install it if it doesn't exist yet, otherwise upgrade it; create the namespace if it's missing; set the value `image.tag` to the contents of `$TAG`; and report success only once the Pods are Ready.

**C6.** Make a Terraform plan using the variable values in `prod.tfvars` and save it to a file named `tfplan`; then, only if planning worked, apply **exactly that saved plan**.

## 📝 Mini mock exam (60 minutes)

**Rules:** set a **60-minute timer**. Closed book: no notes, no terminal. Write your answers in `MY_ANSWERS.md` under **16**. For MCQs, commit to **one** letter (a hedge scores zero). For the debugging questions, **list every bug** (what's wrong and what it causes) and **write the fixed code**.

**Marking:** MCQs 1 mark each (25) + debugging 3 marks each, one per bug found **and** fixed (15) = **40 marks**. Full-marks target: **≥ 36/40 (90%)**.
**Weighting** (the expected exam mix): Kubernetes ~35% · Docker ~15% · Linux/shell/networking/Git ~20% · CI/CD + DevSecOps ~12% · Terraform/Cloud ~10% · Monitoring/GitOps/Helm ~8%.

### Part A · MCQs (25)

**M1.** What does this print?
```bash
MSG="build ok"
for w in $MSG; do echo "[$w]"; done
for w in "$MSG"; do echo "[$w]"; done
```
- A) `[build]` / `[ok]` / `[build]` / `[ok]`
- B) `[build ok]` / `[build]` / `[ok]`
- C) `[build]` / `[ok]` / `[build ok]`
- D) `[build ok]` / `[build ok]`

**M2.** An engineer runs `kubectl get pods` and sees `api-6f7c9-x2k   0/1   CrashLoopBackOff   5 (40s ago)`. `kubectl logs api-6f7c9-x2k` prints nothing, because the container has just been restarted. Which command most directly shows why it crashed?
- A) `kubectl logs api-6f7c9-x2k -p`
- B) `kubectl get ep api -o wide`
- C) `kubectl top pod api-6f7c9-x2k`
- D) `kubectl rollout undo deploy/api`

**M3.** A Flask app inside a container calls `app.run(host="127.0.0.1", port=8000)`. It's started with `docker run -d -p 8000:8000 app`. `docker ps` shows `0.0.0.0:8000->8000/tcp`, but `curl localhost:8000` on the Mac fails with `curl: (52) Empty reply from server`. Which statement best describes the cause?
- A) EXPOSE 8000 is missing, so the port was never opened
- B) The app listens only on the container's own loopback
- C) Published ports can't be reached via localhost on a Mac
- D) -p must be written as 127.0.0.1:8000:8000 on a Mac

**M4.** A Service has `port: 80` and `targetPort: 8080`. Its Pods run the counter image, which listens on **5000**, and they declare `containerPort: 5000`. From another Pod, `curl http://counter` fails with `Connection refused`. Which single change fixes it?
- A) Change the Service's `port` to 5000
- B) Set `containerPort: 8080` on the Pods
- C) Add a `nodePort: 30080` to the Service
- D) Set `targetPort: 5000` on the Service

**M5.** A commit that broke production has already been pushed to the shared `main` branch, and teammates have pulled it. What's the safest way to undo it?
- A) `git reset --hard HEAD~1`, then `git push --force`
- B) `git revert <sha>`, then push the new commit
- C) `git checkout HEAD~1`, then push from there
- D) `git commit --amend`, then push it again

**M6.** A workflow's `test` job runs `pytest --junitxml=report.xml`. A second job, `publish`, has `needs: test` and runs `cat report.xml`, which fails with `No such file or directory`. Why?
- A) The publish job must run actions/checkout to get it
- B) needs: was ignored, so both jobs ran at the same time
- C) Jobs don't share files; pass it as an artifact
- D) The file is in .gitignore, so the runner deleted it

**M7.** A Deployment has 4 replicas with `maxSurge: 1` and `maxUnavailable: 0`. CI deploys a new image tag that **doesn't exist** in the registry. A few minutes later, what do you see?
- A) One new Pod in ImagePullBackOff; 4 old Pods serving
- B) All 4 old Pods replaced, and the app is fully down
- C) Kubernetes rolls back to the old version on its own
- D) The Deployment is deleted after its progress deadline

**M8.** `terraform plan` shows `-/+ resource "aws_db_instance" "main"` with `# forces replacement`. The database holds customer data. Which `lifecycle` setting would make Terraform **refuse** to run a plan like this?
- A) `create_before_destroy = true`
- B) `ignore_changes = all`
- C) `prevent_destroy = true`
- D) `replace_triggered_by = []`

**M9.** A Pod's **readiness** probe starts failing because its database is down, while its **liveness** probe keeps passing. What does Kubernetes do?
- A) Restarts the container until the probe passes
- B) Deletes the Pod and schedules it on a new node
- C) Marks the Deployment as failed and rolls back
- D) Removes the Pod from the Service's endpoints

**M10.** A user's umask is `027`. They create a new **file** with `touch report.txt`. What does `ls -l` show for its permissions?
- A) `-rwxr-x---`
- B) `-rw-r-----`
- C) `-rw-rw-r--`
- D) `-rw-r--r--`

**M11.** A pipeline scan reads `requirements.txt` and reports `requests 2.19.0 — CVE-2018-18074 (HIGH), fixed in 2.20.0`. What kind of scan produced this?
- A) SCA: it checked third-party dependencies
- B) SAST: it analysed the app's own source code
- C) DAST: it attacked the running application
- D) Secret scanning: it searched for leaked keys

**M12.** A Deployment reads `LOG_LEVEL` from a ConfigMap through `envFrom`. You change the ConfigMap to `LOG_LEVEL=debug` with `kubectl apply`. An hour later the app still logs at `info`. Why, and what fixes it?
- A) Env vars are read at start; run kubectl rollout restart
- B) ConfigMaps update only after a kubelet reboot on the node
- C) The ConfigMap must be base64-encoded to be picked up
- D) envFrom works only with Secrets; switch it to a Secret

**M13.** A Dockerfile ends with:
```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "5000"]
```
You run `docker run img --port 8000`. What runs inside the container?
- A) `python app.py --port 5000`; your args are ignored
- B) `--port 8000` alone; ENTRYPOINT is replaced
- C) `python app.py --port 8000`; CMD is replaced
- D) `python app.py`; both CMD and args are dropped

**M14.** From an app server, `nc -zv db.internal 5432` returns `Connection refused` **instantly**. From a second server, the same command **hangs, then times out**. Which reading is most accurate?
- A) Refused = DNS failed; timeout = wrong port number
- B) Refused = firewall drop; timeout = app not running
- C) Both mean the database rejected the password
- D) Refused = no listener; timeout = firewall drop

**M15.** An HPA targets **50%** average CPU utilisation, with `maxReplicas: 10`. The Deployment has **2** Pods, averaging **90%** of their CPU requests. How many replicas will the HPA ask for?
- A) 2 replicas
- B) 3 replicas
- C) 4 replicas
- D) 5 replicas

**M16.** What happens when this runs in an empty folder?
```bash
COUNT=2
if [ $COUNT > 5 ]; then echo "big"; else echo "small"; fi
ls
```
- A) `small`, and `ls` shows nothing new
- B) `big`, and `ls` shows a new file named `5`
- C) An error: `>` isn't allowed inside `[ ]`
- D) `small`, and `ls` shows a new file named `5`

**M17.** Your laptop's `terraform init` installed AWS provider **6.12.0** last month; a teammate's fresh `init` today installed **6.68.0**. Both used `version = "~> 6.0"`. What makes everyone get the same version?
- A) Commit `.terraform.lock.hcl` to Git
- B) Commit the `.terraform/` folder to Git
- C) Commit `terraform.tfstate` to Git
- D) Run `terraform fmt` before each init

**M18.** A database server in a **private** subnet must download security patches from the internet, but nothing on the internet may **start** a connection to it. What should you add?
- A) A route 0.0.0.0/0 → internet gateway for the subnet
- B) An inbound security group rule for 443 from 0.0.0.0/0
- C) A NAT gateway in a public subnet as its default route
- D) An Elastic IP attached directly to the database server

**M19.** Prometheus's **Targets** page shows `counter:5000/metrics` as **DOWN** with `connection refused`, and Grafana's panel for the app shows no data. Which is most likely?
- A) Grafana's data source URL points to the wrong server
- B) Prometheus can't scrape the app's metrics endpoint
- C) The app must push its metrics to Prometheus first
- D) Alertmanager has silenced every alert for the app

**M20.** You run `helm install web ./chart` (revision 1), then `helm upgrade web ./chart --set image.tag=v2` (revision 2), which breaks the app, then `helm rollback web 1`. What does `helm history web` show as the **latest** revision?
- A) Revision 1, marked deployed again
- B) Revision 2, marked as rolled back
- C) Revision 0, a fresh install
- D) Revision 3, "Rollback to 1"

**M21.** A Pod in namespace `shop` calls `http://payments:8080`. The `payments` Service exists in namespace **`billing`**. The call fails with `Could not resolve host: payments`. What fixes it?
- A) Call `billing.payments:8080` instead
- B) Call `payments.billing:8080` instead
- C) Expose `payments` as a NodePort Service
- D) Add `hostNetwork: true` to the shop Pod

**M22.** Which statement about this Dockerfile's **final** image is true?
```dockerfile
FROM python:3.12 AS build
WORKDIR /src
COPY requirements.txt .
RUN pip wheel -r requirements.txt -w /wheels

FROM python:3.12-slim
COPY --from=build /wheels /wheels
RUN pip install /wheels/*
```
- A) It has the slim base plus only the copied wheels
- B) It still contains the full python:3.12 image as a layer
- C) It's larger, because both FROM images are stacked in it
- D) It can't install the wheels, as build tools were dropped

**M23.** A teammate says: "Our DB password is safe because it's in a Kubernetes Secret." The Secret's YAML shows `password: cGFzc3dvcmQxMjM=`. Which statement is accurate?
- A) It's encrypted; only the API server holds the key
- B) It's a hash, so the password can't be recovered
- C) It's base64: anyone who can read it can decode it
- D) It's safe to commit, since Git can't decode base64

**M24.** You delete one Pod of a Deployment that has `replicas: 3`. Which sequence best describes what happens next?
- A) Scheduler creates a Pod → etcd picks a node → kube-proxy runs it
- B) kubelet restarts the same Pod on the same node → scheduler records it
- C) API server restores the old Pod from etcd → kubelet runs it again
- D) ReplicaSet makes a new Pod → scheduler picks node → kubelet runs it

**M25.** The pipeline runs `trivy image --severity CRITICAL --exit-code 1 app:1.4`. The image has **3 HIGH** and **0 CRITICAL** vulnerabilities. What happens?
- A) The step fails, because HIGH counts as a finding
- B) The step fails, since exit-code 1 always fails it
- C) The step passes, but the image push is blocked
- D) The step passes; HIGH ones aren't checked at all

### Part B · Debugging / "fix the code" (5 × 3 marks)
Each file has **3 planted bugs**. For each bug: say **what's wrong**, **what goes wrong because of it**, and **write the fixed file**.

**D1. Shell script** (`release.sh`, run as `bash release.sh push`)
```bash
#!/bin/bash
# release.sh: build the counter image, and push it if asked
set -e

TAG = $(git rev-parse --short HEAD)
IMAGE="ghcr.io/vansh/counter:$TAG"

build() {
  docker build -t "$IMAGE" .
}

build
if [ $1 = "push" ]; then
  docker push "$IMAGE"
  echo 'Pushed $IMAGE'
fi
```

**D2. Dockerfile** (for the counter app; the image builds, but the container won't start, and every code change reinstalls everything)
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
USER appuser
EXPOSE 5000
CMD ["python" "app.py"]
```

**D3. Compose file** (`compose.yaml`; `docker compose up` refuses to start, and even after that the web app can't reach Redis)
```yaml
services:
  web:
    build: .
    ports:
      - "8080:5000"
    environment:
      - REDIS_HOST=localhost
    depends_on:
      - redis
  cache:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
```

**D4. Kubernetes YAML** (`counter.yaml`; `kubectl apply` fails, and after that's fixed it still fails, and even then the Service doesn't work)
```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: counter
spec:
  replicas: 2
  selector:
    matchLabels:
      app: counter
  template:
    metadata:
      labels:
        app: counter-web
    spec:
      containers:
        - name: counter
          image: ghcr.io/vansh/counter:1.0
          ports:
            - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: counter
spec:
  selector:
    app: counter
  ports:
    - port: 80
      targetPort: 8080
```

**D5. GitHub Actions workflow** (`.github/workflows/ci.yaml`)
```yaml
name: CI
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt pytest
      - run: pytest -q

  build:
    needs: tests
    runs-on: ubuntu-latest
    steps:
      - name: Build image
        run: docker build -t ghcr.io/vansh/counter:${{ github.sha }} .
      - name: Log in to GHCR
        run: echo "${{ secret.GHCR_TOKEN }}" | docker login ghcr.io -u vansh --password-stdin
      - name: Push image
        run: docker push ghcr.io/vansh/counter:${{ github.sha }}
```

**⏱ Stop when the timer ends.** Then mark yourself with `16_final-project-mock_ANSWERS.md`. For every question you missed, the answer key names the session to re-read.

---

## 📋 Last-look exam-day sheet (top 30)

Read this on the morning of the exam. Each line is something examiners love to test.

**Foundations**
1. **Lifecycle:** Plan → Code → Build → Test → Release → Deploy → Operate → Monitor. **CI** builds and tests every merge and **reports** failures (it doesn't fix them). **Continuous Delivery** = always releasable, prod needs a human click; **Continuous Deployment** = straight to prod automatically.
2. **Quotes:** the **outer** quotes decide. `'…'` = literal; `"…"` = expands `$` and keeps spaces together; no quotes = expand and **split at spaces**. Always write `"$VAR"`.
3. **Exit codes:** `$?` = the last command's code; a function's code = its `return N` (or its last command). `a && b || c` is **not** if/else. `>` inside `[ ]` is **redirection**; use `-gt`.
4. **Permissions:** r=4, w=2, x=1 (`r-x` = 5, `rw-r-----` = 640). umask subtracts from **666** for files and **777** for folders.
5. **127.0.0.1** = this machine only; **0.0.0.0** = listen on every interface. **Apps in containers must listen on 0.0.0.0.**
6. **Private ranges:** 10.x, 172.16–31.x, 192.168.x. `/24` = 256 addresses, `/16` = 65,536, `/32` = one, `0.0.0.0/0` = everything.
7. **Protocols & codes:** HTTP/HTTPS/SSH (22) use **TCP**; DNS is mostly **UDP 53**. **301** = moved permanently, **302** = temporarily; 404 = not found; 502 = bad gateway; 503 = service unavailable.
8. **Git:** `git status` = **which** files; `git diff` = **what** changed (unstaged; `--staged` for staged). Conflict: edit → `git add` → `git commit`. Undo a pushed commit: **`git revert`**. A PR updates when you push to its branch.

**Docker**
9. Image = program, container = process. A stopped container **keeps its name**; `docker rmi` refuses while any container (even stopped) uses the image.
10. **Cache:** the first changed layer **and every layer after it** rebuild → `COPY requirements.txt` → `RUN pip install` → `COPY . .`.
11. **CMD** = default arguments (replaced by `docker run` args); **ENTRYPOINT** = the fixed program (args are appended). Exec form needs **commas**: `["python", "app.py"]`.
12. **EXPOSE** only documents; **`-p HOST:CONTAINER`** publishes. Volume (Docker-managed, a **name**) vs bind mount (a host **path**). `compose down` keeps volumes; `down -v` deletes them.
13. In Compose, **service names are DNS names**; inside a container `localhost` = **that container**. `depends_on` waits for **start**, not **ready**.

**Kubernetes**
14. **API server** (front door) · **etcd** (the database) · **scheduler** (picks the node) · **controller manager** (makes actual = desired) · **kubelet** (runs Pods on its node) · **kube-proxy** (Service networking).
15. **Deployment → ReplicaSet → Pods**, joined by **labels/selectors**; the selector must match the template labels. `kubectl rollout status / history / undo`.
16. **ClusterIP** (inside only) · **NodePort** (30000–32767 on every node) · **LoadBalancer** (a cloud load balancer). Traffic: Service **`port`** → **`targetPort`** (the container's real port); `containerPort` is informational.
17. **DNS:** same namespace = `svc`; other namespace = `svc.namespace` (full: `svc.namespace.svc.cluster.local`).
18. **Ingress** = L7 (HTTP-level) host/path rules; it needs an **Ingress controller** to work. TLS = a Secret of type `kubernetes.io/tls`.
19. **ConfigMap** (plain settings) vs **Secret** (**base64**, NOT encryption). Env vars are read **at container start** → `kubectl rollout restart` after changing them.
20. **PV** (the actual storage) · **PVC** (a request for storage) · **StorageClass** (creates PVs on demand: dynamic provisioning).
21. **Liveness** fails → **restart**; **readiness** fails → **removed from endpoints** (no restart); **startup** → holds the other probes until the app has started.
22. **HPA** needs **Metrics Server** + **CPU requests**; desired = **ceil(current × actual ÷ target)**; `<unknown>` = no metrics (or no requests).
23. **Troubleshoot:** `get` → `describe` (Events) → `logs` (`--previous`) → `exec`. **CrashLoopBackOff** = the app keeps crashing → logs. **ImagePullBackOff** = wrong name/tag/credentials → describe. **Pending** = can't be scheduled (Insufficient cpu, PVC unbound) → describe.

**Automation, security, infrastructure**
24. **Helm:** chart = package, release = an installed copy; `values.yaml` = defaults, `--set` overrides; `helm upgrade --install`; `helm rollback` creates a **new** revision.
25. **GitHub Actions:** workflow → jobs (**parallel** unless `needs:`; a **fresh runner** each) → steps (`uses:` = an action, `run:` = shell). `${{ secrets.NAME }}`; **artifacts** pass files between jobs.
26. **DevSecOps:** **SAST** = source code · **SCA** = dependencies/CVEs · **DAST** = the running app · **secret scanning** = leaked keys · **image scanning** = Trivy. A **gate** = a non-zero exit code fails the job.
27. **Observability:** metrics / logs / traces. **Prometheus pulls (scrapes)**; Grafana draws dashboards; Alertmanager alerts. **GitOps:** Git = the source of truth; **Argo CD pulls and syncs** (self-heal reverts drift).
28. **Terraform:** `init` (providers + lock file) → `plan` (`+` create, `~` update, `-` destroy, `-/+` replace) → `apply` → `destroy`. Declarative + idempotent.
29. **State** maps code ↔ real resources; it holds **secrets**; never in Git; use a **remote backend with locking**. Variables: default < `TF_VAR_` < `terraform.tfvars` < `-var`.
30. **Cloud:** IaaS (VMs) / PaaS (you bring code) / SaaS (finished software). Region ⊃ AZs (use 2+ AZs for high availability). **Public subnet = route `0.0.0.0/0 → internet gateway`.** **Security group** = stateful, per instance, allow only; **NACL** = stateless, per subnet, allow + deny.

**Exam habits:** underline **NOT / most / first / twice**; commit to one answer; write "differences" as *"X does…, whereas Y…"*; in fix-the-code questions, list **every** bug, then write the **whole** fixed file. Good luck. You've covered the entire syllabus.
