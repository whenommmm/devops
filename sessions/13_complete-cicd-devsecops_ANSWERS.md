# Answer key · Session 13 · Complete CI/CD & DevSecOps

⚠️ Only open after answering.

---

## Warm-up

**W1.** A workflow has two jobs, `build` and `test`, and no `needs:`. `build` runs `docker build -t app:ci .` and `test` runs `docker run app:ci pytest`. The `test` job fails with `Unable to find image 'app:ci' locally`, followed by a pull error. What's the real cause?
- A) Each job gets its own fresh runner machine
- B) `test` lacks `permissions: packages: read`
- C) `docker run` needs `--pull never` in CI jobs
- D) They ran in parallel; adding `needs:` fixes it

**Correct: A.** Every job runs on its **own fresh runner** (a separate virtual machine), so the image built in `build` simply doesn't exist on `test`'s runner. Docker then tries to pull `app:ci` from Docker Hub, which fails. D is the tempting half-truth: without `needs:` they did run in parallel, but adding `needs: build` only changes the **order**; `test` would still run on a different machine with no `app:ci`. You'd have to build in the same job, or pass the image along (push it to a registry, or save it as an artifact). B: no registry permission is involved, because the image was never pushed. C: `--pull never` would just fail sooner with "no such image".

---

**W2.** A step has `env:` with `IMAGE: hello`, and runs `run: echo '$IMAGE:latest'`. What's printed?
- A) `hello:latest`, as env vars always expand
- B) `$IMAGE:latest`, as quotes stop the `$`
- C) `:latest`, as IMAGE isn't set in the shell
- D) An error, as `:` can't follow a variable

**Correct: B.** The **outer quotes decide**: single quotes make everything literal, so bash prints the text `$IMAGE:latest` exactly. A would be right with double quotes (`"$IMAGE:latest"` → `hello:latest`). C is wrong because `env:` really does set `IMAGE` in the step's shell; it just isn't expanded inside `'…'`. D: a colon after a variable is perfectly legal (`$IMAGE:latest` with double quotes works fine).

---

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

**Correct: C.** A function's exit code is its `return N` or, if there's no `return`, its **last command's** exit code. `grep` exits **0** when it finds a match, **1** when it finds no match, and **2** when something goes wrong (like a missing file). No match here, so `check` returns 1 and `echo $?` prints `1`. A confuses "the function ran" with "the function succeeded". B: 2 is for errors (the file exists, so no error). D: `-q` only silences **grep's** own output; `echo` is a separate command.

---

## Check 78

**78.1** Production runs `ghcr.io/vansh/hello-app:latest`, and a bug appears. Why would a tag like `hello-app:3f9c2a1` (a commit SHA) have helped?
- A) It makes the image smaller and faster to pull
- B) It makes Kubernetes always pull a fresh image
- C) It maps the running image to one exact commit
- D) It signs the image, so nobody can tamper with it

**Correct: C.** A SHA tag gives **traceability**: from the running Pod's image tag you can run `git show 3f9c2a1` and see the exact code, author and PR. `latest` moves with every push, so it tells you nothing. A: a tag is just a label; it doesn't change the image's size. B: `latest` is actually the tag that makes `imagePullPolicy` default to `Always`; other tags default to `IfNotPresent`. D: signing is a separate step (cosign); a tag proves nothing about tampering.

---

**78.2** A workflow's push step fails with `denied: permission_denied: write_package` when pushing to `ghcr.io`. The login step succeeded, the job has no `permissions:` block, and the repository's default token is read-only. What's the fix?
- A) Add `permissions: packages: write` to the job
- B) Replace GITHUB_TOKEN with your GitHub password
- C) Add `docker/setup-buildx-action` before the push
- D) Tag the image `latest` instead of the git SHA

**Correct: A.** `GITHUB_TOKEN` can only do what `permissions:` allows. Logging in only needs read access, but pushing a package needs `packages: write` (add `contents: read` too, so checkout still works). B: GHCR doesn't accept your account password, and putting it in a workflow would be a security disaster. C: Buildx changes **how** the image is built, not whether the token may write. D: the tag name has nothing to do with permissions.

---

**78.3** (one line) `docker build -t ghcr.io/Vansh-S/hello-app:1.0 .` fails immediately. Why, and what's the fix?

**Model answer:** *"The repository part of an image name must be lowercase, and `Vansh-S` contains capitals, so Docker rejects it with `invalid reference format: repository name … must be lowercase`; fix it by writing `ghcr.io/vansh-s/hello-app:1.0` (in a workflow, lowercase it, e.g. with bash's `${GITHUB_REPOSITORY_OWNER,,}`)."*

---

## Check 79

**79.1** A deploy job runs `kubectl set image deployment/hello hello=ghcr.io/vansh/hello-app:${{ github.sha }}`. The job is green ✅, but the new Pods are stuck in `ImagePullBackOff`. What should the job have included?
- A) A `kubectl rollout status --timeout` step
- B) A `kubectl apply -f` step before set image
- C) The `environment: production` approval key
- D) A `helm repo update` step before the deploy

**Correct: A.** `kubectl set image` only **submits** the change and returns success immediately. `kubectl rollout status deployment/hello --timeout=120s` waits for the new Pods; when they never become ready it prints `error: timed out waiting for the condition` and exits **1**, which turns the job red (and can trigger an `if: failure()` rollback step). B: applying manifests also returns immediately, so it doesn't verify anything either. C: approval decides **whether** to deploy, not whether the deploy worked. D: `helm repo update` refreshes chart lists; it has nothing to do with a `kubectl` rollout.

---

**79.2** Your GitHub-hosted workflow uses your minikube kubeconfig, stored as a secret. `kubectl get pods` fails with `dial tcp 127.0.0.1:32771: connect: connection refused`. Why?
- A) The kubeconfig secret was base64-encoded twice
- B) On the runner, 127.0.0.1 is the runner itself
- C) minikube refuses connections without a Service
- D) kubectl needs `--insecure` to reach port 32771

**Correct: B.** minikube's kubeconfig says `server: https://127.0.0.1:32771`. `127.0.0.1` is **loopback**: on the runner, it means the runner itself, where nothing listens on 32771, hence "connection refused". Your real cluster sits on a private IP inside Colima, behind your router, and can't be reached from GitHub's datacentre anyway. A: a double-encoded file would fail earlier, with a kubeconfig **parsing** error, not a TCP connection error. C: Services are for reaching Pods; the API server needs no Service from you. D: `--insecure-skip-tls-verify` affects certificate checks, which happen **after** a connection succeeds; here the connection itself was refused.

---

**79.3** Which describes **Continuous Delivery** (as opposed to Continuous Deployment)?
- A) Every passing change goes live with no human step
- B) Code is merged and tested often but never packaged
- C) The cluster pulls each change from Git by itself
- D) Each change is ready; a human approves production

**Correct: D.** In Continuous **Delivery**, every change that passes the pipeline is **releasable**, but a person approves the production release (in GitHub Actions: an `environment` with required reviewers). A is Continuous **Deployment** (fully automatic). B describes only Continuous Integration. C describes pull-based GitOps, which is **how** changes get applied, not whether a human approves.

---

## Check 80

**80.1** Which of these needs the application to be **running** in order to find problems?
- A) A SAST tool such as Semgrep
- B) An SCA tool such as pip-audit
- C) A DAST tool such as OWASP ZAP
- D) A secret scanner such as gitleaks

**Correct: C.** **D**AST = **Dynamic**: it attacks the running app through HTTP. A: SAST is **static**: it reads source code without running it. B: SCA reads dependency files (`requirements.txt`). D: secret scanners read files and Git history.

---

**80.2** pip-audit reports `flask 3.0.3  PYSEC-2026-2151  Fix Versions 3.1.3`. Your own code has no bugs. What's the right fix?
- A) Add the ID to `.trivyignore` so the scan passes
- B) Run SAST on app.py, since the bug is in your code
- C) Rebuild on `alpine` so the CVE count drops
- D) Bump flask to 3.1.3, then rerun tests and SCA

**Correct: D.** This is the course's remediation flow: identify the package and version, check the fixed version, **update the dependency**, run the tests, run SCA again. A: `.trivyignore` is read by **Trivy**, not pip-audit, and ignoring a fixable issue is hiding it, not fixing it. B: the problem is in Flask (a dependency), not your code, so SAST is the wrong tool. C: the base image controls **OS** packages; Flask 3.0.3 would still be installed by pip on Alpine.

---

**80.3** (one line) A teammate pushed a real AWS key to a public GitHub repo, then deleted it in the next commit. Is that enough? What must happen first?

**Model answer:** *"No: the key is still in the earlier commit in Git history, and bots scan public GitHub within minutes, so it must be treated as compromised; first revoke/rotate the key, then remove it from the history, check for unauthorised use, and store the replacement in a secret (environment variable or GitHub Actions secret), never in code."*

---

## Check 81

**81.1** A gate step runs `trivy image --severity HIGH,CRITICAL --exit-code 1 app:ci | tee trivy.txt`. Trivy finds 2 CRITICAL CVEs, yet the step is green. The step has no `shell:` key. Why?
- A) `--severity` must be lowercase: `high,critical`
- B) `tee` exited 0, and the pipe hid Trivy's exit code
- C) `--exit-code 1` only applies when output is JSON
- D) GitHub treats exit code 1 as a warning, not a failure

**Correct: B.** A pipe's exit code is the **last** command's exit code, here `tee`'s (0). With no `shell:` key, GitHub runs the step with `bash -e`, which only stops on a failing command **outside** a pipe; `shell: bash` would add `-o pipefail` and make the step fail. (You saw this on your Mac: `… | tail -3; echo $?` printed 0.) A: Trivy accepts the uppercase names, as the course uses them. C: `--exit-code` works with every output format. D: any non-zero exit code fails a step.

---

**81.2** Which change **reduces the number of CVEs** Trivy reports for an image?
- A) Adding `USER appuser` so the app runs as non-root
- B) Adding `--exit-code 1` to the Trivy scan command
- C) Basing it on `nginx:alpine`, not `nginx:1.27`
- D) Pushing the image to GHCR instead of Docker Hub

**Correct: C.** Fewer installed packages means fewer known CVEs: in your scan, `nginx:1.27` (Debian, 144 packages) had about 707 findings and `nginx:alpine` (71 packages) about 6. A is the classic trap: non-root reduces the **damage** if someone breaks in, but the same packages (and CVEs) are still there. B changes what happens **after** the scan (exit code), not what's found. D: the registry stores the same bytes; the contents decide the findings.

---

## Session quiz

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

**Correct: C.** `--severity` only **filters the report**. Without `--exit-code 1`, Trivy exits 0 even with CRITICAL findings, so `image-scan` succeeds and `push` runs. (The course's own demo workflow has exactly this gap.) A is backwards: `needs:` requires the needed job to **succeed**; if it fails, the dependent job is skipped. B: no such rule exists; with `--exit-code 1`, any finding at the selected severities fails it. D: every `run:` step fails on a non-zero exit code by default (`bash -e`); `shell: bash` only adds `pipefail` for pipes.

---

**Q2.** Your pipeline pushes `ghcr.io/vansh/hello-app:latest` on every merge, then runs `kubectl apply -f k8s/deployment.yaml`, which contains `image: ghcr.io/vansh/hello-app:latest`. The apply prints `deployment.apps/hello unchanged`, and the old code keeps running. What's the best explanation?
- A) The Pod template is unchanged, so no rollout runs
- B) GHCR caches the `latest` tag for an hour after a push
- C) `kubectl apply` only works after a `kubectl create`
- D) The Pods need `imagePullPolicy: Never` to update

**Correct: A.** A Deployment rolls out new Pods only when its **Pod template changes**. The YAML still says `…:latest`, exactly as before, so Kubernetes sees nothing to do (`unchanged`), and the running Pods keep the image they pulled earlier. Unique tags (the commit SHA) fix this, because every build changes the template. B: GHCR serves the new `latest` immediately. C: `kubectl apply` creates **or** updates; no `create` is needed first. D: `Never` means "never pull", which is the opposite of what you want.

---

**Q3.** Your pipeline runs CodeQL, pip-audit, gitleaks and Trivy, and all of them pass. A ZAP scan of the staging site then finds that the login page sends no `Content-Security-Policy` header and reflects user input back into the page. Why did only ZAP find these?
- A) CodeQL scans dependency files only, not app code
- B) pip-audit skips web frameworks like Flask by default
- C) Trivy reports headers only with `--scanners misconfig`
- D) Only DAST sees the live app's real HTTP responses

**Correct: D.** A missing response header and input reflected into a page are properties of the **running** app's HTTP responses. ZAP is the only DAST tool in the list: it sends requests and inspects what comes back. A is backwards: CodeQL is SAST and scans **your code** (scanning dependencies is SCA). B: pip-audit does check Flask, but only for known CVEs in its version, not for how your app behaves. C: Trivy's misconfiguration scanner checks files like Dockerfiles, Kubernetes YAML and Terraform; it never sends HTTP requests to a live site.

---

**Q4.** Which statement about push-based deployment (from GitHub Actions) and pull-based GitOps is **NOT** correct?
- A) Push-based CI must hold credentials for the cluster
- B) Push-based CI reaches a laptop cluster by default
- C) Pull-based agents run inside the target cluster
- D) Pull-based agents only need outbound access to Git

**Correct: B** (it's the false statement). A GitHub-hosted runner is on the internet and **can't** reach a laptop cluster: its address is loopback or a private IP behind your router. You'd need a self-hosted runner, a cloud cluster, or a pull-based agent. A is true: push-based CI keeps the kubeconfig in its secrets. C is true: Argo CD/Flux run as Pods in the cluster. D is true: the agent connects **out** to Git, which is why GitOps works even for local clusters.

---

**L1.** Explain SAST vs SCA in contrast form, naming one tool for each.

**Model answer:** *"SAST (Static Application Security Testing) scans **your own source code**, without running it, for insecure patterns such as SQL injection or hard-coded passwords (e.g. CodeQL, Semgrep, Bandit), whereas SCA (Software Composition Analysis) scans the **third-party dependencies** your app uses, including transitive ones, for known vulnerabilities (CVEs) and licence problems (e.g. pip-audit, Dependabot, Snyk)."*

---

**L2.** What is a security gate, and what exactly makes a pipeline step act as one?

**Model answer:** *"A security gate is a pipeline step that decides whether the pipeline may continue, based on a security policy (e.g. no HIGH/CRITICAL vulnerabilities with a fix). It acts as a gate because the check **exits with a non-zero code** when the policy is broken (e.g. `trivy image --severity HIGH,CRITICAL --exit-code 1`): the step fails, jobs that `need` it are skipped, so nothing is pushed or deployed, and with branch protection's required status checks, the PR can't be merged."*
