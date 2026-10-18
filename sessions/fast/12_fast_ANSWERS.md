# Answer key · Session 12 FAST · CI/CD and GitHub Actions

⚠️ Only open after answering.

---

**1.** A team's pipeline is BUILD → TEST → DEPLOY, and every change that passes goes to production with no human approval. In one run, the test script prints `All 40 tests passed`, then ends with `exit 2` because a temp-file clean-up command failed. What happens, and what is this practice called?
- A) Passes, as the log says all passed; Continuous Deployment
- B) Fails, and DEPLOY is skipped; this is Continuous Deployment
- C) Fails, and DEPLOY awaits approval; this is Continuous Delivery
- D) Passes, as only exit 1 means failure; Continuous Deployment

**Correct: B.** A pipeline judges a step only by its exit code, and any non-zero code (here 2) means failure, so TEST is red and, because of fail fast, DEPLOY never runs. Releasing every passing change with no human approval is Continuous Deployment. A trusts the printed text, which CI ignores. C names the wrong practice: this pipeline has no approval step (that would be Continuous Delivery), and a failed TEST never reaches the deploy stage anyway. D is off by one detail: every non-zero code is a failure, not just 1.

---

**2.** Job `package` has `needs: build`. A `run:` step in `build` builds `dist/app.zip` from the source code, and `build` succeeds. Then `package` runs `ls dist/app.zip` and fails with `No such file or directory`. Which change fixes it?
- A) Add actions/checkout as the first step of package
- B) Upload dist/ in build; read it via `${{ }}` in package
- C) Add `if: always()` to the ls step inside package
- D) Upload dist/ in build; download it in package

**Correct: D.** Each job runs on its own brand-new runner, and `needs:` only sets the order, so the zip made on `build`'s machine does not exist on `package`'s machine. The way to pass a file between jobs is an artifact: `actions/upload-artifact` in `build`, then `actions/download-artifact` in `package`. A brings in the repository's source code, but the zip was built during the run and is not in the repository. B is half right: an expression only pastes a text value into the step before it runs and cannot fetch files, so the download step is still missing. C changes only when the step runs, not which files are on the machine, so `ls` still fails.

---

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

**Correct: A.** `start` prints, then `pytest` exits 1 and its step fails. The `alert` step has `if: failure()`, so it runs precisely because an earlier step failed. The `done` step has the hidden default `if: success()`, so it is skipped. The `test` job has failed, but `report` has `if: always()` at job level, so it runs anyway and prints `report`. B forgets the default `success()` condition that skips steps after a failure. C is true of a dependent job without `if: always()`, but this one has it. D thinks `failure()` is not enough; it is exactly the condition for "run when an earlier step failed".

---

**4.** An engineer pushes a commit to `main`, but the Actions tab shows no run at all, not even a failed one. Which of these is NOT a possible cause?
- A) The file was saved in `.github/workflow/` (no s)
- B) The workflow's branch filter lists only `master`
- C) The `test` job in it has no `runs-on:` line
- D) The file's only trigger is `workflow_dispatch`

**Correct: C.** A missing `runs-on:` is a structure mistake: GitHub still finds and reads the file, and shows a red "Invalid workflow file" error, so a failed entry does appear. A is a real cause: GitHub only reads `.github/workflows/` with the s, so the file is silently ignored. B is a real cause: the file is valid but the filter never matches a push to `main`. D is a real cause: `workflow_dispatch` only adds the manual "Run workflow" button, so a push starts nothing.

---

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

**Correct: B.** Masking replaces the secret's exact value with `***` wherever it appears in the log, so the first line shows `token: ***`. The second line prints a base64-encoded copy, which is different text, so GitHub does not recognise it and the encoded secret leaks (anyone can decode it). A is wrong because passing a secret through `env:` is the normal way to use it, and masking still applies. C assumes masking catches changed copies, but it matches only the exact value. D is wrong because GitHub never blocks commands; it only hides the exact value in the output, which is why you should never echo secrets.

---
