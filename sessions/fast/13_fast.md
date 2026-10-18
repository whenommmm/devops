# Session 13 · Complete CI/CD & DevSecOps · FAST (10-min read)
*How a pipeline builds your image, stores it in a registry, deploys it to Kubernetes, and blocks anything unsafe.*

## 1 · Build the image in the pipeline, tag it, push it to a registry
An image built by hand on a laptop can't be traced back to its code. So the CI/CD (Continuous Integration / Continuous Delivery or Deployment) pipeline builds it on a runner (the fresh machine GitHub gives each job) from the exact commit that passed the tests, then pushes it to a container registry, a server that stores images for Kubernetes to pull.
- Every Git commit has a unique ID, the commit SHA (Secure Hash Algorithm), shortened to 7 characters like `3f9c2a1`; in GitHub Actions it is `${{ github.sha }}`.
- Tagging the image `shop:3f9c2a1` gives traceability: read the tag on a running Pod, run `git show 3f9c2a1`, and you see the exact code.
- `latest` is a mutable (changeable) tag that every push moves to a new image, so it never tells you what is inside; deploy by the SHA tag instead.
- GHCR (GitHub Container Registry) names look like `ghcr.io/OWNER/IMAGE:TAG`; Docker Hub (`docker.io/USER/IMAGE:TAG`) is used when no registry is written.
- To push to GHCR, the job logs in (with `docker/login-action`) using `GITHUB_TOKEN`, a temporary password made for each run, and needs `packages: write` under `permissions:`, or the push is denied.
- The name before the `:` must be lowercase, while the tag after it may mix cases. `${{ github.repository }}` keeps GitHub's capitals, so lowercase it with bash's `${GITHUB_REPOSITORY,,}`.

⚠️ **Exam trap:** Re-pushing `latest` leaves the Deployment file unchanged, so `kubectl apply` prints `unchanged` and nothing redeploys, whereas a new SHA tag changes the Pod template (the Pod description inside it) and starts a rolling update.

## 2 · Deploying to Kubernetes from the pipeline
Deploying by hand invites wrong tags and unnoticed failures, so the pipeline deploys the new SHA itself and then verifies the rollout. To run `kubectl`, the runner needs a kubeconfig (the file saying which cluster to reach and who you are), stored as a secret.
```bash
kubectl set image deployment/shop shop=ghcr.io/riya-k/shop:${{ github.sha }}
kubectl rollout status deployment/shop --timeout=120s
```
- `kubectl set image` gives the container named `shop` (left of `=`) the new image, which starts a rolling update (new Pods gradually replace old ones).
- Like `kubectl apply`, it only submits the change and exits 0 at once, even if every new Pod will fail.
- `kubectl rollout status` waits up to `--timeout`, then exits 0 (`successfully rolled out`) or 1 (`error: timed out waiting for the condition`). That exit code turns the step green or red.
- It exits 1 even though the old Pods still serve users (a rolling update removes them only after new ones are ready).

⚠️ **Exam trap:** A green `kubectl set image` only proves the request was accepted, whereas `kubectl rollout status` proves the new Pods became ready.

## 3 · DevSecOps and shift-left: SAST, SCA and DAST
DevSecOps (Development + Security + Operations) makes security everyone's job by automating checks in every pipeline stage, instead of one audit at the end. Shift-left means moving checks earlier in the pipeline, because a problem fixed early is far cheaper than one fixed in production.
- Each known vulnerability (a weakness an attacker could use) gets a CVE (Common Vulnerabilities and Exposures) ID and a severity: LOW, MEDIUM, HIGH or CRITICAL.
- SAST (Static Application Security Testing) reads your own source code without running it, so it is white-box (it sees inside). It runs at PR (pull request) time and points to the file and line of flaws like SQL (Structured Query Language) injection, where user input changes a database query; its tools include CodeQL and Semgrep.
- SCA (Software Composition Analysis) checks the libraries you depend on for known CVEs, including transitive dependencies (libraries your libraries pull in, like `werkzeug` under Flask). It runs at PR and build time, with tools like pip-audit, Dependabot and Snyk.
- DAST (Dynamic Application Security Testing) attacks the running app from outside without seeing the code, so it is black-box. It runs against staging (a test copy of production) and finds runtime flaws like XSS (Cross-Site Scripting: injected scripts that run in visitors' browsers), but can't name the faulty line. Its tool is ZAP (Zed Attack Proxy) from OWASP (Open Worldwide Application Security Project).

⚠️ **Exam trap:** Static means not running (SAST), whereas dynamic means running (DAST). SAST scans your code, whereas SCA scans other people's libraries.

## 4 · Secret scanning
Secret scanning looks for passwords, API (Application Programming Interface) keys and tokens in files and Git history, because bots find leaked keys within minutes. Tools like gitleaks match known patterns (GitHub tokens start with `ghp_`, AWS (Amazon Web Services) keys with `AKIA`) and exit 1 on a finding, so the step fails.
- Run it as a pre-commit hook (a script Git runs before each commit; exit 1 cancels the commit) and also in CI, because cloning a repo does not copy its hooks.
- Deleting a leaked key in a new commit is not enough, because the old commit keeps it.
- The order is: revoke (cancel) and rotate the key first, then clean code and history, check for misuse, and store the new key in a GitHub Actions secret.

⚠️ **Exam trap:** Cleaning history only hides a key, whereas revoking makes it useless, so revoke first.

## 5 · Image scanning with Trivy, and security gates
Clean code does not mean a clean image, because the base image adds dozens of OS (operating system) packages. Trivy compares every OS and language package in the image with a vulnerability database. A security gate is a step that decides whether the pipeline may continue: a check (the scan), a policy ("no fixable HIGH or CRITICAL") and enforcement (a non-zero exit code).
```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ghcr.io/riya-k/shop:3f9c2a1
```
- `--severity HIGH,CRITICAL` only filters the report; on its own it never fails anything.
- `--exit-code 1` makes Trivy exit 1 if anything is left after the filters. This is the gate: without it, Trivy exits 0 even with CRITICAL findings.
- `--ignore-unfixed` skips findings with Status `affected` (no fix exists yet) and keeps Status `fixed` (a newer version fixes it).
- When the gate exits 1, the step fails, any job that `needs:` it (waits for it to succeed) is skipped (no push, no deploy), and branch protection (required checks on `main`) blocks the PR's merge.
- A smaller base image (`alpine`, `-slim`) has fewer packages, so fewer CVEs.

⚠️ **Exam trap:** In `trivy … --exit-code 1 app | tee report.txt`, the pipe reports only the last command's exit code (`tee` gives 0), and GitHub's default `run:` shell ignores failures inside a pipe, so the gate is off; `shell: bash` adds `-o pipefail` and fixes it.

## ⚡ Must-remember
| Term / command | Meaning |
|---|---|
| `${{ github.sha }}` | Commit ID used as the tag; deploy by it, not `latest` |
| `packages: write` | Lets `GITHUB_TOKEN` push to GHCR |
| `kubectl rollout status --timeout=120s` | Waits; 0 = rolled out, 1 = timed out |
| SAST / SCA / DAST | Your code, white-box / your libraries / running app, black-box |
| `--severity HIGH,CRITICAL` | Filters the report only |
| `--exit-code 1` | Turns the scan into a gate |
| `--ignore-unfixed` | Skips Status `affected` (no fix yet) |
| `\| tee` after a scanner | Hides its exit code (last command wins) |
| Leaked secret | Revoke and rotate first, then clean history |

## 📝 Quiz (5 MCQs; answer in chat as "1 B, 2 D, …")

**1.** A workflow in the repo `Riya-K/ShopApp` has this step, which fails instantly with `invalid reference format`:
```yaml
- run: docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
```
Which change fixes it?
- A) Lowercase the tag, i.e. the part after the colon
- B) Give the job `packages: write` under `permissions:`
- C) Run `docker/login-action` before the build step
- D) Lowercase the name, i.e. the part before the colon

**2.** An engineer rehearses a deploy step in a terminal. The tag `a1b2c3d` was never pushed to the registry.
```bash
kubectl set image deployment/shop shop=ghcr.io/riya-k/shop:a1b2c3d
echo "A=$?"
kubectl rollout status deployment/shop --timeout=30s
echo "B=$?"
```
What is printed?
- A) `A=1` and `B=1`, as both commands see the bad tag
- B) `A=0` and `B=1`, as only rollout status waits
- C) `A=0` and `B=0`, as the old Pods keep serving
- D) `A=1` and `B=0`, as set image rejects the tag

**3.** A team's pipeline runs Semgrep on every PR and OWASP ZAP against staging. A HIGH CVE is announced in `werkzeug`, a library Flask pulls in by itself; `requirements.txt` lists only `flask`. Which tool is built to catch it?
- A) pip-audit, as SCA also checks transitive libraries
- B) Semgrep, as SAST reads every file in the repository
- C) ZAP, as DAST attacks the app that runs werkzeug
- D) None, as werkzeug isn't listed in requirements.txt

**4.** Trivy finds exactly two problems in `shop:3f9c2a1`: one HIGH with Status `affected`, and one MEDIUM with Status `fixed`. Each command below is a GitHub Actions `run:` step with no `shell:` key. Which step turns red?
- A) `trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 shop:3f9c2a1`
- B) `trivy image --severity HIGH,CRITICAL --exit-code 1 shop:3f9c2a1 | tee scan.txt`
- C) `trivy image --severity MEDIUM,HIGH --ignore-unfixed --exit-code 1 shop:3f9c2a1`
- D) `trivy image --severity MEDIUM,HIGH,CRITICAL --ignore-unfixed shop:3f9c2a1`

**5.** A developer pushes a real AWS key starting with `AKIA` to a public GitHub repo. Twenty minutes later they push a commit that deletes the line, yet the CI secret scan, which reads Git history, still fails. Which action must come first?
- A) Rewrite Git history so that no commit contains the key
- B) Revoke the key in AWS and create a replacement key
- C) Check the AWS logs for any use of the leaked key
- D) Move the key into a GitHub Actions secret instead
