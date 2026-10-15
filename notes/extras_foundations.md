# ➕ Small but important: Intro, Linux, Shell, Networking, Git

These are short topics that the subtopic list didn't name but that exams and assignments often ask about. Read one section at a time and try each 🛠 command. **~45 min total.** Quick checks are at the bottom (answers right under them, so cover them up first).

---

## 1 · DevOps roles (Intro lecture)

| Role | What they mainly do |
|---|---|
| **Developer** | Writes the application code |
| **DevOps engineer** | Builds and maintains the **automation**: CI/CD pipelines, infrastructure as code, containers. **Bridges** Dev and Ops |
| **SRE** (**Site Reliability Engineer**) | Keeps production **reliable** using software-engineering methods: sets reliability targets, handles incidents, automates away repetitive manual work |
| **Platform engineer** | Builds an **internal platform** (shared tools and templates) so developers can deploy by themselves |
| **Cloud engineer** | Designs and runs **cloud infrastructure** (AWS, Azure, GCP) |
| **Release manager** | Plans and coordinates **releases** |
| **QA / test engineer** | Writes **automated tests** (QA = Quality Assurance) |
| **DevSecOps / security engineer** | Puts **security checks into the pipeline** (scanning code, images, secrets) |

**Two terms SREs use:**
- **SLO** (**Service Level Objective**): a reliability **target**, e.g., "99.9% of requests succeed".
- **Error budget:** the failure that target **allows** (the other 0.1%). If you use it up, you slow down releases and focus on stability.

**DevOps vs SRE** (an exam favourite): **DevOps is a culture and set of practices. SRE is one specific way to implement it**, with concrete rules like SLOs and error budgets. The famous line: *"SRE implements DevOps."*

**12-factor app** (awareness): a well-known list of 12 rules for building cloud-friendly apps. The one you already know: **store configuration in environment variables, not in the code.**

---

## 2 · Linux extras

### 2a · Links: hard vs soft (symbolic)
A **link** is a second name that points to a file.
- **Soft link** (**symbolic link**, **symlink**): a **shortcut that points to a path**. If the original file is deleted, the symlink **breaks** (it points to nothing). It can point to folders and to other disks. Made with **`ln -s target linkname`**.
- **Hard link:** **another name for the very same file data**. Deleting the original name doesn't delete the data while another hard link still exists. It can't point to folders. Made with **`ln target linkname`**.

`ls -l` shows a symlink with an arrow: `mylink -> /path/to/target`, and the permission string starts with **`l`**.

🛠
```bash
cd ~/devops-practice && echo hello > original.txt
ln -s original.txt shortcut.txt && ls -l shortcut.txt     # l… shortcut.txt -> original.txt
cat shortcut.txt                                          # hello
rm original.txt && cat shortcut.txt                       # error: the symlink is now broken
rm shortcut.txt
```

### 2b · SSH and SSH keys
**SSH** (**Secure Shell**) = logging in to a **remote** computer's terminal over an **encrypted** connection, on **port 22** (TCP).
```bash
ssh user@203.0.113.10          # log in to a server
scp file.txt user@server:/tmp/ # copy a file to a server (scp = secure copy)
```
**SSH keys** let you log in **without a password**, more safely:
- `ssh-keygen` creates a **key pair**:
  - a **private key** (`~/.ssh/id_ed25519`): **secret, never share it**. Its permissions should be **600**.
  - a **public key** (`~/.ssh/id_ed25519.pub`): safe to share.
- You put your **public key** into the server's **`~/.ssh/authorized_keys`** file. When you connect, the server checks that you hold the matching private key.
- **Analogy:** the public key is a **padlock** you hand out; only your private **key** opens it.
- GitHub uses the same idea: you add your **public** key to GitHub to push over SSH.

Common error: `Permissions … are too open` → the private key must be `chmod 600`.

### 2c · Disk, memory, and load (the "Logs & Basic Troubleshooting" subtopic)

| Command | Answers | Notes |
|---|---|---|
| `df -h` | **Disk free**: how full each disk is | `-h` = **h**uman-readable (GB/MB). Fixes "No space left on device" investigations |
| `du -sh *` | **Disk usage**: how big each item in this folder is | `-s` = summary, `-h` = human-readable. Finds what's filling the disk |
| `free -h` | How much **memory** (RAM) is used and free | Linux only (on Mac, use Activity Monitor or `top`) |
| `top` / `htop` | Live **CPU and memory** use per process | `htop` = a friendlier version |
| `uptime` | How long the machine has been on, plus the **load average** | Load average = how many processes are waiting for the CPU, over the last 1, 5, and 15 minutes. If it's consistently higher than the number of CPU cores, the machine is overloaded |
| `dmesg` | **Kernel messages** (hardware problems, the out-of-memory killer) | **OOM** = **Out Of Memory**. The kernel's **OOM killer** kills a process when RAM runs out: `dmesg \| grep -i oom` |
| `journalctl -u nginx` | Logs of one systemd service | `-f` to follow live |
| `tail -f /var/log/syslog` | The system log, live | On Red Hat systems it's `/var/log/messages` |

**A troubleshooting routine:**
1. Is the service running? (`systemctl status`)
2. What do the logs say? (`journalctl`, `tail`)
3. Is the disk full? (`df -h`)
4. Is memory or CPU exhausted? (`free -h`, `top`)
5. Is the port listening? (`ss -tulpn` on Linux, `lsof -i` on Mac)

🛠 (Mac): `df -h`, `du -sh ~/devops-practice/*`, `uptime`, `top -l 1 | head -12`

### 2d · Other commands that appear in questions

| Command | Does | Example |
|---|---|---|
| `find` | Search for files by name, size, or age | `find /var/log -name "*.log" -size +100M` |
| `sort` / `uniq` | Sort lines / remove repeated adjacent lines (`uniq -c` counts them) | `sort access.log \| uniq -c` |
| `cut` | Pick out columns | `cut -d: -f1 /etc/passwd` (field 1, split on `:`) |
| `awk` | Process text **by columns** | `awk '{print $1}' access.log` (print the 1st column) |
| `sed` | **Find-and-replace** text in a stream | `sed 's/old/new/g' file.txt` (`g` = every match) |
| `tar` | Pack or unpack an **archive** (many files in one) | `tar -czf backup.tar.gz folder/` (c = create, z = gzip-compress, f = file); `tar -xzf …` (x = extract) |
| `nohup cmd &` | Keep a command running after you log out | `nohup ./server.sh &` |
| `jobs` / `fg` / `bg` | List background jobs / bring one to the front / resume one in the background | |
| `which` / `whereis` | Where a command's program file is (PATH!) | `which python3` |

**Special permissions** (awareness): the **sticky bit** (on `/tmp`) means users can delete only their **own** files in a shared folder. **setuid** makes a program run with its **owner's** powers (e.g., `passwd`).

---

## 3 · Shell scripting extras

### 3a · Output streams and `/dev/null`
Every command has **three streams** (numbered channels):

| Number | Name | What it is |
|---|---|---|
| 0 | **stdin** (standard input) | What goes **in** (keyboard, or a pipe) |
| 1 | **stdout** (standard output) | Normal output |
| 2 | **stderr** (standard error) | **Error messages** |

```bash
ls /nope 2> errors.txt          # send only ERRORS into a file
ls /nope > out.txt 2>&1         # send errors to the same place as normal output (both into out.txt)
ls /nope &> all.txt             # bash shortcut for "both into the file"
command > /dev/null             # throw normal output away
command > /dev/null 2>&1        # silence everything
```
**`/dev/null`** is a special "black hole" file: whatever you write to it disappears. It's used to silence output.
⚠️ **Order matters:** `> file 2>&1` sends both to the file, but `2>&1 > file` does **not** (stderr still goes to the screen).

### 3b · `case` (a neater multi-way if)
```bash
case "$1" in
  start)   echo "Starting…" ;;
  stop)    echo "Stopping…" ;;
  *)       echo "Usage: $0 {start|stop}"; exit 1 ;;   # * = anything else
esac
```
`;;` ends each branch, and `esac` (case backwards) closes it, like `fi` for `if`.

### 3c · Arrays (short)
```bash
SERVERS=("web1" "web2" "web3")
echo "${SERVERS[0]}"            # web1 (counting starts at 0)
for s in "${SERVERS[@]}"; do echo "$s"; done    # every item
echo "${#SERVERS[@]}"           # 3 = how many items
```

### 3d · Here-documents
`cat > file <<'EOF' … EOF` writes several lines into a file (you used it for the Git hook). `EOF` is just a marker word. Quoting it (`'EOF'`) stops `$variables` inside from being expanded. Quotes again!

### 3e · Debugging a script
`bash -x script.sh` (or `set -x` inside the script) prints **every command before it runs**, with variables filled in. It's the fastest way to see where a script goes wrong.

---

## 4 · Networking extras

### 4a · The OSI model (7 layers): which layer is which
The **OSI model** (**Open Systems Interconnection**) is a standard way of describing networking as **7 layers**, from the physical wire up to the app. Exams ask "which layer does X belong to?"

| # | Layer | What it handles | Examples |
|---|---|---|---|
| 7 | **Application** | What apps use | **HTTP, HTTPS, DNS, SSH, FTP** |
| 6 | Presentation | Data format, **encryption** | TLS/SSL (often placed here), JPEG |
| 5 | Session | Opening and closing conversations | |
| 4 | **Transport** | Delivery between programs, **ports** | **TCP, UDP** |
| 3 | **Network** | Addressing and routing between networks | **IP**, routers, ICMP (ping) |
| 2 | Data link | Delivery on the local network | **MAC addresses**, switches |
| 1 | Physical | Cables, radio signals | Ethernet cable, Wi-Fi radio |

**Memory trick (top → bottom): "All People Seem To Need Data Processing."**
The simpler **TCP/IP model** has 4 layers: **Application** (OSI 5–7), **Transport** (4), **Internet** (3), **Network Access/Link** (1–2).

**Load balancers:**
- **L4** (layer 4) load balancers route by **IP and port** only.
- **L7** load balancers can route by **HTTP details** like the path (`/api`) or hostname. **Kubernetes Ingress is L7.**

### 4b · Subnet masks
Another way to write CIDR (from the networking session):

| CIDR | Subnet mask |
|---|---|
| `/8` | `255.0.0.0` |
| `/16` | `255.255.0.0` |
| `/24` | **`255.255.255.0`** |
| `/32` | `255.255.255.255` |

Each `255` = one fixed number. For `/24`, there are 256 addresses, but **254 are usable for devices**: the first is the **network address** and the last is the **broadcast address** (which means "everyone on this network").

### 4c · MAC address, DHCP, NAT, firewall, forward proxy
- **MAC address** (**Media Access Control**): the **hardware ID** burned into a network card (like `a4:83:e7:…`). It's used on the **local** network (layer 2). IP addresses are used **across** networks.
- **DHCP** (**Dynamic Host Configuration Protocol**): **automatically gives a device an IP address** when it joins a network. That's how your Mac got its `192.168.x.x` from the Wi-Fi router.
- **NAT** (**Network Address Translation**): lets **many private IPs share one public IP**. The router rewrites addresses as traffic goes out and comes back. (Docker's `-p` uses a form of NAT.)
- **Firewall:** filters traffic by rules ("allow port 443 from anywhere; block port 22 except from the office").
- **Forward proxy vs reverse proxy:**
  - A **forward proxy** sits **in front of users** and makes requests **on their behalf** (e.g., a company proxy that filters employees' web access). The **website** sees the proxy, not the user.
  - A **reverse proxy** sits **in front of servers** and receives requests **on their behalf** (nginx, Ingress). The **user** sees the proxy, not the servers.


### 4d · `ss` / `netstat`: listing ports and connections (asked in the course assignment)
`ss` = **socket statistics** (Linux). `netstat` = **network statistics** (older; same flags). On a Mac, use `lsof`.

| Flag | Meaning |
|---|---|
| `-t` | TCP only |
| `-u` | UDP only |
| `-l` | **listening** sockets only |
| `-a` | **all** (listening + established connections) |
| `-n` | **numeric**: show port numbers and IPs, not names (8000 instead of `irdmi`) |
| `-p` | show the **process** (name + PID) using each socket (needs sudo) |
| `-K` | **kill** matching sockets (dangerous) |

- Listening TCP ports with process names, numeric → **`ss -tlnp`**
- TCP **and** UDP → **`ss -tulpn`** (also `netstat -tulpn`)
- Mac: `lsof -iTCP -sTCP:LISTEN -P -n`

**Technique:** map each requirement to a letter; reject options with a missing or extra letter.

---

## 5 · Git extras

| Command / feature | What it does |
|---|---|
| **Git hooks** | Scripts Git runs at certain moments (`pre-commit`, `commit-msg`, `pre-push`, server-side `pre-receive`). A non-zero exit code **blocks** the action. Secret scanners use pre-commit. See the end of `sessions/01_git-github.md` |
| `git cherry-pick <commit>` | **Copy one specific commit** from another branch onto the current branch (e.g., grab just one bug fix) |
| `git restore file.txt` | **Throw away** unstaged changes to a file (go back to the last committed version) |
| `git restore --staged file.txt` | **Unstage** a file (take it out of the box, keep the changes) |
| `git reflog` | A log of **everywhere HEAD has been**, including commits "lost" after a `reset --hard`. Used to **recover** them |
| `git blame file.txt` | Shows **who last changed each line**, and in which commit |
| `git show <commit>` | Shows **one commit's** details and changes |
| `git rm` / `git mv` | Delete / rename a file **and** stage that change |
| `git config --global` vs `--local` | Settings for **all** your repos vs **this** repo only |
| **Detached HEAD** | When you check out a **specific commit** instead of a branch. New commits there belong to no branch, so create one (`git switch -c new-branch`) if you want to keep them |
| **Squash merge** | GitHub merge option that combines all of a PR's commits into **one** commit on main |
| **Branch protection rules** | GitHub settings that block pushing straight to `main`, and require PR reviews and passing CI checks before merging |
| **CODEOWNERS** | A file naming who **must review** changes to certain folders |
| **GitHub Issues** | GitHub's to-do / bug tracker, linked to PRs |
| HTTPS vs SSH remotes | `https://github.com/u/r.git` (log in with a token) vs `git@github.com:u/r.git` (log in with an SSH key) |

---

## ✅ Quick checks (cover the answers first)

1. You delete a file that a symlink points to. What happens to the symlink?
2. Which file on a server lists the public keys allowed to log in?
3. `./deploy.sh > /dev/null 2>&1`: what appears on screen?
4. An app crashed, and `dmesg` shows "Out of memory: Killed process". What happened?
5. Which OSI layer do TCP and UDP belong to? And HTTP?
6. What's `/24` as a subnet mask, and how many usable host addresses does it have?
7. You need just one bug-fix commit from a teammate's branch, not the whole branch. Which command?
8. After `git reset --hard`, you realise you lost a commit. Which command helps you find it?

**Answers:** (1) It breaks and points to nothing. (2) `~/.ssh/authorized_keys`. (3) Nothing: both normal output and errors go to `/dev/null`. (4) RAM ran out, and the kernel's OOM killer killed the process. (5) TCP/UDP = layer 4, Transport. HTTP = layer 7, Application. (6) `255.255.255.0`, with 254 usable hosts. (7) `git cherry-pick <commit>`. (8) `git reflog`.
