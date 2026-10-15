# Session 12 · CI/CD and GitHub Actions (Oct 16, Session 2)

**Time:** ~90–100 min · **Concepts:** 74 → 77 · **Answers:** `MY_ANSWERS.md` or say "CI/CD questions" · **Answer key:** `12_cicd-github-actions_ANSWERS.md`

**Before you start:** today uses your real GitHub account through **`gh`**, the **GitHub CLI** (Command-Line Interface).
```bash
gh auth status                     # expect "✓ Logged in…" and Token scopes incl. 'workflow' (lets you push .github/workflows/)
git --version                      # git 2.x
python3 --version                  # Python 3.9 or newer
```
> ⚠️ Never delete the **`minikube`** or **`localstack`** containers. Today doesn't need Docker or Kubernetes. Text after `#` in commands is a note for you: don't type it (your zsh would pass it on as arguments), or run `setopt interactivecomments` first.

## 🔁 Warm-up (Helm + the weak list)
**W1.** Release `web` is at revision 5. You run `helm rollback web 3`, then `helm history web`. What is the newest line?
- A) Revision 3: deployed, "Rollback to 3"
- B) Revision 4: deployed, "Rollback to 3"
- C) Revision 5: superseded, "Rollback to 3"
- D) Revision 6: deployed, "Rollback to 3"

**W2.** A chart's values.yaml has `replicaCount: 1`. You installed with `helm install shop ./chart -f values-prod.yaml` (which sets `replicaCount: 4`). A week later you run `helm upgrade shop ./chart --set image.tag=2.0`, with no `-f`. How many replicas run after the upgrade?
- A) 4, because Helm keeps the earlier -f file
- B) 1, because only this run's values apply
- C) 4, since --set changes only the named key
- D) 0, because no replicaCount was supplied

**W3.** `app.log` exists but contains no line with `ERROR`. What's the **last** line printed?
```bash
check_log() {
  echo "checking app.log"
  grep -q "ERROR" app.log
}
check_log
echo $?
```
- A) `0`, because the echo inside succeeded
- B) `0`, because the function has no return
- C) `1`, from grep finding no match
- D) Nothing, since -q makes grep silent

## Concept 74 · CI, CD, and the pipeline
**Words you'll learn:** integration, CI (Continuous Integration), Continuous Delivery, Continuous Deployment, pipeline, stage, trigger, unit test, artifact, smoke test, fail fast, environment (dev / staging / production), promotion, deployment strategy, recreate, rolling update, blue-green, canary, Jenkins, Jenkinsfile, agent, GitLab CI.

**Problem:** five developers merge three weeks of separate work at once: conflicts everywhere, a broken build, and nobody knows whose change did it (**integration hell**; to **integrate** = combine everyone's changes).
- **CI (Continuous Integration)** = merge small changes into main often, and **every push or PR automatically triggers a build and automated tests** (a **unit test** = a small piece of code that checks one small part, e.g. "does `GET /` return 200?").
  - ⚠️ **CI DETECTS and REPORTS problems; it never fixes code.** A developer reads the red ❌ log and fixes it.
  - **Analogy:** a quality-control checkpoint on a factory conveyor belt: faulty items are flagged and sent back, but the inspector repairs nothing.

| "CD" can mean… | **Continuous Delivery** | **Continuous Deployment** |
|---|---|---|
| Every passing change is… | **ready to release** | **released** automatically |
| Last step (to production) | **a human approves** (⚠️ the only difference) | **automatic**, no human |

- A **pipeline** = an automated sequence of **stages** that every change passes through (like an assembly line), started by a **trigger** (an event such as a push or PR):
  - `SOURCE (git push) → BUILD (pip install) → TEST (pytest; lint = style checks) → PACKAGE (docker build) → SCAN (security) → PUBLISH (docker push) → DEPLOY (helm upgrade) → VERIFY`
  - A **smoke test** = a quick "is it alive?" check, e.g. `curl -f …/health`.
- **Fail fast:** cheap checks first. **Exit codes decide** (shell scripting): 0 = ✅; non-zero = ❌ and every later stage is skipped (like `set -e`). `pytest` exits 0 (all pass), 1 (a test failed), 5 (no tests found).
- ⚠️ **CI trusts the exit code, not the text:** a script that prints "3 tests FAILED" but ends with `exit 0` is **green**.
- **Artifact** = a file a pipeline produces and keeps (test report, image, chart). **Build once, deploy many:** the *same* artifact moves (**promotion**) through each **environment**, a separate copy of the app: **dev** (try things) → **staging** (dress rehearsal) → **production** (real users). A **deployment strategy** = how v1 is replaced by v2:

| Strategy | How | Downtime? | Rollback | Extra resources / risk |
|---|---|---|---|---|
| **Recreate** | stop all v1, then start v2 | **Yes** | redeploy the old one (slow) | none / high |
| **Rolling** (K8s default) | replace a few at a time (`maxSurge`, `maxUnavailable`) | No | `kubectl rollout undo`, `helm rollback` | a little / medium |
| **Blue-green** | two full environments, then ONE switch (e.g. Service selector `blue` → `green`) | No | **instant switch back** | **double** / low |
| **Canary** | 5% of users get v2; watch metrics; then 25 → 50 → 100% | No | send the 5% back to v1 | a little / **lowest** |

(Coal miners' canaries fell ill from gas before the miners did.) **Tools:**
- **GitHub Actions** (`.github/workflows/*.yml` in YAML; jobs run on a **runner**)
- **Jenkins** (a `Jenkinsfile` in **Groovy**, a programming language; jobs run on an **agent**; a server you maintain, 1000s of plugins)
- **GitLab CI** (`.gitlab-ci.yml`). **Argo CD** is CD-only for Kubernetes (Session 14).

### 🛠 Do (~5 min): a pipeline is just exit codes
```bash
mkdir -p ~/devops-practice/cicd && cd ~/devops-practice/cicd   # practice folder
cat > mini-pipeline.sh <<'EOF'
#!/bin/bash
set -e                                   # stop at the first failing command
echo "== BUILD";  python3 -c "print('building...')"
echo "== TEST";   python3 -c "import sys; print('1 test failed'); sys.exit(1)"
echo "== DEPLOY"; echo "deploying to production"
EOF
bash mini-pipeline.sh; echo "pipeline exit code: $?"           # run it, then print its exit code
```
(`cat > file <<'EOF' … EOF` writes the lines in between into the file.) Expect `1 test failed` and `pipeline exit code: 1`: **DEPLOY never prints**. Change `sys.exit(1)` to `sys.exit(0)` and rerun: `deploying to production` now prints with exit code 0, although the text said "failed".

### ✅ Check 74
**74.1** A team's pipeline builds, tests, and deploys every passing commit to **staging** automatically. Production deploys happen only when the release manager clicks **Approve**. Which practice is this?
- A) Continuous Delivery, with a manual prod gate
- B) Continuous Deployment, as staging is automatic
- C) Continuous Integration only, no CD at all
- D) Continuous Deployment with a canary release

**74.2** A test script prints `3 tests FAILED` but always ends with `exit 0`. What does the CI pipeline do?
- A) Marks the step failed, because it reads FAILED
- B) Marks it passed and moves on to later stages
- C) Pauses and asks a developer to read the output
- D) Fixes the failing tests, then reruns the stage

**74.3** Which strategy keeps two full environments and switches all traffic at once, so rollback is instant?
- A) Canary
- B) Rolling update
- C) Blue-green
- D) Recreate

## Concept 75 · GitHub Actions: the parts and how they fit
**Words you'll learn:** GitHub Actions, workflow, `.github/workflows/`, event (trigger), `push`, `pull_request`, `workflow_dispatch`, `schedule`, cron (recap), UTC, job, step, action, GitHub Marketplace, `actions/checkout`, `actions/setup-python`, runner, GitHub-hosted runner, `ubuntu-latest`, self-hosted runner, workflow run.

**GitHub Actions** = GitHub's built-in CI/CD service: it runs your automation on GitHub's computers when something happens in your repo (free for public repos).
```
WORKFLOW = one YAML file in .github/workflows/ (any other folder: silently ignored); on: push = the EVENT (WHEN)
 └─ JOBS (e.g. test, lint): run IN PARALLEL, each on its own fresh RUNNER (runs-on: ubuntu-latest = WHERE)
     └─ STEPS, in order, on that same machine:  uses: actions/checkout@v7 (an ACTION)  or  run: pytest (shell)
```
**Analogy: a restaurant kitchen preparing a banquet.** The workflow is the plan, the event is the booking arriving, jobs are separate cooking stations (each with its own clean counter, working at once), steps are one station's instructions in order, actions are ready-made sauces from a supplier, and the runner is the counter and cook.
- **Events** (`on:`): `push` (add `branches: [main]` as a filter); `pull_request` (a PR is opened **or new commits are pushed to its branch**; it tests the merged result); `workflow_dispatch` (the manual **Run workflow** button, or `gh workflow run`); `schedule` (cron).
  - **Cron** = `minute hour day-of-month month day-of-week`, 24-hour clock, 0 = Sunday (`1-5` = Mon–Fri).
  - ⚠️ It's in **UTC** (Coordinated Universal Time, the world's reference clock) unless you set `timezone:`; **IST** (Indian Standard Time) = UTC + 5:30, so `"30 2 * * 1-5"` = 08:00 IST. Default branch only, at most every 5 minutes, may start late.
- **Job** = steps on **one runner**. Jobs run **in parallel** unless `needs:` makes one wait. **Each job gets a brand-new machine**, so job A's files don't exist in job B. **Step** = one task: `run:` (shell commands) **or** `uses:` (an action). Steps share the runner's files; after a failure, the rest are skipped.
- **Action** = a reusable, packaged step someone already wrote, from the **GitHub Marketplace** (GitHub's catalogue), written `owner/repo@version` (older tutorials show `@v3`/`@v4`).
  - **`actions/checkout`** copies (clones) your repo's code onto the empty runner, so it's almost always step 1; `actions/setup-python` installs Python; `upload-artifact`, `download-artifact` and `cache` come in Concept 77.
- **Runner** = the machine that runs a job.
  - **GitHub-hosted:** a fresh **VM** (virtual machine) per job, deleted afterwards, picked by a label like `ubuntu-latest` (or `windows-latest`, `macos-latest`).
  - **Self-hosted:** your own machine with GitHub's runner program (`runs-on: self-hosted`); not wiped between jobs, can reach your private network.
- **On `git push`:** match workflows' `on:` → a fresh runner per job → steps run, each exit code checked → logs, ✅/❌ and artifacts appear in the Actions tab and on the PR → VM destroyed. One execution = a **workflow run**.

### 🛠 Do (~5 min): look at real runs (read-only)
```bash
gh run list --repo actions/checkout --limit 5   # the newest 5 runs in GitHub's own checkout repo
gh workflow list --repo actions/checkout        # its workflows (YAML files): name, active, ID
```
Expect columns `STATUS TITLE WORKFLOW BRANCH EVENT ID ELAPSED AGE`; one push often starts several workflows, one row each. **Translate these cron lines** (answers in the answer key under "Do 75"): 1. `0 18 * * *` · 2. `30 3 * * 1` · 3. `*/15 * * * *`

### ✅ Check 75
**75.1** A workflow has jobs `lint` and `test`, with no `needs:`. `lint` creates `report.txt`; `test` runs `cat report.txt`. What happens?
- A) test waits for lint, then prints the report
- B) test fails: it runs on a separate fresh VM
- C) test prints the report, as jobs share a disk
- D) GitHub runs lint again inside the test job

**75.2** Which schedule runs a workflow at **9:00 AM IST** (UTC + 5:30), Monday to Friday? No `timezone:` key is set.
- A) `cron: "0 9 * * 1-5"`
- B) `cron: "30 3 * * 0-4"`
- C) `cron: "0 21 * * 1-5"`
- D) `cron: "30 3 * * 1-5"`

**75.3** (one line) What does `actions/checkout` do, and why is it almost always the first step?

## Concept 76 · Workflow YAML in depth
**Words you'll learn:** `name`, `on`, branch filter, `jobs`, job ID, `runs-on`, `steps`, `uses`, `run`, `with`, `env`, `needs`, `if`, `strategy` / `matrix`, `fail-fast`, expression (`${{ }}`), context (`github`, `matrix`, `env`, `secrets`, `runner`), `github.sha`, `github.ref`, status functions (`success()`, `failure()`, `always()`), `run: |`, default shell, actionlint.

This is the workflow you'll push in Concept 77; the comments explain each key.
```yaml
name: CI                                   # name shown in the Actions tab
on:                                        # the triggers
  push:
    branches: [main]                       # branch filter: pushes to main only
  pull_request:
    branches: [main]                       # PRs INTO main
  workflow_dispatch:                       # the manual "Run workflow" button
jobs:
  test:                                    # job ID (your choice; needs: refers to it)
    runs-on: ubuntu-latest                 # REQUIRED: a fresh GitHub-hosted Ubuntu VM
    timeout-minutes: 10                    # stop the job if it hangs
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]   # 2 parallel copies: test (3.11), test (3.12)
    steps:
      - name: Get the code                 # optional label in the log
        uses: actions/checkout@v7          # uses = run an action
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v7
        with:                              # with = the action's inputs
          python-version: ${{ matrix.python-version }}   # ${{ }} = an expression
          cache: pip                       # reuse downloaded packages next time
      - name: Install dependencies
        run: |                             # run = shell; | = several lines, one script
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      - name: Run tests                    # --junitxml also writes a report file (the artifact)
        run: pytest -v --junitxml=reports/junit-${{ matrix.python-version }}.xml
      - name: Upload test report
        if: always()                       # upload even when the tests failed
        uses: actions/upload-artifact@v7
        with:
          name: test-report-${{ matrix.python-version }}   # unique name per matrix copy
          path: reports/
          retention-days: 7
  summary:
    needs: test                            # only after BOTH test copies pass; else skipped
    runs-on: ubuntu-latest                 # a different, brand-new VM
    steps:
      - name: Download all test reports    # files don't carry over between jobs
        uses: actions/download-artifact@v8
        with:
          path: all-reports
      - name: Show what was downloaded
        run: ls -R all-reports             # -R = list recursively
```
- Also: a top-level `env:` sets variables for every job; `if: github.ref == 'refs/heads/main'` on a job runs it only for main; **`${{ github.sha }}`** = the full 40-character commit ID (a unique image tag).
  - **JUnit XML** = a standard test-report format (**XML** = eXtensible Markup Language).
  - ⚠️ **One step = `uses` OR `run`, never both.** To run a command after an action, add a second step.
- An **expression** `${{ … }}` is **filled in by GitHub BEFORE the step runs**. **Analogy:** a mail-merge letter, whose blanks are filled before anyone reads it.
  - A **context** is a group of values: `github` (`sha`, `ref` = `refs/heads/main`, `ref_name` = `main`, `event_name`, `actor`), `matrix`, `env`, `vars`, `secrets`, `runner` (`runner.os`), `needs`, `steps`.
  - **`if:` status functions:** `success()` = all earlier passed (**the default**, so steps after a failure are skipped); `failure()` = something failed (send an alert); `always()` = run anyway (upload reports); `cancelled()`.
- ⚠️ **`${{ env.X }}` vs `$X` in `run:`:** GitHub pastes `${{ env.X }}` in **before bash starts**; `$X` is expanded **by bash** with the shell quote rules. So `echo '$X'` prints a literal `$X`, but `echo '${{ env.X }}'` prints the value.
- **Matrix:** one job per **combination** (2 versions × 2 `os` values with `runs-on: ${{ matrix.os }}` = 4 jobs). `fail-fast: true` (default) cancels the other copies when one fails.
  - ⚠️ **Quote versions:** YAML reads `3.10` as the number 3.1.
- **`needs:`** waits for **all** matrix copies (`needs: [test, lint]` = both); dependants are skipped on failure unless `if: always()`.
- **Pipe trap:** a Linux `run:` without `shell:` runs as `bash -e {0}` (`-e` = `set -e`) but **without `pipefail`**, so `pytest | tee out.txt` **passes when tests fail** (a pipe's exit code is the last command's, `tee`). Fix: `shell: bash` (adds `-o pipefail`: any failing part fails the pipe).
- **Common bugs:** a wrong folder (`.github/workflow/`, `github/workflows/`) or a branch filter that doesn't match (`[master]` when you push `main`) → **nothing happens**, no run appears.
  - Bad indentation/tabs, `on: pushh`, no `runs-on`, `uses` + `run` in one step, `needs:` a missing job, or steps without `-` → **"Invalid workflow file"** with a line number, and no jobs run.
  - Unquoted `3.10` → Python 3.1, setup fails.

### 🛠 Do (~10 min)
`gh api` calls GitHub's API as you, `--jq .content` keeps one field of the **JSON** (JavaScript Object Notation) reply, and `base64 -d` decodes it: read GitHub's starter workflow (`name: Python application`, a `build` job) and **find** its trigger, job ID, a `uses` step, a `with:` input, and a `run: |`.

**Bug hunt:** in `broken-ci.yml`, add four bugs: under `push:` change `[main]` to `[master]`; delete the `test` job's `runs-on:` line; add `run: python --version` under `uses: actions/setup-python@v7`; change `needs: test` to `needs: tset`.
```bash
cd ~/devops-practice/cicd && code ci.yml   # paste the workflow above and save (Concept 77 pushes it)
gh api repos/actions/starter-workflows/contents/ci/python-app.yml --jq .content | base64 -d   # GitHub's starter workflow
cp ci.yml broken-ci.yml && code broken-ci.yml    # make the four bugs, save
brew install actionlint                          # (optional) a free checker for workflow files
actionlint ci.yml; echo "exit code: $?"          # the good file: no output, exit code 0
actionlint broken-ci.yml; echo "exit code: $?"   # the broken copy: errors with line numbers, exit code 1
```
Expect errors about the missing `runs-on`, the step mixing `uses` and `run`, and the unknown job `tset`. The `master` bug is valid YAML, so no checker can see it: only a human catches logic bugs.

### ✅ Check 76
**76.1** This step fails validation. Why?
```yaml
      - name: Set up Python
        uses: actions/setup-python@v7
        run: python --version
```
- A) `run` must come before `uses` in a step
- B) `name` is not allowed when `uses` is set
- C) A step has `uses` or `run`, never both
- D) `uses` needs a `with:` block before `run`

**76.2** Job `test` has a matrix with `python-version: ["3.11", "3.12"]` and `os: [ubuntu-latest, windows-latest]`, plus `runs-on: ${{ matrix.os }}`. Job `build` has `needs: test` and no matrix. If everything passes, how many job runs happen in total?
- A) 5: four matrix copies, then the build job
- B) 3: one per Python version, then the build job
- C) 4: matrix copies only; needs: adds no new job
- D) 6: the build job runs once per test copy

**76.3** (one line) Why does `python-version: [3.10, 3.11]` (no quotes) in a matrix cause trouble?

## Concept 77 · Secrets, artifacts, and a real build & test pipeline
**Words you'll learn:** secret, repository secret, environment secret, `secrets` context, masking, `GITHUB_TOKEN`, `permissions`, configuration variable (`vars`), artifact (recap), `actions/upload-artifact`, `actions/download-artifact`, retention, cache, `actions/cache`, cache key, virtual environment (venv), pytest, test client, JUnit XML report, `gh repo create`, `gh run watch`, `gh run view`, `--log-failed`, `gh run download`, `gh secret set`.

**Problem:** pipelines log in to things (Docker Hub, a cloud). A password typed into the workflow YAML is committed to Git, readable by anyone with repo access (the world, if public), and stays in the history forever. **Analogy:** a hotel's safe-deposit box: the hotel keeps it, your workflow can use what's inside, but nobody can open it to look.
- A **secret** = an encrypted value stored by GitHub (not in your files) that workflows can use but **nobody can read back**.
  - Create it in repo **Settings → Secrets and variables → Actions**, or `gh secret set NAME`; use it via `env:`, e.g. `env: { TOKEN: "${{ secrets.DOCKERHUB_TOKEN }}" }` then `echo "$TOKEN" | docker login -u me --password-stdin`.
- **Masking:** the exact value shows as **`***`** in logs. ⚠️ A changed copy (base64, reversed) is **not** masked, so never echo secrets.
  - A secret that **doesn't exist** becomes an **empty string** (no error until, say, the login fails).
  - Secrets are **not given** to PRs from a **fork** (someone else's copy of your repo). Levels: repository, **environment**, organization.
- **`GITHUB_TOKEN`:** an automatic secret made for each run (a **token** = a temporary password for a program), for the GitHub API on **this repo** (PR comments, pushing to **GHCR**, GitHub Container Registry); it expires when the job ends.
  - Limit it with **`permissions:`** (e.g. `contents: read`, `packages: write`).
- **Variables** (`${{ vars.NAME }}`, `gh variable set`) hold non-secret settings such as `REGISTRY=ghcr.io`: **visible** in logs and readable later, whereas secrets are **masked** and write-only.
- **Artifacts** (`upload-artifact`: `name`, `path`, `retention-days`) are kept **90 days** by default (**retention**).
  - ⚠️ Since v4 **each name must be unique within a run** (put the matrix value in it). Use **`if: always()`** so failed runs still save their reports.
- A **cache** (`actions/cache`, or setup-python's `cache: pip`) reuses downloads; its **key** often uses `hashFiles('requirements.txt')`, a fingerprint that changes when the file changes.

| | **Artifact** | **Cache** |
|---|---|---|
| Purpose | **keep and share outputs** (reports, builds) | **speed up future runs** |
| Used by | people, and later jobs in the **same run** | **later runs** with the same key |
| Lifetime | 90 days by default | removed after 7 days unused; 10 GB per repo |
| If missing | the downloading job **fails** | the run is just **slower** (a cache miss) |

### 🛠 Do (~35 min): a real pipeline on your GitHub
**A. App and tests** (Session 03's Flask app plus a `/health` page). **pytest** runs every `test*` function in `test_*.py`; each **`assert`** must be true. Flask's **test client** is a pretend browser (no server needed); `monkeypatch` sets an env var for one test only; `b"…"` = bytes.
```bash
mkdir -p ~/devops-practice/ci-demo && cd ~/devops-practice/ci-demo   # project folder
cat > app.py <<'EOF'
from flask import Flask
import os
app = Flask(__name__)
@app.route("/")
def home():
    env = os.environ.get("APP_ENV", "dev")
    return f"Hello from the {env} container!\n"
@app.route("/health")
def health():
    return {"status": "ok"}
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
EOF
cat > test_app.py <<'EOF'
from app import app
def test_home_returns_200():
    response = app.test_client().get("/")
    assert response.status_code == 200
def test_home_uses_app_env(monkeypatch):
    monkeypatch.setenv("APP_ENV", "ci")
    response = app.test_client().get("/")
    assert b"Hello from the ci container!" in response.data
def test_health_is_ok():
    response = app.test_client().get("/health")
    assert response.status_code == 200
    assert response.get_json() == {"status": "ok"}
EOF
printf 'flask==3.0.3\npytest==8.3.3\n' > requirements.txt                 # pinned libraries
printf '.venv/\n__pycache__/\n.pytest_cache/\nreports/\n' > .gitignore    # files Git must NOT commit
python3 -m venv .venv && source .venv/bin/activate   # venv = this project's private Python libraries; prompt shows (.venv)
pip install -r requirements.txt && pytest -v         # expect "3 passed"
```
**B–D. Workflow, GitHub, watch**
```bash
mkdir -p .github/workflows && cp ../cicd/ci.yml .github/workflows/ci.yml   # or paste Concept 76's workflow into it
git init -b main                 # ⚠️ without -b main your Git names the branch master: branches: [main] never matches
git add .
git status                       # new: app.py, test_app.py, requirements.txt, .gitignore, .github/ (NOT .venv/, reports/)
git commit -m "Flask app with tests and CI workflow"
gh repo create ci-demo --private --source=. --push   # private repo from this folder (.), set as origin, pushed now
gh run watch                     # follow the run live ("no in progress runs" = it already finished)
gh run list                      # STATUS ✓, WORKFLOW CI, BRANCH main, EVENT push
gh run view                      # pick the run: JOBS ✓ and ARTIFACTS test-report-3.11, test-report-3.12
gh run download -n test-report-3.12 -D downloaded && head -c 300 downloaded/junit-3.12.xml   # fetch + peek
```
Expect `✓ Created repository <you>/ci-demo`, then `✓ test (3.11)` and `✓ test (3.12)` in parallel, `✓ summary`, completed with 'success'; the XML shows `failures="0"`.
- Asked for a password on push? Run **`gh auth setup-git`** once, then `git push -u origin main`. (Name taken? Use `ci-demo-2`.)

**E–F. Break it, read the ❌, fix it.** In `app.py` change `Hello from the` to `Hi from the`, and **don't** test locally:
```bash
git commit -am "Change greeting" && git push && gh run watch   # -a = stage changed files; test ❌ (fail-fast may cancel the other copy), summary skipped
gh run view --log-failed         # only the failed steps' logs
```
Expect `E  AssertionError: assert b'Hello from the ci container!' in b'Hi from the ci container!\n'`, `1 failed, 2 passed`, and `##[error]Process completed with exit code 1.` (`>` marks the failing line, `E` says why; reports still upload thanks to `if: always()`.)
- ⚠️ CI only reported it; a human decides whether to fix the code or the test.
- Change `Hi` back to `Hello`, run `pytest -q` (3 passed), then `git commit -am "Restore greeting" && git push && gh run watch` → 'success'.

**G. A secret, and masking.** The `cat >>` block **appends** a step to the end of the `summary` job.
```bash
gh secret set DEMO_SECRET        # hidden prompt: type a FAKE value like pretend-password-123 (--body would hit shell history)
gh secret list                   # DEMO_SECRET and a date, never the value
cat >> .github/workflows/ci.yml <<'EOF'
      - name: Use the demo secret
        env:
          DEMO_SECRET: ${{ secrets.DEMO_SECRET }}
        run: |
          if [ -n "$DEMO_SECRET" ]; then echo "secret is set"; fi
          echo "value: $DEMO_SECRET"
EOF
git commit -am "Demo: use a secret" && git push && gh run watch
gh run view --log | grep -E "secret is set|value:"   # pick the newest run
```
Expect `secret is set` and **`value: ***`** (printing is OK only because it's fake). **Keep `ci-demo`** for Session 13. (To delete later: `gh auth refresh -s delete_repo`, then `gh repo delete <you>/ci-demo`.)

### ✅ Check 77
**77.1** A matrix job (Python 3.11 and 3.12) uploads its report with `name: test-report`, the same name in both copies. The 3.11 copy uploads fine; the 3.12 copy fails at the upload step. Why?
- A) Only one job per run may upload artifacts
- B) The 3.12 report path didn't match any files
- C) Artifact names must be unique within a run
- D) Artifacts expire as soon as a job finishes

**77.2** A workflow step uses `${{ secrets.DOCKER_TOKEN }}`, but nobody ever created that secret. What happens at that step?
- A) The run is rejected before any job starts
- B) The expression becomes an empty string
- C) GitHub prompts the repo owner for the value
- D) The text secrets.DOCKER_TOKEN is used as-is

**77.3** (one line) Explain **artifact vs cache** in GitHub Actions.

## ➕ Small but important (GitHub Actions): read before the quiz
| Item | What it is |
|---|---|
| **`workflow_dispatch` inputs** | a manual run asks questions (`inputs: environment: { type: choice, options: [staging, production] }`), read as `${{ inputs.environment }}`; CLI: `gh workflow run ci.yml -f environment=staging` |
| **`concurrency`** | `concurrency: { group: deploy-${{ github.ref }}, cancel-in-progress: true }`: runs in one group never overlap; the older one is cancelled |
| **`timeout-minutes`** · **`continue-on-error`** · **path filters** · **`[skip ci]`** | stops a job that runs too long (default **360 min = 6 h**) · `continue-on-error: true`: a failure there doesn't fail the run · `paths: ['src/**']` / `paths-ignore: ['docs/**']` under `push:` · `[skip ci]` in a commit message skips push and PR workflows |
| **Default env vars** / **`GITHUB_OUTPUT`, `GITHUB_ENV`, `GITHUB_STEP_SUMMARY`** | every step gets `GITHUB_SHA`, `GITHUB_REF`, `GITHUB_REF_NAME`, `GITHUB_REPOSITORY`, `GITHUB_ACTOR`, `GITHUB_RUN_ID`, `GITHUB_WORKSPACE`, `RUNNER_OS`, `CI=true` / `echo "tag=v1" >> "$GITHUB_OUTPUT"` in step `id: meta` → `${{ steps.meta.outputs.tag }}`; `GITHUB_ENV` sets env vars for later steps; `GITHUB_STEP_SUMMARY` adds Markdown to the run page |
| **Status badge** · **branch protection / rulesets** | `![CI](https://github.com/<you>/ci-demo/actions/workflows/ci.yml/badge.svg)` in the README · rules that **require status checks to pass before merging**, require reviews, block force-pushes (private repos need GitHub Pro) |
| **Environments + required reviewers** | `environment: production` on a job + required reviewers = the job **pauses until a person approves** (Continuous Delivery); own secrets too |
| **Reusable workflows** / **composite actions** | callee `on: workflow_call`, caller **job** `uses: owner/repo/.github/workflows/build.yml@main` / your own `action.yml` with `runs: using: composite` |
| **Self-hosted runners** / **pinning** | **never on public repos** (a stranger's PR runs code on your machine), not wiped between jobs, `runs-on: [self-hosted, linux]` / `uses: actions/checkout@<40-character SHA>` is safest, since a tag can be moved (**SHA** = Secure Hash Algorithm) |
| **`services:`** / **secrets in `if:`** | `services: db: image: postgres:16` = extra container for tests / can't test `secrets.X` in `if:`: copy it to `env:`, test `env.X` |

**Quick checks:** (1) How do you make a production deploy job wait for a person's approval? (2) How do you stop two deployments of the same branch from running at the same time?
*Answers:*
- (1) Add `environment: production` to the job, and give that environment a **required reviewers** protection rule in Settings → Environments. The job pauses until a reviewer approves (Continuous Delivery).
- (2) Add `concurrency:` with a group name such as `deploy-${{ github.ref }}`, so runs in the same group can't overlap. `cancel-in-progress: true` also cancels the older one.

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory
**Q1.** You run `git init` (with no `-b`), commit, and push using `gh repo create ci-demo --private --source=. --push`. Your workflow has `on: push: branches: [main]`. No run ever appears in the Actions tab. Why?
- A) Pushes made by gh don't trigger workflows
- B) The pushed branch is master, not main
- C) A private repo must enable Actions first
- D) The first run needs workflow_dispatch

**Q2.** One test fails in this workflow. Which of the `echo` lines get printed?
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: pytest
      - run: echo "tests done"
      - if: always()
        run: echo "cleanup"
  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploying"
```
- A) `cleanup` only; `deploy` is skipped
- B) `tests done` and `cleanup`; no deploy
- C) `cleanup` and `deploying`, nothing else
- D) None: the whole run stops at pytest

**Q3.** What does this step print?
```yaml
env:
  APP_ENV: ci
jobs:
  show:
    runs-on: ubuntu-latest
    steps:
      - run: echo '$APP_ENV' '${{ env.APP_ENV }}'
```
- A) `ci ci`, because GitHub fills in both before bash
- B) `$APP_ENV` and the raw `${{…}}` text, both literal
- C) `ci` and the raw `${{…}}` text: bash expands one
- D) `$APP_ENV ci`, since GitHub only fills in `${{ }}`

**Q4.** A step is `run: pytest | tee test-output.txt` (no `shell:` key, Linux runner). Two tests fail. What happens to the step?
- A) It fails, since bash -e stops on any error
- B) It fails, since GitHub adds -o pipefail
- C) It passes: the pipe's exit code is tee's
- D) It fails, as tee passes on pytest's code

**L1.** Write the full forms of **CI** and **CD**, and explain **Continuous Delivery vs Continuous Deployment** in contrast form.

**L2.** Explain **job vs step** in GitHub Actions: where each one runs, and what they share.
