# Answer key · Session 01 · Git & GitHub

⚠️ Only open after writing your answers in `MY_ANSWERS.md`.

---

## Warm-up

**W1.** `curl -I http://site.com/old` returns `301` with `Location: https://site.com/new`. What's true?
- A) The page moved temporarily, so keep using /old
- B) The page moved permanently, so links should be updated to /new
- C) The server failed and redirected you to an error page
- D) You must log in before you can see /old

**Correct: B.** 301 = **moved permanently**. The `Location` header gives the new address. A describes **302** (temporary). C would be a 5xx code. D is the typical reason for a **302** redirect to a login page.

---

**W2.** `NAME=Riya`. What does `echo 'Hi "$NAME"'` print?
- A) `Hi "Riya"`
- B) `Hi Riya`
- C) `Hi "$NAME"`
- D) `Hi $NAME`

**Correct: C.** The **outer** quotes are **single**, so nothing inside is expanded: `$NAME` stays literally `$NAME`, and the inner `"` are printed as normal characters. A would need outer **double** quotes: `echo "Hi \"$NAME\""` or `echo "Hi '$NAME'"`.

---

**W3.** Which pair mostly uses UDP?
- A) DNS lookups and live video calls
- B) HTTPS websites and SSH logins
- C) Plain HTTP websites and DNS lookups
- D) SSH logins and live video calls

**Correct: A.** DNS questions are tiny and need speed, and live video would rather skip a lost frame than wait. HTTP, HTTPS, and SSH all need every byte to arrive in order, so they use **TCP** (Transmission Control Protocol). That rules out B, C, and D.

---

## Check 30

**30.1** Which statement is true?
- A) You need GitHub to make commits
- B) Git works offline; GitHub hosts repositories online
- C) Git and GitHub are two names for the same program
- D) GitHub is a command you run in the Terminal

**Correct: B.** Git is a tool on your computer that works with no internet. GitHub is an online hosting service. A is wrong because commits are made locally by Git. C confuses the tool with the website. D: the command is `git` (GitHub's own tool is called `gh`).

---

**30.2** You edit `app.txt` and run `git commit -m "update"` without `git add`. Git says `no changes added to commit`. Why?
- A) Commits need an internet connection
- B) The change was never staged
- C) The commit message is too short
- D) app.txt isn't tracked by GitHub

**Correct: B.** A commit saves **only what's in the staging area**. The change stayed in the working directory. Fix: `git add app.txt`, then commit (or `git commit -am "update"` for already-tracked files).

---

**30.3** What's inside `.git`, and what happens if you delete it?
**Model answer:** *"`.git` holds the repository's entire history (all commits, branches, and settings). If you delete it, the current files remain, but all the history is lost and the folder is no longer a Git repo."*

---

## Check 31

**31.1** `main` = A, B. `feature` adds C, D. `main` gets no new commits. `git switch main && git merge feature`. What happens?
- A) A new merge commit is created joining both lines
- B) Fast-forward: the `main` label moves to D, and no new commit is made
- C) A merge conflict, because two branches exist
- D) The `feature` branch is deleted automatically

**Correct: B.** Main didn't move, so Git just slides the label forward (a **fast-forward**). A happens only when **both** branches have new commits. C: conflicts need the **same lines changed differently**, not just two branches existing. D: branches are only deleted when you ask (`git branch -d`).

---

**31.2** Which command creates `fix-bug` and switches to it?
- A) `git branch fix-bug`
- B) `git switch fix-bug`
- C) `git switch -c fix-bug`
- D) `git merge fix-bug`

**Correct: C.** `-c` = create. A **creates** the branch but **doesn't switch** to it. B only switches to a branch that **already exists**. D merges.

---

## Check 32

**32.1** A teammate rebases the shared `develop` branch and force-pushes. Likely result?
- A) Nothing bad, because rebase is always safe
- B) Everyone else's copy no longer matches, causing confusing conflicts and duplicate commits
- C) Git automatically refuses to rebase shared branches
- D) The `develop` branch is deleted on GitHub

**Correct: B.** Rebase **rewrites** commits (new IDs), so everyone else still has the old ones. A ignores the golden rule. C: Git doesn't stop you (some GitHub settings can block force-pushes, but Git itself won't). D: the branch is overwritten, not deleted.

---

**32.2** Which part is the incoming branch's version?
**Model answer:** *"Below the `=======`, down to the `>>>>>>> branch-name` line. Above it, after `<<<<<<< HEAD`, is your current branch's version."*

---

## Check 33

**33.1** Which is true?
- A) `git fetch` immediately changes the files in your folder
- B) `git pull` is `git fetch` followed by a merge
- C) `git push` downloads your teammates' new commits
- D) `git clone` copies only the latest files, without history

**Correct: B.** A: fetch **only downloads** and doesn't touch your files. C: push **uploads**. D: clone copies the **whole history**.

---

**33.2** A bad commit is already pushed and shared. Safest undo?
- A) `git reset --hard HEAD~1`, then force-push
- B) `git revert <commit-id>`, then push
- C) Delete the repository and create it again
- D) `git stash`, then push

**Correct: B.** `revert` adds a **new** commit that undoes the bad one, so history isn't rewritten and teammates aren't disrupted. A rewrites shared history (the same danger as 32.1). C is extreme. D only affects **uncommitted** local changes. (`HEAD~1` means "one commit before HEAD".)

---

## Session quiz

**Q1.** `echo v1 > app.txt` → `git add app.txt` → `echo v2 > app.txt` → `git commit -m "first"`. What's in the commit?
- A) `v1`
- B) `v2`
- C) Both
- D) Nothing, the commit fails

**Correct: A.** `git add` copied the file **as it was at that moment** (`v1`) into the staging area. The later change to `v2` was **never staged**, so the commit saves `v1`, and `git status` would still show app.txt as modified. B is the common mistake: assuming commit takes the current file. D: the commit succeeds because something **was** staged.

---

**Q2.** `main` = A–B–E. `feature` from B = C–D. `git switch feature && git rebase main`. Result?
- A) A–B–C–D–E, with a merge commit on top
- B) A–B–E–C'–D', where C' and D' are new copies of C and D
- C) Unchanged, because rebase only changes `main`
- D) A–B–E only, because C and D are discarded

**Correct: B.** Rebase replays your commits **on top of** main's latest commit, creating **new copies** (new hashes), with no merge commit. A mixes up merge and rebase. C: rebase changes the **current** branch (feature), not main. D: your commits are kept, just re-created.

---

**Q3.** A reviewer asks for a change on your open PR. What do you do?
- A) Close the PR and open a new one for each change
- B) Merge the PR first, then fix the code in `main`
- C) Email the changed file to the reviewer
- D) Push a new commit to `feature-x`; the PR updates automatically

**Correct: D.** A PR tracks the **branch**, so new commits on it appear in the PR automatically. B skips the review, which is the whole point of a PR.

---

**Q4.** Which belongs in `.gitignore`?
- A) `app.py`
- B) `README.md`
- C) `.env`, containing the database password
- D) `Dockerfile`

**Correct: C.** Secrets must **never** be committed. Once pushed, they're in the history and visible to anyone with access. A, B, and D are normal project files that **should** be tracked.

---

**L1.** fetch vs pull
**Model answer:** *"`git fetch` downloads new commits from the remote but leaves your files and branch unchanged, whereas `git pull` downloads them and immediately merges them into your current branch (pull = fetch + merge)."*

---

**L2.** Same line changed in two branches: what happens, and the three steps?
**Model answer:** *"Git stops with a merge conflict and writes both versions into the file between `<<<<<<<`, `=======`, and `>>>>>>>` markers. (1) Edit the file to the correct final content and remove the markers. (2) `git add config.txt` to mark it resolved. (3) `git commit` to complete the merge."*
