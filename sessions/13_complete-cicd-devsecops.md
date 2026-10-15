# Session 13 · Complete CI/CD & DevSecOps (Oct 17, Session 1)

**Time:** ~110 min · **Concepts:** 78 → 81 · **Answers:** `MY_ANSWERS.md` or say "DevSecOps questions" · **Answer key:** `13_complete-cicd-devsecops_ANSWERS.md`

**Before you start:** (**CI/CD** = Continuous Integration / Continuous Delivery or Deployment; **PR** = pull request.)
```bash
colima status                 # Docker must be running
minikube status               # Kubernetes must be running (minikube start if it isn't)
docker images hello-app       # your Flask image from Session 03 (missing? rebuild it below)
cd ~/devops-practice/hello-app && docker build -t hello-app:1.0 .   # only if hello-app:1.0 is missing (files: Session 03, Concept 38)
```
> ⚠️ **Never remove the `minikube` or `localstack` containers.**
> Text after `#` in commands is a note for you: don't type it (your zsh would pass it on as arguments), or run `setopt interactivecomments` first.

## 🔁 Warm-up
**W1.** A workflow has two jobs, `build` and `test`, and no `needs:`. `build` runs `docker build -t app:ci .` and `test` runs `docker run app:ci pytest`. The `test` job fails with `Unable to find image 'app:ci' locally`, followed by a pull error. What's the real cause?
- A) Each job gets its own fresh runner machine
- B) `test` lacks `permissions: packages: read`
- C) `docker run` needs `--pull never` in CI jobs
- D) They ran in parallel; adding `needs:` fixes it

**W2.** A step has `env:` with `IMAGE: hello`, and runs `run: echo '$IMAGE:latest'`. What's printed?
- A) `hello:latest`, as env vars always expand
- B) `$IMAGE:latest`, as quotes stop the `$`
- C) `:latest`, as IMAGE isn't set in the shell
- D) An error, as `:` can't follow a variable

**W3.** A pre-commit hook contains:
```bash
check() { grep -q "password" config.txt; }
check
echo $?
```
`config.txt` exists and does **not** contain the word "password". What's printed?
- A) `0`, because the function itself ran fine
- B) `2`, because grep reports a missing match
- C) `1`, because grep found no matching line
- D) Nothing, because `-q` silences `echo` too

## Concept 78 · Building the image in the pipeline, and pushing it to a registry
**Words you'll learn:** pipeline stage, container registry (recap), GHCR, Docker Hub, commit SHA, short SHA, semantic versioning (semver), `latest` (recap), traceability, immutable, digest, `GITHUB_TOKEN`, `permissions:`, `packages: write`, access token, `docker/login-action`, Buildx, `docker/build-push-action`, image naming rules.

**Problem:** after the tests pass, Arjun builds `shop:latest` **on his laptop** and pushes it. When production breaks, nobody knows which commit is inside, whether his laptop had uncommitted changes, or what ran before (`latest` was overwritten).
- So let the **pipeline** build the image on a clean runner, from the exact tested commit, tagged with that commit's ID, and push it to a **registry** that Kubernetes pulls from.
- A **pipeline stage** = one named phase (test, build, scan, push, deploy); a failed stage stops the rest.
- **Analogy: batch numbers in a medicine factory.** A faulty batch is traced and recalled by its number; a commit-ID tag is the batch number stamped on your image.
```
git push ─► fresh runner: checkout ─► tests (fail = STOP) ─► docker build ─► tag :3f9c2a1 ─► login + push
        ─► registry (GHCR / Docker Hub) ─► Kubernetes PULLS it (a wrong name/tag = ImagePullBackOff)
```
- **Commit SHA:** a commit's 40-character ID, made by **SHA-1** (**Secure Hash Algorithm 1**); the first 7 characters are the **short SHA** (`3f9c2a1`, as in `git log --oneline`). In Actions, `${{ github.sha }}` is the full SHA.
- **Semantic versioning (semver)** = **MAJOR.MINOR.PATCH** (`2.4.1`): MAJOR for breaking changes, MINOR for features, PATCH for bug fixes; usually built from a Git tag like `v2.4.1`.

| Tag style | Example | Good for | Weakness |
|---|---|---|---|
| **Commit SHA** | `hello-app:3f9c2a1` | **Traceability**: one unique tag per commit | not human-friendly |
| **Semver** | `hello-app:2.4.1` | releases people talk about | must be bumped; can be reused by mistake |
| **`latest`** | `hello-app:latest` | quick local tests | **moves** with every push |

- **Traceability** = following the trail back: running Pod → `kubectl get deploy hello -o jsonpath='{..image}'` → `…:3f9c2a1` → `git show 3f9c2a1`.
- **Immutable** = never changes once created (a SHA tag behaves so; `latest` is **mutable**); the **digest** (`@sha256:…`) pins the exact bytes. Push two tags (`:sha` + `:latest`), and **deploy by SHA or semver, never by `latest`**.
- ⚠️ **Trap:** a Deployment says `image: hello-app:latest`; CI pushes a new `latest` and runs `kubectl apply`. The YAML didn't change → `deployment.apps/hello unchanged`, **no rollout**, old Pods keep running. A new SHA tag changes the Pod template, which triggers a rolling update.
- **Registries:** **GHCR** (**GitHub Container Registry**, `ghcr.io/OWNER/IMAGE:TAG`, used by the course) and **Docker Hub** (`docker.io/USER/IMAGE:TAG`, the default; anonymous pulls can hit `429 Too Many Requests`).
  - Also cloud ones (Amazon **ECR** = Elastic Container Registry, Google Artifact Registry, Azure **ACR** = Azure Container Registry), and self-hosted **Harbor**.
- **Logging in:** for GHCR, use the automatic **`GITHUB_TOKEN`** (a temporary password made per run); the job needs **`permissions: packages: write`** (images are "packages"), since the token is often **read-only by default**.
  - For Docker Hub, use an **access token** (a revocable, limited password-like key) saved as a secret such as `DOCKERHUB_TOKEN`.
- **Actions:** `docker/login-action@v4` runs `docker login`; `docker/setup-buildx-action` sets up **Buildx** (Docker's newer builder, with caching and multi-CPU builds); `docker/build-push-action@v7` builds, applies every tag in `tags:`, and pushes when `push: true`.
  - Pushed images appear under your profile → **Packages**, **private** at first (make it public, or Kubernetes needs an `imagePullSecret`).
- **Naming rules:** the **repository** part (before `:`) must be **lowercase** (`a–z 0–9 . _ - /`); the **tag** may mix case (letters, digits, `_ . -`, up to 128 characters, not starting with `.` or `-`).
  - Error: `invalid reference format: repository name (VanshSrivastava/hello-app) must be lowercase`.
  - ⚠️ `${{ github.repository }}` keeps GitHub's capitals (`Vansh-S/HelloApp`): lowercase it in bash with **`${GITHUB_REPOSITORY,,}`**.

### 🛠 Do (~10 min): tag an image the way CI does
```bash
cd ~/devops-practice/hello-app
git init -b main && git add . && git commit -m "hello-app: ready for CI"   # "Reinitialized…" / "nothing to commit" is fine
SHA=$(git rev-parse --short HEAD)
echo "$SHA"                                       # short SHA of HEAD, e.g. 3f9c2a1 (yours differs)
docker build -t hello-app:"$SHA" .                # tag = your SHA ("…" lets $SHA expand)
docker images hello-app                           # new tag; often the SAME image ID as before: one image, two labels
docker tag hello-app:"$SHA" Hello-App:"$SHA"      # error: repository name (library/Hello-App) must be lowercase
docker tag hello-app:"$SHA" hello-app:Feature_1   # works: uppercase is fine in the TAG
docker rmi hello-app:Feature_1                    # removes only that extra label (CI would push ghcr.io/<you-in-lowercase>/hello-app:<sha>)
```

### ✅ Check 78
**78.1** Production runs `ghcr.io/vansh/hello-app:latest`, and a bug appears. Why would a tag like `hello-app:3f9c2a1` (a commit SHA) have helped?
- A) It makes the image smaller and faster to pull
- B) It makes Kubernetes always pull a fresh image
- C) It maps the running image to one exact commit
- D) It signs the image, so nobody can tamper with it

**78.2** A workflow's push step fails with `denied: permission_denied: write_package` when pushing to `ghcr.io`. The login step succeeded, the job has no `permissions:` block, and the repository's default token is read-only. What's the fix?
- A) Add `permissions: packages: write` to the job
- B) Replace GITHUB_TOKEN with your GitHub password
- C) Add `docker/setup-buildx-action` before the push
- D) Tag the image `latest` instead of the git SHA

**78.3** (one line) `docker build -t ghcr.io/Vansh-S/hello-app:1.0 .` fails immediately. Why, and what's the fix?

## Concept 79 · Deploying to Kubernetes from the pipeline
**Words you'll learn:** deploy stage, API server (recap), kubeconfig, cluster credentials, base64 (recap), `kubectl set image`, `helm upgrade --install` (recap), `--wait`, `kubectl rollout status`, `--timeout`, `kubectl rollout undo` (recap), `if: failure()`, environment, required reviewers, Continuous Delivery, Continuous Deployment, push-based deployment, pull-based deployment, GitOps (preview), self-hosted runner, kind.

**Problem:** someone still types `kubectl set image …` by hand; at midnight Arjun deploys the wrong tag to the wrong namespace, and nobody checks whether the rollout worked. The last mile must be automated **and verified**.
- Every `kubectl` command is an HTTPS request to the **API server** (**API** = Application Programming Interface). So the runner needs **tools** (`kubectl`/`helm`, preinstalled on `ubuntu-latest`), **credentials** (a kubeconfig stored as a secret), and **a network path** to the API server.
- A **kubeconfig** (`~/.kube/config`) lists **clusters** (API server address + the **CA**, Certificate Authority, certificate to trust it), **users** (**cluster credentials**: a client certificate or token), and **contexts** (cluster + user + namespace).
  - **Analogy:** a hotel key card that encodes the building, who you are, and your floor; whoever holds it gets in, so guard it.
- Teams **base64-encode** it into one line (encoding, not encryption), save it as a secret like `KUBECONFIG_B64`, and a step runs `echo "${{ secrets.KUBECONFIG_B64 }}" | base64 -d > ~/.kube/config`.
- **Three ways to update:**
  - **apply manifests** (declarative: `sed` the SHA into `k8s/deployment.yaml`, then `kubectl apply -f k8s/`);
  - **set the image** (imperative, quick: `kubectl set image deployment/hello hello=ghcr.io/vansh/hello-app:${{ github.sha }}`);
  - **Helm** (`helm upgrade --install hello ./chart --set image.tag=${{ github.sha }} --wait --timeout 3m`).
- In `hello=IMAGE`, the left side is the **container's** name in the Pod template, not the Deployment's. `--install` creates the release if missing; `--wait` fails if Pods aren't ready within `--timeout` (Helm v4's `--rollback-on-failure`, Helm 3's `--atomic`, also rolls back).
- ⚠️ `kubectl set image` / `apply` only **submit** the change and return success at once, even if every new Pod will crash: a green ✅ over a broken deploy.
  - **`kubectl rollout status deployment/hello --timeout=120s`** waits: `successfully rolled out` = exit 0; `error: timed out waiting for the condition` (or `exceeded its progress deadline`) = exit 1, so the step goes red.
  - A later step with **`if: failure()`** can run `kubectl rollout undo deployment/hello`.
- A deploy job with **`environment: production`** gets that environment's own secrets and can need **required reviewers**: it pauses ("Waiting for review") until someone clicks **Approve and deploy**. That's **Continuous Delivery** (a human approves production), whereas **Continuous Deployment** releases with no human step.
- Everything above is **push-based**: the CI runner applies changes, so CI holds the cluster credentials and must reach **in** to the API server, and manual `kubectl` edits (drift) go unnoticed.
- **Pull-based** (**GitOps**, full table in Session 14): an agent **inside** the cluster (e.g. Argo CD) pulls from Git, needs only **outbound** access to Git, keeps credentials in the cluster, and detects drift; CI only updates Git.
- **Why GitHub's runners can't reach your minikube:** your kubeconfig says `server: https://127.0.0.1:32771`, and on the runner `127.0.0.1` is **the runner itself** (`connection refused`). The real address (`192.168.49.2`) is a **private IP** inside Colima behind your router, and the certificate files are on your Mac's disk.
  - Options:
    - a **self-hosted runner** on your Mac (connects **out** to GitHub; ⚠️ private repos only),
    - a cloud cluster (**EKS** = Elastic Kubernetes Service, **GKE** = Google Kubernetes Engine, **AKS** = Azure Kubernetes Service; costs money),
    - a throwaway **kind** (**K**ubernetes **in** **D**ocker) cluster on the runner (a test only), or a GitOps agent (Session 14).
  - In class: **Actions builds + scans + pushes to GHCR; deploying to local Kubernetes is practised manually.**

### 🛠 Do (~10 min): rehearse a pipeline's deploy commands on minikube
```bash
kubectl config view --minify | grep server                          # server: https://127.0.0.1:<port> (loopback)
kubectl create namespace s13 && kubectl create deployment web --image=nginx:1.27 --replicas=2 -n s13   # 2 Pods; container named nginx
kubectl rollout status deployment/web -n s13                        # "successfully rolled out"
kubectl set image deployment/web nginx=nginx:1.28 -n s13            # container nginx → image nginx:1.28
kubectl rollout status deployment/web -n s13 --timeout=60s; echo "exit code: $?"   # exit code: 0 → a pipeline continues
kubectl set image deployment/web nginx=nginx:9.99 -n s13            # a bad release: this tag doesn't exist
kubectl rollout status deployment/web -n s13 --timeout=30s; echo "exit code: $?"   # timed out… exit code: 1 → step red
kubectl get pods -n s13                                             # 2 old Pods Running + 1 ImagePullBackOff: the old version kept serving
kubectl rollout undo deployment/web -n s13 && kubectl rollout status deployment/web -n s13   # back to nginx:1.28: rolled out
kubectl delete namespace s13                                        # clean up
```

### ✅ Check 79
**79.1** A deploy job runs `kubectl set image deployment/hello hello=ghcr.io/vansh/hello-app:${{ github.sha }}`. The job is green ✅, but the new Pods are stuck in `ImagePullBackOff`. What should the job have included?
- A) A `kubectl rollout status --timeout` step
- B) A `kubectl apply -f` step before set image
- C) The `environment: production` approval key
- D) A `helm repo update` step before the deploy

**79.2** Your GitHub-hosted workflow uses your minikube kubeconfig, stored as a secret. `kubectl get pods` fails with `dial tcp 127.0.0.1:32771: connect: connection refused`. Why?
- A) The kubeconfig secret was base64-encoded twice
- B) On the runner, 127.0.0.1 is the runner itself
- C) minikube refuses connections without a Service
- D) kubectl needs `--insecure` to reach port 32771

**79.3** Which describes **Continuous Delivery** (as opposed to Continuous Deployment)?
- A) Every passing change goes live with no human step
- B) Code is merged and tested often but never packaged
- C) The cluster pulls each change from Git by itself
- D) Each change is ready; a human approves production

## Concept 80 · DevSecOps and shift-left: SAST, SCA, DAST, IAST, secret and IaC scanning
**Words you'll learn:** vulnerability, exploit, DevSecOps, shift-left, CVE, NVD, CVSS, SAST, SQL injection, SCA, direct dependency, transitive dependency, license compliance, DAST, XSS, OWASP, ZAP, IAST, instrumentation, secret scanning, push protection, IaC scanning, misconfiguration.

**Problem:** a bank's security team audits the app two weeks before launch and finds 60 problems in code written months ago. And in 2021 **Log4Shell** (**CVE-2021-44228**, scored 10.0) hit the Log4j library inside millions of Java apps; companies spent weeks just finding which apps contained it.
- Lessons: find problems **early**, and know **what your software is made of**.
- **DevSecOps** = **Dev**elopment + **Sec**urity + **Op**erations: security is everyone's job, with automated checks **in every pipeline stage**. **Shift-left** = move checks earlier (left on the timeline), where fixes are cheap.
  - **Analogy:** spell-checking while you type instead of after printing 1,000 brochures; and, like airport security, several layers each catch different things.
```
PLAN ─► CODE ─────────► BUILD ───────────► TEST ──────► RELEASE ─────► DEPLOY ─► OPERATE
        pre-commit       SAST, SCA, secret   DAST on      security       signed     rescans,
        secret scan      + image + IaC scan  staging      gate           images     monitoring
◄──────────── shift-left: move checks this way, where fixes are cheap ────────────
```
- A **vulnerability** = a weakness an attacker could use; an **exploit** = the actual attack. **CVE** (**Common Vulnerabilities and Exposures**) gives each publicly known vulnerability an ID, `CVE-YEAR-NUMBER` (its passport number).
  - The **NVD** (**National Vulnerability Database**, US government) adds details and scores.
  - **CVSS** (**Common Vulnerability Scoring System**) scores 0.0–10.0: **LOW** 0.1–3.9 · **MEDIUM** 4.0–6.9 · **HIGH** 7.0–8.9 · **CRITICAL** 9.0–10.0. Gates use these words ("block HIGH and CRITICAL").
- **SAST** (**Static Application Security Testing**; static = **not running**) reads **your source code** for dangerous patterns, reporting the **file and line**.
  - E.g. `cursor.execute("SELECT … WHERE name = '" + name + "'")`: typing `' OR '1'='1` returns all users. That's **SQL injection** (user input changes a database command); fix with a parameterised query `cursor.execute("… WHERE name = %s", (name,))`.
  - Tools: **CodeQL** (GitHub's; its job needs `security-events: write`), **SonarQube** (with a pass/fail "quality gate"), **Semgrep**, **Bandit** (Python). Weakness: **false positives**; blind to runtime settings.
- **SCA** (**Software Composition Analysis**) checks **third-party libraries**: **direct dependencies** (you list them: `flask`) and **transitive dependencies** (dependencies of dependencies: Flask pulls in `werkzeug`, `jinja2`, `click`…).
  - It checks them against CVE databases (GitHub Advisory Database, **OSV** = Open Source Vulnerabilities), often plus **license compliance** (is each library's legal licence allowed?).
  - Tools: **Dependabot**, **Snyk**, **OWASP Dependency-Check**, **`pip-audit`**, **`npm audit`**, `trivy fs .`.
  - Course flow: **find → identify package + version → check the fixed version → update → run tests → run SCA again.**
- **DAST** (**Dynamic Application Security Testing**; dynamic = **running**) attacks the **running** app over HTTP from outside (**black-box**), usually on staging: injection strings, **XSS** (**Cross-Site Scripting**: getting a site to send an attacker's script to other users), missing security headers like `Content-Security-Policy`.
  - **OWASP** (**Open Worldwide Application Security Project**, a non-profit) makes **ZAP** (**Zed Attack Proxy**), e.g. `docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t https://staging.example.com`; also Burp Suite. It can't point to a line of code.
- **IAST** (**Interactive** AST, awareness): an agent **inside** the running app (**instrumentation** = adding watching code) sees the request **and** the code that handled it.
- **Secret scanning** finds passwords, API keys and tokens in files and Git history: **gitleaks**, **TruffleHog**, detect-secrets (match patterns like `ghp_…`, `AKIA…`); GitHub's **push protection** **blocks the `git push`**.
  - Run it as a **pre-commit hook** and in CI (hooks aren't shared on clone).
  - If a secret leaks: **revoke/rotate first** → remove it from code/history → check for misuse → store the new one safely (env var + Actions secret). Deleting it in a new commit is **not** enough.
- **IaC** (**Infrastructure as Code**) **scanning** checks Terraform/Kubernetes files for **misconfiguration** (an unsafe setting: SSH open to `0.0.0.0/0`, a public bucket, a root/privileged container): **Checkov**, **tfsec** (now `trivy config .`), **KICS**.

| | **SAST** | **SCA** | **DAST** | **Secret scanning** | **Image scanning** (81) |
|---|---|---|---|---|---|
| Scans | your source code | your dependencies | the running app (HTTP) | files + Git history | the built image |
| App running? | ❌ | ❌ | ✅ **Yes** | ❌ | ❌ |
| When | code / PR | code / PR / build | after deploying to staging | pre-commit + CI | after `docker build` |
| Finds | bad code patterns | known CVEs in libraries | real runtime weaknesses | leaked credentials | CVEs in OS + library packages |
| Examples | CodeQL, SonarQube, Semgrep, Bandit | Dependabot, Snyk, pip-audit | OWASP ZAP, Burp Suite | gitleaks, TruffleHog | Trivy, Grype, Docker Scout |

> ⚠️ **Traps:** static = not running (SAST); dynamic = running (DAST). SCA ≠ SAST (libraries vs your code): a clean SAST says nothing about a vulnerable Flask. Each scan is one layer.

### 🛠 Do (~10 min): run an SCA scan and a secret scan, both in containers
```bash
cd ~/devops-practice/hello-app && cat requirements.txt   # flask==3.0.3 (from Session 03)
docker run --rm -v "$PWD":/src -w /src python:3.12-slim sh -c "pip install -q pip-audit && pip-audit -r requirements.txt"; echo "exit code: $?"
```
`--rm` = throwaway container; `-v "$PWD":/src` bind-mounts this folder; `-w /src` starts there.
- Expect (after a pip-as-root warning) `Found 2 known vulnerabilities in 1 package`, `flask 3.0.3   PYSEC-2026-2151 3.1.3` and **`exit code: 1`**: a CI step would fail here (an SCA gate). Trivy calls the same issue `CVE-2026-27205`. Now fix it the course's way:
```bash
sed -i '' 's/flask==3.0.3/flask==3.1.3/' requirements.txt   # edit in place (-i '' is the macOS form)
docker run --rm -v "$PWD":/src -w /src python:3.12-slim sh -c "pip install -q pip-audit && pip-audit -r requirements.txt"; echo "exit code: $?"   # No known vulnerabilities found, exit code: 0
docker build -t hello-app:2.0 .                                  # rebuild with the fixed Flask
docker run -d --rm -p 8090:5000 --name hello2 hello-app:2.0      # smoke test: start it…
sleep 2; curl http://localhost:8090                              # …expect "Hello from the … container!"
docker rm -f hello2                                              # stop it
mkdir -p ~/devops-practice/s13-secrets && cd ~/devops-practice/s13-secrets
printf 'GITHUB_TOKEN = "ghp_4fT9xQ2mZ7LbV1nK8sR3wY6pD0cH5jE2aU9i"\n' > settings.py      # a FAKE token
docker run --rm -v "$PWD":/scan ghcr.io/gitleaks/gitleaks:v8.30.1 dir -v /scan; echo "exit code: $?"   # scan the folder (image already on your Mac)
cd ~ && rm -rf ~/devops-practice/s13-secrets                     # clean up
```
gitleaks shows `RuleID: github-pat`, `File: /scan/settings.py`, `leaks found: 1` and **`exit code: 1`**: a pre-commit hook would cancel the commit; CI would fail the step.

### ✅ Check 80
**80.1** Which of these needs the application to be **running** in order to find problems?
- A) A SAST tool such as Semgrep
- B) An SCA tool such as pip-audit
- C) A DAST tool such as OWASP ZAP
- D) A secret scanner such as gitleaks

**80.2** pip-audit reports `flask 3.0.3  PYSEC-2026-2151  Fix Versions 3.1.3`. Your own code has no bugs. What's the right fix?
- A) Add the ID to `.trivyignore` so the scan passes
- B) Run SAST on app.py, since the bug is in your code
- C) Rebuild on `alpine` so the CVE count drops
- D) Bump flask to 3.1.3, then rerun tests and SCA

**80.3** (one line) A teammate pushed a real AWS key to a public GitHub repo, then deleted it in the next commit. Is that enough? What must happen first?

## Concept 81 · Container image scanning with Trivy, and security gates
**Words you'll learn:** image scanning, OS package, language package, Trivy, vulnerability database, severity, Status (`fixed` / `affected`), Fixed Version, `--severity`, `--exit-code`, `--ignore-unfixed`, `.trivyignore`, security gate, policy, required status check, minimal base image, SBOM, SPDX, CycloneDX, image signing, cosign, least privilege.

**Problem:** Riya's code passes SAST and SCA, but `python:3.12-slim` also holds a small Debian Linux (~87 **OS packages** like `openssl`, plus Python and pip), and new CVEs appear weekly. **Clean source code does not mean a clean image**: scan the thing you ship.
- **Trivy** (Aqua Security, free) scans images (`trivy image`), folders (`trivy fs`), IaC (`trivy config`). It unpacks the image's **layers**, lists **OS packages** (`dpkg`/`apk` records) and **language packages** (Python, Node, Java), and compares each version with its **vulnerability database** (~130 MB, downloaded on the first run).
- A report (real `nginx:1.27`): `Total: 707 (UNKNOWN: 37, LOW: 210, MEDIUM: 286, HIGH: 163, CRITICAL: 11)`, then rows of **Library** · **Vulnerability** (CVE ID) · **Severity** · **Status** (**`fixed`** = a newer package fixes it; **`affected`** = no fix yet; also `will_not_fix`, `fix_deferred`) · **Installed Version** · **Fixed Version**.
- **Flags:** `--severity HIGH,CRITICAL` (show and count only these) · **`--exit-code 1`** (exit 1 if anything is found after filters) · `--ignore-unfixed` (skip findings with no fix yet)
  - `--scanners vuln` (vulnerabilities only, faster) · `-q` (quiet) · `--format json|sarif|cyclonedx` + `--output file` (machine-readable / GitHub Security tab / SBOM)
  - **`.trivyignore`** (accepted-risk IDs, one per line, read from the current folder).
- ⚠️ **Number-one gate mistake:** `--severity HIGH,CRITICAL` **only filters the report**; without `--exit-code 1` Trivy exits **0** even with CRITICALs (the course's own demo scan never blocks). **The gate is `--exit-code 1`.**

**Analogy:** a scan is a smoke detector, a gate is the fire door: the detector finds smoke, the door decides whether anyone gets through. A **security gate** = a step that decides whether the pipeline may continue: a **check** (scan) + a **policy** (the rule, e.g. "no fixable HIGH/CRITICAL") + **enforcement** (a **non-zero exit code**).
```
docker build ─► trivy image --severity HIGH,CRITICAL --exit-code 1 app:sha
                   exit 0 ─► push to GHCR ─► deploy
                   exit 1 ─► step ✗ → job ✗ → jobs that need it SKIPPED → red PR check → merge blocked
```
- Blocking uses what you know: a non-zero exit fails the step; **`needs:`** skips dependants; **branch protection** "Require status checks to pass before merging" makes chosen jobs **required status checks**, and a red one disables the merge button.
  - In a PR: SCA ✗ → `image` skipped → merge blocked; fix it **on the same branch** and push, and the PR re-runs (no new PR needed).
  - Course mindset: **Detect → Evaluate → Enforce → Remediate → Deploy**. Every stage can gate (tests, SAST, SCA, secrets, build, image scan, push, rollout); HIGH/CRITICAL is a **classroom threshold**, each organisation sets its own policy.
- ⚠️ **Four ways to switch a gate off:** (1) no `--exit-code 1`; (2) a pipe, `trivy … | tee report.txt` (the pipe's exit code is `tee`'s; only `shell: bash` adds `-o pipefail`); (3) `continue-on-error: true`; (4) scanning one image and **pushing a rebuilt** one: build once, scan it, push that same image.
- **Fewer findings:** a **minimal base image** (`alpine`, `-slim`, distroless) = fewer packages; **multi-stage builds** keep build tools out; **rebuild regularly** (images never patch themselves); **pin versions**; **scheduled rescans** (`on: schedule`).
  - ⚠️ `USER appuser` (non-root) is great practice but **doesn't lower the CVE count**: it lowers the **damage**.
- **SBOM** (**Software Bill of Materials**) = an ingredients list of every package in an image, in **SPDX** (Software Package Data Exchange) or **CycloneDX** (OWASP) format: `trivy image --format cyclonedx --output sbom.json hello-app:2.0`.
- **Image signing** with **cosign** (Sigstore) proves who built an image and that it's unchanged (`cosign sign` / `cosign verify`).
- **Least privilege:** `permissions: contents: read` at the top and more only where needed; short-lived `GITHUB_TOKEN`; prod secrets behind an approval environment; **OIDC** (**OpenID Connect**: temporary cloud credentials) instead of stored keys.

### 🛠 Do (~15 min): scan images with Trivy on your Mac
```bash
brew install trivy && trivy --version                          # Version: 0.75.0 or newer
export DOCKER_HOST="unix://$HOME/.colima/default/docker.sock"  # ⚠️ Trivy needs this to find Colima's Docker (this Terminal only; rerun if an image "can't be found")
trivy image nginx:1.27                                         # first run downloads the DB (~130 MB); Total: ~707 (numbers drift daily)
trivy image -q --severity HIGH,CRITICAL nginx:1.27    | grep Total   # Total: 174 (HIGH: 163, CRITICAL: 11)
trivy image -q --severity HIGH,CRITICAL nginx:alpine  | grep Total   # Total: 3 (HIGH: 3, CRITICAL: 0): fewer packages (71 vs 144), far fewer CVEs
trivy image -q --severity HIGH,CRITICAL hello-app:1.0 | grep Total   # Total: 44 (HIGH: 44, CRITICAL: 0)
trivy image -q --scanners vuln --pkg-types library hello-app:1.0 | grep -i flask   # Flask … CVE-2026-27205 │ LOW │ fixed │ 3.0.3 │ 3.1.3
trivy image -q --scanners vuln --pkg-types library hello-app:2.0 | grep -i flask   # flask-3.1.3 with count 0: the SCA fix shows here too
trivy image -q --severity HIGH,CRITICAL --exit-code 1 nginx:alpine; echo "exit code: $?"          # 1: the gate blocks
trivy image -q --severity CRITICAL --exit-code 1 nginx:alpine; echo "exit code: $?"               # 0: a looser policy passes
trivy image -q --severity HIGH,CRITICAL --exit-code 1 nginx:alpine | tail -3; echo "exit code: $?"   # 0! tail's code (gate mistake 2)
trivy image -q --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 hello-app:2.0; echo "exit code: $?"   # 0: its HIGHs are all "affected"
```
**The complete secure pipeline** (test → SCA → build → Trivy gate → push to GHCR). First create `~/devops-practice/hello-app/test_app.py`:
```python
from app import app

def test_home_page():
    response = app.test_client().get("/")   # Flask's pretend browser
    assert response.status_code == 200
    assert b"Hello" in response.data
```
Then create `.github/workflows/secure-pipeline.yml` in the same folder (`mkdir -p .github/workflows` first):
```yaml
name: secure-pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]                   # every PR into main is gated
permissions:
  contents: read                       # least privilege: every job starts read-only
jobs:
  test:                                # stages 1-2: tests + SCA
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
      - name: Install dependencies
        run: pip install -r requirements.txt pytest pip-audit
      - name: Unit tests
        run: pytest -q                       # a failing test exits non-zero → stop
      - name: "SCA: scan dependencies for known CVEs"
        run: pip-audit -r requirements.txt   # a known vulnerability → exit 1 → stop
  image:                               # stages 3-5: build, scan gate, push
    needs: test                        # only if tests AND SCA passed
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write                  # only this job may push to GHCR
    steps:
      - uses: actions/checkout@v7
      - name: Work out the image name    # ,, = lowercase; >> $GITHUB_ENV keeps $IMAGE for later steps
        run: echo "IMAGE=ghcr.io/${GITHUB_REPOSITORY_OWNER,,}/hello-app:${GITHUB_SHA}" >> "$GITHUB_ENV"
      - name: Build the image
        run: docker build -t "$IMAGE" .
      - name: "Security gate: Trivy image scan"   # the socket mount lets Trivy see the image just built
        run: |
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            ghcr.io/aquasecurity/trivy:0.75.0 image \
            --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 \
            "$IMAGE"
      - name: Log in to GHCR
        if: github.event_name == 'push'      # PRs are built and scanned, never pushed
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Push the image
        if: github.event_name == 'push'
        run: docker push "$IMAGE"            # the SAME image that passed the scan
```
**Optional, on your GitHub** (creates a public repo):
- Keep `flask==3.1.3` (or the SCA step fails: the gate working); test locally with `docker run --rm -v "$PWD":/src -w /src python:3.12-slim sh -c "pip install -q -r requirements.txt pytest && pytest -q"` (`1 passed`).
- Then `git add . && git commit -m "Add secure pipeline" && gh repo create hello-app-ci --public --source=. --push` and `gh run watch`; the image appears under **Packages**.
- To see a PR blocked: protect `main` (Settings → Branches, "Require status checks to pass", pick `test` and `image`), then `git switch -c old-flask`, set `flask==3.0.3`, commit, `git push -u origin old-flask`, `gh pr create --fill`: `test` goes red at SCA, `image` is skipped, merge is blocked.

### ✅ Check 81
**81.1** A gate step runs `trivy image --severity HIGH,CRITICAL --exit-code 1 app:ci | tee trivy.txt`. Trivy finds 2 CRITICAL CVEs, yet the step is green. The step has no `shell:` key. Why?
- A) `--severity` must be lowercase: `high,critical`
- B) `tee` exited 0, and the pipe hid Trivy's exit code
- C) `--exit-code 1` only applies when output is JSON
- D) GitHub treats exit code 1 as a warning, not a failure

**81.2** Which change **reduces the number of CVEs** Trivy reports for an image?
- A) Adding `USER appuser` so the app runs as non-root
- B) Adding `--exit-code 1` to the Trivy scan command
- C) Basing it on `nginx:alpine`, not `nginx:1.27`
- D) Pushing the image to GHCR instead of Docker Hub

## ➕ Small but important (CI/CD + DevSecOps): read before the quiz
| Item | What it is |
|---|---|
| **Dependabot** | GitHub's dependency bot: **alerts**, **security updates** (PRs bumping to the fixed version), **version updates** (set in `.github/dependabot.yml`: `version: 2`, `updates:` with `package-ecosystem: "pip"` or `"github-actions"`, `directory: "/"`, `schedule: interval: "weekly"`) |
| **CodeQL** / **SARIF** | Settings → Code security → Code scanning → **Default setup** (no YAML), results in the **Security** tab / **SARIF** (Static Analysis Results Interchange Format) = standard JSON for scan results (`trivy image --format sarif`) |
| **OWASP Top 10** | the ten most critical web-app risks (2021, 2025); **Broken Access Control** is #1; also **Injection**, **Security Misconfiguration**, vulnerable/outdated components |
| **CVE vs CWE** | a **CVE** is one specific vulnerability (CVE-2021-44228); a **CWE** (**Common Weakness Enumeration**) is a **type** of weakness (CWE-89 = SQL injection) |
| **False positive / negative** / **`.trivyignore`** | reports a problem that isn't real (common in SAST) / misses a real one; `.trivyignore`: one ID per line with a reason, `CVE-2026-27205 exp:2026-12-31` expires |
| **Pinning actions** | a tag (`@v7`, `@v7.0.1`) can be moved; `@<full commit SHA>  # v7.0.1` can't (in March 2025 `tj-actions/changed-files` tags were moved to code that leaked secrets; SHA pins were safe) |
| **`permissions:` rule** | listing **any** permission sets **every unlisted one to `none`**; keys: `contents`, `packages`, `security-events`, `pull-requests`, `id-token` (OIDC) |
| **Fork PRs** / **masking** | PRs from a fork get **no secrets** and a read-only token / secret values show as `***` in logs (still never echo them) |
| **`continue-on-error: true`** / **`if: failure()`, `if: always()`** | the step may fail without failing the job (**disables** a gate) / run only after a failure (rollback, alert), or no matter what (upload the report) |
| **Scheduled rescans** | `on: schedule: - cron: "0 2 * * *"` = daily at 02:00 **UTC** (Coordinated Universal Time); catches new CVEs in unchanged images |
| **Signed commits** / **branch protection** | `git commit -S` with a **GPG** (GNU Privacy Guard) or SSH key shows **Verified** / rules on `main`: require PRs, approvals, **status checks**, signed commits; block force-pushes |
| **Other scanners** | image: **Grype**, **Docker Scout** (`docker scout cves IMAGE`), Snyk Container, Clair; SCA: `npm audit`, `pip-audit` (both exit non-zero on findings) |

**Quick checks:** (1) A job needs to push to GHCR and you write `permissions: packages: write` only. `actions/checkout` now fails on a private repo. Why? (2) Why pin `uses:` to a full commit SHA instead of `@v7`?
*Answers:* (1) Listing any permission sets all unlisted ones to `none`, so `contents` became `none` and the code can't be read. Add `contents: read`. (2) A tag can be moved to different (possibly malicious) code; a full commit SHA always means the same code.

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory
**Q1.** A workflow has these jobs:
```yaml
  image-scan:
    needs: docker-build
    runs-on: ubuntu-latest
    steps:
      - run: trivy image --severity HIGH,CRITICAL myapp:${{ github.sha }}
  push:
    needs: image-scan
```
The scan log shows `Total: 4 (HIGH: 3, CRITICAL: 1)`, yet `push` runs and publishes the image. Which statement best explains this?
- A) `needs:` waits for a job to finish, not for it to succeed
- B) Trivy fails a build only when CRITICALs exceed HIGHs
- C) Trivy exits 0 by default; a gate needs `--exit-code 1`
- D) A `run:` step's exit code counts only with `shell: bash`

**Q2.** Your pipeline pushes `ghcr.io/vansh/hello-app:latest` on every merge, then runs `kubectl apply -f k8s/deployment.yaml`, which contains `image: ghcr.io/vansh/hello-app:latest`. The apply prints `deployment.apps/hello unchanged`, and the old code keeps running. What's the best explanation?
- A) The Pod template is unchanged, so no rollout runs
- B) GHCR caches the `latest` tag for an hour after a push
- C) `kubectl apply` only works after a `kubectl create`
- D) The Pods need `imagePullPolicy: Never` to update

**Q3.** Your pipeline runs CodeQL, pip-audit, gitleaks and Trivy, and all of them pass. A ZAP scan of the staging site then finds that the login page sends no `Content-Security-Policy` header and reflects user input back into the page. Why did only ZAP find these?
- A) CodeQL scans dependency files only, not app code
- B) pip-audit skips web frameworks like Flask by default
- C) Trivy reports headers only with `--scanners misconfig`
- D) Only DAST sees the live app's real HTTP responses

**Q4.** Which statement about push-based deployment (from GitHub Actions) and pull-based GitOps is **NOT** correct?
- A) Push-based CI must hold credentials for the cluster
- B) Push-based CI reaches a laptop cluster by default
- C) Pull-based agents run inside the target cluster
- D) Pull-based agents only need outbound access to Git

**L1.** Explain SAST vs SCA in contrast form, naming one tool for each.

**L2.** What is a security gate, and what exactly makes a pipeline step act as one?
