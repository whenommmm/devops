# Brief: model answers for the question-bank LABS (exam is TODAY; the student has ~3 hours)

The exam's questions come from `/Users/vanshsrivastava/DevOPS/repo-questions/devops-heros/DevOps-Exam-Practice-Quentions/`. Each file ends with 3 "Practical / Lab Questions". The student must be able to WRITE the commands/config for each lab in the exam. The student is a beginner (3rd-year college student, new to DevOps), so answers must be exact, short, and explained in plain words.

## Output
For each session file you are given, write ONE file: `/Users/vanshsrivastava/DevOPS/exam-bank/<same file name as the question file>` (e.g. `exam-bank/session-12-ingress-configmaps-secrets.md`). Do not edit the question files (they're in a separate git repo).

## Shape of each output file
```
# Labs · <session title>

## Lab 1 · <the task, restated in one short line>
**Answer (what to write in the exam):**
```bash
<commands, in order, with realistic names; ≤ ~15 lines>
```
(If the lab needs a manifest/Dockerfile/HCL/workflow/script, give it as a separate short block, block-style YAML, minimal but complete and valid.)
**What each line does:**
- `<command or key line>`: plain-words meaning + key flags (1 line each; merge trivial ones)
**Verify:** `<command>` → what you should see (1–2 lines).
**🧠 In simple words:** 2–3 plain sentences: what the whole thing does and why. End with "In short: …".

## Lab 2 · …
## Lab 3 · …
```
- Conceptual labs (draw a diagram / explain / compare / roadmap): give a compact model answer: a small ASCII diagram and/or a short table/bullets that would score full marks, then the 🧠 paragraph.
- Total length per lab: ~25–45 lines. Exact and exam-ready beats long.

## Rules
- Write the commands an examiner expects on a standard **Linux** lab machine / Kubernetes cluster (e.g. `ip addr`, `ss -tulpn`, `useradd`, `systemctl`, `journalctl`). If the student's Mac differs, add ONE short "(Mac: …)" note, nothing more.
- Kubernetes: use `kubectl` commands and short manifests; use `--dry-run=client -o yaml` generators where they save writing. Include verification (`get`, `describe`, `get endpoints`, `rollout status`, `logs`, `exec … -- env`).
- Never hard-code credentials in manifests/workflows (use Secrets / `${{ secrets.X }}`), since the labs say so.
- Define every acronym on first use; no game-dev analogies; no bare cross-references.
- Correctness matters most. You MAY validate safely: `kubectl --context minikube apply --dry-run=client -f <file>` (client dry-run only, never create anything), `bash -n script.sh`, `docker compose -f x config` in a scratch folder, `terraform fmt`/`validate` in a scratch folder under `/private/tmp/claude-501/-Users-vanshsrivastava-DevOPS/7b00419d-5c0a-448c-9109-e46490c11da0/scratchpad/`. Don't create cluster resources, containers, or cloud resources; don't touch the `kind-stockpilot` context; don't run git; don't push.
- Time limit: finish within ~10 minutes. Prefer writing over extensive testing.

## Also
Check the 10 MCQ answer keys in your files; if any key looks wrong, report it (don't edit).

When done, report: files written, labs per file, anything you validated, and any doubtful key.
