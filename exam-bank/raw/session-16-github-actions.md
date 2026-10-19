# Labs · Session 16 · GitHub Actions

> GitHub Actions = GitHub's built-in CI/CD (Continuous Integration / Continuous Delivery). A **workflow** is a YAML file in `.github/workflows/`; it has triggers (`on:`), **jobs** (run on a **runner** VM chosen by `runs-on:`), and each job has **steps** (`uses:` = run a ready-made action, `run:` = run shell commands).
> Example app used below: a tiny Node.js app with `package.json` scripts `"build": "mkdir -p dist && cp index.js sum.js dist/"` and `"test": "node --test"`, plus a committed `package-lock.json`.

## Lab 1 · Workflow on push: checkout, install dependencies, build
**Answer (what to write in the exam):**
```bash
mkdir -p .github/workflows
nano .github/workflows/ci.yml            # paste the YAML below
git add .github/workflows/ci.yml
git commit -m "ci: add build workflow"
git push origin main                     # the push itself triggers the workflow
gh run list --limit 1                    # or: GitHub repo → Actions tab
gh run watch                             # follow it live
```
`.github/workflows/ci.yml`:
```yaml
name: CI

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build
```
**What each line does:**
- `on: push` (+ `branches: main`): run the workflow on every push to main.
- `jobs: build:` / `runs-on: ubuntu-latest`: one job called `build` on a fresh GitHub-hosted Ubuntu VM.
- `uses: actions/checkout@v4`: copies your repo code onto the runner (the VM starts empty).
- `uses: actions/setup-node@v4` + `node-version: 22`: installs Node 22; `cache: npm` caches downloads to speed up runs.
- `run: npm ci`: clean install of exact versions from `package-lock.json` (fails if the lock file is missing).
- `run: npm run build`: runs the build script, which creates `dist/`.
- (Python version: `actions/setup-python@v5` with `python-version: "3.12"`, then `pip install -r requirements.txt`.)

**Verify:** Actions tab → run "CI" has a green ✅ and the `build` job log shows each step passing; `gh run list` → `completed  success  CI  main  push`.

**🧠 In simple words:** Every time you push, GitHub starts a clean Linux machine, downloads your code, installs Node and your dependencies, and builds the app, so you know the code builds on a machine other than yours. In short: push → checkout → install → build, automatically.

## Lab 2 · Add a test step; show a failing test fails the run and a fixed test passes
**Answer (what to write in the exam):**
```bash
# 1) add the Test step (below) after Build in ci.yml
# 2) commit a WRONG test (expects 6) and push
git add . && git commit -m "test: add sum test (wrong on purpose)" && git push
gh run list --limit 1                    # → completed  failure  ❌
gh run view --log-failed                 # shows: ✖ adds 2 + 3 ... expected: 6, actual: 5
# 3) fix the test (expect 5) and push again
git commit -am "test: fix expected value" && git push
gh run list --limit 1                    # → completed  success  ✅
```
Step to add in `ci.yml` (after Build):
```yaml
      - name: Test
        run: npm test
```
`sum.test.js` (uses Node's built-in test runner, no extra packages):
```js
const test = require('node:test');
const assert = require('node:assert');
const sum = require('./sum');            // sum.js: module.exports = (a, b) => a + b;

test('adds 2 + 3', () => {
  assert.strictEqual(sum(2, 3), 6);      // WRONG on purpose → fails. Fix: 5
});
```
**What each line does:**
- `run: npm test`: runs `node --test`, which finds `*.test.js` files and runs them.
- A failing assertion makes `npm test` exit with code 1 → the step fails → the job and the whole run turn red ❌; later steps are skipped.
- `gh run view --log-failed`: prints only the log of the failed step (same as clicking the red step on the website).
- After fixing the expected value to 5, `npm test` exits 0 → every step green → run ✅.

**Verify:** Actions tab shows two runs: "test: add sum test" ❌ (Test step red, `fail 1`) and "test: fix expected value" ✅ (`pass 1, fail 0`).

**🧠 In simple words:** The test step is an automatic check on every push: if any test fails, its non-zero exit code makes GitHub mark the run as failed, so broken code is caught before anyone uses it. Fixing the code or test and pushing again turns it green. In short: exit code 1 = red ❌, exit code 0 = green ✅.

## Lab 3 · Upload a build artifact downloadable from the Actions run
**Answer (what to write in the exam):**
```bash
# add the upload step (below) as the LAST step of the build job, then:
git commit -am "ci: upload dist as artifact" && git push
gh run watch                                   # wait for ✅
gh run download <run-id> -n web-dist -D web-dist   # download into ./web-dist
ls web-dist                                    # index.js  sum.js
```
Step to add at the end of `ci.yml`:
```yaml
      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: web-dist
          path: dist/
          retention-days: 7
```
**What each line does:**
- `uses: actions/upload-artifact@v4`: zips files from the runner and stores them with the run (the runner VM is deleted after the job).
- `name: web-dist`: the artifact's name shown on the run page (must be unique within the run).
- `path: dist/`: what to upload (the build output).
- `retention-days: 7`: auto-delete after 7 days (default is 90).
- It is the last step, so it only runs if build AND tests passed (a failed step skips the rest).
- `gh run download <run-id> -n web-dist -D web-dist`: downloads that artifact into folder `web-dist` (get the id from `gh run list`).

**Verify:** open the run page → **Artifacts** section at the bottom → `web-dist` with a size and a download button; the job log shows `Artifact web-dist has been successfully uploaded!`.

**🧠 In simple words:** The runner machine is thrown away after the job, so anything it built is lost unless you save it. The upload-artifact step saves the build folder with the run so anyone can download exactly what CI built and tested. In short: build → test → upload `dist/` → download it from the run page.
