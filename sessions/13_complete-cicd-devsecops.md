# Session 13 · Complete CI/CD & DevSecOps (Oct 17, Session 1)

**Time:** ~110 min · **Concepts:** 78 → 81 · **Answers:** `MY_ANSWERS.md` or say "DevSecOps questions" · **Answer key:** `13_complete-cicd-devsecops_ANSWERS.md`

**Before you start:** today you join Docker + Kubernetes + GitHub Actions + security. (**CI/CD** = Continuous Integration / Continuous Delivery or Deployment; **PR** = pull request.)
```bash
colima status                 # Docker running?
minikube status               # Kubernetes running?
docker images hello-app       # your Flask image
cd ~/devops-practice/hello-app && docker build -t hello-app:1.0 .   # only if hello-app:1.0 is missing
```
**What each line does:**
- `colima status`: checks Colima, the small Linux VM that runs Docker on your Mac; it should say it's running (if not, `colima start`).
- `minikube status`: checks your local Kubernetes cluster; expect `host`, `kubelet` and `apiserver` all `Running`. If not, run `minikube start` and wait a minute or two.
- `docker images hello-app`: lists local images named `hello-app`, your Flask "Hello from the … container!" app. You want a row with TAG `1.0`.
- `cd … && docker build -t hello-app:1.0 .`: **only if that row is missing**. Rebuilds the image; `-t` = name:tag; `.` = use the current folder (it needs the `app.py`, `requirements.txt` and `Dockerfile` you wrote there).
⚠️ **Never remove the `minikube` or `localstack` containers.** Text after `#` in commands is a note for you: don't type it (your zsh would pass it on as arguments), or run `setopt interactivecomments` first.

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

**The problem.** Your GitHub Actions pipeline from last time (checkout, install, pytest) **tests** the code, but production runs an **image**. In Riya's team, Arjun builds `shop:latest` **on his laptop** after the tests pass, and pushes it. When production breaks, nobody knows **which commit** is inside, whether his laptop had **uncommitted changes**, or which image ran **before** (`latest` was overwritten, so the old one is gone).

**The idea:** let the **pipeline** build the image, on a **clean runner**, from the **exact commit that passed the tests**, labelled with **that commit's ID**. Then it pushes the image to a **registry**, where Kubernetes pulls it from (a wrong name or tag there = `ImagePullBackOff`). Each named phase of the pipeline (test, build, scan, push, deploy) is a **pipeline stage**; if one fails, the stages after it don't run.

**Analogy: batch numbers in a medicine factory.** If one batch is faulty, its number tells the factory exactly which production run made it, and only that batch is recalled. A commit-ID tag is the batch number on your image.
### Choosing the tag
- **Commit SHA:** every Git commit has a 40-character ID calculated from its contents (by **SHA-1**, Secure Hash Algorithm 1), so people say "the commit SHA". Its first 7 characters, like `3f9c2a1`, are the **short SHA** (what `git log --oneline` shows). In GitHub Actions, `${{ github.sha }}` is the full SHA of the commit that started the run.
- **Semantic versioning (semver)** is the format **MAJOR.MINOR.PATCH**, e.g. `2.4.1`: MAJOR goes up for a change that breaks compatibility, MINOR for a new feature, PATCH for a bug fix.

| Tag style | Example | Good for | Weakness |
|---|---|---|---|
| **Commit SHA** | `hello-app:3f9c2a1` | one unique tag per commit | not human-friendly |
| **Semver** | `hello-app:2.4.1` | releases people talk about | someone must bump it; can be reused by mistake |
| **`latest`** | `hello-app:latest` | quick local tests | **moves** with every push, so says nothing about what's inside |

- **Traceability** = following the trail from the running thing back to its source: running Pod → `kubectl get deploy hello -o jsonpath='{..image}'` → `…/hello-app:3f9c2a1` → `git show 3f9c2a1` (the exact code, author and date).
- **Immutable** = never changes once created. A SHA tag is used for one build only, so it behaves as immutable; `latest` is **mutable** (moved to a new image on every push). The **digest** (`@sha256:…`) goes further and pins the image's exact bytes.
- **The rule:** you may push two tags (`:3f9c2a1` and `:latest`), but **deploy by SHA or semver, never by `latest`**.

⚠️ **Trap: `latest` and Kubernetes.** A Deployment says `image: hello-app:latest`. CI pushes a new `latest` and runs `kubectl apply`. The YAML text **didn't change**, so kubectl prints `deployment.apps/hello unchanged`, **no rollout starts**, and the old Pods keep running. A new SHA tag changes the Pod template, and a template change is what triggers a rolling update.
### Registries, and how the pipeline logs in
A **container registry** is a server that stores images, which you `docker push` to and `docker pull` from. The course uses **GHCR** (**GitHub Container Registry**, `ghcr.io/OWNER/IMAGE:TAG`, next to your code); **Docker Hub** (`docker.io/USER/IMAGE:TAG`) is the default when a name has no registry in front. The pipeline must log in before it can push:
- **GHCR:** for every run, GitHub creates a temporary password called **`GITHUB_TOKEN`** (written `${{ secrets.GITHUB_TOKEN }}`), valid only while the job runs. What it may do is set by the workflow's **`permissions:`** block, and pushing an image needs **`packages: write`** (GitHub calls images "packages"). On many repos the token is **read-only by default**, so without that line the push is **denied**.
- **Docker Hub:** use an **access token**, a password-like key you create in Docker Hub's settings and can limit or **revoke** (cancel) at any time, so it's safer than your real password. Save it as a repository secret, e.g. `DOCKERHUB_TOKEN`.
- **Ready-made actions:** `docker/login-action` runs `docker login`; `docker/build-push-action` builds, applies every tag you list, and pushes. (`docker/setup-buildx-action` prepares **Buildx**, Docker's newer builder with extras like caching: it changes **how** you build, not whether you may push.)
- The first push creates a **private** package (your profile → **Packages**). Make it public, or Kubernetes needs a pull secret to download it.
### Image naming rules (a favourite "fix the code" question)
- The **repository** part (everything before the `:`) must be **lowercase** (`a–z 0–9 . _ - /`); the **tag** (after the `:`) may mix upper and lower case.
- Break the rule and you get: `invalid reference format: repository name (VanshSrivastava/hello-app) must be lowercase`.
- ⚠️ `${{ github.repository }}` is `Owner/RepoName` **exactly as written on GitHub** (e.g. `Vansh-S/HelloApp`), so it can break the rule. In bash, `${GITHUB_REPOSITORY,,}` gives the value in lowercase (`,,` = lowercase).
### 🛠 Do (~10 min): tag an image the way CI does
```bash
cd ~/devops-practice/hello-app
git init -b main && git add . && git commit -m "hello-app: ready for CI"   # make a commit
SHA=$(git rev-parse --short HEAD); echo "$SHA"    # e.g. 3f9c2a1
docker build -t hello-app:"$SHA" .                # tag = your SHA
docker images hello-app                           # see the new tag
docker tag hello-app:"$SHA" Hello-App:"$SHA"      # error: must be lowercase
docker tag hello-app:"$SHA" hello-app:Feature_1   # works
docker rmi hello-app:Feature_1                    # remove that label
```
**What each line does:**
- `cd ~/devops-practice/hello-app`: the folder with your Flask app and Dockerfile.
- `git init -b main && git add . && git commit -m "…"`: makes the folder a Git repo with first branch `main` (`-b`), stages every file (`.`), and commits (`-m` = message), so there's a commit SHA to use. `Reinitialized…` or `nothing to commit` just means it was done before: fine.
- `SHA=$(git rev-parse --short HEAD); echo "$SHA"`: `$( … )` captures a command's output into the variable `SHA`; `git rev-parse --short HEAD` = the 7-character short SHA of `HEAD`, the commit you're on. Prints e.g. `3f9c2a1`.
- `docker build -t hello-app:"$SHA" .`: builds the image tagged with that SHA, as CI does; the double quotes still let `$SHA` expand.
- `docker images hello-app`: shows the new tag, often with the **same IMAGE ID** as `1.0` (nothing changed, so Docker reused its cache): one image, two labels.
- `docker tag hello-app:"$SHA" Hello-App:"$SHA"`: tries to add a name with capitals in the **repository** part → error: `repository name (library/Hello-App) must be lowercase`.
- `docker tag hello-app:"$SHA" hello-app:Feature_1`: works, because upper case is allowed in the **tag** (after the `:`).
- `docker rmi hello-app:Feature_1`: removes only that extra label (`Untagged: …`); the image stays, since other tags still point to it.
*(`docker tag SOURCE NEWNAME` gives an existing image another name, without copying it. CI would name it `ghcr.io/<you-in-lowercase>/hello-app:<sha>` and upload it with `docker push`.)*

**🧠 In simple words:** You went into the folder that holds your Flask app and its Dockerfile and turned it into a Git repository with one commit, so your code now has a commit SHA (short for Secure Hash Algorithm), the unique ID Git calculates for every commit. You saved the short, 7-character version of that ID in a variable called `SHA` and built your image with it as the tag, exactly the way a CI (Continuous Integration) pipeline labels each build with the commit it came from, so anyone can later trace a running image back to its exact code. When you listed your images, the new tag usually showed the same image ID as your older `1.0` tag, because nothing in the code had changed and Docker reused its cache (the saved results of earlier builds), so it is one image wearing two labels. Next you used `docker tag`, which gives an existing image an extra name without copying it, to try a name with capital letters in the repository part (the part before the colon), and Docker refused with a "must be lowercase" error, because that part may only use lowercase letters. Giving it the tag `Feature_1` worked, because capitals are allowed in the tag part after the colon. Finally you removed that extra label, and only the label disappeared, because the image itself stays as long as other tags still point to it. In short: label every image with the commit ID it was built from, and keep the name before the colon lowercase, while the tag after it may mix cases.

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

**The problem.** The image is in the registry, but someone still types `kubectl set image …` by hand. At midnight Arjun deploys the wrong tag into the wrong namespace, and nobody notices for an hour, because nobody checked whether the rollout worked. The last mile (registry → cluster) must be automated **and verified**.
### What a pipeline needs in order to deploy
Every `kubectl` command is an HTTPS request to the cluster's **API server**, its front door (every tool and component talks to the cluster only through it). So the runner needs what your Mac has: the **tools** (`kubectl`/`helm`, already on `ubuntu-latest`), **credentials** (a kubeconfig, stored as a secret) and **a network path** to the API server's address.

A **kubeconfig** (by default `~/.kube/config`) is the file `kubectl` reads to know where to connect and who you are. It lists **clusters** (each API server's address, plus the certificate used to trust it), **users** (the **cluster credentials**: a client certificate or a token that proves who you are) and **contexts** (which user talks to which cluster, in which default namespace).

**Analogy: a hotel key card.** It encodes the building (cluster), who you are (user) and your floor (namespace). Whoever holds it gets in, so guard it like a password. Because it has many lines, teams **base64-encode** it into one line (an encoding, **not** encryption), save it as a secret like `KUBECONFIG_B64`, and a step decodes it: `echo "${{ secrets.KUBECONFIG_B64 }}" | base64 -d > ~/.kube/config`.
### Updating the app, then verifying it
A pipeline changes the running image in one of three ways:
- **apply manifests:** write the SHA into `k8s/deployment.yaml` (e.g. with `sed`), then `kubectl apply -f k8s/` (declarative)
- **set the image:** `kubectl set image deployment/hello hello=ghcr.io/vansh/hello-app:${{ github.sha }}` (imperative; in `hello=IMAGE`, the left side is the **container's** name in the Pod template)
- **Helm:** `helm upgrade --install hello ./chart --set image.tag=${{ github.sha }} --wait` (`--install` creates the release if it's missing; `--wait` fails if the Pods don't become ready)

⚠️ `kubectl set image` and `kubectl apply` only **submit** the change. They report success **immediately**, even if every new Pod is about to fail, so the pipeline would show a green ✅ over a broken deployment.

The fix is **`kubectl rollout status deployment/hello --timeout=120s`**, which waits and watches the rollout. It ends with `deployment "hello" successfully rolled out` (**exit code 0**) or, if the rollout isn't done within `--timeout`, with `error: timed out waiting for the condition` (**exit code 1**). A non-zero exit code fails the step, so a broken rollout now turns the pipeline **red**. A later step with **`if: failure()`** (run this step only if an earlier one failed) can then undo it: `kubectl rollout undo deployment/hello`.
### Environments, approval, and Continuous Delivery vs Deployment
An **environment** in GitHub Actions is a named deploy target such as `staging` or `production` (Settings → Environments). A job with `environment: production` can use secrets that belong **only** to that environment, and can have **required reviewers**: the job **pauses** ("Waiting for review") until a listed person clicks **Approve and deploy**. That's the difference between:
- **Continuous Delivery:** every change that passes the pipeline is **ready** to release, but a **human approves** the production deploy.
- **Continuous Deployment:** every passing change is **released automatically**, with **no human step**.
### Push-based vs pull-based deployment (preview of Session 14)
Everything above is **push-based**: the CI runner **pushes** changes **into** the cluster with `kubectl` or `helm`. The alternative is **pull-based**, better known as **GitOps**: an agent running **inside** the cluster (such as Argo CD) watches a Git repo and **pulls** changes in by itself; CI only updates Git.
- **Push-based:** the cluster credentials live in CI secrets, outside the cluster; CI must reach **in** to the API server; manual `kubectl` edits (**drift**) go unnoticed until the next deploy.
- **Pull-based:** the credentials never leave the cluster; the agent only reaches **out** to Git; drift is detected and can be reverted.
### Why GitHub's runners can't reach your minikube
Your kubeconfig says `server: https://127.0.0.1:32771`. `127.0.0.1` is **loopback** ("this machine"), so on a GitHub runner it means **the runner itself**, where nothing is listening: `connection refused`. And the cluster's real address (`192.168.49.2`) is a **private IP** inside Colima, behind your home router, which the internet can't reach. Ways around it:
- a **self-hosted runner**: GitHub's runner program on your Mac. It connects **out** to GitHub to ask for jobs, so kubectl runs on your Mac (⚠️ private repos only: on a public one, a stranger's PR could run code on your machine)
- a **cloud cluster** (Amazon EKS, Google GKE, Azure AKS), reachable but paid; or a throwaway **kind** (**K**ubernetes **in** **D**ocker) cluster on the runner, which only **tests** the manifests; or a **GitOps agent** inside minikube (like Argo CD, which pulls from Git, so nothing needs to reach in)

In class: **GitHub Actions builds, scans and pushes to GHCR; deploying to local Kubernetes is practised by hand** (below).
### 🛠 Do (~10 min): rehearse a pipeline's deploy commands on minikube
```bash
kubectl config view --minify | grep server                          # the API server address
kubectl create namespace s13 && kubectl create deployment web --image=nginx:1.27 --replicas=2 -n s13   # 2 Pods
kubectl rollout status deployment/web -n s13                        # wait until ready
kubectl set image deployment/web nginx=nginx:1.28 -n s13            # a good release
kubectl rollout status deployment/web -n s13 --timeout=60s; echo "exit code: $?"   # exit code: 0
kubectl set image deployment/web nginx=nginx:9.99 -n s13            # a bad release
kubectl rollout status deployment/web -n s13 --timeout=30s; echo "exit code: $?"   # exit code: 1
kubectl get pods -n s13                                             # 1 ImagePullBackOff
kubectl rollout undo deployment/web -n s13 && kubectl rollout status deployment/web -n s13   # back to 1.28
kubectl delete namespace s13                                        # clean up
```
**What each line does:**
- `kubectl config view --minify | grep server`: prints your kubeconfig; `--minify` = only the parts for the current context; `grep server` keeps the API server line: `server: https://127.0.0.1:<port>`, a loopback address that only your Mac can reach.
- `kubectl create namespace s13 && kubectl create deployment web --image=nginx:1.27 --replicas=2 -n s13`: makes a practice namespace, then a Deployment `web` with 2 Pods (`--replicas=2`) of `nginx:1.27`; `-n` = in namespace `s13`. kubectl names the container after the image: `nginx`.
- `kubectl rollout status deployment/web -n s13`: waits until the rollout is finished: `deployment "web" successfully rolled out`.
- `kubectl set image deployment/web nginx=nginx:1.28 -n s13`: what a pipeline's deploy step does: container `nginx` (left of `=`) gets image `nginx:1.28`, which starts a rolling update. It returns at once.
- `kubectl rollout status … --timeout=60s; echo "exit code: $?"`: waits at most 60 s, then prints the exit code (`;` = run the next command no matter what; `$?` = the last exit code). `exit code: 0` → a pipeline would continue.
- `kubectl set image … nginx=nginx:9.99 …`: a bad release (this tag doesn't exist), yet the command still succeeds.
- `kubectl rollout status … --timeout=30s; echo …`: after 30 s, `error: timed out waiting for the condition` and `exit code: 1` → in a pipeline this step goes red.
- `kubectl get pods -n s13`: 2 old Pods Running + 1 new Pod in `ImagePullBackOff`: the rolling update kept the old version serving.
- `kubectl rollout undo … && kubectl rollout status …`: puts the Deployment back to its previous Pod template (`nginx:1.28`), then waits for `successfully rolled out`.
- `kubectl delete namespace s13`: deletes the practice namespace and everything in it.

**🧠 In simple words:** First you looked inside your kubeconfig (the file that tells `kubectl` which cluster to talk to and who you are) and saw that the cluster's API (Application Programming Interface) server, its front door, sits at a loopback address, `127.0.0.1`, which only your own Mac can reach, so a GitHub runner out on the internet could not use it. Then you made a practice namespace called `s13` with a Deployment named `web` running two NGINX Pods, and waited until they were ready. Next you did exactly what a pipeline's deploy step does: you switched the image to `nginx:1.28`, and because `kubectl set image` returns straight away, you followed it with `kubectl rollout status`, which waited, saw the update finish, and ended with exit code 0, meaning a pipeline would carry on. After that you released a broken version, `nginx:9.99`, a tag that does not exist, and the set-image command still reported success, because it only submits the change and never checks whether it works. This time the rollout status command gave up after 30 seconds with "timed out waiting for the condition" and exit code 1, which is exactly the signal that turns a pipeline step red. Listing the Pods showed the two old Pods still running and one new Pod stuck in `ImagePullBackOff` (Kubernetes could not download its image), because the rolling update waits for new Pods to become ready before removing old ones, so the old version kept serving. You then rolled back to the previous version, `nginx:1.28`, waited for it to finish, and deleted the namespace to clean up. In short: changing the image only submits a request, so a pipeline must also wait for the rollout and check it, letting a failure stop the pipeline and a rollback bring back the last good version.

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

**The problem.** **Security at the end:** two weeks before a bank's launch, a separate security team audits the app and finds 60 problems in code written months ago. **And the code you didn't write:** in 2021 a flaw nicknamed **Log4Shell** was found in **Log4j**, a logging library inside millions of Java apps, and companies spent weeks just finding **which** of their apps contained it. Lessons: find problems **early**, and know **what your software is made of**.
### DevSecOps and shift-left
**DevSecOps** = **Dev**elopment + **Sec**urity + **Op**erations: security becomes **everyone's job**, with checks **automated in every stage of the pipeline** instead of one big review at the end.

**Shift-left** means **moving checks earlier** (to the left on the timeline below), because the earlier a problem is found, the cheaper the fix: minutes in your editor, versus an incident in production.

**Analogy:** spell-checking as you type, instead of spotting the typo after printing 1,000 brochures. And like airport security, there are **several layers**, each catching different things.
```
PLAN ─► CODE ─────────► BUILD ───────────► TEST ──────► RELEASE ─────► DEPLOY ─► OPERATE
        pre-commit       SAST, SCA, secret   DAST on      security       signed     rescans,
        secret scan      + image + IaC scan  staging      gate           images     monitoring
◄──────────── shift-left: move checks this way, where fixes are cheap ────────────
```
### The vocabulary of vulnerabilities
- A **vulnerability** is a **weakness** in software that an attacker could use; an **exploit** is the actual attack that uses it.
- **CVE** (**Common Vulnerabilities and Exposures**) gives every publicly known vulnerability an ID, `CVE-YEAR-NUMBER` (Log4Shell is `CVE-2021-44228`): its passport number, the same in every tool.
- **CVSS** (**Common Vulnerability Scoring System**) scores the danger from 0.0 to 10.0, which maps to a **severity** word: **LOW** (0.1–3.9), **MEDIUM** (4.0–6.9), **HIGH** (7.0–8.9), **CRITICAL** (9.0–10.0). Gates use these words ("block HIGH and CRITICAL").
### SAST: scanning your own source code
**SAST** = **Static Application Security Testing**. **Static** = **without running the program**: the tool reads your **source code** like a strict proof-reader, looking for dangerous patterns, e.g. a database query built by gluing in what the user typed:
```python
cursor.execute("SELECT * FROM users WHERE name = '" + name + "'")   # ⚠️ SQL injection
```
If a user types `' OR '1'='1`, the condition is true for every row and the query returns **all users**. This attack is **SQL injection**: user input changes the meaning of a database command. SAST reports it with the **file and line number**; the fix is a parameterised query, which treats the input only as data: `cursor.execute("… WHERE name = %s", (name,))`.
- **Tools:** **CodeQL** (GitHub's), **SonarQube**, **Semgrep**, **Bandit** (Python).
- **Weakness:** **false positives** (warnings about code that's actually safe), and it can't see how the running app behaves.
### SCA: scanning the libraries you depend on
**SCA** = **Software Composition Analysis**: checking **what your software is made of**, which is mostly **open-source libraries** written by other people.
- A **direct dependency** is a library **you** list (`flask` in `requirements.txt`).
- A **transitive dependency** is a dependency **of** a dependency: Flask pulls in `werkzeug`, `jinja2`, `click` and others. You never typed them, but they're in your app, and so are their vulnerabilities.

SCA compares every package version (transitive ones too) with databases of known vulnerabilities, and often checks **license compliance** (is each library's legal licence allowed in your product?). **Tools:** **Dependabot** (built into GitHub), **Snyk**, **`pip-audit`** (Python), **`npm audit`** (Node.js). **The course's fix flow:** find it → identify the package and version → check the fixed version → update the dependency → run the tests → run SCA again.
### DAST and IAST: testing the running app
**DAST** = **Dynamic Application Security Testing**. **Dynamic** = **while running**: the tool attacks your **running** app (usually on staging) **from the outside**, like a hacker, without seeing the code:
- it sends crafted HTTP requests: injection strings, and scripts that test for **XSS** (**Cross-Site Scripting**: tricking a site into sending an attacker's script to other users' browsers)
- it checks the responses, e.g. for missing **security headers** such as `Content-Security-Policy`
- the best-known tool is **ZAP** (**Zed Attack Proxy**) from **OWASP** (**Open Worldwide Application Security Project**, a non-profit that publishes free security tools and guides). DAST finds real behaviour, but **can't point to the line of code** at fault.

**IAST** (**Interactive** AST, awareness) puts a small agent **inside** the running app (adding watching code like this is called **instrumentation**), so it sees both the request and the exact code that handled it.
### Secret scanning and IaC scanning
**Secret scanning** looks for **passwords, API keys and tokens** written into files or Git history. Tools like **gitleaks** and **TruffleHog** match known patterns (GitHub tokens start with `ghp_`, AWS keys with `AKIA`), and GitHub's **push protection** **blocks the `git push`** itself. Run a scanner as a **pre-commit hook** (a script Git runs before each commit: if it exits 1, the commit is cancelled) **and** in CI, since hooks aren't shared when someone clones the repo.

**If a real secret leaks,** deleting it in a new commit is **not** enough: the old commit still has it, and bots scan public GitHub for keys within minutes. Treat it as **compromised**: **revoke/rotate it first** → remove it from code and history → check for misuse → keep the new one in a secret (an environment variable filled from a GitHub Actions secret), never in code.

**IaC scanning** (awareness): **IaC** (**Infrastructure as Code**) means describing servers and clusters in files (Terraform, Kubernetes YAML). IaC scanners check those files for a **misconfiguration** (an unsafe setting) **before** anything is created: SSH open to the whole internet (`0.0.0.0/0`), a public storage bucket, a container running as root. Tools: **Checkov**, `trivy config .`.
### The comparison table (memorise this)
| | **SAST** | **SCA** | **DAST** | **Secret scanning** | **Image scanning** (81) |
|---|---|---|---|---|---|
| Scans | your source code | your dependencies | the running app (HTTP) | files + Git history | the built image |
| App running? | ❌ | ❌ | ✅ **Yes** | ❌ | ❌ |
| When | code / PR | code / PR / build | after deploying to staging | pre-commit + CI | after `docker build` |
| Finds | bad code patterns | known CVEs in libraries | real runtime weaknesses | leaked credentials | CVEs in OS + library packages |
| Examples | CodeQL, SonarQube, Semgrep | Dependabot, Snyk, pip-audit | OWASP ZAP, Burp Suite | gitleaks, TruffleHog | Trivy, Grype, Docker Scout |

⚠️ **Traps:** static = not running (SAST); dynamic = running (DAST). SCA ≠ SAST (libraries vs your code): a clean SAST result says nothing about a vulnerable Flask. Each scan is one layer.
### 🛠 Do (~10 min): run an SCA scan and a secret scan, both in containers
Both tools run in throwaway containers (`--rm` = delete afterwards; `-v "$PWD":/src` = bind-mount this folder at `/src`; `-w /src` = start there). Ignore pip's warning about running as root.
```bash
cd ~/devops-practice/hello-app && cat requirements.txt   # flask==3.0.3
docker run --rm -v "$PWD":/src -w /src python:3.12-slim sh -c "pip install -q pip-audit && pip-audit -r requirements.txt"; echo "exit code: $?"   # exit code: 1
sed -i '' 's/flask==3.0.3/flask==3.1.3/' requirements.txt   # the fix
docker run --rm -v "$PWD":/src -w /src python:3.12-slim sh -c "pip install -q pip-audit && pip-audit -r requirements.txt"; echo "exit code: $?"   # exit code: 0
docker build -t hello-app:2.0 .                                  # rebuild with the fixed Flask
mkdir -p ~/devops-practice/s13-secrets && cd ~/devops-practice/s13-secrets
printf 'GITHUB_TOKEN = "ghp_4fT9xQ2mZ7LbV1nK8sR3wY6pD0cH5jE2aU9i"\n' > settings.py      # a FAKE token
docker run --rm -v "$PWD":/scan ghcr.io/gitleaks/gitleaks:v8.30.1 dir -v /scan; echo "exit code: $?"   # exit code: 1
cd ~ && rm -rf ~/devops-practice/s13-secrets                     # clean up
```
**What each line does:**
- `cd ~/devops-practice/hello-app && cat requirements.txt`: shows the pinned libraries: `flask==3.0.3`, the version from when you first built the image.
- `docker run --rm -v "$PWD":/src -w /src python:3.12-slim sh -c "…"; echo "exit code: $?"`: a throwaway Python container with this folder (`$PWD`) mounted at `/src`. `sh -c "…"` runs the quoted commands inside: install pip-audit quietly (`-q`), then check every package listed in the file (`-r`). Expect `flask 3.0.3  PYSEC-2026-2151  3.1.3` (package, advisory ID, fixed version) and `exit code: 1`.
- `sed -i '' 's/flask==3.0.3/flask==3.1.3/' requirements.txt`: the fix. Edits the file in place (`-i ''` = the macOS form, no backup file); `s/old/new/` = replace the text, so Flask is bumped to the fixed version.
- the second `docker run … pip-audit …`: the same scan again: `No known vulnerabilities found`, `exit code: 0`.
- `docker build -t hello-app:2.0 .`: rebuilds the image with the fixed Flask as `hello-app:2.0` (Trivy scans it in the next section).
- `mkdir -p … && cd …`: a scratch folder for the secret-scan demo.
- `printf 'GITHUB_TOKEN = "ghp_…"\n' > settings.py`: writes a file containing a **fake** GitHub token; it starts with `ghp_` like real ones, which is the pattern scanners look for.
- `docker run --rm -v "$PWD":/scan ghcr.io/gitleaks/gitleaks:v8.30.1 dir -v /scan; echo …`: runs gitleaks from its official image on the folder mounted at `/scan`; `dir` = scan a plain folder (not Git history); `-v` = verbose, show each finding. Expect `RuleID: github-pat`, `leaks found: 1`, `exit code: 1`.
- `cd ~ && rm -rf ~/devops-practice/s13-secrets`: leave the folder and delete it (`-r` = with everything inside, `-f` = don't ask).
Both scanners **exit 1** when they find something, so in CI the step would **fail** (a gate), and in a pre-commit hook gitleaks would cancel the commit. (`PYSEC-…` is the Python advisory ID; Trivy calls the same Flask issue `CVE-2026-27205`.)

**🧠 In simple words:** You ran two kinds of security scanner, each inside a throwaway container that is deleted afterwards with your folder shared into it, so nothing had to be installed on your Mac. First you looked at your app's list of libraries and saw that it pins Flask at version 3.0.3. Then you ran pip-audit, an SCA (Software Composition Analysis) tool that checks the libraries you depend on against databases of known vulnerabilities, and it reported that Flask 3.0.3 has a known problem which is fixed in 3.1.3, ending with exit code 1. You fixed it the proper way, by editing the requirements file to bump Flask to 3.1.3, and the same scan then found no known vulnerabilities and ended with exit code 0, after which you rebuilt your image as `hello-app:2.0` with the safe Flask inside. For the second scanner you made a scratch folder and wrote a file containing a fake GitHub token that starts with `ghp_`, the same pattern real tokens have. gitleaks, a secret scanner that hunts for passwords and keys left in files, read the folder as plain files, recognised the text as a GitHub personal access token (PAT), reported one leak and ended with exit code 1, and then you deleted the scratch folder. Both tools exit with 1 when they find something, so in a pipeline that step would fail and stop everything after it, and in a pre-commit hook (a script Git runs before each commit) gitleaks would cancel the commit. In short: a scanner protects you because a finding makes it exit with 1, which turns it into a gate that stops unsafe code from moving forward.

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

**The problem.** Riya's code passes SAST and SCA, but the image isn't just her code: `python:3.12-slim` contains a small Debian Linux with about 87 **OS packages** (libraries like `openssl`, tools like `bash`), plus Python and pip, and new CVEs are found in packages like these every week. **Clean source code does not mean a clean image**, so scan the thing you actually ship.
### Trivy: what it does
**Trivy** (from Aqua Security) is a free scanner: `trivy image` scans images (`trivy fs` scans folders, `trivy config` IaC files). It unpacks the image's **layers** (the stacked read-only filesystem pieces, one per Dockerfile instruction), lists the installed **OS packages** and **language packages** (Python, Node, Java libraries), and compares each name + version with its **vulnerability database** (a big list of CVEs and the versions they affect, downloaded on the first run).

Each row of the report shows **Library** · **Vulnerability** (the CVE ID) · **Severity** · **Status** · **Installed Version** · **Fixed Version**. The **Status** tells you whether you can act: **`fixed`** = a newer version of the package fixes it (upgrading or rebuilding helps); **`affected`** = no fix exists yet.

| Flag | What it does |
|---|---|
| `--severity HIGH,CRITICAL` | show and count only these severities |
| `--exit-code 1` | exit with code 1 if anything is found (after the filters) |
| `--ignore-unfixed` | skip findings with no fix yet (upgrading can't fix them anyway) |
| `-q` / `--format sarif` / `cyclonedx` | quiet (no progress lines) / output for GitHub's Security tab / an SBOM |
| `.trivyignore` | a file of accepted-risk IDs to skip, one per line, read from the current folder |

⚠️ **The number-one gate mistake:** `--severity HIGH,CRITICAL` **only filters the report**. Without `--exit-code 1`, Trivy exits **0** even with CRITICAL findings, so the pipeline carries on (the course's own demo workflow has exactly this gap). **The gate is `--exit-code 1`.**
### Security gates
A **security gate** is a pipeline step that **decides whether the pipeline may continue**. It has three parts: a **check** (the scan), a **policy** (the rule for what's acceptable, e.g. "no HIGH or CRITICAL findings that have a fix") and **enforcement** (a **non-zero exit code** when the policy is broken).

**Analogy:** a scan is a smoke detector; a gate is the fire door. The detector finds smoke, but the door decides whether anyone gets through.
```
docker build ─► trivy image --severity HIGH,CRITICAL --exit-code 1 app:sha
                   exit 0 ─► push to GHCR ─► deploy
                   exit 1 ─► step ✗ → job ✗ → jobs that need it SKIPPED → red PR check → merge blocked
```
- **Exit codes:** a step whose command exits non-zero **fails**. **`needs:`** (makes one job wait for another): a job that needs a failed job is **skipped**, so push and deploy never run.
- **Branch protection:** a rule on `main`, "Require status checks to pass before merging". The jobs you pick (e.g. `test`, `image`) become **required status checks**, and while one is red, the PR's merge button is disabled. Fix it **on the same branch** and push: the PR re-runs its checks (no new PR needed).
- Any stage can gate (tests, SAST, SCA, secrets, build, image scan, push, rollout). HIGH/CRITICAL is a **classroom threshold**: each organisation sets its own policy.

⚠️ **Four ways to accidentally switch a gate off:**
1. **No `--exit-code 1`:** Trivy exits 0.
2. **A pipe:** `trivy … --exit-code 1 app | tee report.txt`. A pipe's exit code is the **last** command's (`tee` → 0). GitHub's default `run:` shell doesn't notice a failure inside a pipe; only `shell: bash` (which adds `-o pipefail`) does.
3. **`continue-on-error: true`** on the step: the step may fail, but the job carries on.
4. **Scanning one image and pushing another**, e.g. a later job **rebuilds** before pushing. Build once, scan that image, push that same image.
### Fewer findings, SBOM and signing
- **Minimal base image** (`alpine`, `-slim`): fewer packages = fewer things that can be vulnerable. **Multi-stage builds** keep build tools out of the final image. **Rebuild regularly** (a built image never patches itself) and **rescan on a schedule** (new CVEs appear daily).
- ⚠️ **Trap: non-root.** `USER appuser` is excellent practice but **doesn't lower the CVE count** (the same packages are installed); it lowers the **damage** an attacker can do.
- **SBOM** (**Software Bill of Materials**): an **ingredients list** of every package and version in an image, in a standard format (**SPDX** or **CycloneDX**). When the next Log4Shell appears, you search your SBOMs and know in minutes which images are affected: `trivy image --format cyclonedx --output sbom.json hello-app:2.0`.
- **Image signing** with **cosign**: a digital signature proving **who built** an image and that it **hasn't been changed** since (`cosign sign` in CI, `cosign verify` before deploying).
### 🛠 Do (~15 min): scan images with Trivy on your Mac
```bash
brew install trivy && trivy --version                          # 0.75.0 or newer
export DOCKER_HOST="unix://$HOME/.colima/default/docker.sock"  # ⚠️ point Trivy at Colima
trivy image -q --severity HIGH,CRITICAL nginx:1.27    | grep Total   # Total: 174
trivy image -q --severity HIGH,CRITICAL nginx:alpine  | grep Total   # Total: 3
trivy image -q --severity HIGH,CRITICAL hello-app:1.0 | grep Total   # Total: 44
trivy image -q --severity HIGH,CRITICAL --exit-code 1 nginx:alpine; echo "exit code: $?"          # 1: blocked
trivy image -q --severity CRITICAL --exit-code 1 nginx:alpine; echo "exit code: $?"               # 0: passes
trivy image -q --severity HIGH,CRITICAL --exit-code 1 nginx:alpine | tail -3; echo "exit code: $?"   # 0! the pipe
trivy image -q --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 hello-app:2.0; echo "exit code: $?"   # 0
```
**What each line does:**
- `brew install trivy && trivy --version`: installs Trivy with Homebrew and prints its version: `Version: 0.75.0` or newer.
- `export DOCKER_HOST="unix://$HOME/.colima/default/docker.sock"`: ⚠️ tells Trivy where Colima's Docker socket is (the file Docker listens on), so it can find your local images. `export` lasts for this Terminal tab only: rerun it in a new tab, or if an image "can't be found".
- `trivy image -q --severity HIGH,CRITICAL nginx:1.27 | grep Total`: scans the image; `-q` = quiet (no progress lines); `--severity` = show and count only HIGH and CRITICAL; `grep Total` keeps just the summary. The first run downloads the vulnerability DB (~130 MB). Expect `Total: 174 (HIGH: 163, CRITICAL: 11)`.
- `… nginx:alpine | grep Total`: `Total: 3`: the Alpine image has far fewer packages (71 vs 144), so far fewer CVEs.
- `… hello-app:1.0 | grep Total`: your own image: `Total: 44 (HIGH: 44, CRITICAL: 0)`. The numbers drift daily as new CVEs are published.
- `… --exit-code 1 nginx:alpine; echo "exit code: $?"`: `--exit-code 1` = exit with 1 if anything is found after the filters: this makes it a gate. Its 3 findings give `exit code: 1`: blocked.
- `… --severity CRITICAL --exit-code 1 nginx:alpine; echo …`: a looser policy (CRITICAL only) finds nothing to block: `exit code: 0`.
- `… --exit-code 1 nginx:alpine | tail -3; echo …`: `tail -3` = show the last 3 lines. `exit code: 0`! A pipe reports only the **last** command's exit code (`tail`'s), so piping the gate's output switches the gate off.
- `… --ignore-unfixed --exit-code 1 hello-app:2.0; echo …`: `--ignore-unfixed` = skip findings with no fix yet. `exit code: 0`, because all its HIGHs have Status `affected` (no fix exists).

The same image passes or fails depending on the **policy**, and the last line is why the pipeline below uses `--ignore-unfixed`: it blocks the problems you **can** fix.

**🧠 In simple words:** You installed Trivy, a free tool that scans container images for known vulnerabilities, and told it where Colima's Docker socket is (the file Docker listens on) so it could find the images on your Mac. Then you scanned three images while counting only HIGH and CRITICAL problems: the full `nginx:1.27` image had 174, the small Alpine-based `nginx:alpine` had only 3 because it contains far fewer packages, and your own `hello-app:1.0` had 44 (these numbers change daily as new vulnerabilities are published). Next you added `--exit-code 1`, which makes Trivy exit with 1 whenever anything is found, and that turned the scan into a gate: `nginx:alpine` with its 3 findings was blocked with exit code 1. When you loosened the policy to CRITICAL only, the same image had nothing to block, so it passed with exit code 0. Then you sent the gate's output through a pipe into `tail`, and the exit code became 0 even though the findings were still there, because a pipe reports only the last command's exit code, and `tail` itself succeeded. Finally you scanned your fixed `hello-app:2.0` with `--ignore-unfixed`, which skips problems that have no fix yet, and it passed with exit code 0 because all its HIGH findings had the status "affected", meaning no fixed version exists. This shows that the same image can pass or fail depending on the policy, which is why the secure pipeline that follows uses `--ignore-unfixed`, so that it blocks only the problems you can actually fix. In short: a scan only becomes a gate when it exits with 1, and what it blocks depends on the rules you give it.

### The complete secure pipeline (test → SCA → build → Trivy gate → push to GHCR)
First create `~/devops-practice/hello-app/test_app.py`:
```python
from app import app

def test_home_page():
    response = app.test_client().get("/")   # Flask's pretend browser sends GET /
    assert response.status_code == 200      # the page loaded (200 OK)
    assert b"Hello" in response.data        # and it contains "Hello"
```
Then create `.github/workflows/secure-pipeline.yml` in the same folder (`mkdir -p .github/workflows` first):
```yaml
name: secure-pipeline                  # name shown in the Actions tab
on:
  push:
    branches: [main]                   # run on every push to main…
  pull_request:
    branches: [main]                   # …and on every PR into main (the gate)
permissions:
  contents: read                       # least privilege: every job starts read-only
jobs:
  test:                                # stages 1-2: tests + SCA
    runs-on: ubuntu-latest             # a fresh GitHub-hosted Linux runner
    steps:
      - uses: actions/checkout@v7      # get the code onto the runner
      - uses: actions/setup-python@v7  # install Python…
        with:
          python-version: "3.12"       # …this version
      - name: Install dependencies
        run: pip install -r requirements.txt pytest pip-audit   # app libraries + test and SCA tools
      - name: Unit tests
        run: pytest -q                       # a failing test exits non-zero → stop
      - name: "SCA: scan dependencies for known CVEs"
        run: pip-audit -r requirements.txt   # a known vulnerability → exit 1 → stop
  image:                               # stages 3-5: build, scan gate, push
    needs: test                        # only if tests AND SCA passed
    runs-on: ubuntu-latest
    permissions:                       # a job's own list replaces the top one…
      contents: read                   # …so repeat read access for checkout
      packages: write                  # only this job may push to GHCR
    steps:
      - uses: actions/checkout@v7
      - name: Work out the image name    # ,, = lowercase; >> $GITHUB_ENV keeps $IMAGE for later steps
        run: echo "IMAGE=ghcr.io/${GITHUB_REPOSITORY_OWNER,,}/hello-app:${GITHUB_SHA}" >> "$GITHUB_ENV"
      - name: Build the image
        run: docker build -t "$IMAGE" .      # tagged with the full commit SHA
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
          registry: ghcr.io                  # leave out → Docker Hub
          username: ${{ github.actor }}      # the user who triggered the run
          password: ${{ secrets.GITHUB_TOKEN }}   # the automatic token
      - name: Push the image
        if: github.event_name == 'push'
        run: docker push "$IMAGE"            # the SAME image that passed the scan
```
*Why `$GITHUB_ENV`: each `run:` step is a new shell, so a normal variable would be lost; a `NAME=value` line appended to that file becomes a variable in **all later steps**. The Trivy step runs Trivy from its official image, and mounting `/var/run/docker.sock` lets it ask the runner's Docker for the image just built.*

**Optional, on your GitHub** (creates a public repo): keep `flask==3.1.3` (or the SCA step fails: the gate working), then `git add . && git commit -m "Add secure pipeline" && gh repo create hello-app-ci --public --source=. --push` and `gh run watch`; the image appears under **Packages**. To see a PR blocked, protect `main` (Settings → Branches, "Require status checks to pass", pick `test` and `image`), then push a branch with `flask==3.0.3` and open a PR: `test` goes red at SCA, `image` is skipped, merge is blocked.
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

| Item | What it means |
|---|---|
| **Dependabot** | GitHub's dependency bot: **alerts** you to vulnerable dependencies and opens **PRs** that bump them to the fixed version (regular updates are set in `.github/dependabot.yml`) |
| **CodeQL** / **SARIF** | GitHub's SAST engine, switched on in Settings → Code security, results in the **Security** tab / **SARIF** = a standard JSON format for scan results, so GitHub can show them (`trivy image --format sarif`) |
| **OWASP Top 10** | OWASP's list of the ten most critical web-app risks: **Broken Access Control** is number 1; others include **Injection** and **Security Misconfiguration** |
| **CVE vs CWE** | a **CVE** is one specific vulnerability (CVE-2021-44228 in Log4j); a **CWE** (Common Weakness Enumeration) is a **type** of weakness (CWE-89 = SQL injection) |
| **False positive / negative** | the tool reports a problem that isn't real (common in SAST) / misses a real one (why you use several layers) |
| **`.trivyignore` entries** | write a reason next to each ignored ID; `CVE-2026-27205 exp:2026-12-31` stops ignoring it after that date |
| **Pinning actions** | `@v7` is a **tag**, and tags can be moved to different code; `@<full commit SHA>` can never change (in 2025 a popular action's tags were moved to code that leaked secrets; SHA-pinned workflows were safe) |
| **Least privilege** / **`permissions:` rule** | give each job only the permissions it needs / as soon as you list **any** permission, **every one you didn't list becomes `none`** |
| **Fork PRs** / **masking** | PRs from a fork get **no secrets** and a read-only token / GitHub shows secret values as `***` in logs (still never `echo` them) |
| **`if: always()`** | run a step no matter what happened before it (e.g. upload the scan report) |
| **Scheduled rescans** | `on: schedule: - cron: "0 2 * * *"` = every day at 02:00 **UTC** (GitHub's clock); catches new CVEs in unchanged images |
| **Signed commits** / **other scanners** | `git commit -S` (GPG or SSH key) shows **Verified**; branch protection can require it / image: **Grype**, **Docker Scout**; SCA: `npm audit` (exits non-zero on findings, like `pip-audit`) |

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
### ✍️ Write the command
**C1.** Build an image from the Dockerfile in the current folder, named `hello-app` with the tag `3f9c2a1` (a short commit SHA).

**C2.** Give the local image `hello-app:3f9c2a1` a second name for GHCR (GitHub Container Registry) under the owner `vansh-s`, keeping the same tag, then upload it there (you're already logged in).

**C3.** Scan the image `hello-app:3f9c2a1` with Trivy as a security gate: report only HIGH and CRITICAL vulnerabilities, and make the command fail (exit code 1) if it finds any.

**C4.** Check the Python packages listed in `requirements.txt` for known vulnerabilities (SCA, Software Composition Analysis).
