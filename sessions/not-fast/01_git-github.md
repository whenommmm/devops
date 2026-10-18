# Session 01 · Git & GitHub ("Oct 6", Session 2)

**Time:** ~90 min · **Concepts:** 30 → 33 · **Answers go in:** `MY_ANSWERS.md` · **Answer key:** `01_git-github_ANSWERS.md` (open it only after answering)

---

## 🔁 Warm-up (your weak spots from last session)

**W1.** `curl -I http://site.com/old` returns `301` with the header `Location: https://site.com/new`. What's true?
- A) The page moved temporarily, so keep using /old
- B) The page moved permanently, so links should be updated to /new
- C) The server failed and redirected you to an error page
- D) You must log in before you can see /old

**W2.** `NAME=Riya`. What does `echo 'Hi "$NAME"'` print?
- A) `Hi "Riya"`
- B) `Hi Riya`
- C) `Hi "$NAME"`
- D) `Hi $NAME`

**W3.** Which pair of things mostly uses **UDP** (User Datagram Protocol)?
- A) DNS lookups and live video calls
- B) HTTPS websites and SSH logins
- C) Plain HTTP websites and DNS lookups
- D) SSH logins and live video calls

---

## Concept 30 · Version control, Git vs GitHub, repositories & commits

### The problem
Without a tool, people save versions of their work like this: `report_final.docx`, `report_final_v2.docx`, `report_REALLY_final.docx`. Now imagine:
- 10 developers editing the **same** code files at the same time
- needing to know **who** changed a line, **when**, and **why**
- something breaking today and needing to **go back** to yesterday's working version

Doing that by copying folders is impossible. You need **version control**.

### Version control
A **version control system (VCS)** is a tool that **records every change made to a set of files over time**. You can:
- see the full history
- go back to any earlier version
- let many people work on the same files in parallel without overwriting each other

### Git
**Git** is the most widely used version control system. It was created in 2005 by Linus Torvalds, the same person who created Linux.

Git is a **distributed** VCS: **every developer has a complete copy of the whole history on their own computer**. So Git works **offline**, and if one copy is lost, the others still have everything.

The older style is **centralised** (one central server holds the history, and everyone must connect to it). The best-known example is SVN (Subversion). Exam awareness only.

### GitHub
**GitHub** is a **website and online service** (owned by Microsoft) that **hosts Git repositories on the internet** and adds collaboration features on top:
- **pull requests**: asking for your changes to be reviewed and merged (Concept 33)
- **issues**: a to-do / bug list
- **GitHub Actions**: automation, which you'll use in the CI/CD session

Alternatives that do the same job: **GitLab**, **Bitbucket**.

| | **Git** | **GitHub** |
|---|---|---|
| What it is | A **tool** (program) installed on your computer | A **website/service** on the internet |
| Works offline? | ✅ Yes | ❌ No, needs the internet |
| Job | Tracks versions and history | Stores repositories online, shares them, adds teamwork features |
| Made by | Linus Torvalds (open source) | GitHub Inc. / Microsoft |

**Analogy:** Git is a **camera** that takes snapshots of your project. GitHub is an **online photo album** where you upload those snapshots to share them and let others comment.

### Repository
A **repository** (usually shortened to **repo**) is a **project folder that Git is tracking**. It contains:
- your normal files
- a **hidden folder called `.git`** where Git stores the **entire history**: every snapshot, every message, every branch

⚠️ If you delete the `.git` folder, your files stay, but **all the history is gone**, and the folder is no longer a repo.

### Commit
A **commit** is a **saved snapshot of the project at one moment**, plus:
- a **message** describing the change (`"Fix login button"`)
- the **author** and the **date**
- a unique ID called a **hash**: a long string like `a1b2c3d4e5…` that Git calculates from the content, like a fingerprint. You usually see only the first 7 characters: `a1b2c3d`.

### The three areas (very important)

```
 WORKING DIRECTORY  ──git add──►  STAGING AREA  ──git commit──►  REPOSITORY (.git)
 (your files as you            (a waiting area: the          (the permanent history
  edit them right now)          changes you've chosen          of saved snapshots)
                                for the NEXT commit)
```

- **Working directory:** the actual files you see and edit.
- **Staging area** (also called the **index**): a waiting area where you **choose which changes go into the next commit**. You don't have to commit everything you changed.
- **Repository:** the saved history.

**Analogy: packing a parcel.** Your room full of stuff is the **working directory**. The open box you're putting chosen items into is the **staging area**. Sealing the box, labelling it, and putting it on the shelf is the **commit**.

A file can be:
- **untracked**: Git has never been told about it
- **modified**: changed since the last commit
- **staged**: added to the box
- **committed**: safely saved in history

### The basic commands

| Command | What it does |
|---|---|
| `git init` | Start tracking this folder (creates `.git`) |
| `git status` | Show what's changed, what's staged, and what's untracked. **Run it constantly.** |
| `git add file.txt` / `git add .` | Move changes into the staging area (`.` = everything in this folder) |
| `git commit -m "message"` | Save the staged changes as a snapshot (`-m` = **m**essage) |
| `git log --oneline` | Show the history, one line per commit |
| `git diff` | Show changes **not yet staged** |
| `git diff --staged` | Show changes that **are** staged |

**HEAD** is Git's name for **"where you are right now"**: the commit (and branch) you currently have checked out. You'll see it in `git log`.

### 🛠 Do (~8 min)

**One-time setup.** Tell Git who you are (it's stamped on every commit), and make VS Code the editor Git opens when it needs you to type something:
```bash
git config --global user.name        # if this prints nothing, set it on the next line
git config --global user.name "Your Name"
git config --global user.email "you@example.com"     # use your GitHub email
git config --global core.editor "code --wait"
```
*(With `code --wait`, whenever Git needs you to edit a message, it opens a VS Code tab. Save it and **close the tab** to continue.)*

**Your first repo:**
```bash
mkdir -p ~/devops-practice/git-demo && cd ~/devops-practice/git-demo
git init -b main                 # -b main = name the first branch "main"
git status                       # nothing yet
echo "version 1" > app.txt
git status                       # app.txt is UNTRACKED
git add app.txt
git status                       # now STAGED ("Changes to be committed")
git commit -m "Add app.txt"
echo "version 2" >> app.txt
git diff                         # shows the new line (not staged yet)
git add app.txt
git commit -m "Update app.txt to version 2"
git log --oneline                # two commits, newest first
ls -a                            # see the hidden .git folder
```

### ✅ Check 30

**30.1** Which statement is true?
- A) You need GitHub to make commits
- B) Git works offline; GitHub hosts repositories online
- C) Git and GitHub are two names for the same program
- D) GitHub is a command you run in the Terminal

**30.2** You edit `app.txt` and run `git commit -m "update"` **without** running `git add`. Git says `no changes added to commit`. Why?
- A) Commits need an internet connection
- B) The change was never staged
- C) The commit message is too short
- D) app.txt isn't tracked by GitHub

**30.3** (one line) What's inside the hidden `.git` folder, and what happens if you delete it?

---

## Concept 31 · Branching & merging

### Branch
A **branch** is a **separate line of work**. It lets you make changes **without touching the main version** until you're ready.

Technically, a branch is just a **movable label that points to a commit**. Each new commit you make on that branch moves the label forward.

The default branch is called **`main`** (older repos call it `master`).

**Why branches:**
- Build a new feature without breaking the working version.
- Several people work on different things at the same time.
- Experiment safely: if it fails, just delete the branch.

**Analogy:** writing a story. Your branch is a **photocopy of the story** where you try a new ending. If the new ending is good, you **merge** it back into the main story. If not, you throw the copy away.

### Branch commands

| Command | What it does |
|---|---|
| `git branch` | List branches (`*` marks the one you're on) |
| `git switch -c feature-login` | **Create** a new branch **and** switch to it (`-c` = create). Older equivalent: `git checkout -b feature-login` |
| `git switch main` | Switch to an existing branch |
| `git merge feature-login` | Bring that branch's changes **into the branch you're currently on** |
| `git branch -d feature-login` | Delete a branch you've finished with (`-d` = delete) |

### Merging: two kinds

**1. Fast-forward merge.** If `main` **hasn't changed** since you created your branch, Git just **slides the `main` label forward** to your latest commit. No new commit is created.
```
Before:  main → B          feature → D        (history: A ─ B ─ C ─ D)
After:   main → D          (main simply moved forward)
```

**2. Three-way merge (merge commit).** If **both** `main` and your branch got new commits, Git creates a **new "merge commit"** that combines both. It's called "three-way" because Git looks at three points: the **common starting point** and the **two branch tips**.
```
          C ─ D        (feature)
         /     \
A ─ B ─ E ───── M      (main: M is the new merge commit joining both lines)
```

See the history as a picture with:
```bash
git log --oneline --graph --all
```

### 🛠 Do (~6 min)
```bash
cd ~/devops-practice/git-demo
git switch -c feature-login
echo "login page" > login.txt
git add login.txt && git commit -m "Add login page"
git switch main
ls                                 # login.txt is NOT here: it only exists on the branch
git merge feature-login            # fast-forward (main hadn't changed)
ls                                 # now it is
git log --oneline --graph --all

git switch -c feature-search
echo "search page" > search.txt
git add search.txt && git commit -m "Add search page"
git switch main
echo "home page" > home.txt
git add home.txt && git commit -m "Add home page"
git merge --no-edit feature-search # both moved → a merge commit (--no-edit = use the default message)
git log --oneline --graph --all    # see the fork and the join
```

### ✅ Check 31

**31.1** `main` has commits A, B. You create `feature`, add commits C and D. `main` gets **no** new commits. You run `git switch main` then `git merge feature`. What happens?
- A) A new merge commit is created joining both lines
- B) Fast-forward: the `main` label moves to D, and no new commit is made
- C) A merge conflict, because two branches exist
- D) The `feature` branch is deleted automatically

**31.2** Which command creates a branch called `fix-bug` **and** switches to it?
- A) `git branch fix-bug`
- B) `git switch fix-bug`
- C) `git switch -c fix-bug`
- D) `git merge fix-bug`

---

## Concept 32 · Rebase vs merge, and merge conflicts

### Rebase
**Rebase** means: **take your branch's commits and replay them on top of the latest `main`**, as if you'd started your work from there.

```
Before:                             After  git switch feature && git rebase main:
        C ─ D   (feature)
       /                            A ─ B ─ E ─ C' ─ D'   (feature)
A ─ B ─ E       (main)                      ↑
                                           main
```
- The result is a **straight line**: a "linear" history with **no merge commit**.
- **C' and D' are new copies** of C and D, with **new hashes**. Rebase **rewrites** commits.

### Merge vs rebase (exam favourite)

| | **Merge** | **Rebase** |
|---|---|---|
| History shape | Shows the real fork and join | One straight, tidy line |
| Creates a merge commit? | Yes (unless fast-forward) | No |
| Rewrites existing commits? | **No**: history is preserved exactly | **Yes**: your commits get new IDs |
| Safe on shared branches? | ✅ Yes | ⚠️ **No**: never rebase commits others already have |

**Analogy:** **merge** is two roads joined at a junction, so you can see where they split and met. **Rebase** is rebuilding your road so it starts at the end of the main road, so it looks like one straight road.

**The golden rule of rebase:** **never rebase commits that are already pushed and shared with others.** Their copies still have the old commits, and the new copies conflict with them, which creates confusion and duplicate commits. Rebasing rewrites history, so if you then upload it, you have to **force-push** (`git push --force`). Force-pushing **overwrites the branch on GitHub with your version**, wiping whatever was there.

Rebase is great for **tidying up your own private branch** before you share it.

Rebase commands:
- `git rebase main`: replay the current branch onto main
- `git rebase --continue`: carry on after you've fixed a conflict
- `git rebase --abort`: cancel and go back to how things were

### Merge conflicts
A **merge conflict** happens when **two branches changed the same line(s) of the same file in different ways**. Git can't guess which version you want, so it **stops and asks a human**. A conflict is **not an error**. It's Git asking you to decide.

Git writes **both** versions into the file, between **conflict markers**:
```
<<<<<<< HEAD
color = blue
=======
color = green
>>>>>>> feature-green
```
- Between `<<<<<<< HEAD` and `=======`: **your current branch's** version (HEAD = where you are).
- Between `=======` and `>>>>>>> feature-green`: the **incoming** branch's version (the one you're merging in).

**To resolve it:**
1. **Edit the file** to the final content you want (blue, green, or a mix), and **delete all three marker lines**.
2. `git add settings.txt`: tells Git "this file is resolved".
3. `git commit` to finish the merge (or `git rebase --continue` if you were rebasing).

`git status` lists conflicted files as **"both modified"**. To give up and go back: `git merge --abort`.

**How teams avoid conflicts:** merge small changes **often**. That's one of the reasons for Continuous Integration from Day 1.

*(VS Code highlights conflicts and shows clickable buttons: **Accept Current Change / Accept Incoming Change / Accept Both**.)*

### 🛠 Do (~8 min)

**Make a conflict on purpose:**
```bash
cd ~/devops-practice/git-demo
echo "color = red" > settings.txt
git add settings.txt && git commit -m "Add settings"

git switch -c feature-blue
echo "color = blue" > settings.txt
git commit -am "Make it blue"        # -a = stage all tracked, modified files, then commit

git switch main
git switch -c feature-green
echo "color = green" > settings.txt
git commit -am "Make it green"

git switch main
git merge --no-edit feature-blue     # fine (fast-forward)
git merge feature-green              # CONFLICT!
git status                           # "both modified: settings.txt"
code settings.txt                    # look at the markers; keep "color = green", delete marker lines, save
git add settings.txt
git commit --no-edit                 # finishes the merge
git log --oneline --graph --all
```

**Try a rebase:**
```bash
git switch -c feature-docs
echo "docs" > docs.txt
git add docs.txt && git commit -m "Add docs"
git switch main
echo "footer" > footer.txt
git add footer.txt && git commit -m "Add footer"
git switch feature-docs
git rebase main                      # replay "Add docs" on top of "Add footer"
git log --oneline --graph --all      # feature-docs is now a straight line above main
git switch main && git merge feature-docs   # fast-forward, because it's directly ahead now
```

### ✅ Check 32

**32.1** A teammate rebases the shared `develop` branch, which everyone has already pulled, then force-pushes it. What's the likely result?
- A) Nothing bad, because rebase is always safe
- B) Everyone else's copy no longer matches, causing confusing conflicts and duplicate commits
- C) Git automatically refuses to rebase shared branches
- D) The `develop` branch is deleted on GitHub

**32.2** (one line) In a conflict block, which part is the version from the branch you're **merging in**: above or below the `=======`?

---

## Concept 33 · Remotes, push/pull, forks, pull requests, and the basic workflow

### Remote
A **remote** is **a copy of your repository stored somewhere else**, usually on GitHub. Your local repo remembers it by a short name, by convention **`origin`**. `git remote -v` lists your remotes.

### Moving commits between your Mac and GitHub

| Command | Direction | What it does |
|---|---|---|
| `git clone <url>` | GitHub → your Mac | Download a **full copy** of a repo, **including all its history** |
| `git push` | your Mac → GitHub | **Upload** your new commits |
| `git fetch` | GitHub → your Mac | **Download** new commits from GitHub but **don't change your files yet**. It just updates Git's knowledge of what's on GitHub |
| `git pull` | GitHub → your Mac | **Download AND merge** them into your current branch. **`pull` = `fetch` + `merge`** |

**Analogy:** `fetch` is checking your mailbox and bringing the letters inside, still sealed. `pull` is bringing them in **and** opening them and filing them away.

`git push -u origin my-branch`: the first time you push a new branch, `-u` links your local branch to the GitHub branch, so later you can just type `git push`.

### Fork
A **fork** is **your own copy of someone else's repository, on GitHub** (under your account). You use it to contribute to projects you're **not allowed to write to directly**: you change your fork, then ask the original project to accept your changes.

| | Fork | Clone | Branch |
|---|---|---|---|
| Where the copy lives | On **GitHub**, under your account | On **your computer** | **Inside** the same repo |
| Copies what | A whole repo | A whole repo + history | A line of work within one repo |
| Typical use | Contributing to others' projects | Getting a repo onto your machine to work on | Developing a feature |

### Pull request (PR)
A **pull request** is **a request on GitHub to merge one branch into another**, usually your feature branch into `main`. It's the place where:
- teammates **review** the changes line by line and **comment**
- **automatic checks** (CI tests, from GitHub Actions) run and show ✅ or ❌
- someone clicks **Merge** once it's approved

If reviewers ask for changes, you just **push more commits to the same branch**, and the PR **updates automatically**.

*(GitLab calls the same thing a "merge request".)*

### The basic Git workflow (GitHub Flow)
The standard way teams work:
```
1. main is always working and deployable
2. git switch -c my-feature          → create a branch for your change
3. commit small changes              → git add / git commit
4. git push -u origin my-feature     → upload the branch
5. open a Pull Request               → review + automatic tests
6. merge into main                   → (and delete the branch)
7. main is deployed                  → this is where CI/CD takes over
```

### Short extras (MCQ-level)

| Command / file | Meaning |
|---|---|
| **`.gitignore`** | A file listing what Git should **never track**: logs, build output, `node_modules`, and **secrets like `.env` files with passwords** (a security must) |
| `git revert <commit>` | Creates a **new commit that undoes** an old one. History is kept, so it's **safe for shared branches** |
| `git reset` | **Moves the branch back** to an earlier commit, removing later commits from history. `--soft` keeps the changes staged; `--hard` **throws them away**. Use only for **local, unshared** work |
| `git stash` | Temporarily **put aside** uncommitted changes (to switch branches), and bring them back later with `git stash pop` |
| `git tag v1.0` | Put a permanent **label** on a commit, usually a release version |

### 🛠 Do (~10 min): a real pull request on your GitHub

```bash
cd ~/devops-practice/git-demo
gh repo create git-demo --private --source=. --remote=origin --push
git remote -v                        # origin → your new GitHub repo
git switch -c add-readme
echo "# Git demo" > README.md
echo "Practice repo for my DevOps exam." >> README.md
git add README.md && git commit -m "Add README"
git push -u origin add-readme
gh pr create --title "Add README" --body "My first pull request"
gh pr view --web                     # opens the PR in your browser: look at the "Files changed" tab
gh pr merge --merge --delete-branch  # merge it, and delete the branch
git switch main
git pull                             # bring the merged result down to your Mac
git log --oneline --graph --all
```
*(`gh` is GitHub's official command-line tool. You're already logged in, so you don't need to set anything up.)*

### ✅ Check 33

**33.1** Which statement is true?
- A) `git fetch` immediately changes the files in your folder
- B) `git pull` is `git fetch` followed by a merge
- C) `git push` downloads your teammates' new commits
- D) `git clone` copies only the latest files, without history

**33.2** You committed a bad change that's **already pushed and shared** with the team. What's the safest way to undo it?
- A) `git reset --hard HEAD~1`, then force-push
- B) `git revert <commit-id>`, then push
- C) Delete the repository and create it again
- D) `git stash`, then push

---

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory

**Q1.** In a new repo, you run these commands in order:
```bash
echo v1 > app.txt
git add app.txt
echo v2 > app.txt
git commit -m "first"
```
What content of `app.txt` is saved in the commit?
- A) `v1`
- B) `v2`
- C) Both `v1` and `v2`
- D) Nothing, because the commit fails

**Q2.** History: `main` is A–B–E. `feature` branched from B and has C–D. You run `git switch feature` then `git rebase main`. Which describes `feature` afterwards?
- A) A–B–C–D–E, with a merge commit on top
- B) A–B–E–C'–D', where C' and D' are new copies of C and D
- C) Unchanged, because rebase only changes `main`
- D) A–B–E only, because C and D are discarded

**Q3.** You open a PR from `feature-x` into `main`. A reviewer asks you to change one function. What do you do?
- A) Close the PR and open a new one for each change
- B) Merge the PR first, then fix the code in `main`
- C) Email the changed file to the reviewer
- D) Push a new commit to `feature-x`; the PR updates automatically

**Q4.** Which file should be listed in `.gitignore`?
- A) `app.py`
- B) `README.md`
- C) `.env`, containing the database password
- D) `Dockerfile`

**L1.** Explain the difference between `git fetch` and `git pull` in the contrast form ("fetch does…, whereas pull…").

**L2.** Two branches changed the same line of `config.txt` differently. What does Git do when you merge, and what three steps do you take to finish the merge?

---

## ➕ Extra (from the course assignment): Git hooks

A **hook** is a script Git runs automatically at a certain moment. Hooks live in `.git/hooks/` and must be executable.
- **`pre-commit`** runs before a commit is created, on your machine. **Exit 0 → the commit goes ahead; non-zero → the commit is cancelled.**
- `commit-msg` checks the message. `pre-push` runs before a push. **`pre-receive`** runs on the **server** when a push arrives and can reject the **push**.
- Secret scanners (gitleaks, detect-secrets) install as a **pre-commit hook**: if they find a password or key, they print "Possible secret detected!" and `exit 1`, so the commit fails. This is DevSecOps "shift-left".
- Plain Git has no built-in secret scanning. `.gitignore` only lists files to skip; it never inspects file contents.
- Hooks are not shared through push/clone (they live in `.git/`), so teams also run the same checks in CI.
