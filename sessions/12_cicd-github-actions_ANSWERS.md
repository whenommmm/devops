# Answer key · Session 12 · CI/CD and GitHub Actions

⚠️ Only open after answering.

---

## Warm-up

**W1.** Release `web` is at revision 5. You run `helm rollback web 3`, then `helm history web`. What is the newest line?
- A) Revision 3: deployed, "Rollback to 3"
- B) Revision 4: deployed, "Rollback to 3"
- C) Revision 5: superseded, "Rollback to 3"
- D) Revision 6: deployed, "Rollback to 3"

**Correct: D.** A rollback always **adds a new revision** (5 + 1 = 6) that copies revision 3's settings, and revision 5 becomes `superseded`. A imagines revision 3 being re-labelled in place. B forgets that revision 4 already exists. C mixes up the old revision's new status (superseded) with the new line.

---

**W2.** A chart's values.yaml has `replicaCount: 1`. You installed with `helm install shop ./chart -f values-prod.yaml` (which sets `replicaCount: 4`). A week later you run `helm upgrade shop ./chart --set image.tag=2.0`, with no `-f`. How many replicas run after the upgrade?
- A) 4, because Helm keeps the earlier -f file
- B) 1, because only this run's values apply
- C) 4, since --set changes only the named key
- D) 0, because no replicaCount was supplied

**Correct: B.** Each `helm upgrade` starts from the chart's values.yaml plus **only the values given in that command** (unless you add `--reuse-values`). The `-f values-prod.yaml` from the install isn't remembered, so `replicaCount` falls back to the default 1. A and C describe what people expect, which is the trap. D: the chart's default (1) is always there.

---

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

**Correct: C.** A function's exit code is its `return N`, **or else its last command's**. The last command is `grep`, which exits **1** when it finds no match. A: the `echo` ran earlier, so it isn't the last command. B: with no `return`, the last command decides, not an automatic 0. D: `-q` silences grep's **output**, but `echo $?` still prints the code. (Today's link: a CI step that ran this function would turn **red**, because its exit code is non-zero.)

---

## Check 74

**74.1** A team's pipeline builds, tests, and deploys every passing commit to **staging** automatically. Production deploys happen only when the release manager clicks **Approve**. Which practice is this?
- A) Continuous Delivery, with a manual prod gate
- B) Continuous Deployment, as staging is automatic
- C) Continuous Integration only, no CD at all
- D) Continuous Deployment with a canary release

**Correct: A.** Everything is automated and always releasable, but a **human approves** the step into production. That's Continuous **Delivery**. B: Continuous **Deployment** means production itself is automatic, and automatic staging doesn't make it so. C: there clearly is CD, because deployment to staging is automated. D: canary is a deployment **strategy** (small share of users first) and says nothing about approval.

---

**74.2** A test script prints `3 tests FAILED` but always ends with `exit 0`. What does the CI pipeline do?
- A) Marks the step failed, because it reads FAILED
- B) Marks it passed and moves on to later stages
- C) Pauses and asks a developer to read the output
- D) Fixes the failing tests, then reruns the stage

**Correct: B.** CI tools judge a step **only by its exit code**, and 0 means success, so the broken code continues down the pipeline (you saw this in the Concept 74 Do). A: CI doesn't read the text. C: pipelines don't pause for that (only for explicit approval gates). D: **CI never fixes code**. It detects and reports, and people fix (a weak-list point).

---

**74.3** Which strategy keeps two full environments and switches all traffic at once, so rollback is instant?
- A) Canary
- B) Rolling update
- C) Blue-green
- D) Recreate

**Correct: C.** Blue-green runs a complete "blue" (live) and "green" (new) environment and flips all traffic in one switch, so rolling back is flipping it back. A: canary shifts traffic **gradually**, starting with a small share. B: rolling replaces copies **a few at a time** within one environment. D: recreate stops everything first, giving downtime and no instant rollback.

---

## Do 75 (cron translations)
1. `0 18 * * *` = **every day at 18:00 UTC** (= 23:30 IST).
2. `30 3 * * 1` = **every Monday at 03:30 UTC** (= 09:00 IST).
3. `*/15 * * * *` = **every 15 minutes** (`*/15` in the minute field = 0, 15, 30, 45).

---

## Check 75

**75.1** A workflow has jobs `lint` and `test`, with no `needs:`. `lint` creates `report.txt`; `test` runs `cat report.txt`. What happens?
- A) test waits for lint, then prints the report
- B) test fails: it runs on a separate fresh VM
- C) test prints the report, as jobs share a disk
- D) GitHub runs lint again inside the test job

**Correct: B.** Each job gets its **own brand-new runner VM**, and with no `needs:` the two jobs even run **at the same time**. `report.txt` exists only on lint's machine, so `cat` fails (`No such file or directory`, exit code 1). A: without `needs:` there's no waiting, and even with it the file wouldn't be copied: you'd need upload-artifact + download-artifact. C: only **steps** within one job share a disk. D: GitHub never re-runs jobs inside other jobs.

---

**75.2** Which schedule runs a workflow at **9:00 AM IST** (UTC + 5:30), Monday to Friday? No `timezone:` key is set.
- A) `cron: "0 9 * * 1-5"`
- B) `cron: "30 3 * * 0-4"`
- C) `cron: "0 21 * * 1-5"`
- D) `cron: "30 3 * * 1-5"`

**Correct: D.** GitHub's schedules use **UTC** by default: 09:00 IST − 5:30 = **03:30 UTC**, written minute first (`30 3`), and `1-5` = Monday to Friday. A forgets the UTC conversion (that's 2:30 PM IST). B has the right time, but `0-4` means **Sunday** to Thursday (0 = Sunday). C is the 12-hour clock trap: 21 = 9 **PM**.

---

**75.3** (one line) What does `actions/checkout` do, and why is it almost always the first step?

**Model answer:** *"`actions/checkout` clones the repository's code onto the runner. Each job starts on a fresh, empty VM, so without it the later steps would have no code to install, build, or test."*

---

## Check 76

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

**Correct: C.** A step either runs an action (`uses`) **or** shell commands (`run`), never both. Fix it by splitting it into two steps. actionlint reports it as `unexpected key "uses" for step to run shell command`. A: key order inside a step doesn't matter. B: `name` is allowed on any step. D: `with:` is optional, and adding it wouldn't make `uses` + `run` legal.

---

**76.2** Job `test` has a matrix with `python-version: ["3.11", "3.12"]` and `os: [ubuntu-latest, windows-latest]`, plus `runs-on: ${{ matrix.os }}`. Job `build` has `needs: test` and no matrix. If everything passes, how many job runs happen in total?
- A) 5: four matrix copies, then the build job
- B) 3: one per Python version, then the build job
- C) 4: matrix copies only; needs: adds no new job
- D) 6: the build job runs once per test copy

**Correct: A.** The matrix makes every combination: 2 versions × 2 operating systems = 4 copies of `test`. `build` then runs **once**, after all four succeed. B forgets the `os` dimension. C: `build` is a separate job, so it adds one run. D: `needs:` waits for all copies, and doesn't multiply the dependent job (it has no matrix of its own).

---

**76.3** (one line) Why does `python-version: [3.10, 3.11]` (no quotes) in a matrix cause trouble?

**Model answer:** *"YAML reads the unquoted 3.10 as the number 3.1, so setup-python is asked for Python 3.1 and fails. Version numbers should be quoted, as in `"3.10"`."*

---

## Check 77

**77.1** A matrix job (Python 3.11 and 3.12) uploads its report with `name: test-report`, the same name in both copies. The 3.11 copy uploads fine; the 3.12 copy fails at the upload step. Why?
- A) Only one job per run may upload artifacts
- B) The 3.12 report path didn't match any files
- C) Artifact names must be unique within a run
- D) Artifacts expire as soon as a job finishes

**Correct: C.** Since upload-artifact v4, artifacts **can't be overwritten** (unless you set `overwrite: true`), so a second artifact with the same name in one run fails with a conflict. Fix: `name: test-report-${{ matrix.python-version }}`. A: any number of jobs may upload, as long as the names differ. B: a missing path only **warns** by default (`if-no-files-found: warn`), and wouldn't fail the step. D: artifacts are kept for 90 days by default.

---

**77.2** A workflow step uses `${{ secrets.DOCKER_TOKEN }}`, but nobody ever created that secret. What happens at that step?
- A) The run is rejected before any job starts
- B) The expression becomes an empty string
- C) GitHub prompts the repo owner for the value
- D) The text secrets.DOCKER_TOKEN is used as-is

**Correct: B.** A secret that doesn't exist evaluates to **an empty string**, with no error at that point. The step continues, and something later fails confusingly (for example, a Docker login with no password). A: GitHub doesn't check secret names before running. C: runs never stop to prompt for input. D: expressions are always evaluated, never left as literal text.

---

**77.3** (one line) Explain **artifact vs cache** in GitHub Actions.

**Model answer:** *"An artifact stores files a run produces, such as test reports or build outputs, so people can download them and later jobs in the same run can use them (kept 90 days by default), whereas a cache stores reusable downloads, such as pip packages, under a key, so future runs finish faster. A missing cache only slows a run down."*

---

## Session quiz

**Q1.** You run `git init` (with no `-b`), commit, and push using `gh repo create ci-demo --private --source=. --push`. Your workflow has `on: push: branches: [main]`. No run ever appears in the Actions tab. Why?
- A) Pushes made by gh don't trigger workflows
- B) The pushed branch is master, not main
- C) A private repo must enable Actions first
- D) The first run needs workflow_dispatch

**Correct: B.** Your Git has no default branch name configured, so `git init` creates **`master`**. The push went to `master`, and the branch filter only accepts `main`, so the (perfectly valid) workflow never matches. Fix: `git branch -M main` then `git push -u origin main`, or start with `git init -b main`. A: `gh` pushes with normal Git, and those pushes trigger workflows. C: Actions are enabled by default on new repos, private ones included. D: `workflow_dispatch` is an extra, optional trigger. Push triggers don't need a manual run first.

---

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

**Correct: A.** `pytest` exits 1, so its step fails. The next step has the default condition `success()`, so `tests done` is **skipped**. The `if: always()` step still runs and prints `cleanup`. The job fails, so `deploy` (which `needs: test`) is **skipped**. B: steps after a failure don't run unless they have `if: always()` (or `failure()`). C: a job whose `needs` failed never starts. D forgets that `always()` steps run even after a failure.

---

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

**Correct: D.** Two layers work in order. First, **GitHub** replaces every `${{ … }}` in the script text, so bash receives `echo '$APP_ENV' 'ci'`. Then **bash** runs it, and **the outer quotes decide**: single quotes keep `$APP_ENV` literal. So it prints `$APP_ENV ci`. A forgets that bash never expands inside single quotes. B forgets that GitHub's substitution happens **before** bash and ignores shell quotes. C has both layers backwards.

---

**Q4.** A step is `run: pytest | tee test-output.txt` (no `shell:` key, Linux runner). Two tests fail. What happens to the step?
- A) It fails, since bash -e stops on any error
- B) It fails, since GitHub adds -o pipefail
- C) It passes: the pipe's exit code is tee's
- D) It fails, as tee passes on pytest's code

**Correct: C.** With no `shell:` key, GitHub runs `bash -e {0}`, **without `pipefail`**. A pipeline's exit code is **its last command's**, and `tee` succeeded (0), so the step passes even though pytest exited 1. A: `-e` reacts to the **pipeline's** exit code, which is tee's 0. B: `-o pipefail` is added only when you write `shell: bash` explicitly. That's the fix. D: `tee` just copies text and returns its own exit code.

---

**L1.** Write the full forms of **CI** and **CD**, and explain **Continuous Delivery vs Continuous Deployment** in contrast form.

**Model answer:** *"CI = Continuous Integration; CD = Continuous Delivery or Continuous Deployment. In Continuous Delivery, every change that passes the pipeline is automatically built, tested, and kept ready to release, but a human approves the final deployment to production, whereas in Continuous Deployment every change that passes all stages is released to production automatically, with no manual step."*

---

**L2.** Explain **job vs step** in GitHub Actions: where each one runs, and what they share.

**Model answer:** *"A job is a group of steps that runs on its own fresh runner VM, and jobs run in parallel unless `needs:` orders them, sharing files only through artifacts, whereas a step is a single command (`run`) or action (`uses`) inside a job; steps run in order on the same runner and share its files and workspace."*
