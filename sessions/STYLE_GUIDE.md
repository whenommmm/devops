# Style guide for writing session files (follow exactly)

## The student
- 3rd-year college student, **zero prior knowledge of DevOps tools**. Treat them as **new to tech**.
- Exam: Mon 19 Oct 2026. Expected format: **MCQs + debugging/"fix the code" questions**. Goal: **full marks**.
- Already learned (you can link back to these): DevOps lifecycle/CALMS/CI-CD basics, monolith vs microservices, Linux (files, permissions, processes, signals SIGTERM/SIGKILL, daemons, systemd, env vars, PATH, package managers), shell scripting (variables, quotes, `$( )`, args, if/`[ ]`, loops, exit codes, `set -e`, functions, cron), networking (IP, private ranges, CIDR, 127.0.0.1 vs 0.0.0.0, ports, TCP vs UDP, DNS records, HTTP methods + status codes, HTTPS/TLS/certificates, load balancer, reverse proxy, curl/dig/nc/lsof), Git (commits, staging, branches, merge vs rebase, conflicts, remotes, PRs, hooks), Docker (containers vs VMs, namespaces/cgroups, client/daemon/registry, image vs container = program vs process, lifecycle, `-p`/`-d`/`-e`, PID 1, Dockerfile instructions, layers/cache, CMD vs ENTRYPOINT, multi-stage, volumes vs bind mounts, user-defined networks + DNS by name, YAML, Docker Compose).

## The student's Mac (all practicals must work here)
- macOS, Apple Silicon (arm64), 16 GB RAM. **No Linux VM for the student, no cloud accounts.**
- Installed via Homebrew: **Docker CLI + Colima** (Docker runs in an Ubuntu VM; no Docker Desktop), docker compose, **minikube** (running, **docker driver**, as a container inside Colima), **kubectl** (context `minikube`), **Helm v4**, **Terraform v1.16**, `gh` (GitHub CLI, logged in), git, python3.
- A **LocalStack** container (an AWS emulator) runs on `127.0.0.1:4566`. It's usable for Terraform AWS practice.
- ⚠️ Tell the student to **never delete the `minikube` or `localstack` containers**.
- minikube tips: `minikube status`; `minikube start` if stopped; `minikube addons enable ingress` / `metrics-server`. With the docker driver on macOS, NodePort/Ingress aren't directly reachable from the Mac, so use `kubectl port-forward`, `minikube service <name> --url`, or `minikube tunnel` (explain whichever you use).
- Practice folder: `~/devops-practice/` (create subfolders per session).

## Writing rules (the student has complained about each of these, so they're strict)
1. **Explain every technical word where it first appears**, in plain words. **Give the full form of every acronym** on first use (e.g., "HPA (Horizontal Pod Autoscaler)", "etcd (pronounced 'et-cee-dee', from the Linux /etc folder + 'd' for distributed)").
2. Each concept starts with a **"Words you'll learn:"** list, then explains step by step: **problem first (a short story), then the idea, then how it works, then details.**
3. **Everyday analogies** (restaurant, post office, hotel, factory…). **Do NOT use game-development analogies.**
4. **Every command is broken into its parts**, and you show **roughly what output to expect** and what it means.
5. **Link back** to things the student already knows (e.g., "a Pod's IP is private, like the private ranges from the networking session").
6. **About 4 concepts per session.** Never split a concept into "A/B" parts. Number concepts with plain numbers (given in your task).
7. Every session has a **"➕ Small but important"** section (placed just before the quiz): commonly examined smaller topics, commands, and flags, as a table, plus 2 quick checks with answers. **Don't skip small topics** that exams or course assignments might ask about.
8. Depth must stay the same from the first concept to the last. No thinning out at the end.
9. Use tables and ASCII diagrams generously. Use the **exam-trap callouts** (⚠️) for commonly confused pairs.

## File structure (copy the shape of `02_docker-fundamentals.md` and `04_docker-networking-volumes-compose.md`)
```
# Session NN · Title (Day, Session N)
**Time:** · **Concepts:** a → b · **Answers:** `MY_ANSWERS.md` or say "<name> questions" · **Answer key:** `NN_slug_ANSWERS.md`
**Before you start:** (tool checks, e.g. `minikube status`)
## 🔁 Warm-up (3 MCQs from earlier topics and the weak list below)
## Concept X · Title   (Words you'll learn → Steps → 🛠 Do → ✅ Check X with 2–3 questions: MCQs + maybe 1 one-liner)
… (about 4 concepts)
## ➕ Small but important
## 📝 Session quiz (exactly 4 MCQs + 2 one-liners), from memory
```

## MCQ rules (strict; check every question before writing it)
1. **All four options roughly the same length** (within ~10 characters). If one option has a reason ("…, because …"), every option gets a reason of similar length.
2. **The correct option must NOT be the longest.** Across a quiz, make it the shortest or middle length at least half the time.
3. **No giveaway words** (e.g., don't write "the file containing the password"). The right option should look as ordinary as the others.
4. **Distractors must be genuinely tempting:** true of a neighbouring concept, half right, off by one detail, a common misconception, or a near-miss command/flag. No silly or opposite options.
5. **Vary the correct letter** (A/B/C/D) across each quiz.
6. Prefer **multi-step, scenario, "predict the output", "which is NOT", and debugging** stems over one-step recall. Questions should be **hard** (full-marks level), in the style of course assignments like: *"An engineer runs X and sees error Y. Which statement best describes the cause?"*

## Warm-up material: the student's weak spots (use 3 per session; rotate)
- Quotes: the OUTER quotes decide (`'…'` literal, `"…"` expands `$`, no quotes = split at spaces). The weakest topic.
- A function's exit code = its `return N` (or its last command); `$?`
- `git status` (which files) vs `git diff` (what changed); a PR updates when you push to its branch; conflict fix = edit → `git add` → `git commit`
- 301 (permanent) vs 302 (temporary); HTTP/HTTPS/SSH use TCP, DNS mostly UDP; 127.0.0.1 vs 0.0.0.0
- `>` inside `[ ]` is redirection; `a && b || c` isn't if/else; cron uses 24-hour time
- Stopped containers keep their name; `docker rmi` refuses while containers use the image
- Permission numbers (r-x = 5); umask on files starts from 666
- Plus the previous session's topics (e.g., the K8s II warm-up uses K8s I)

## Answer-key file (`NN_slug_ANSWERS.md`)
For **every** warm-up, check, and quiz question, in this exact format (the student insisted):
```
**Q1.** <full question, including any code>
- A) …
- B) …
- C) …
- D) …

**Correct: X.** <why it's right>. <why each tempting wrong option is wrong>
```
One-liners: question, then **Model answer:** in full-marks wording (use "X does…, whereas Y…" for differences).

## Verifying commands (allowed, carefully)
You may run commands to check syntax or real output (e.g., `kubectl apply --dry-run=client -f`, `terraform validate` in a temp folder, `kubectl explain`, `helm template`).
- If you create anything (pods, namespaces, files), **use names starting with `claude-check-` and delete them afterwards**.
- **Never touch** the `minikube`/`localstack` containers or other existing resources.
- **Never push** to GitHub, create GitHub repos, or change global config.

## LENGTH CAP (user, 2026-10-15; strict)
The user said sessions were far too long ("don't make it 500 lines long"). **Each session file must be at most ~330 lines** (target 280–330), i.e. roughly **50–70 lines per concept**. Keep every term defined (one line each), full forms, one analogy per concept, essential tables and diagrams, and the practical commands. Show only the 1–3 key lines of expected output. Cut repetition, long stories (2–3 sentences max), restated tables, and multiple analogies. "Small but important" is one table of at most ~15 rows.
