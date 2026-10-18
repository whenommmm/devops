# Session 12 · CI/CD and GitHub Actions · FAST (10-min read)
*How every push gets built, tested and released automatically, and how GitHub Actions runs that for you.*

## 1 · CI, CD and the pipeline
When developers merge weeks of separate work on one day, the code clashes and nobody knows whose change broke what. **CI (Continuous Integration)** fixes this: developers merge small changes into the main branch often, and every push or PR (pull request, a request to merge a branch) automatically triggers a build and automated tests, so a break shows up within minutes and points at one commit. **CD** then carries the tested change to users, and it has two meanings that differ only in the last step.
- **Continuous Delivery** means every change that passes the pipeline is ready to release, but a human approves the final step into production.
- **Continuous Deployment** means every change that passes the pipeline goes to production automatically, with no human step at all.
- A **pipeline** is an automated sequence of **stages** (for example build → test → package → deploy) that every change passes through, started by a **trigger** such as a push.
- **Fail fast** means quick checks go first, and once a stage fails the later stages never run, so nothing broken is deployed.
- A pipeline judges each step **only by its exit code**: 0 means the step succeeded, and any other number means it failed.
- CI only detects and reports problems; a developer reads the red ❌ log and fixes the code.

⚠️ **Exam trap:** Continuous Delivery keeps a human approval before production, whereas Continuous Deployment releases with no approval (automatic deploys to staging alone do not make it Continuous Deployment).
⚠️ **Exam trap:** CI trusts the exit code, not the text: printing "3 tests FAILED" then `exit 0` is green, whereas printing "all passed" then `exit 2` is red.

## 2 · GitHub Actions: workflows, jobs, steps and runners
**GitHub Actions** is GitHub's built-in CI/CD service: it runs your automation on GitHub's computers whenever something happens in your repository. Think of a restaurant kitchen: the workflow is the plan, jobs are separate stations with their own clean counters, and steps are one station's instructions in order.
- A **workflow** is one YAML (YAML Ain't Markup Language) file saved in exactly `.github/workflows/`, with the dot and the s; a file anywhere else is silently ignored.
- The `on:` key lists the **triggers**: `push` (commits are pushed), `pull_request` (a PR is opened or gets new commits), `workflow_dispatch` (a manual "Run workflow" button) and `schedule` (cron times, read in UTC, Coordinated Universal Time, not your local time).
- A **job** is a group of steps that runs on one runner, and jobs run **in parallel** unless `needs:` makes one wait for another.
- A **step** is one task inside a job, and steps run in order on the **same** runner, sharing its files.
- A **runner** is the machine that runs a job: a **GitHub-hosted runner** is a fresh VM (virtual machine) such as `ubuntu-latest`, created for each job and deleted afterwards, whereas a **self-hosted runner** is your own machine and is not wiped between jobs.

⚠️ **Exam trap:** `uses:` runs an **action** (a ready-made reusable step such as `actions/checkout`, which copies your repo's code onto the empty runner), whereas `run:` runs shell commands; a step has one of them, never both.
⚠️ **Exam trap:** steps in one job share a disk, whereas each job gets a brand-new machine, so a file made in one job does not exist in another; `needs:` only orders jobs and copies nothing.

## 3 · A workflow file, line by line
Learn this small file's shape, because "fix the workflow" questions test structure. Indent with spaces, never tabs, and start every list item with `-`.
```yaml
on:
  push:
    branches:
      - main
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: pytest --junitxml=reports/junit.xml
      - if: always()
        uses: actions/upload-artifact@v7
        with:
          name: test-report
          path: reports/
```
- The `on:` block means "run only when commits are pushed to `main`".
- `test:` names the job, `runs-on: ubuntu-latest` (required) picks a fresh Ubuntu Linux runner, and `steps:` is a list of `-` items.
- `uses: actions/checkout@v7` brings in the code, and `run: pytest …` runs the tests (pytest is Python's test tool) and writes a report file.
- `if: always()` uploads the report even when the tests failed, and `with:` passes the action its inputs.

More rules:
- Every step has a hidden default `if: success()`, so after a failure the next steps are skipped; `if: failure()` runs a step only when an earlier one failed, and `if: always()` runs it no matter what.
- `needs: test` on another job makes it wait for `test`, and if `test` fails that job is skipped, unless the job itself has `if: always()`.
- A **matrix** (`strategy:` → `matrix:` with a list such as Python `"3.11"` and `"3.12"`) runs the same job once per value, in parallel; two lists of two values give four jobs.
- `${{ … }}` is an **expression**: GitHub replaces it with its value before the step runs, and `run: |` lets one step hold several shell lines.

⚠️ **Exam trap:** a wrong folder, a non-matching branch filter (`master` when you push `main`) or a missing trigger gives no run at all, whereas a structure mistake (no `runs-on`, `uses` plus `run` in one step, `needs:` naming a missing job) shows a red "Invalid workflow file" error.

## 4 · Secrets, artifacts and the build & test pipeline
A password typed into the YAML gets committed to Git, where everyone with access can read it forever. A **secret** is an encrypted value that GitHub stores for you (added in the repo's Settings or with `gh secret set NAME`) which workflows can use but nobody can read back, like a hotel's safe-deposit box.
- A step reads a secret as `${{ secrets.NAME }}`, usually handed to the shell through an `env:` block.
- **Masking:** when a secret's exact value appears in a log, GitHub prints `***` instead, but a changed copy (for example base64-encoded, rewritten as letters and digits) is not recognised and leaks.
- An **artifact** is a file the pipeline keeps, such as a test report: `actions/upload-artifact` saves it, a later job in the same run fetches it with `actions/download-artifact`, and it stays 90 days by default.
- A typical **build & test pipeline** is checkout → `actions/setup-python` → `pip install -r requirements.txt` (pip is Python's package installer) → `pytest` (exit code 1 if any test fails) → upload the report with `if: always()`, and a deploy job with `needs:` runs only if all of that passed.

⚠️ **Exam trap:** an artifact keeps and shares outputs with people and later jobs of the same run, whereas a **cache** keeps downloads such as pip packages to speed up future runs.

## ⚡ Must-remember
| Term | Meaning |
|---|---|
| Continuous Delivery vs Deployment | human approves production vs fully automatic |
| Exit code 0 / non-zero | step passed / step failed (text is ignored) |
| `.github/workflows/` | the only folder GitHub reads workflow files from |
| `on:` | triggers: `push`, `pull_request`, `schedule` (UTC), `workflow_dispatch` |
| `needs:` | makes a job wait; without it jobs run in parallel |
| `uses:` vs `run:` | run an action vs run shell commands (one per step) |
| `if: always()` | run the step or job even after a failure |
| `${{ secrets.X }}` | reads a secret; its exact value logs as `***` |
| Artifact vs cache | keep outputs of this run vs speed up future runs |

## 📝 Quiz (5 MCQs; answer in chat as "1 B, 2 D, …")

**1.** A team's pipeline is BUILD → TEST → DEPLOY, and every change that passes goes to production with no human approval. In one run, the test script prints `All 40 tests passed`, then ends with `exit 2` because a temp-file clean-up command failed. What happens, and what is this practice called?
- A) Passes, as the log says all passed; Continuous Deployment
- B) Fails, and DEPLOY is skipped; this is Continuous Deployment
- C) Fails, and DEPLOY awaits approval; this is Continuous Delivery
- D) Passes, as only exit 1 means failure; Continuous Deployment

**2.** Job `package` has `needs: build`. A `run:` step in `build` builds `dist/app.zip` from the source code, and `build` succeeds. Then `package` runs `ls dist/app.zip` and fails with `No such file or directory`. Which change fixes it?
- A) Add actions/checkout as the first step of package
- B) Upload dist/ in build; read it via `${{ }}` in package
- C) Add `if: always()` to the ls step inside package
- D) Upload dist/ in build; download it in package

**3.** `pytest` finds one failing test. Which `echo` lines appear across the whole run?
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "start"
      - run: pytest
      - if: failure()
        run: echo "alert"
      - run: echo "done"
  report:
    needs: test
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: echo "report"
```
- A) start, alert, report: done is skipped after a failure
- B) start, alert, done, report: only the pytest step fails
- C) start, alert: report is skipped because test has failed
- D) start, done, report: alert would need `if: always()` too

**4.** An engineer pushes a commit to `main`, but the Actions tab shows no run at all, not even a failed one. Which of these is NOT a possible cause?
- A) The file was saved in `.github/workflow/` (no s)
- B) The workflow's branch filter lists only `master`
- C) The `test` job in it has no `runs-on:` line
- D) The file's only trigger is `workflow_dispatch`

**5.** A repo secret `DEPLOY_TOKEN` exists. What does the log of this step show?
```yaml
      - env:
          TOKEN: ${{ secrets.DEPLOY_TOKEN }}
        run: |
          echo "token: $TOKEN"
          echo "$TOKEN" | base64
```
- A) The real value twice, since `env:` removes the mask
- B) `token: ***`, then the base64 text in plain view
- C) `token: ***`, then `***` again on the base64 line
- D) Nothing, since GitHub blocks echo of a secret
