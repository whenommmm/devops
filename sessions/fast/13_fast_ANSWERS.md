# Answer key · Session 13 · Complete CI/CD & DevSecOps · FAST

⚠️ Only open after answering.

---

**1.** A workflow in the repo `Riya-K/ShopApp` has this step, which fails instantly with `invalid reference format`:
```yaml
- run: docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
```
Which change fixes it?
- A) Lowercase the tag, i.e. the part after the colon
- B) Give the job `packages: write` under `permissions:`
- C) Run `docker/login-action` before the build step
- D) Lowercase the name, i.e. the part before the colon

**Correct: D.** `${{ github.repository }}` is filled in exactly as the repo is written on GitHub, so the image name becomes `ghcr.io/Riya-K/ShopApp:<sha>`. The part before the colon must be lowercase, so Docker rejects the name before it builds anything. Lowercasing it (in bash, `${GITHUB_REPOSITORY,,}`) gives `ghcr.io/riya-k/shopapp:<sha>`, which works. A is the near-miss: it picks the wrong side of the colon. The tag may mix cases, and a commit SHA is already lowercase letters and digits anyway. B: `packages: write` only matters when you push, and this step only builds. A missing permission also shows up as `denied` at push time, not as `invalid reference format`. C: logging in is needed to push, not to build, and it doesn't change the image name.

---

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

**Correct: B.** `kubectl set image` only submits the change. Kubernetes accepts it without checking whether the tag exists, so it exits 0 straight away (`A=0`). `kubectl rollout status` then waits. The new Pod can't download its image, so it never becomes ready, and after 30 seconds the command prints `error: timed out waiting for the condition` and exits 1 (`B=1`). In a pipeline, that is what turns the step red. A is wrong because set image never checks the registry, so it can't "see" the bad tag. C is half right: the old Pods do keep serving, because a rolling update removes them only after new Pods are ready. But rollout status asks whether the new version finished rolling out, and it didn't, so it exits 1. D has both exit codes the wrong way round.

---

**3.** A team's pipeline runs Semgrep on every PR and OWASP ZAP against staging. A HIGH CVE is announced in `werkzeug`, a library Flask pulls in by itself; `requirements.txt` lists only `flask`. Which tool is built to catch it?
- A) pip-audit, as SCA also checks transitive libraries
- B) Semgrep, as SAST reads every file in the repository
- C) ZAP, as DAST attacks the app that runs werkzeug
- D) None, as werkzeug isn't listed in requirements.txt

**Correct: A.** `werkzeug` is a transitive dependency: it gets installed because Flask needs it, even though you never listed it. SCA (Software Composition Analysis) tools like pip-audit work out the full list of libraries, transitive ones included, and compare each version with the databases of known CVEs, so this is exactly their job. B: Semgrep is SAST (Static Application Security Testing). It reads your own source code for dangerous patterns, but werkzeug's code isn't in your repository (pip installs it), and SAST doesn't match library versions against CVEs. C: ZAP is DAST (Dynamic Application Security Testing). It is black-box: it attacks the running app from outside and only sees the responses, not which library versions are inside. D is the common misconception, because checking transitive libraries is the whole point of SCA.

---

**4.** Trivy finds exactly two problems in `shop:3f9c2a1`: one HIGH with Status `affected`, and one MEDIUM with Status `fixed`. Each command below is a GitHub Actions `run:` step with no `shell:` key. Which step turns red?
- A) `trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 shop:3f9c2a1`
- B) `trivy image --severity HIGH,CRITICAL --exit-code 1 shop:3f9c2a1 | tee scan.txt`
- C) `trivy image --severity MEDIUM,HIGH --ignore-unfixed --exit-code 1 shop:3f9c2a1`
- D) `trivy image --severity MEDIUM,HIGH,CRITICAL --ignore-unfixed shop:3f9c2a1`

**Correct: C.** Apply the filters in order. `--severity MEDIUM,HIGH` keeps both findings. `--ignore-unfixed` then drops the HIGH, because its Status is `affected` (no fix exists yet). The MEDIUM, with Status `fixed`, is still left, so `--exit-code 1` makes Trivy exit 1 and the step fails. A is the tempting "standard gate" command, but here it finds nothing: `--severity HIGH,CRITICAL` drops the MEDIUM and `--ignore-unfixed` drops the HIGH, so Trivy exits 0. B: Trivy itself does exit 1 (the HIGH stays, because nothing ignores unfixed findings). But the pipe reports `tee`'s exit code, which is 0, and GitHub's default `run:` shell ignores failures inside a pipe, so the step is green. D keeps the MEDIUM, but there is no `--exit-code 1`, so Trivy only prints the report and exits 0.

---

**5.** A developer pushes a real AWS key starting with `AKIA` to a public GitHub repo. Twenty minutes later they push a commit that deletes the line, yet the CI secret scan, which reads Git history, still fails. Which action must come first?
- A) Rewrite Git history so that no commit contains the key
- B) Revoke the key in AWS and create a replacement key
- C) Check the AWS logs for any use of the leaked key
- D) Move the key into a GitHub Actions secret instead

**Correct: B.** The key sat on a public repo for twenty minutes, and bots find leaked keys within minutes, so someone may already have a copy. Only revoking it (and rotating to a new key) makes every copy useless, which is why it comes first. A is a real step, but it comes second: cleaning history only hides the key from people who haven't copied it yet. C is also a real step, but you do it after revoking, so an attacker can't keep using the key while you look through the logs. D is the trap: the leaked key is compromised forever, so moving it into a secret doesn't protect anything. Only the new key goes into a GitHub Actions secret.

---
