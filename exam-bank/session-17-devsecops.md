# Labs · Session 17 · DevSecOps

> DevSecOps = adding security checks into every stage of the CI/CD (Continuous Integration / Continuous Delivery) pipeline ("shift left" = check early). All workflows below live in `.github/workflows/` of a repo that has a `Dockerfile` and a Node.js app with `package-lock.json`.

## Lab 1 · Pipeline that builds a Docker image and pushes it to a registry using repository secrets
**Answer (what to write in the exam):**
```bash
# 1) Docker Hub → Account settings → Personal access tokens → create token (Read & Write)
# 2) GitHub repo → Settings → Secrets and variables → Actions → New repository secret
gh secret set DOCKERHUB_USERNAME          # (CLI way) paste your Docker Hub username
gh secret set DOCKERHUB_TOKEN             # paste the access token (never your password)
# 3) add .github/workflows/docker-publish.yml (below), then:
git add .github/workflows/docker-publish.yml && git commit -m "ci: build and push image" && git push
gh run watch
docker pull <your-dockerhub-user>/web:<commit-sha>    # proves the image is in the registry
```
`.github/workflows/docker-publish.yml`:
```yaml
name: Docker publish

on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  build-push:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/web:${{ github.sha }}
```
**What each line does:**
- `${{ secrets.DOCKERHUB_TOKEN }}`: reads an encrypted repository secret at run time; GitHub masks it as `***` in logs, and it is never in the YAML or Git history.
- `permissions: contents: read`: least privilege; the built-in `GITHUB_TOKEN` can only read the repo.
- `docker/login-action@v3`: runs `docker login` with the secrets.
- `docker/build-push-action@v6`: builds from the `Dockerfile` in `context: .`; `push: true` uploads it; `tags:` = image name, tagged with the commit SHA so every build is traceable.
- GHCR (GitHub Container Registry) instead: `registry: ghcr.io`, `username: ${{ github.actor }}`, `password: ${{ secrets.GITHUB_TOKEN }}`, add `packages: write` under `permissions:`, tag `ghcr.io/<owner>/web:<sha>`.

**Verify:** run is ✅, log shows `Login Succeeded` and `pushing ... done` with the username shown as `***`; Docker Hub → repository `web` shows a tag equal to the commit SHA.

**🧠 In simple words:** On every push GitHub builds the Docker image and uploads it to Docker Hub. The login details are stored as encrypted repository secrets and injected only while the job runs, so no password is ever written in the code. In short: secrets in Settings → `${{ secrets.X }}` in YAML → login → build → push.

## Lab 2 · Add SAST, dependency scanning and secret scanning; make the security check block later stages
**Answer (what to write in the exam):**
```bash
# 1) add .github/workflows/secure-ci.yml (below) and push
git add . && git commit -m "ci: add security job" && git push
gh run watch                                  # security ✅ → build runs
# 2) GitHub → Settings → Branches → Add branch protection rule (or Rules → Rulesets) for "main":
#    tick "Require a pull request before merging" + "Require status checks to pass",
#    search and add the check "security" → Save
# 3) test it: open a PR that adds a vulnerable package (e.g. npm install lodash@4.17.15)
gh pr create --fill
gh pr checks                                  # security ❌ fail, build - skipped, merge blocked
```
`.github/workflows/secure-ci.yml`:
```yaml
name: Secure CI

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

permissions:
  contents: read

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code (full history for secret scan)
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: SAST with Semgrep
        run: docker run --rm -v "$PWD:/src" semgrep/semgrep semgrep scan --config auto --error

      - name: Dependency scan with npm audit
        run: npm audit --audit-level=high

      - name: Secret scan with gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  build:
    needs: security
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Build image
        run: docker build -t web:${{ github.sha }} .
```
**What each line does:**
- SAST (Static Application Security Testing) = Semgrep reads the source code for insecure patterns without running it; `--error` makes it exit 1 if it finds anything. (Alternatives: CodeQL, or `bandit -r .` for Python.)
- Dependency scan = `npm audit --audit-level=high` checks `package-lock.json` against known CVEs (Common Vulnerabilities and Exposures) and fails on HIGH or CRITICAL. (Python: `pip-audit -r requirements.txt`; any language: `trivy fs --exit-code 1 .`)
- Secret scan = gitleaks searches the code and Git history (`fetch-depth: 0`) for passwords, keys and tokens.
- `needs: security`: `build` waits for `security` and is **skipped** if it fails, so nothing later runs.
- Branch protection "Require status checks to pass" + check `security`: GitHub refuses to merge a PR into main until that job is green.

**Verify:** a PR with `lodash@4.17.15` (or a committed fake token) shows `security` ❌, `build` skipped, and the merge button says "Required statuses must pass"; after removing it the checks are ✅ and merge is allowed.

**🧠 In simple words:** The security job checks three things on every push and PR: your own code (SAST), the libraries you use (dependency scan), and leaked passwords or keys (secret scan). Because the build job `needs` it and branch protection requires it, a failed check stops the pipeline and blocks the merge. In short: security job first, everything else `needs:` it, and main only accepts green PRs.

## Lab 3 · Container image scanning with a gate that blocks deployment above a severity threshold
**Answer (what to write in the exam):**
```bash
# Threshold agreed: no HIGH or CRITICAL vulnerability that has a fix available.
# Same check by hand on a lab machine (exit code 1 = gate closed):
docker build -t web:test .
trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed web:test
echo $?                                        # 1 = vulnerabilities found, 0 = clean
# In CI: add .github/workflows/image-gate.yml (below), push, then:
gh run watch                                   # scan ❌ → push + deploy skipped
```
`.github/workflows/image-gate.yml`:
```yaml
name: Image scan gate

on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  build-scan-push:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Build image (local only)
        run: docker build -t web:${{ github.sha }} .

      - name: Trivy scan (gate)
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: web:${{ github.sha }}
          severity: CRITICAL,HIGH
          ignore-unfixed: true
          exit-code: '1'

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Push image (only if scan passed)
        run: |
          docker tag web:${{ github.sha }} ${{ secrets.DOCKERHUB_USERNAME }}/web:${{ github.sha }}
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/web:${{ github.sha }}

  deploy:
    needs: build-scan-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy
        run: echo "Deploying web:${{ github.sha }} (e.g. helm upgrade / kubectl set image)"
```
**What each line does:**
- `trivy image`: lists the OS and library packages inside the image and matches them against known CVEs.
- `severity: CRITICAL,HIGH` (`--severity`): the agreed threshold; only these levels count.
- `exit-code: '1'` (`--exit-code 1`): fail the step if any such CVE is found (default is 0 = report only, never blocks).
- `ignore-unfixed: true` (`--ignore-unfixed`): skip CVEs with no patch yet, so the gate is actionable.
- Scan runs BEFORE login/push, so a vulnerable image never reaches the registry; `deploy` has `needs: build-scan-push`, so it is skipped when the scan fails.
- `environment: production`: optional extra gate; add "Required reviewers" to it in Settings → Environments.
- Pin actions to a fixed version (or commit SHA) so a moved tag cannot change what runs.

**Verify:** with an old base image (e.g. `FROM node:14`) the log shows a table of HIGH/CRITICAL CVEs, the Trivy step is ❌, push and `deploy` show "skipped"; switch to a current base (e.g. `FROM node:22-alpine`) → `Total: 0 (HIGH: 0, CRITICAL: 0)` → push and deploy ✅.

**🧠 In simple words:** Trivy looks inside the built image for software with known security holes. If it finds any HIGH or CRITICAL ones that can be fixed, it exits with code 1, which fails the job, and because deploy `needs` that job, nothing is pushed or deployed. In short: build → scan with a severity threshold → only a clean image is pushed and deployed.
