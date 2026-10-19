# Labs · Session 5 · Git & GitHub

Acronyms used: PR = Pull Request (a request on GitHub to merge one branch into another, with review) · CLI = Command-Line Interface · `gh` = the official GitHub CLI · PAT = Personal Access Token (a password-like token for GitHub) · SSH = Secure Shell · HEAD = "the commit/branch you are on right now".

---

## Lab 1 · Create a repo with a README, make 3 meaningful commits, push to GitHub

**Answer (what to write in the exam):**
```bash
mkdir todo-app && cd todo-app
git init                                   # (one-time setup: git config --global user.name/user.email)
echo "# Todo App - a tiny to-do list program" > README.md
git add README.md
git commit -m "Add README with project description"
echo 'print("Todo App started")' > app.py
git add app.py
git commit -m "Add app entry point"
echo "__pycache__/" > .gitignore
git add .gitignore
git commit -m "Add .gitignore for Python cache files"
git branch -M main
git remote add origin https://github.com/<your-username>/todo-app.git
git push -u origin main
```
Before the push, create an **empty** repo named `todo-app` on GitHub (web: **+ → New repository**, no README) or with `gh repo create todo-app --public`. Log in with `gh auth login`, an SSH key, or a PAT entered at the password prompt; **never** put a token inside the remote URL or a file.

**What each line does:**
- `git init`: turns the folder into a Git repository (creates the hidden `.git` folder).
- `git add <file>`: puts the file in the **staging area** (the list of changes going into the next commit).
- `git commit -m "msg"`: saves a snapshot of the staged changes with a message. Three separate commits = three meaningful steps in history.
- `.gitignore`: lists files Git should never track (here, Python cache files).
- `git branch -M main`: renames the current branch to `main` (`-M` = force rename, even if it was `master`).
- `git remote add origin <url>`: saves the GitHub repo address under the short name `origin`.
- `git push -u origin main`: uploads `main` to GitHub; `-u` (set upstream) links local `main` to `origin/main`, so later a plain `git push`/`git pull` works.

**Verify:** `git log --oneline` → 3 commits (newest first); `git remote -v` → `origin https://github.com/<you>/todo-app.git (fetch/push)`; `git status` → `Your branch is up to date with 'origin/main'`; the GitHub page shows the 3 files and README text.

**🧠 In simple words:** you make a folder into a Git repo, save your work in three small labelled snapshots (commits), then connect the repo to an empty GitHub repo and upload it. Small commits with clear messages make history easy to read and undo. In short: **init → add → commit (×3) → branch -M main → remote add → push -u.**

---

## Lab 2 · Feature branch → change → push → review diff → merge into main through a PR

**Answer (what to write in the exam):**
```bash
git switch -c feature/add-greeting              # (older Git: git checkout -b feature/add-greeting)
echo 'print("Hello! Welcome to Todo App")' >> app.py
git add app.py
git commit -m "Add welcome greeting"
git push -u origin feature/add-greeting
git diff main...feature/add-greeting            # review: what the feature adds since it left main
gh pr create --base main --head feature/add-greeting \
  --title "Add welcome greeting" --body "Prints a greeting when the app starts"
gh pr diff                                      # review the PR's diff
gh pr merge --merge --delete-branch             # merge PR into main, delete the branch
git switch main
git pull origin main
git log --oneline --graph
```

**Web-UI alternative to `gh` (equally valid in the exam):** after the push, open the repo on GitHub → click **Compare & pull request** → set **base: main**, **compare: feature/add-greeting** → add title/description → **Create pull request** → open the **Files changed** tab to review the diff (a reviewer can comment/approve) → **Merge pull request** → **Confirm merge** → **Delete branch**. Then locally: `git switch main && git pull`.

**What each line does:**
- `git switch -c <name>`: creates a new branch from the current commit and moves to it (`-c` = create).
- `>> app.py`: appends a line to the file (`>` would overwrite it).
- `git push -u origin feature/add-greeting`: uploads the branch to GitHub (main stays untouched).
- `git diff main...feature/add-greeting`: **three dots** = only the changes made on the feature branch since it split from main (exactly what the PR will show). Lines starting `+` are added, `-` removed. (Two dots `main..feature` in `git diff` compares the two branch tips directly.)
- `gh pr create --base main --head <branch>`: opens a PR asking to merge the feature branch into main.
- `gh pr diff` / `gh pr view`: show the PR's changes and status for review.
- `gh pr merge --merge --delete-branch`: merges with a merge commit (other options: `--squash`, `--rebase`) and deletes the branch on GitHub and locally.
- `git switch main && git pull origin main`: bring the merged result down to your local main.

**Verify:** `gh pr view` / the GitHub PR page shows **Merged**; `git log --oneline --graph` on main shows `Merge pull request #1 from <you>/feature/add-greeting` above "Add welcome greeting"; `cat app.py` contains the greeting line.

**🧠 In simple words:** you never work directly on main; you make a separate branch, commit there, push it, and open a Pull Request so the change can be reviewed (the diff) before it goes into main. After approval, GitHub merges it and you pull the updated main. In short: **branch → commit → push → PR → review diff → merge → pull main.**

---

## Lab 3 · Create a merge conflict (same line, two branches), resolve it, commit, verify history

**Answer (what to write in the exam):**
```bash
git switch main
echo "Welcome to Todo App" > greeting.txt
git add greeting.txt && git commit -m "Add greeting file"
git switch -c conflict-demo
echo "Welcome to Todo App - branch version" > greeting.txt
git commit -am "Change greeting on branch"
git switch main
echo "Welcome to Todo App - main version" > greeting.txt
git commit -am "Change greeting on main"
git merge conflict-demo                          # → CONFLICT
git status                                       # → both modified: greeting.txt
echo "Welcome to Todo App - merged version" > greeting.txt   # or edit the file by hand
git add greeting.txt                             # mark the conflict as resolved
git commit -m "Merge conflict-demo: resolve greeting conflict"
git log --oneline --graph --all
```

**What you see at the conflict:**
```text
$ git merge conflict-demo
Auto-merging greeting.txt
CONFLICT (content): Merge conflict in greeting.txt
Automatic merge failed; fix conflicts and then commit the result.

$ cat greeting.txt
<<<<<<< HEAD
Welcome to Todo App - main version
=======
Welcome to Todo App - branch version
>>>>>>> conflict-demo
```
- `<<<<<<< HEAD` to `=======` → **your** side (the branch you are on: main).
- `=======` to `>>>>>>> conflict-demo` → the **incoming** side (the branch being merged).
- To resolve: delete all three marker lines and keep the text you want (one side, the other, or a mix). The file must contain **no** `<<<<<<<`, `=======`, `>>>>>>>` lines.

**Final history:**
```text
$ git log --oneline --graph --all
*   c4f1a9e (HEAD -> main) Merge conflict-demo: resolve greeting conflict
|\
| * 7b2d3e0 (conflict-demo) Change greeting on branch
* | 5e8a1f2 Change greeting on main
|/
* 2a9c6d4 Add greeting file
* ...      (older commits from Labs 1–2 continue below; your hashes will differ)
```

**What each line does:**
- `git commit -am "msg"`: `-a` stages all changes to already-tracked files, then commits (saves a separate `git add`).
- `git merge conflict-demo`: tries to combine conflict-demo into main; both changed the **same line**, so Git stops and asks you to decide.
- `git status`: lists `Unmerged paths … both modified: greeting.txt`.
- `git add greeting.txt`: tells Git "this file is fixed". `git commit` then creates the **merge commit** (two parents). (`git merge --abort` cancels the merge instead.)
- `git log --oneline --graph --all`: one line per commit, with an ASCII graph showing the two branches splitting and joining.

**Verify:** `git status` → `nothing to commit, working tree clean`; `cat greeting.txt` → only the merged line, no markers; the graph shows a merge commit with two lines joining (`|\` … `|/`).

**🧠 In simple words:** a conflict happens when two branches change the same line and Git cannot know which one is right. Git marks both versions in the file; you pick the final text, remove the markers, stage the file and commit, which finishes the merge. In short: **merge → CONFLICT → edit file and remove markers → git add → git commit → git log --oneline --graph.**
