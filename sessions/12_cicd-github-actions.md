# Session 12 · CI/CD and GitHub Actions (Oct 16, Session 2)

**Time:** ~90–100 min · **Concepts:** 74 → 77 · **Answers:** `MY_ANSWERS.md` or say "CI/CD questions" · **Answer key:** `12_cicd-github-actions_ANSWERS.md`

**Before you start:** today's practical uses **your real GitHub account** through **`gh`**, the **GitHub CLI** (Command-Line Interface): GitHub's own tool for doing GitHub things from the Terminal. `gh auth status` should show `✓ Logged in to github.com …` with Token scopes including `'workflow'` (the permission to push files into `.github/workflows/`); `git --version` should show 2.x, and `python3 --version` 3.9 or newer.

⚠️ Never delete the **`minikube`** or **`localstack`** containers; today doesn't need Docker or Kubernetes. Text after `#` in commands is a note for you: don't type it (your zsh would pass it on as arguments), or run `setopt interactivecomments` first.

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

### The problem, and CI
Five developers each work on their own branch for **three weeks**. On "merge day" the branches clash everywhere, the app doesn't even build, and nobody knows whose change broke what. This is called **integration hell** (to **integrate** = to combine everyone's changes into one shared codebase). Releasing is just as risky: someone follows a printed checklist by hand, skips one step, and the site goes down.

> **CI (Continuous Integration)** = developers **merge small changes into the main branch often**, and **every push or pull request automatically triggers a build and automated tests**. Because each change is small, a break is noticed **within minutes**, and you know **which commit** caused it.

- A **unit test** is a small piece of code that checks one small part of the app, e.g. "does `GET /` return 200?". You'll write some with pytest later today.
- ⚠️ **CI DETECTS and REPORTS problems; it never fixes code.** It runs the checks and shows a red ❌ with the error, and **a developer** reads the log and fixes it. **Analogy:** a quality-control checkpoint on a factory conveyor belt: a faulty item is flagged and sent back, but the inspector repairs nothing.

### CD: two different things
After CI, the tested change has to reach users. "CD" can mean either of these, and ⚠️ they differ **only in the last step**:

| | **Continuous Delivery** | **Continuous Deployment** |
|---|---|---|
| Every change that passes the pipeline… | is **ready to release** at any moment | **is released**, automatically |
| The last step (to production) | **a human approves** (clicks a button) | **no human**: it goes out on its own |

### The pipeline
> A **pipeline** is an automated sequence of **stages** that every change passes through, from the push until it's running for users, like an assembly line where each station does one job in order. It's started by a **trigger**: an event such as a push or a PR.

`SOURCE (git push) → BUILD (pip install) → TEST (pytest, lint) → PACKAGE (docker build) → SCAN (security checks) → PUBLISH (docker push) → DEPLOY (helm upgrade) → VERIFY (smoke test)`
- **Lint** = automatic checks for style and common mistakes (like `helm lint`). A **smoke test** is a quick "is it alive?" check after deploying, e.g. `curl -f https://shop.example.com/health`.
- **Fail fast:** put the quick, cheap checks first. If an early stage fails, **the later stages don't run at all**, so nothing broken gets packaged or deployed.
- **How does a pipeline know a step failed? Exit codes** (shell scripting): **0 = success, anything else = failure**. A CI tool checks **only** that number: 0 → ✅ and the next step runs; non-zero → ❌ and every later stage is skipped, exactly like `set -e`. `pytest` exits 0 when all tests pass and 1 when any test fails.
- ⚠️ **CI trusts the exit code, not the text.** A script that prints "3 tests FAILED" but ends with `exit 0` gives a **green** pipeline.

### Artifacts, environments and deployment strategies
- An **artifact** is a file the pipeline produces and keeps afterwards: a test report, a packaged chart, a Docker image.
- An **environment** is a separate copy of the whole app for one purpose: **dev** (developers try things; it breaks often), **staging** (a near-identical copy of production: the dress rehearsal), and **production** (the real thing, with real users).
- **Build once, deploy many:** build the artifact **once**, then **promote** (move) **the same artifact** from dev to staging to production. Rebuilding it for production would ship something that was never tested.
- A **deployment strategy** is the way the old version (v1) is replaced by the new one (v2):

| Strategy | How | Downtime? | Rollback | Extra resources / risk |
|---|---|---|---|---|
| **Recreate** | stop all v1, then start v2 | **Yes** | redeploy the old one (slow) | none / high |
| **Rolling** (K8s default) | replace a few Pods at a time (`maxSurge` = extra Pods allowed, `maxUnavailable` = Pods allowed down) | No | `kubectl rollout undo`, `helm rollback` | a little / medium |
| **Blue-green** | two full environments: "blue" is live, "green" gets v2 and is tested; then ONE switch (e.g. a Service selector `blue` → `green`) moves all traffic | No | **instant switch back** | **double** / low |
| **Canary** | 5% of users get v2 (the "canary" that meets problems first); watch the error rates, then 25 → 50 → 100% | No | send the 5% back to v1 | a little / **lowest** |

**CI/CD tools:** **GitHub Actions** (built into GitHub; today's tool). **Jenkins**: a server you install and maintain yourself, with 1000s of plugins; its pipeline file is a **`Jenkinsfile`** written in **Groovy** (a programming language), and its jobs run on an **agent** (a worker machine). **GitLab CI**: built into GitLab, configured in `.gitlab-ci.yml`. **Argo CD** is deploy-only (CD) for Kubernetes: it runs inside the cluster and keeps it matching what's in a Git repo.

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
- `cat > file <<'EOF' … EOF` writes the lines in between into the file; `sys.exit(1)` ends the Python line with exit code 1, like a failing test run. Expect `1 test failed` and `pipeline exit code: 1`: **DEPLOY never prints**, because `set -e` stopped the script, exactly as a CI server would.
- Now make the test **lie**: change `sys.exit(1)` to `sys.exit(0)` and rerun. `deploying to production` prints, with exit code 0, although the text said "failed".

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

### The parts, from biggest to smallest
> **GitHub Actions** is GitHub's built-in CI/CD service: it runs your automation **on GitHub's computers** whenever something happens in your repository. It's free for public repos, and private repos get a free monthly allowance of minutes.

```
.github/workflows/ci.yml              ← WORKFLOW: one YAML file = one workflow
      on: push                        ← EVENT (trigger): WHEN it runs
      jobs:
        test:                         ← JOB: runs on its own fresh RUNNER machine
          runs-on: ubuntu-latest      ← RUNNER: WHERE it runs
          steps:                      ← STEPS: run in order, on that same machine
            - uses: actions/checkout@v7      ← a step that uses an ACTION
            - run: pytest                    ← a step that runs a shell command
        lint:                         ← another JOB: runs IN PARALLEL with "test"
```
**Analogy: a restaurant kitchen preparing a banquet.** The **workflow** is the plan ("when a booking arrives, do all this"), and the **event** is the booking arriving. **Jobs** are separate cooking stations, each with its **own clean counter**, working at the same time. **Steps** are one station's instructions, done in order. **Actions** are ready-made sauces bought from a supplier, used as one step. The **runner** is the counter and cook doing the work.

### Each part in detail
- **Workflow:** a YAML file in **exactly** the folder `.github/workflows/` (note the dot, and the **s**), with any name ending in `.yml`. A file anywhere else is **silently ignored**.
- **Event** (the `on:` key) = **when** the workflow runs:
  - `push`: commits are pushed. Add `branches: [main]` to limit it to some branches.
  - `pull_request`: a PR is opened, **or new commits are pushed to its branch**. GitHub tests what the code would look like **after merging**.
  - `workflow_dispatch`: someone clicks **Run workflow** in the Actions tab, or types `gh workflow run` (a manual trigger).
  - `schedule`: at times written in **cron**: five fields, `minute hour day-of-month month day-of-week`, 24-hour clock, day-of-week 0 = Sunday (so `1-5` = Monday to Friday). ⚠️ It's in **UTC** (**Coordinated Universal Time**, the world's reference clock) unless you add a `timezone:` key. **IST** (Indian Standard Time) = UTC + 5:30, so **subtract 5 h 30 min**: `"30 2 * * 1-5"` = 02:30 UTC = **08:00 IST**, Monday to Friday.
- **Job:** a group of steps that runs on **one runner**. Jobs run **in parallel** unless `needs:` makes one wait for another. **Each job gets a brand-new machine**, so files made in job A **don't exist** in job B (artifacts pass files between jobs: job A uploads them, job B downloads them).
- **Step:** one task inside a job, **either** `run:` (shell commands) **or** `uses:` (an action). Steps run in order on the **same** runner, so they share its files. If one fails, the rest are skipped.
- **Action:** a reusable, packaged step someone has already written, shared on the **GitHub Marketplace** (GitHub's catalogue of actions) and written `owner/repo@version`. **`actions/checkout`** copies (clones) your repo's code onto the runner, which starts empty, so it's almost always step 1. **`actions/setup-python`** installs the Python version you ask for. (`@v7` is a version tag, like an image tag; older tutorials show `@v3` or `@v4`.)
- **Runner:** the machine that runs a job. A **GitHub-hosted runner** is a fresh **VM** (virtual machine) that GitHub creates for **each job** and deletes afterwards, picked with a label such as `ubuntu-latest` (or `windows-latest`, `macos-latest`). A **self-hosted runner** is **your own** machine with GitHub's runner program installed (`runs-on: self-hosted`): you maintain it, it's **not wiped** between jobs, and it can reach your private network.
- **What happens on `git push`:** GitHub finds every workflow whose `on:` matches → starts a fresh runner for each job → runs the steps in order, checking each exit code → shows the logs, the ✅/❌ and any artifacts in the **Actions** tab and on the PR → destroys the VM. One execution of a workflow is called a **workflow run**.

### 🛠 Do (~5 min): look at real runs (read-only)
```bash
gh run list --repo actions/checkout --limit 5   # the newest 5 runs in GitHub's own (public) checkout repo
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

### A full workflow, line by line (you'll push this one in Concept 77)
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
- **JUnit XML** is a standard test-report format that CI tools can read (**XML** = eXtensible Markup Language, a tag-based format like HTML). `--junitxml=…` makes pytest write one, and that file is the run's **artifact**.
- ⚠️ **One step = `uses` OR `run`, never both.** To run a command after an action, add a second step.

### Expressions, contexts and `if:`
- An **expression** is anything inside **`${{ … }}`**. **GitHub works it out and replaces it with plain text BEFORE the step runs**, so the shell never sees `${{ }}`, only the result. **Analogy:** a mail-merge letter, whose blanks are filled in before anyone reads it.
- A **context** is a group of values you can read in expressions: `github` (`github.sha` = the full 40-character commit ID, handy as a unique image tag; `github.ref` = `refs/heads/main`; `github.ref_name` = `main`; `github.event_name` = `push`), `matrix` (the current matrix value), `env` (values from `env:` blocks), `vars` and `secrets` (stored settings and hidden passwords, explained below), and `runner` (`runner.os` = `Linux`).
- A top-level **`env:`** sets environment variables for every job (it can also go on one job or one step). **`if:`** on a job or step runs it only when a condition is true, e.g. `if: github.ref == 'refs/heads/main'`.
- **Status functions** in `if:`: **`success()`** = every earlier step succeeded (**the default**, which is why steps after a failure are skipped); **`failure()`** = an earlier step failed (e.g. send an alert); **`always()`** = run anyway, even after a failure (e.g. upload test reports).
- ⚠️ **`${{ env.X }}` vs `$X` inside `run:`.** `${{ env.X }}` is filled in **by GitHub, before bash starts**. `$X` is expanded **by bash, while running**, using the shell's quote rules. So `echo '$X'` prints a literal `$X` (single quotes), but `echo '${{ env.X }}'` prints the value, because GitHub already pasted it inside the quotes.

### Matrix, `needs:` and a pipe trap
- A **matrix** runs the job **once per combination**: `python-version: ["3.11", "3.12"]` × `os: [ubuntu-latest, windows-latest]` (with `runs-on: ${{ matrix.os }}`) = **4 jobs**, in parallel.
- **`fail-fast: true`** (the default) cancels the other copies still running when one fails. Set `false` to let them all finish.
- ⚠️ **Quote version numbers.** YAML reads `3.10` (unquoted) as the **number 3.1**, so setup-python would look for Python 3.1. Write `"3.10"`.
- **`needs: test`** waits for **all** matrix copies of `test`, and `needs: [test, lint]` waits for both. If a needed job fails, the dependent job is **skipped**, unless it has `if: always()`.
- **The pipe trap:** on Linux runners, a `run:` without a `shell:` key runs as **`bash -e {0}`** (`{0}` = the script file GitHub writes from your `run:` text). **`-e`** is `set -e`: stop at the first failing command. But **`pipefail` is off**, and in a pipe (`a | b`) bash reports **only the last command's** exit code. So `run: pytest | tee out.txt` **passes even when tests fail**, because `tee` succeeded. Fix: add **`shell: bash`**, which adds `-o pipefail` (a pipe fails if **any** part fails), or don't pipe test commands.

### Common workflow bugs
- **Nothing happens, no run appears:** a wrong folder (`.github/workflow/` without the s, or `github/workflows/` without the dot), or a branch filter that doesn't match (`branches: [master]` when you push `main`: the file is valid, it just never matches).
- **"Invalid workflow file"** with a line number, and no jobs run: bad indentation or tabs, `on: pushh`, no `runs-on`, `uses` + `run` in one step, `needs:` a job that doesn't exist, or steps without `-` (`steps` must be a YAML list).
- **Setup fails:** unquoted `python-version: 3.10` asks for Python **3.1**.

### 🛠 Do (~10 min)
**Bug hunt:** in `broken-ci.yml`, add four bugs: under `push:` change `[main]` to `[master]`; delete the `test` job's `runs-on:` line; add `run: python --version` under `uses: actions/setup-python@v7`; change `needs: test` to `needs: tset`. **actionlint** (optional) is a free checker for workflow files, like `helm lint` for GitHub Actions.
```bash
cd ~/devops-practice/cicd && code ci.yml   # paste the workflow above and save (you push it to GitHub later today)
gh api repos/actions/starter-workflows/contents/ci/python-app.yml --jq .content | base64 -d   # GitHub's starter workflow
cp ci.yml broken-ci.yml && code broken-ci.yml    # make the four bugs, save
brew install actionlint                          # (optional) install the checker
actionlint ci.yml; echo "exit code: $?"          # the good file: no output, exit code 0
actionlint broken-ci.yml; echo "exit code: $?"   # the broken copy: errors with line numbers, exit code 1
```
- `gh api` calls GitHub's API as you; `--jq .content` keeps one field of the **JSON** (JavaScript Object Notation) reply; GitHub sends file contents base64-encoded, so `base64 -d` decodes them. You get `name: Python application` with one job, `build`: **find** its trigger, its job ID, a `uses` step, a `with:` input, and a `run: |`.
- actionlint reports the missing `runs-on`, the step mixing `uses` and `run`, and the unknown job `tset`. **The `master` bug is valid YAML**, so no checker can see it: only a human catches logic bugs.

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

### Secrets
Pipelines log in to things: Docker Hub to push images, a cloud account to deploy. A password typed into the workflow YAML is **committed to Git**: anyone with access to the repo can read it (the whole world, for a public repo), and it stays in the Git history **forever**, even after you delete the line.

> A **secret** is an encrypted value stored by GitHub (not in your files) that workflows can use but **nobody can read back**. **Analogy:** a hotel's safe-deposit box: the hotel keeps it, and your workflow can use what's inside, but nobody can open it to look.

- **Create one** in your repo's **Settings → Secrets and variables → Actions**, or with `gh secret set NAME` (it asks for the value and hides your typing). After saving, even you can only replace or delete it.
- **Use one** by passing it to a step through `env:` (Part G below does exactly this): `env: DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}`, then `run: echo "$DOCKERHUB_TOKEN" | docker login -u myuser --password-stdin` (`--password-stdin` reads the password from the pipe, so it never appears on the command line).
- **Masking:** if a secret's exact value appears in a log, GitHub shows **`***`** instead. ⚠️ It catches only the **exact** text: a changed copy (base64-encoded, reversed) is **not** masked and leaks. Never echo secrets.
- A secret that **doesn't exist** becomes an **empty string**, with no error at that point; the failure shows up later (e.g. "login failed").
- Secrets are **not given** to workflows started by a PR from a **fork** (someone else's copy of your repo), or a stranger's PR could print them. They exist at three levels: repository, **environment** (➕) and organization.
- **`GITHUB_TOKEN`** is a secret GitHub **creates automatically for every run** (a **token** = a temporary password for a program). It lets the workflow use the GitHub API **for this repo**, e.g. comment on a PR or push an image to **GHCR** (GitHub Container Registry), and it expires when the job ends. Limit what it may do with **`permissions:`** (e.g. `contents: read`, `packages: write`).
- **Configuration variables** (`${{ vars.NAME }}`, created with `gh variable set`) hold **non-secret** settings such as `REGISTRY=ghcr.io`. They're **visible** in logs and readable later, whereas secrets are **masked** and write-only.

### Artifacts and cache
- **`actions/upload-artifact`** saves files from a job (inputs: `name`, `path`, `retention-days`). People download them from the run page, and later jobs in the **same run** fetch them with **`actions/download-artifact`**. They're kept **90 days** by default: that's the **retention** period.
- ⚠️ Since v4, **each artifact name must be unique within a run**, so in a matrix put the matrix value into the name. Add **`if: always()`** so the report is saved even when tests fail, which is exactly when you need it.
- A **cache** keeps **downloads that rarely change** (like Python packages) so **future runs** are faster: use `actions/cache`, or simply `cache: pip` on setup-python. Its **key** usually includes `hashFiles('requirements.txt')`, a fingerprint of that file, so when requirements.txt changes, a fresh cache is built.

| | **Artifact** | **Cache** |
|---|---|---|
| Purpose | **keep and share outputs** (reports, builds) | **speed up future runs** |
| Used by | people, and later jobs in the **same run** | **later runs** with the same key |
| Lifetime | 90 days by default | removed after 7 days unused; 10 GB per repo |
| If missing | the downloading job **fails** | the run is just **slower** (a cache miss) |

### 🛠 Do (~35 min): a real pipeline on your GitHub
**A. App and tests.** The small Flask "Hello from the … container!" app you containerised earlier, plus a small `/health` page. **pytest** (Python's most popular test tool) runs every function whose name starts with `test` in files named `test_*.py`, and each **`assert`** line must be true or the test fails. Flask's **test client** is a **pretend browser**: it sends requests straight to the app, with no server needed.
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
python3 -m venv .venv && source .venv/bin/activate   # create and switch on a venv; your prompt now shows (.venv)
pip install -r requirements.txt && pytest -v         # expect "3 passed"
```
- `monkeypatch` is a pytest helper that changes something (here the env var `APP_ENV`) **only during that one test**. `response.data` is the page as **bytes**, so the expected text is written `b"…"`.
- `python3 -m venv .venv` creates a **virtual environment** (**venv**): a private folder holding this project's own Python libraries, so they don't mix with your Mac's Python.

**B–D. Workflow, GitHub, watch.**
```bash
mkdir -p .github/workflows && cp ../cicd/ci.yml .github/workflows/ci.yml   # or paste the full workflow from above into it
git init -b main                 # ⚠️ without -b main your Git names the branch master: branches: [main] never matches
git add .
git status                       # new: app.py, test_app.py, requirements.txt, .gitignore, .github/ (NOT .venv/, reports/)
git commit -m "Flask app with tests and CI workflow"
gh repo create ci-demo --private --source=. --push   # private repo from this folder (.), set as origin, pushed now
gh run watch                     # follow the run live ("no in progress runs" = it already finished)
gh run list                      # STATUS ✓, WORKFLOW CI, BRANCH main, EVENT push
gh run view                      # pick the run: JOBS ✓ and ARTIFACTS test-report-3.11, test-report-3.12
gh run download -n test-report-3.12 -D downloaded && head -c 300 downloaded/junit-3.12.xml   # fetch one artifact + peek
```
- Expect `✓ Created repository <you>/ci-demo`, then `✓ test (3.11)` and `✓ test (3.12)` side by side (in parallel), then `✓ summary`, and the run completed with 'success'. The XML shows `failures="0"`.
- Asked for a password on push? Run **`gh auth setup-git`** once (it lets Git use your `gh` login), then `git push -u origin main`. (Name taken? Use `ci-demo-2`.)

**E–F. Break it, read the ❌, fix it.** In `app.py`, change `Hello from the` to `Hi from the`, and **don't** test locally this time:
```bash
git commit -am "Change greeting" && git push && gh run watch   # -a = stage changed files; test ❌, summary skipped
gh run view --log-failed         # only the logs of the steps that failed
```
- Expect `E  AssertionError: assert b'Hello from the ci container!' in b'Hi from the ci container!\n'`, then `1 failed, 2 passed` and `##[error]Process completed with exit code 1.`
- **How to read it:** **which test** failed, **which line** (`>` marks it), and **why** (the `E` lines). The last line is the key: **exit code 1**, so the step failed. When one test copy fails, `fail-fast` may cancel the other, and `summary` is skipped (it `needs` test). The reports still upload, thanks to `if: always()`.
- ⚠️ CI only reported the problem; a human decides whether to fix the code or the test. Here it was a mistake: change `Hi` back to `Hello`, run `pytest -q` (3 passed), then `git commit -am "Restore greeting" && git push && gh run watch` → 'success'.

**G. A secret, and masking.** The `cat >>` block **appends** (adds) a step to the end of the `summary` job:
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
`[ -n "$DEMO_SECRET" ]` = "is the variable non-empty?". Expect `secret is set` and **`value: ***`**: GitHub hid the value. (Printing it is OK only because it's fake.) **Keep `ci-demo`** for the next session. (To delete it later: `gh auth refresh -s delete_repo`, then `gh repo delete <you>/ci-demo`.)

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
| **`workflow_dispatch` inputs** | A manual run can ask questions, e.g. `inputs: environment: { type: choice, options: [staging, production] }`, read as `${{ inputs.environment }}`. From the Terminal: `gh workflow run ci.yml -f environment=staging` |
| **`concurrency`** | `concurrency: { group: deploy-${{ github.ref }}, cancel-in-progress: true }`: runs in the same **group** never overlap, and the older one is cancelled |
| **`timeout-minutes`** · **`continue-on-error`** | Stop a job that runs too long (default **360 min = 6 h**) · `continue-on-error: true` = a failure in that step or job doesn't fail the run |
| **Path filters** · **`[skip ci]`** | `paths: ['src/**']` under `push:` runs only when those files change (`paths-ignore:` = the opposite) · `[skip ci]` in a commit message skips the push and PR workflows |
| **Default env vars** · **`GITHUB_OUTPUT`** | Every step automatically gets `GITHUB_SHA`, `GITHUB_REF`, `GITHUB_ACTOR`, `RUNNER_OS`, `CI=true`, … · pass a value to later steps: `echo "tag=v1" >> "$GITHUB_OUTPUT"` in a step with `id: meta`, then `${{ steps.meta.outputs.tag }}` |
| **Branch protection / rulesets** | Repo rules that **require status checks to pass before merging** (and reviews, no force-pushes), so a red CI really blocks the merge. Private repos need GitHub Pro for this |
| **Environments + required reviewers** | `environment: production` on a job, plus required reviewers in Settings → Environments: the job **pauses until a person approves** (Continuous Delivery's manual gate). Environments can hold their own secrets |
| **Reusable workflows** | One pipeline shared by many repos: the called workflow has `on: workflow_call`, and the caller **job** says `uses: owner/repo/.github/workflows/build.yml@main` |
| **Self-hosted runner security** | **Never on public repos**: a stranger's PR would run its code on your machine. They aren't wiped between jobs; pick them with labels, e.g. `runs-on: [self-hosted, linux]` |
| **Pin actions to a commit** | `uses: actions/checkout@<40-character commit SHA>` is safest, because a tag like `v7` can be moved to different code (**SHA** = Secure Hash Algorithm, which makes Git's commit IDs) |
| **Secrets in `if:`** | You can't test `secrets.X` directly in `if:`: copy it into `env:` and test `env.X` |

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

### ✍️ Write the command
**C1.** Turn the current folder into a new Git repository whose first branch is named `main` (so `branches: [main]` will match).

**C2.** In one command, create a **private** GitHub repository named `ci-demo` from the current folder, set it as the remote, and push your commits to it.

**C3.** Your latest workflow run failed. Show only the logs of the steps that failed.

**C4.** Write the workflow's `on:` block so it runs on every push to `main` and on every pull request into `main`.
