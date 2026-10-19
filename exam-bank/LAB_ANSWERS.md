# Lab ANSWERS · all 57 labs, in session order

*Solve them first in [LAB_QUESTIONS.md](LAB_QUESTIONS.md). Each answer: the commands/config to write, what each line does, how to verify, and a plain summary.*


## Session 1 · DevOps intro & roadmap

### Lab 1 · Draw a CI/CD workflow from code commit to deployment (build, test, package, deploy, monitor)
**Scenario used:** an online shop web app kept in a Git repo on GitHub.
CI = Continuous Integration (every push is built and tested automatically). CD = Continuous Delivery/Deployment (every tested build is automatically made ready for release / released).

**Answer (what to draw in the exam):**
```text
 Developer
    │  git commit + git push  (or opens a Pull Request)
    ▼
 [0 SOURCE]   Git repo (GitHub) ── push event triggers the pipeline
    │
    ▼
 [1 BUILD]    install dependencies, compile the code          ┐
    │                                                          │  CI
    ▼                                                          │
 [2 TEST]     unit tests + lint + security scan                │
    │         any failure → pipeline STOPS, developer notified ┘
    ▼
 [3 PACKAGE]  docker build → tag image shop:1.4.0 → push to registry
    │
    ▼
 [4 DEPLOY]   deploy to staging → smoke test → (approval) → production   CD
    │
    ▼
 [5 MONITOR]  metrics, logs, alerts (Prometheus + Grafana)
    │
    └──── feedback (bugs, alerts, slow pages) ────▶ back to Developer
```
**What each stage does:**
| Stage | What happens | Example tool | Output |
|---|---|---|---|
| Build | fetch code, install dependencies, compile | GitHub Actions / Jenkins | a build that compiles |
| Test | run automated tests and code checks | pytest / Jest, SonarQube | pass or fail report |
| Package | wrap the app + its dependencies into one versioned artifact | Docker, image registry | `shop:1.4.0` image |
| Deploy | release the same artifact to staging, then production | kubectl / Helm / Argo CD | app live for users |
| Monitor | watch health, errors, response time; alert on problems | Prometheus, Grafana | dashboards + alerts |

- **Fail fast:** a failed stage stops the pipeline, so broken code never reaches users.
- **Build once, deploy many:** the same image goes to staging and to production.
- **Continuous Delivery** = a human clicks "approve" before production; **Continuous Deployment** = production release is fully automatic.

**🧠 In simple words:** Every time a developer pushes code, a pipeline automatically builds it, tests it, packs it into a container image, deploys it, and then watches it in production. If any step fails, the pipeline stops and tells the developer, and monitoring feeds problems back so the next change fixes them. In short: commit → build → test → package → deploy → monitor → feedback, all automatic.

### Lab 2 · Monolith vs microservices for an online shop: 2 operational benefits + 2 challenges of microservices
**Answer (what to write in the exam):**
```text
 MONOLITH                                MICROSERVICES
 ┌──────────────────────────────┐              [ API Gateway ]
 │ ONE app, ONE deploy           │       ┌──────────┬─────┴────┬───────────┐
 │ UI | Catalog | Cart |         │    Catalog     Cart      Orders     Payment
 │ Orders | Payment              │    service    service    service    service
 └──────────────┬───────────────┘      │          │          │           │
          one shared database         DB         DB         DB          DB
```
| Point | Monolith | Microservices |
|---|---|---|
| Code & deploy | one codebase, one deploy unit | many small services, each deployed on its own |
| Scaling | copy the whole app | scale only the busy service (e.g. Catalog) |
| Failure | one bug can crash the whole shop | failure stays mostly inside one service |
| Tech stack | one language/framework | each service can choose its own |
| Ops effort | simple: one app, one log, one pipeline | complex: many pipelines, network calls, logs |
| Best for | small team, early product | big team, high traffic, fast-changing features |

**2 operational benefits of microservices**
1. **Independent deployment:** the Payment team can release a fix without redeploying Catalog or Cart, so releases are smaller, faster and less risky.
2. **Independent scaling + fault isolation:** on a sale day you scale only Catalog/Cart; if Payment crashes, customers can still browse and add to cart.

**2 challenges of microservices**
1. **Operational complexity:** many services means many pipelines, containers, configs and service-to-service network calls (needs Docker, Kubernetes, service discovery, retries/timeouts).
2. **Monitoring and debugging are harder:** one customer order passes through 4 services, so you need centralised logging, metrics and distributed tracing to find which one failed. (Also: keeping data consistent across separate databases, e.g. order saved but payment failed.)

**🧠 In simple words:** A monolith is the whole shop in one program, easy to start but every change redeploys everything and one bug can take down the whole site. Microservices split the shop into small services (catalog, cart, orders, payment) that deploy and scale separately, at the cost of much more infrastructure and harder troubleshooting. In short: microservices buy independent deploys and scaling, and you pay for it in operational complexity.

### Lab 3 · One-page DevOps roadmap for a small team that deploys manually
**Answer (what to write in the exam):**
```text
 TODAY: code on laptops → copy files to server by hand (FTP/SSH) → hope it works
 GOAL : push code → tested automatically → deployed automatically → monitored

 Phase 1 ──▶ Phase 2 ──▶ Phase 3 ──▶ Phase 4 ──▶ Phase 5
 Version     Automated   Container-  CI/CD       Monitoring
 control     tests       ization     pipeline    + alerts
 (wk 1-2)    (wk 3-4)    (wk 5-6)    (wk 7-8)    (wk 9-10)
```
| # | Practice | What to do | Tool | Why this comes at this position |
|---|---|---|---|---|
| 1 | Version control | put all code + config in Git; branches + Pull Request (PR) reviews; protect `main` | Git, GitHub | single source of truth with history and rollback; every later step is triggered by Git |
| 2 | Automated tests | write unit tests for key flows; run them with one command | pytest / Jest | a safety net that catches bugs early; automating deploys without tests only ships bugs faster |
| 3 | Containerization | write a Dockerfile; same image on laptop, test and prod | Docker | removes "works on my machine"; gives one versioned artifact to deploy |
| 4 | CI/CD | on every push: build → test → build image → deploy to staging → prod | GitHub Actions | removes manual, error-prone deploys; fast, repeatable releases |
| 5 | Monitoring | collect metrics + logs, dashboards, alerts on errors/downtime | Prometheus, Grafana | find problems before users do; feedback for the next change |

**How the team measures progress** (DORA = DevOps Research and Assessment metrics): deployment frequency ↑, lead time for changes ↓, change failure rate ↓, MTTR (Mean Time To Recovery) ↓.

**🧠 In simple words:** The roadmap moves the team from hand-copied deployments to an automated flow, one step at a time, where each step needs the one before it: Git first because everything starts from code history, tests next so automation has a safety net, Docker so every environment is identical, then a CI/CD pipeline to automate build-test-deploy, and finally monitoring to see how production behaves. In short: Git → tests → Docker → CI/CD → monitoring, each one making the next safe.

---

## Session 2 · Linux

### Lab 1 · Make a project folder (app, logs, config), then list, copy, move, search and delete files
**Answer (what to write in the exam):**
```bash
mkdir -p ~/project/{app,logs,config}           # create project + 3 subfolders
cd ~/project
echo 'print("hello")' > app/main.py            # create sample files
printf 'INFO started\nERROR db timeout\n' > logs/app.log
echo "port=8080" > config/app.conf
touch app/notes.txt                            # empty file
ls -la ; ls -R                                 # LIST: details + hidden, then recursive
cp config/app.conf config/app.conf.bak         # COPY a file
cp -r config config_backup                     # COPY a whole folder
mv app/notes.txt logs/                         # MOVE a file (mv also renames)
grep -rn "ERROR" .                             # SEARCH text inside files
find . -name "*.conf"                          # SEARCH files by name
rm config/app.conf.bak                         # DELETE a file
rm -r config_backup                            # DELETE a folder and its contents
```
**What each line does:**
- `mkdir -p ~/project/{app,logs,config}`: `-p` creates parent folders and gives no error if they exist; `{a,b,c}` (brace expansion) makes all three in one go.
- `echo ... >` / `printf ... >`: write text into a new file (`>` overwrites, `>>` appends); `touch`: create an empty file (or update its timestamp).
- `ls -la`: long format (permissions, owner, size, date) including hidden `.` files; `ls -R`: list every subfolder too.
- `cp src dst`: copy a file; `cp -r`: copy a directory recursively.
- `mv src dst`: move or rename (no `-r` needed for folders).
- `grep -rn "ERROR" .`: `-r` search all files under `.`, `-n` show line numbers (add `-i` to ignore case).
- `find . -name "*.conf"`: find files by name pattern starting at the current folder.
- `rm file`: delete a file (no recycle bin); `rm -r dir`: delete a folder; `rmdir dir` deletes only an empty folder.

**Verify:** `ls -R ~/project` → `app: main.py`, `config: app.conf`, `logs: app.log notes.txt`; `grep -rn ERROR .` → `./logs/app.log:2:ERROR db timeout`.

**🧠 In simple words:** We build a small project tree with one command, put a few sample files in it, and then use the core file commands: `ls` to see, `cp` to duplicate, `mv` to relocate, `grep`/`find` to search by content or by name, and `rm` to remove. In short: mkdir → touch/echo → ls → cp → mv → grep/find → rm.

### Lab 2 · Create user examuser + group devops, add the user to the group, set ownership and permissions on a project directory
**Answer (what to write in the exam):**
```bash
sudo groupadd devops                            # create the group
sudo useradd -m -s /bin/bash examuser           # create user with home dir + bash shell
sudo passwd examuser                            # set password interactively (never hard-code it)
sudo usermod -aG devops examuser                # ADD user to devops (keep other groups)
id examuser                                     # check membership
sudo mkdir -p /opt/project
sudo chown -R examuser:devops /opt/project      # owner = examuser, group = devops
sudo chmod -R 770 /opt/project                  # rwx owner, rwx group, nothing for others
sudo chmod g+s /opt/project                     # new files inherit group devops
ls -ld /opt/project
sudo -u examuser touch /opt/project/test.txt    # test: examuser can write
```
**What each line does:**
- `groupadd devops`: creates a new group (stored in `/etc/group`).
- `useradd -m -s /bin/bash examuser`: `-m` creates `/home/examuser`, `-s` sets the login shell (stored in `/etc/passwd`).
- `passwd examuser`: prompts for a password, so it is never written into a script.
- `usermod -aG devops examuser`: `-G` sets supplementary groups, `-a` = append. Without `-a` the user is removed from every other group.
- `chown -R user:group dir`: change owner and group, `-R` = for everything inside too.
- `chmod 770`: digits are owner/group/others; r=4, w=2, x=1, so 7 = rwx and 0 = no access. On a directory, x means "can enter it".
- `chmod g+s`: setgid bit on a directory, so files created inside belong to group `devops` automatically.
- `sudo -u examuser touch ...`: run a command as examuser to prove the permissions work.

**Verify:** `id examuser` → `uid=1001(examuser) gid=1001(examuser) groups=1001(examuser),1002(devops)`; `ls -ld /opt/project` → `drwxrws--- 2 examuser devops ... /opt/project`.
(Mac: there is no `useradd`/`groupadd`; practise inside a Linux VM or `docker run -it --rm ubuntu bash`, where you are root so drop `sudo`.)

**🧠 In simple words:** We create a team group and a user, put the user in the group, and hand the project folder to that user and group. Permissions 770 let the owner and the team read, write and enter the folder while everyone else is locked out, and the setgid bit keeps new files in the team's group. In short: groupadd → useradd → usermod -aG → chown → chmod.

### Lab 3 · Check a Linux service's status, read its logs, and diagnose a simulated failure
**Service used:** nginx (any service works the same way: `ssh`, `cron`, `docker`).
**Answer (what to write in the exam):**
```bash
systemctl status nginx --no-pager                  # 1. state: active (running)?
echo "broken_line;" | sudo tee /etc/nginx/conf.d/bad.conf   # 2. SIMULATE: bad config
sudo systemctl restart nginx                       #    -> "Job for nginx.service failed"
systemctl status nginx --no-pager                  # 3. Active: failed (Result: exit-code)
journalctl -u nginx -n 30 --no-pager               # 4. last 30 log lines of this service
journalctl -xeu nginx --since "10 min ago"         #    jump to the end, with explanations
sudo nginx -t                                      # 5. nginx's own config test -> names file + line
sudo tail -n 20 /var/log/nginx/error.log           #    the app's own log file
sudo rm /etc/nginx/conf.d/bad.conf                 # 6. FIX the cause
sudo systemctl restart nginx
systemctl is-active nginx                          # 7. -> active
ss -tulpn | grep ':80'                             #    nginx is listening on port 80 again
sudo systemctl enable nginx                        #    start automatically at boot
```
**What each line does:**
- `systemctl status <svc>`: shows Active state (active/inactive/failed), main PID (process ID) and the latest log lines.
- `sudo tee file`: writes text into a root-owned file (plain `>` would fail because the redirect is not run by sudo).
- `journalctl -u nginx`: logs for one systemd unit; `-n 30` last 30 lines, `-f` follow live, `--since` time filter, `-x` add explanations, `-e` jump to the end, `--no-pager` print without paging (`-xeu` is exactly what systemd suggests after a failed restart).
- `nginx -t`: tests config syntax; here it prints `unknown directive "broken_line" in /etc/nginx/conf.d/bad.conf:1`.
- `systemctl restart / is-active / enable`: restart the service, print its one-word state, start it at boot.
- `ss -tulpn`: lists listening sockets: TCP (Transmission Control Protocol) `-t`, UDP (User Datagram Protocol) `-u`, listening `-l`, owning process `-p`, numeric ports `-n`.

**Diagnosis notes to write:** Symptom: site down, `status` = failed. Evidence: journal shows `[emerg] unknown directive "broken_line"`. Cause: bad config line. Fix: remove the line, `nginx -t` OK, restart. Proof: `is-active` = active, port 80 listening.
**Verify:** `systemctl status nginx` → `Active: active (running)`; `curl -I http://localhost` → `HTTP/1.1 200 OK`.
(Mac: no systemd; use `brew services list`, `launchctl list | grep <name>` and `log show --last 10m`.)

**🧠 In simple words:** systemd runs services, `systemctl` asks it whether a service is running and starts/stops it, and `journalctl` shows what the service logged. We break nginx on purpose with a bad config line, see it fail in `status`, read the exact error in the logs, confirm it with `nginx -t`, fix it and prove it is back. In short: status → logs → test config → fix → restart → verify.

---

## Session 3 · Shell scripting

Exit-code rule used in all three scripts: `0` = success, `1` = called wrongly (usage error), `2` = bad input or a problem found. Error messages go to stderr with `>&2`.

### Lab 1 · Script that takes a directory path and prints how many files and subdirectories it has; handle a missing/invalid argument
**Answer (what to write in the exam):**
```bash
cat > count_items.sh      # paste the script below, then Ctrl+D
chmod +x count_items.sh
./count_items.sh /etc     # valid directory
./count_items.sh          # missing argument -> usage, exit 1
./count_items.sh /nope    # invalid directory -> error, exit 2
echo $?                   # show the exit code of the last run
```
```bash
#!/bin/bash
# count_items.sh - count files and subdirectories in a directory
# Usage: ./count_items.sh <directory>

if [ $# -ne 1 ]; then                       # exactly one argument?
  echo "Usage: $0 <directory>" >&2
  exit 1                                    # 1 = called wrongly
fi

DIR="$1"
if [ ! -d "$DIR" ]; then                    # exists AND is a directory?
  echo "Error: '$DIR' is not a valid directory" >&2
  exit 2                                    # 2 = invalid input
fi

FILES=$(find "$DIR" -mindepth 1 -maxdepth 1 -type f | wc -l)
DIRS=$(find "$DIR" -mindepth 1 -maxdepth 1 -type d | wc -l)

echo "Directory      : $DIR"
echo "Files          : $FILES"
echo "Subdirectories : $DIRS"
exit 0
```
**What each line does:**
- `#!/bin/bash`: shebang, tells the OS (Operating System) to run the file with Bash.
- `$#` = number of arguments, `-ne 1` = "not equal to 1"; `$0` = script name, `$1` = first argument.
- `[ ! -d "$DIR" ]`: true if the path is NOT a directory (covers "does not exist" and "is a file"); quotes keep paths with spaces safe.
- `find "$DIR" -mindepth 1 -maxdepth 1 -type f`: list items directly inside DIR (not DIR itself, not deeper) that are regular files; `-type d` = directories. Hidden items are counted too.
- `| wc -l`: count the lines, i.e. the items; `$( … )` stores a command's output in a variable.
- `>&2` sends the message to stderr; `exit N` sets the script's exit code.

**Verify:** `./count_items.sh /etc` → `Files : 90`, `Subdirectories : 110` (numbers vary); `./count_items.sh; echo $?` → `Usage: ./count_items.sh <directory>` then `1`.
(Mac: works as is; BSD `wc` pads the number with spaces, which is harmless. Remove `-maxdepth 1` to count recursively.)

**🧠 In simple words:** The script first checks that it got exactly one argument and that the argument is a real directory, and stops with a helpful message and a non-zero exit code if not. Then it uses `find` to list the files and the folders directly inside and `wc -l` to count them. In short: validate the input, count with find | wc -l, print, exit 0.

### Lab 2 · Script that checks a supplied file exists and whether it is readable, writable and executable
**Answer (what to write in the exam):**
```bash
chmod +x check_file.sh
touch demo.txt && chmod 640 demo.txt     # sample file: rw- r-- ---
./check_file.sh demo.txt                 # R yes, W yes, X no
./check_file.sh check_file.sh            # all YES (script is executable)
./check_file.sh missing.txt ; echo $?    # error, exit 2
```
```bash
#!/bin/bash
# check_file.sh - report whether a file exists and its r/w/x permissions
# Usage: ./check_file.sh <file>

if [ $# -ne 1 ]; then
  echo "Usage: $0 <file>" >&2
  exit 1
fi

FILE="$1"
if [ ! -e "$FILE" ]; then
  echo "Error: '$FILE' does not exist" >&2
  exit 2
fi

echo "Exists     : YES ($FILE)"
if [ -r "$FILE" ]; then echo "Readable   : YES"; else echo "Readable   : NO"; fi
if [ -w "$FILE" ]; then echo "Writable   : YES"; else echo "Writable   : NO"; fi
if [ -x "$FILE" ]; then echo "Executable : YES"; else echo "Executable : NO"; fi
exit 0
```
**What each line does:**
- `[ ! -e "$FILE" ]`: `-e` = exists (any type); use `-f` instead if it must be a regular file, not a folder.
- `-r` / `-w` / `-x`: true if the CURRENT user can read / write / execute the file.
- `if …; then …; else …; fi`: one-line if/else so each check prints its own clear YES or NO.

**Verify:** `./check_file.sh demo.txt` →
`Exists : YES (demo.txt)`, `Readable : YES`, `Writable : YES`, `Executable : NO`; then `chmod +x demo.txt` and re-run → `Executable : YES`.
(Note: run as a normal user; root always gets YES for read and write.)

**🧠 In simple words:** After checking the argument, the script stops if the file does not exist; otherwise it runs three separate tests and prints a YES/NO line for read, write and execute permission. The answers are for whoever runs the script, which is why `chmod` changes the result. In short: check arg → `-e` exists → `-r`, `-w`, `-x` each with if/else.

### Lab 3 · Reusable system-health script (variables, conditions, loops, functions) reporting disk, memory and uptime
**Answer (what to write in the exam):**
```bash
chmod +x health.sh
./health.sh            # default: warn when a disk is >= 80% full
./health.sh 50         # custom threshold
./health.sh abc        # bad argument -> usage, exit 1
echo $?                # 0 = healthy, 2 = a disk is over the threshold
```
```bash
#!/bin/bash
# health.sh - reusable system health report (disk, memory, uptime)
# Usage: ./health.sh [disk-alert-percent]     default: 80

THRESHOLD="${1:-80}"        # use $1 if given, else 80
STATUS=0                    # 0 = healthy, 2 = a disk is over the threshold

if ! [[ "$THRESHOLD" =~ ^[0-9]+$ ]] || [ "$THRESHOLD" -gt 100 ]; then
  echo "Usage: $0 [disk-alert-percent 0-100]" >&2
  exit 1
fi

header() {
  echo; echo "===== $1 ====="
}

check_disk() {
  header "Disk usage (alert at ${THRESHOLD}%)"
  while read -r fs size used avail pct mount; do
    if [ "${pct%\%}" -ge "$THRESHOLD" ]; then
      echo "WARNING  $mount is $pct full ($fs)"
      STATUS=2
    else
      echo "OK       $mount is $pct full ($fs)"
    fi
  done < <(df -hP | grep '^/dev/')
}

check_memory() {
  header "Memory"
  if command -v free > /dev/null; then
    free -h                               # Linux
  else
    top -l 1 | grep PhysMem               # macOS has no 'free'
  fi
}

check_uptime() {
  header "Uptime and load average"
  uptime
}

echo "Health report for $(hostname) at $(date)"
for check in check_disk check_memory check_uptime; do
  $check
done

echo
if [ "$STATUS" -eq 0 ]; then echo "RESULT: HEALTHY"; else echo "RESULT: WARNING"; fi
exit "$STATUS"
```
**What each line does:**
- `THRESHOLD="${1:-80}"`: **variable** with a default, so the script is reusable with any limit.
- `[[ "$THRESHOLD" =~ ^[0-9]+$ ]]`: **condition** with a regular expression: only digits allowed.
- `header()`, `check_disk()`, …: **functions**, each does one job; `$1` inside a function is the function's own argument.
- `df -hP | grep '^/dev/'`: disk usage, `-h` human sizes (G/M), `-P` one line per disk in fixed columns; grep keeps real disks only.
- `while read -r fs size used avail pct mount`: **loop** that splits each df line into named variables.
- `${pct%\%}`: strip the trailing `%` from `85%` so `-ge` (greater or equal) can compare numbers.
- `done < <(…)`: feed df into the loop without a pipe, so `STATUS=2` set inside the loop is still visible after it.
- `free -h`: RAM (Random Access Memory) total/used/free; `uptime`: how long the system has run + load average (1, 5, 15 min).
- `for check in …; do $check; done`: **loop** that calls every check function in turn; `exit "$STATUS"` lets cron or CI react to warnings.

**Verify:** `./health.sh` → `OK / is 26% full (/dev/sda1)`, a `free -h` table (`Mem: 7.7Gi 2.1Gi …`), an `uptime` line, `RESULT: HEALTHY`; `echo $?` → `0`. `./health.sh 1` → `WARNING …`, `RESULT: WARNING`, exit `2`.
(Mac: no `free`; the script falls back to `top -l 1 | grep PhysMem`, or use `vm_stat`.)

**🧠 In simple words:** The script keeps its settings in variables, puts each health check (disk, memory, uptime) in its own function, loops over the disks to compare each one against a threshold, and loops over the functions to run them all. It prints a clear report and exits 0 when healthy or 2 when a disk is too full, so it can be reused from cron or a pipeline. In short: variables + functions + loops + if, producing a disk/memory/uptime report with a meaningful exit code.

---

## Session 4 · Networking

Acronyms used: IP = Internet Protocol · DNS = Domain Name System (turns names into IP addresses) · TCP = Transmission Control Protocol (reliable, ordered) · UDP = User Datagram Protocol (fast, no delivery guarantee) · ICMP = Internet Control Message Protocol (what `ping` uses) · HTTP/HTTPS = HyperText Transfer Protocol (Secure = encrypted with TLS, Transport Layer Security) · CIDR = Classless Inter-Domain Routing (the `/24` notation) · TTL = Time To Live · PID = Process ID.

---

### Lab 1 · Show the IP address, subnet, default route and listening ports

**Answer (what to write in the exam):**
```bash
ip addr show            # short form: ip a   → IP address + subnet prefix of each interface
ip -4 addr show eth0    # only the IPv4 address of one interface (name may be ens33 / enp0s3)
ip route show           # routing table; the "default" line is the default gateway
ip route show default   # only the default route
sudo ss -tulpn          # all listening TCP/UDP ports + which program owns each one
```

**How to read the output (write this as your "interpretation"):**
```text
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> ...
    inet 127.0.0.1/8 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.1.25/24 brd 192.168.1.255 scope global dynamic eth0
```
- `eth0` = the network card; `UP` = it is switched on. `lo` = loopback (the machine talking to itself, 127.0.0.1).
- `inet 192.168.1.25/24` → **IP address** = 192.168.1.25; **/24 prefix** = subnet mask 255.255.255.0, so the first 24 bits (192.168.1) are the **network part**. Network = 192.168.1.0/24 → 254 usable host addresses (.1 to .254).
- `brd 192.168.1.255` = broadcast address (sends to every host in the subnet). `dynamic` = address came from DHCP (Dynamic Host Configuration Protocol).

```text
$ ip route
default via 192.168.1.1 dev eth0 proto dhcp metric 100
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.25
```
- `default via 192.168.1.1` → **default gateway** = 192.168.1.1 (the router). Any traffic not for our own subnet is sent there.
- Second line: hosts in 192.168.1.0/24 are reached **directly** on eth0, no router needed.

```text
$ sudo ss -tulpn
Netid State   Local Address:Port   Peer Address:Port  Process
tcp   LISTEN  0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=812,fd=3))
tcp   LISTEN  127.0.0.1:5432       0.0.0.0:*          users:(("postgres",pid=990,fd=5))
udp   UNCONN  127.0.0.53%lo:53     0.0.0.0:*          users:(("systemd-resolve",pid=601,fd=12))
```
- Flags: `-t` TCP, `-u` UDP, `-l` only listening sockets, `-p` show process name/PID (needs `sudo`), `-n` show numbers (port 22) not names (ssh).
- `0.0.0.0:22` → SSH (Secure Shell) is listening on **all** interfaces, so other machines can connect. `127.0.0.1:5432` → PostgreSQL accepts **local-only** connections. `udp ... :53` → local DNS resolver.

**Verify:** `ip a` shows an `inet x.x.x.x/NN` line on your main interface; `ip route` shows exactly one `default via …` line; `ss -tulpn` shows `LISTEN` rows.
(Mac: `ifconfig`, `netstat -rn`, `lsof -iTCP -sTCP:LISTEN -n -P`.)

**🧠 In simple words:** `ip a` tells you "who am I on the network" (your IP and how big your subnet is), `ip route` tells you "where do I send traffic that is not local" (the default gateway), and `ss -tulpn` tells you "which programs are waiting for connections, on which ports". These three are the first things you check on any server. In short: **ip a = my address + subnet, ip route = my gateway, ss -tulpn = my open ports.**

---

### Lab 2 · Troubleshoot a web service with ping, ss and curl (reachability vs closed port vs HTTP error)

**Answer (what to write in the exam):**
```bash
# Setup: a simple web server on the target machine (192.168.1.50), port 8080
python3 -m http.server 8080 &

# Step 1 – Is the host reachable at all? (network layer)
ping -c 4 192.168.1.50
# Step 2 – Is anything listening on the port? (run ON the server)
sudo ss -tulpn | grep :8080
# Step 3 – Can I connect, and what HTTP status comes back? (application layer)
curl -v http://192.168.1.50:8080/
curl -I http://192.168.1.50:8080/
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.1.50:8080/missing.html

# Simulate the 3 failures:
#  a) wrong/unreachable IP:     ping -c 4 192.168.1.99
#  b) closed port:              kill %1   (stop the server), then curl again
#  c) HTTP error:               curl -I http://192.168.1.50:8080/missing.html  → 404
```

**Decision table (the key part of the answer):**

| What you see | Diagnosis | Where the problem is |
|---|---|---|
| `ping`: `100% packet loss` or `Destination Host Unreachable`; `curl`: `Connection timed out` / `No route to host` | **Reachability problem** (host down, wrong IP, bad route, firewall dropping packets) | Network (IP/routing) |
| `ping` OK (`0% packet loss`), but `curl`: `Failed to connect … Connection refused`, and `ss` shows **nothing** `LISTEN` on :8080 | **Closed port** (service stopped / crashed / listening on a different port or only on 127.0.0.1) | Transport (TCP port) |
| `curl` connects (`* Connected to 192.168.1.50 port 8080`) but status is **4xx** (e.g. 404 Not Found, 403 Forbidden) or **5xx** (500 Internal Server Error, 503 Service Unavailable) | **HTTP-level problem** (wrong path, permissions, app bug, backend down) | Application (HTTP) |
| `curl` gets `HTTP/1.0 200 OK` | Service is healthy | none |

**What each line does:**
- `python3 -m http.server 8080 &`: starts a tiny web server on port 8080 in the background (`&`), so we have something to test.
- `ping -c 4 <ip>`: sends 4 ICMP echo requests (`-c` = count; without it Linux pings forever). Read `4 received, 0% packet loss` and `time=… ms`.
- `ss -tulpn | grep :8080`: shows if a process is in `LISTEN` state on port 8080 (`grep` filters the line).
- `curl -v URL`: verbose; shows each step: DNS lookup, `Connected to … port 8080` (TCP worked), the request sent (`>`), and the response status (`< HTTP/1.0 200 OK`).
- `curl -I URL`: sends a HEAD request, prints **only the response headers** (first line = status code).
- `curl -s -o /dev/null -w "%{http_code}\n"`: silent, throws the body away, prints just the status number (e.g. `404`).

**Verify:** healthy case → `ping` shows `0% packet loss`, `ss` shows `LISTEN … :8080 … python3`, `curl -I` shows `HTTP/1.0 200 OK`. Note: some networks block ICMP, so a failed `ping` with a working `curl` means "ping blocked", not "host down".

**🧠 In simple words:** you test from the bottom up: first "can I reach the machine?" (ping), then "is the program actually listening on that port?" (ss + curl connect), then "does the program answer correctly?" (HTTP status code). Each failure type gives a different message, so the first step that fails tells you the layer to fix. In short: **ping fails = network, connection refused = closed port, 4xx/5xx = the web app itself.**

---

### Lab 3 · Resolve a domain with nslookup/dig, then check HTTP/HTTPS with curl

**Answer (what to write in the exam):**
```bash
nslookup example.com                 # DNS lookup: which IP(s) does the name map to?
dig example.com                      # full DNS answer (record type, TTL)
dig +short example.com               # only the IP address(es)
curl -I http://example.com           # HTTP status over port 80
curl -I https://example.com          # HTTPS status over port 443 (TLS)
curl -IL http://google.com           # -L follows redirects (301 → final 200)
curl -sv -o /dev/null https://example.com 2>&1 | grep -E "Connected to|SSL|HTTP/"
```

**How to read the output (record these as your results):**
```text
$ nslookup example.com
Server:   127.0.0.53              ← the DNS server that answered
Address:  127.0.0.53#53           ← its IP, port 53
Non-authoritative answer:         ← came from a cache, not the domain's own DNS server
Name:     example.com
Address:  104.20.23.154           ← the A record (IPv4 address) = DNS RESULT

$ dig example.com     (ANSWER SECTION)
example.com.   300   IN   A   104.20.23.154
   name        TTL  class type  value     ← TTL 300 = cache this answer for 300 seconds

$ dig +short example.com
104.20.23.154

$ curl -I https://example.com
HTTP/2 200                                ← HTTP STATUS = 200 OK, HTTPS works
$ curl -I http://google.com
HTTP/1.1 301 Moved Permanently            ← 3xx = redirect; Location: header says where
```
(IP addresses change over time and by location; write down whatever your machine shows.)

**What each line does:**
- `nslookup <name>` / `dig <name>`: ask the DNS server for the name's A record (IPv4); `dig` shows more detail (TTL, record type, which server answered, query time).
- `dig +short`: prints only the answer, handy for scripts.
- `curl -I http://…` vs `https://…`: tests port 80 (plain) vs port 443 (encrypted). `-I` = headers only.
- `curl -L`: follow redirects until the final page.
- `curl -sv … | grep`: shows `Connected to example.com (104.20.23.154) port 443`, the TLS/SSL handshake lines, and the HTTP status line, which proves DNS → TCP → TLS → HTTP all worked.

**Status code families:** 2xx success (200 OK) · 3xx redirect (301, 302) · 4xx client error (400, 401, 403, 404) · 5xx server error (500, 502, 503).

**Verify:** `dig +short` returns at least one IP (empty output = DNS failure, e.g. `NXDOMAIN` = name does not exist), and `curl -I https://…` first line is `HTTP/… 200`.

**🧠 In simple words:** before your computer can talk to a website it must turn the name into an IP address (DNS); `nslookup`/`dig` show you that step on its own. Then `curl` makes a real web request and the status code tells you if the website answered correctly over HTTP and over HTTPS. In short: **dig/nslookup = name → IP, curl -I = is the website answering, and with which status code.**

---

## Session 5 · Git & GitHub

Acronyms used: PR = Pull Request (a request on GitHub to merge one branch into another, with review) · CLI = Command-Line Interface · `gh` = the official GitHub CLI · PAT = Personal Access Token (a password-like token for GitHub) · SSH = Secure Shell · HEAD = "the commit/branch you are on right now".

---

### Lab 1 · Create a repo with a README, make 3 meaningful commits, push to GitHub

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

### Lab 2 · Feature branch → change → push → review diff → merge into main through a PR

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

### Lab 3 · Create a merge conflict (same line, two branches), resolve it, commit, verify history

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

---

## Session 6–7 · Docker & Dockerfile

### Lab 1 · Run Nginx on host port 8080 → container port 80, curl it, read its logs
**Answer (what to write in the exam):**
```bash
docker pull nginx                              # download the image (optional: run pulls it anyway)
docker run -d --name web -p 8080:80 nginx      # start the container in the background
docker ps                                      # confirm it is "Up" with 0.0.0.0:8080->80/tcp
curl http://localhost:8080                     # fetch the page through the mapped port
docker logs web                                # see Nginx's access log (your curl request)
docker logs -f --tail 5 web                    # optional: follow live, last 5 lines (Ctrl+C to stop)
docker rm -f web                               # clean up: stop + delete the container
```
(On your Mac use 9080: `-p 9080:80` and `curl http://localhost:9080`, because 8080 is taken.)

**What each line does:**
- `docker pull nginx`: downloads the `nginx:latest` image from Docker Hub (the default registry).
- `docker run -d --name web -p 8080:80 nginx`: creates and starts a container. `-d` = detached (runs in background), `--name web` = a fixed name so later commands can use it, `-p HOST:CONTAINER` = publish port: traffic to host port 8080 goes to container port 80 (where Nginx listens).
- `docker ps`: lists running containers; the PORTS column proves the mapping.
- `curl http://localhost:8080`: curl (Client URL) sends an HTTP (HyperText Transfer Protocol) request from the host.
- `docker logs web`: prints what the container wrote to stdout/stderr (standard output / standard error). Nginx sends its access log there. `-f` = follow, `--tail N` = only the last N lines.
- `docker rm -f web`: `-f` force-removes a running container (stop + delete in one step).

**Verify:** `curl` prints HTML containing `<title>Welcome to nginx!</title>`; `docker logs web` shows a line like `"GET / HTTP/1.1" 200 615 "-" "curl/8.x"`.

**🧠 In simple words:** We start the official Nginx web server inside a container and "punch a hole" from the host's port 8080 to the container's port 80, so a request to localhost:8080 reaches Nginx. Then we prove it works with curl and read the container's output with `docker logs`, which shows our request was served. In short: `-p 8080:80` = host port first, container port second; `docker logs` = the container's stdout/stderr.

### Lab 2 · Dockerfile (FROM, WORKDIR, COPY, CMD) for an HTML page → build, run, verify
**Answer (what to write in the exam):**
```bash
mkdir mysite && cd mysite
echo '<h1>Hello from my Docker image</h1>' > index.html
# create the Dockerfile shown below (e.g. with nano Dockerfile / vi Dockerfile)
docker build -t mysite:1.0 .                         # build the image from ./Dockerfile
docker images mysite                                 # image is listed with tag 1.0
docker run -d --name mysite -p 8080:80 mysite:1.0    # run it
curl http://localhost:8080                           # → <h1>Hello from my Docker image</h1>
```
(On your Mac use 9080: `-p 9080:80` and `curl http://localhost:9080`.)

`Dockerfile`:
```dockerfile
FROM nginx:alpine
WORKDIR /usr/share/nginx/html
COPY index.html .
CMD ["nginx", "-g", "daemon off;"]
```

**What each line does:**
- `FROM nginx:alpine`: base image = Nginx on Alpine Linux (small). Every Dockerfile starts with FROM.
- `WORKDIR /usr/share/nginx/html`: sets (and creates if missing) the working directory inside the image; later COPY/RUN/CMD use it. This is the folder Nginx serves HTML (HyperText Markup Language) files from.
- `COPY index.html .`: copies `index.html` from the build context (your folder) into the WORKDIR (`.`), replacing Nginx's default page.
- `CMD ["nginx", "-g", "daemon off;"]`: default command when a container starts. `daemon off;` keeps Nginx in the foreground; if the main process exits, the container stops.
- `docker build -t mysite:1.0 .`: `-t` = name:tag for the image, `.` = build context (the folder sent to the builder, which also holds the Dockerfile).
- `docker run -d --name mysite -p 8080:80 mysite:1.0`: run your own image exactly like Lab 1.

**Verify:** `curl http://localhost:8080` prints `<h1>Hello from my Docker image</h1>`; `docker exec mysite ls /usr/share/nginx/html` lists `index.html`.

**🧠 In simple words:** A Dockerfile is a recipe: start from a ready Nginx image, go to the folder Nginx serves pages from, put our page there, and say which command to run at start. `docker build` turns the recipe into an image, and `docker run` starts a container from it, which we test with curl. In short: Dockerfile → `docker build` → image → `docker run` → container.

### Lab 3 · Add a .dockerignore and an environment variable, rebuild, run, explain
**Answer (what to write in the exam):**
```bash
cd mysite
touch .env debug.log                       # junk files that must NOT go into the image
printf '.git\n.env\n*.log\nDockerfile\n.dockerignore\n' > .dockerignore
# update the Dockerfile as shown below
docker build --build-arg APP_VERSION=2.0 -t mysite:2.0 .
docker rm -f mysite                        # remove the old container (name is reused)
docker run -d --name mysite -p 8080:80 mysite:2.0
docker exec mysite env | grep APP_         # → APP_ENV=production, APP_VERSION=2.0
docker exec mysite ls -a /usr/share/nginx/html   # no .env, no debug.log, no Dockerfile
docker run -d --name mysite-stg -p 8081:80 -e APP_ENV=staging mysite:2.0   # override at run time
```
(On your Mac use 9080: `-p 9080:80`.)

`.dockerignore`:
```text
.git
.env
*.log
Dockerfile
.dockerignore
```

`Dockerfile`:
```dockerfile
FROM nginx:alpine
ARG APP_VERSION=1.0
ENV APP_ENV=production
ENV APP_VERSION=${APP_VERSION}
WORKDIR /usr/share/nginx/html
COPY . .
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**What each added part does:**
- `.dockerignore`: lists paths left OUT of the build context, so `COPY . .` never copies them. Gives smaller, faster builds and keeps secrets (`.env`) and junk (`*.log`, `.git`) out of the image. (Docker still reads the Dockerfile itself; it just is not copied in.)
- `ARG APP_VERSION=1.0`: a build-time variable with a default; changed with `--build-arg APP_VERSION=2.0`. It exists only during the build.
- `ENV APP_ENV=production` / `ENV APP_VERSION=${APP_VERSION}`: environment variables saved in the image, so every container gets them at run time (the second line copies the ARG value into the image).
- `COPY . .`: copies the whole folder minus the ignored paths.
- `EXPOSE 80`: documents the port the app listens on (it does not publish it; `-p` does).
- `-e APP_ENV=staging`: overrides an ENV value for one container without rebuilding.

**Verify:** `docker exec mysite env | grep APP_` → `APP_ENV=production` and `APP_VERSION=2.0`; `ls -a` shows only `.`, `..`, `index.html` and `50x.html` (no `.env`, no `debug.log`). `docker exec mysite-stg env | grep APP_ENV` → `APP_ENV=staging`.

**🧠 In simple words:** `.dockerignore` works like `.gitignore` for builds: it stops secrets and junk files from ever being sent into the image. ENV bakes a setting into the image that the app can read at run time, ARG is a knob you turn only while building, and `-e` lets you change a setting per container without rebuilding. In short: `.dockerignore` = keep files out; ARG = build-time value; ENV = run-time value baked in; `-e` = override at run.

---

## Session 8 · Docker networking, volumes & Compose

### Lab 1 · Custom network + two containers that talk to each other by name
**Answer (what to write in the exam):**
```bash
docker network create appnet                                    # user-defined bridge network
docker run -d --name web    --network appnet nginx              # container 1: web server
docker run -d --name client --network appnet alpine sleep 3600  # container 2: a client that stays up
docker network ls                                               # appnet is listed (driver: bridge)
docker network inspect appnet                                   # "Containers": web and client, with IPs
docker exec client ping -c 2 web                                # name "web" resolves to its IP
docker exec client wget -qO- http://web                         # fetch Nginx's page BY NAME
docker rm -f web client && docker network rm appnet             # clean up
```

**What each line does:**
- `docker network create appnet`: makes a new network; default driver = `bridge` (a private virtual switch on the host). A user-defined bridge gives built-in DNS (Domain Name System): containers find each other by container name.
- `--network appnet`: attaches the container to that network instead of the default `bridge`.
- `alpine sleep 3600`: tiny Linux image; `sleep` keeps it running for 1 hour so we can `exec` into it.
- `docker network inspect appnet`: shows the subnet and which containers are attached, with their IPs.
- `docker exec client ping -c 2 web`: runs ping inside `client`; `-c 2` = send 2 packets. Success proves name → IP resolution.
- `docker exec client wget -qO- http://web`: HTTP (HyperText Transfer Protocol) request to the `web` container on port 80; `-q` quiet, `-O-` print the page to the screen.
- No `-p` is needed: containers on the same network reach each other's ports directly; `-p` is only for access from the host.

**Verify:** ping shows `PING web (172.18.0.2)` and `0% packet loss`; wget prints HTML containing `Welcome to nginx!`.

**🧠 In simple words:** We build a private network and put two containers on it. Because it is a user-defined bridge network, Docker runs a small DNS server for it, so `client` can reach the other container just by typing `web` instead of an IP address that may change. (On the default `bridge` network, names do NOT resolve; only IPs work.) In short: `docker network create` + `--network` on both containers = they talk by name.

### Lab 2 · Database with a named volume: add data, delete + recreate container, data survives
**Answer (what to write in the exam):**
```bash
docker volume create dbdata                       # named volume (optional: -v would create it)
docker run -d --name db -e POSTGRES_PASSWORD="$DB_PASSWORD" \
  -v dbdata:/var/lib/postgresql/data postgres:16-alpine
sleep 8                                           # wait until Postgres is ready
docker exec db psql -U postgres -c "CREATE TABLE notes(msg text); INSERT INTO notes VALUES ('hello');"
docker rm -f db                                   # delete the container
docker volume ls                                  # dbdata is STILL there
docker run -d --name db -e POSTGRES_PASSWORD="$DB_PASSWORD" \
  -v dbdata:/var/lib/postgresql/data postgres:16-alpine   # brand-new container, same volume
sleep 5
docker exec db psql -U postgres -c "SELECT * FROM notes;"   # → hello
```
- Set the password once in the shell first (not saved in any file you submit): `export DB_PASSWORD=pass`.
- Better wait than `sleep`: `until docker exec db pg_isready -U postgres -h localhost; do sleep 1; done` (`-h localhost` checks over TCP (Transmission Control Protocol, a real network connection), which only opens when the real server is up).

Same thing as the one-liner already verified on your Mac:
```bash
docker run -d --name db -e POSTGRES_PASSWORD=pass -v dbdata:/var/lib/postgresql/data postgres:16-alpine && sleep 8 && docker exec db psql -U postgres -c "CREATE TABLE notes(msg text); INSERT INTO notes VALUES ('hello');" && docker rm -f db && docker run -d --name db -e POSTGRES_PASSWORD=pass -v dbdata:/var/lib/postgresql/data postgres:16-alpine && sleep 5 && docker exec db psql -U postgres -c "SELECT * FROM notes;"
```

**What each line does:**
- `docker volume create dbdata`: a named volume = storage managed by Docker that lives outside any container.
- `-e POSTGRES_PASSWORD=...`: the Postgres image refuses to start without it (it sets the `postgres` superuser password).
- `-v dbdata:/var/lib/postgresql/data`: mounts the volume on Postgres's data directory, so the database files are written into the volume, not into the container's own layer.
- `sleep 8`: Postgres needs a few seconds to initialise on first start; `psql` before that fails ("is the server running?").
- `docker exec db psql -U postgres -c "..."`: psql = PostgreSQL's command-line client; `-U postgres` = user, `-c` = run this SQL (Structured Query Language) and exit.
- `docker rm -f db`: deletes the container. Volumes are NOT deleted unless you use `docker rm -v` or `docker volume rm`.
- The second `docker run`: a fresh container, same volume, so it finds the existing database (no re-initialisation).

**Verify:** the last command prints:
```text
  msg
-------
 hello
(1 row)
```
`docker volume inspect dbdata` shows its `Mountpoint`. Clean up: `docker rm -f db && docker volume rm dbdata`.

**🧠 In simple words:** A container's own files disappear when the container is deleted, so we keep the database files in a named volume instead. We create a table, throw the container away, start a brand-new one pointing at the same volume, and the row is still there. In short: data lives in the volume, not the container, so deleting the container does not delete the data.

### Lab 3 · compose.yaml for web + database: start, check they talk, logs, shut down
**Answer (what to write in the exam):**
```bash
mkdir webapp && cd webapp
echo 'DB_PASSWORD=change-me' > .env        # secret lives here, not in compose.yaml
echo '.env' >> .gitignore                  # never commit it
# create compose.yaml (below)
docker compose config                      # check the file + see ${DB_PASSWORD} filled in
docker compose up -d                       # create network + volume, start db then web
docker compose ps                          # both services "running"; web shows 0.0.0.0:8080->80
curl http://localhost:8080                 # web answers from the host
docker compose exec web ping -c 2 db       # web reaches db BY SERVICE NAME
docker compose exec db pg_isready -U postgres     # → accepting connections
docker compose logs db                     # logs of one service (logs -f = follow)
docker compose down                        # stop + remove containers and network (volume kept)
```
(On your Mac use 9080: `"9080:80"` in compose.yaml and `curl http://localhost:9080`.)

`compose.yaml`:
```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      - db
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - dbdata:/var/lib/postgresql/data

volumes:
  dbdata:
```

**What each line does:**
- `services:` → `web`, `db`: each service becomes a container; the service name is also its DNS (Domain Name System) name on the network Compose creates (`<folder>_default`).
- `ports: "8080:80"`: publish host port 8080 → container port 80 (same as `-p`). `db` has no `ports`: only other services need to reach it.
- `depends_on: - db`: start `db` before `web`. It waits for the container to START, not for Postgres to be READY (for that, add a `healthcheck` + `condition: service_healthy`).
- `POSTGRES_PASSWORD: ${DB_PASSWORD}`: Compose reads `DB_PASSWORD` from the `.env` file in the same folder, so no password is hard-coded in the YAML (YAML Ain't Markup Language) file.
- `volumes: - dbdata:/var/lib/postgresql/data` + top-level `volumes: dbdata:`: a named volume (created as `<folder>_dbdata`) so data survives `down`/`up`.
- `docker compose up -d` / `ps` / `exec` / `logs` / `down`: start in background / list / run a command in a service / show output / tear down. `down -v` also deletes the volume.

**Verify:** `docker compose ps` shows `web` and `db` Up; ping prints `0% packet loss`; `pg_isready` prints `/var/run/postgresql:5432 - accepting connections`; after `down`, `docker compose ps` is empty but `docker volume ls` still lists `webapp_dbdata`.

**🧠 In simple words:** Instead of typing several long `docker run` commands, we describe the whole app (a web server, a database, their network and storage) in one `compose.yaml`, and Compose builds all of it with `up` and removes it with `down`. Compose puts both services on one network where they find each other by service name, and the password comes from `.env` so it is never written in the config. In short: `compose.yaml` = the app's blueprint; `up -d` to start, `logs` to debug, `down` to stop.

---

## Session 9 · Kubernetes fundamentals

### Lab 1 · Connect to a cluster and inspect nodes, namespaces and resources with kubectl
**Answer (what to write in the exam):**
```bash
kubectl config get-contexts               # which clusters can I talk to? (* = current)
kubectl config use-context minikube       # pick the cluster
kubectl cluster-info                      # where the API server and CoreDNS run
kubectl version                           # client + server versions
kubectl get nodes -o wide                 # all machines in the cluster
kubectl describe node minikube            # one node in detail
kubectl get ns                            # all namespaces
kubectl get pods -n kube-system           # the cluster's own system Pods
kubectl get all -A                        # common objects in ALL namespaces
kubectl api-resources                     # every object type the API knows
```
**What each line does (record this in the exam):**
- `kubectl`: the command-line tool (CLI, command-line interface) that sends requests to the Kubernetes API (application programming interface) server.
- `config get-contexts` / `use-context`: a *context* = cluster + user + namespace saved in `~/.kube/config`; this lists them and switches to one.
- `cluster-info`: shows the control-plane (API server) URL, e.g. `https://127.0.0.1:32771`, and the CoreDNS (cluster DNS, Domain Name System) URL. Proves you are connected.
- `version`: client version and server version (they should be within one minor version).
- `get nodes -o wide`: NAME, STATUS (`Ready`), ROLES (`control-plane`), VERSION, plus INTERNAL-IP, OS image, kernel and container runtime (`-o wide` = extra columns).
- `describe node`: labels, capacity (CPU (central processing unit) / memory), conditions (MemoryPressure, DiskPressure, Ready), and the Pods running on that node.
- `get ns`: namespaces = logical folders for objects: `default`, `kube-system` (cluster components), `kube-public`, `kube-node-lease`.
- `get pods -n kube-system`: shows etcd, kube-apiserver, kube-scheduler, kube-controller-manager, kube-proxy, coredns running as Pods (`-n` = namespace).
- `get all -A`: Pods, Services, Deployments, ReplicaSets, DaemonSets etc. in every namespace (`-A` = `--all-namespaces`). It does NOT include ConfigMaps/Secrets.
- `api-resources`: NAME, SHORTNAMES (`po`, `svc`, `deploy`, `ns`), APIVERSION, NAMESPACED (true/false) and KIND for every resource type.

**Verify:** `kubectl get nodes` → node `minikube` with STATUS `Ready`; `kubectl cluster-info` → "Kubernetes control plane is running at https://…".

**🧠 In simple words:** kubectl never touches containers directly; it only talks to the API server of the cluster selected by the current context. These commands answer "which cluster am I on, which machines does it have, which namespaces exist, what is running, and what object types can I create". In short: context → cluster-info → nodes → namespaces → all objects → resource types.

### Lab 2 · Deploy a simple web app and inspect it with get, describe and logs until it is healthy
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=2 --dry-run=client -o yaml > web.yaml
kubectl apply -f web.yaml                     # (or run the create command without --dry-run)
kubectl rollout status deployment/web         # waits until all replicas are ready
kubectl get deployment web                    # READY 2/2
kubectl get pods -l app=web -o wide           # Pod names, STATUS Running, IP, NODE
kubectl describe deployment web               # replicas, image, events
kubectl describe pod <web-pod-name>           # State Running, Ready True, Events
kubectl logs deployment/web                   # nginx startup output
kubectl port-forward deployment/web 8080:80   # then in a 2nd terminal: curl localhost:8080
```
**What each line does:**
- `create deployment … --dry-run=client -o yaml > web.yaml`: generates the Deployment manifest (YAML file) without creating anything; `--replicas=2` = two Pods.
- `apply -f web.yaml`: sends the file to the API server; the Deployment creates a ReplicaSet, which creates 2 Pods with label `app=web`.
- `rollout status`: blocks until "deployment "web" successfully rolled out" = healthy.
- `get … -l app=web -o wide`: `-l` filters by label; `-o wide` adds Pod IP (Internet Protocol address) and NODE.
- `describe`: human-readable details; the **Events** section at the bottom shows Scheduled → Pulling → Pulled → Created → Started (or errors like ImagePullBackOff).
- `logs deployment/web`: prints the container's stdout of one Pod of the Deployment (add `-f` to follow).
- `port-forward`: tunnels your laptop's port 8080 to container port 80 so `curl localhost:8080` returns the nginx welcome page.

**Verify:** `kubectl get pods -l app=web` → 2 Pods, READY `1/1`, STATUS `Running`, RESTARTS `0`; `curl localhost:8080` → "Welcome to nginx!".

**🧠 In simple words:** You describe the app (image + number of copies) and Kubernetes starts and watches the Pods for you. `get` gives the quick status, `describe` explains why something is or is not working (read the Events), and `logs` shows what the app itself printed. In short: apply → rollout status → get (Running, 1/1) → describe (Events) → logs.

### Lab 3 · Draw the cluster architecture and give the role of each component
**Answer (what to write in the exam):**
```
  you: kubectl apply / get   (CLI)
                 |  HTTPS (REST API calls)
                 v
+------------------- CONTROL PLANE (master) --------------------+
|                                                               |
|  kube-scheduler --->  +----------------+  <--- controller-    |
|  (picks a node)       | kube-apiserver |       manager        |
|                       |  (front door)  |  (keeps desired =    |
|                       +----------------+   actual state)      |
|                               |                              |
|                               v                              |
|                        +-------------+                       |
|                        |    etcd     |  (key-value store,    |
|                        +-------------+   all cluster state)  |
+---------------------------------------------------------------+
                 |  (only the API server talks to etcd)
      +----------+-----------+
      v                      v
+-- WORKER NODE 1 ---+  +-- WORKER NODE 2 ---+
| kubelet            |  | kubelet            |
| kube-proxy         |  | kube-proxy         |
| container runtime  |  | container runtime  |
| [Pod] [Pod]        |  | [Pod]              |
+--------------------+  +--------------------+
```
**One line per component:**
- **kubectl**: the CLI the user runs; it turns commands into REST (HTTP-based) calls to the API server. It is a client, not part of the cluster.
- **kube-apiserver**: the single front door of the cluster; authenticates and validates every request and is the ONLY component that reads/writes etcd.
- **etcd**: distributed key-value database that stores the whole cluster state (desired and current); back it up.
- **kube-scheduler**: watches for new Pods with no node and chooses the best node (resources, taints, affinity); it does not start the Pod.
- **kube-controller-manager**: runs control loops (Deployment, ReplicaSet, Node, Job controllers) that keep actual state equal to desired state, e.g. recreate a crashed Pod.
- **kubelet**: agent on every node; receives Pods assigned to its node, tells the container runtime to start them, and reports health back to the API server.
- (extra) **kube-proxy**: on every node, programs network rules so Service IPs route to Pods. **Container runtime** (containerd): actually pulls images and runs containers.

**Flow of `kubectl apply -f web.yaml`:** kubectl → API server (validate) → etcd (save) → controller-manager creates Pods → scheduler assigns each Pod a node → kubelet on that node starts the containers → status written back via API server.

**Verify (minikube):** `kubectl get pods -n kube-system` → etcd-minikube, kube-apiserver-minikube, kube-scheduler-minikube, kube-controller-manager-minikube, kube-proxy-…, coredns-….

**🧠 In simple words:** The control plane is the "brain": the API server receives every request, etcd remembers everything, the scheduler decides where Pods go, and the controller manager keeps fixing differences between what you asked for and what exists. Worker nodes are the "muscle": kubelet runs the Pods there. In short: kubectl → API server ↔ etcd; scheduler + controllers decide; kubelet on each node does the work.

---

## Session 10 · Pods, ReplicaSets & Deployments

### Lab 1 · Create an Nginx Pod and inspect its status, labels, IP address and logs
**Answer (what to write in the exam):**
```bash
kubectl run nginx --image=nginx:1.27 --port=80 --labels="app=nginx,tier=web"
kubectl get pod nginx                          # STATUS Running, READY 1/1
kubectl get pod nginx -o wide --show-labels    # IP, NODE, LABELS
kubectl get pod nginx -o jsonpath='{.status.podIP}{"\n"}'   # only the IP
kubectl describe pod nginx                     # full details + Events
kubectl logs nginx                             # container output
kubectl label pod nginx env=dev                # add one more label
kubectl get pods -l app=nginx --show-labels    # select Pods by label
```
Same Pod as a manifest (`kubectl apply -f nginx-pod.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
    tier: web
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```
**What each line does:**
- `kubectl run nginx --image=nginx:1.27`: creates ONE bare Pod named `nginx`; `--port=80` documents the container port; `--labels` attaches key=value tags.
- `get pod nginx`: STATUS goes Pending → ContainerCreating → Running; READY `1/1` = 1 of 1 containers ready.
- `-o wide --show-labels`: adds IP (the Pod's internal IP, Internet Protocol address), NODE, and a LABELS column `app=nginx,tier=web`.
- `-o jsonpath='{.status.podIP}'`: prints just one field from the Pod object.
- `describe pod`: node, IP, labels, container state, image, and **Events** (Scheduled, Pulling, Pulled, Created, Started).
- `logs nginx`: nginx startup lines ("Configuration complete; ready for start up"); add `-f` to follow, `--previous` for a crashed container.
- `label` / `get -l app=nginx`: add a label, then use a *label selector* to find Pods that match it.

**Verify:** `kubectl get pod nginx -o wide --show-labels` → `nginx 1/1 Running 0 … 10.244.0.x minikube … app=nginx,tier=web`.

**🧠 In simple words:** A Pod is the smallest thing Kubernetes runs: one or more containers sharing one IP. Labels are tags that let you find and select Pods later (Services and Deployments use them). In short: run → get -o wide --show-labels (status, IP, labels) → describe (Events) → logs.

### Lab 2 · Create a Deployment with 2 replicas, scale to 5, scale back to 2, verify each step
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=2   # or: kubectl apply -f web-deploy.yaml
kubectl rollout status deployment/web
kubectl get deployment web             # READY 2/2
kubectl scale deployment web --replicas=5
kubectl rollout status deployment/web
kubectl get deployment web             # READY 5/5
kubectl get rs -l app=web              # DESIRED 5  CURRENT 5  READY 5
kubectl scale deployment web --replicas=2
kubectl get deployment web             # READY 2/2
kubectl get pods -l app=web            # exactly 2 Pods left (3 Terminating, then gone)
```
Manifest (`web-deploy.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```
**What each line does:**
- `create deployment … --replicas=2`: Deployment → creates a ReplicaSet → which creates 2 Pods labelled `app=web`.
- `spec.replicas`: desired Pod count; `selector.matchLabels` must match `template.metadata.labels` (that is how the ReplicaSet finds its Pods).
- `scale deployment web --replicas=5`: changes `spec.replicas`; the ReplicaSet controller starts 3 more Pods. Scaling to 2 deletes 3.
- `get deployment`: READY `x/y`, UP-TO-DATE, AVAILABLE. `get rs`: DESIRED / CURRENT / READY of the ReplicaSet.
- Declarative way: edit `replicas: 5` in the file and `kubectl apply -f web-deploy.yaml` again.

**Verify:** `kubectl get deploy web -o jsonpath='{.spec.replicas} {.status.readyReplicas}{"\n"}'` → `2 2`, then `5 5`, then `2 2`.

**🧠 In simple words:** You never create or delete Pods by hand; you change one number (`replicas`) and the ReplicaSet controller adds or removes Pods until reality matches. If a Pod dies, it is replaced automatically. In short: create with 2 → scale to 5 → scale to 2, checking READY each time.

### Lab 3 · Deploy v1, rolling-update to v2, verify, roll back to v1
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:1.26 --replicas=3          # version 1
kubectl annotate deployment web kubernetes.io/change-cause="v1 nginx:1.26"
kubectl rollout status deployment/web
kubectl set image deployment/web nginx=nginx:1.27                      # version 2
kubectl annotate deployment web kubernetes.io/change-cause="v2 nginx:1.27" --overwrite
kubectl rollout status deployment/web                                  # wait for v2
kubectl rollout history deployment/web                                 # REVISION 1 and 2
kubectl get rs -l app=web                                              # new RS 3 Pods, old RS 0
kubectl rollout undo deployment/web                                    # back to v1 (or --to-revision=1)
kubectl rollout status deployment/web
kubectl get deployment web -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```
**What each line does:**
- `create deployment … nginx:1.26`: version 1; the container is auto-named `nginx` (from the image name).
- `annotate … kubernetes.io/change-cause`: text shown in the CHANGE-CAUSE column of `rollout history` (replaces the old deprecated `--record` flag).
- `set image deployment/web nginx=nginx:1.27`: `<container-name>=<new-image>`; changing the Pod template starts a **rolling update**: a new ReplicaSet scales up while the old one scales down (default maxSurge 25%, maxUnavailable 25%), so the app stays up.
- `rollout status`: waits until "successfully rolled out"; `rollout history`: lists revisions.
- `get rs`: two ReplicaSets: the new one has 3 Pods, the old one 0 (kept for rollback).
- `rollout undo`: scales the previous ReplicaSet back up (it becomes revision 3); `--to-revision=1` picks an exact one.

**Verify:** after `set image` the jsonpath prints `nginx:1.27`; after `undo` it prints `nginx:1.26`; `kubectl rollout history deployment/web` shows revisions 2 and 3 (revision 1 was re-used as 3).

**🧠 In simple words:** A Deployment upgrades Pods a few at a time, so users never see downtime, and it keeps the old ReplicaSet so you can go back with one command. In short: set image → rollout status → rollout history → rollout undo.

---

## Session 11 · Kubernetes Services

### Lab 1 · Deploy an app with several replicas, add a ClusterIP Service that selects its Pods, verify endpoints
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3
kubectl rollout status deployment/web
kubectl get pods -l app=web -o wide --show-labels       # note the 3 Pod IPs
kubectl expose deployment web --port=80 --target-port=80   # ClusterIP is the default type
kubectl get svc web                                     # TYPE ClusterIP, CLUSTER-IP 10.x, 80/TCP
kubectl describe svc web                                # Selector app=web, Endpoints
kubectl get endpoints web                               # the same 3 Pod IPs, each :80
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://web
```
Same Service as a manifest (`kubectl apply -f web-svc.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```
**What each line does:**
- `create deployment … --replicas=3`: 3 nginx Pods, each labelled `app=web`.
- `expose deployment web`: creates Service `web` that copies the Deployment's selector `app=web`; `--port` = Service port, `--target-port` = container port.
- `selector: app: web`: the Service sends traffic to every *ready* Pod with this label. A typo here = no endpoints.
- `get svc`: shows the stable virtual CLUSTER-IP (only reachable inside the cluster).
- `get endpoints web`: list of `PodIP:port` behind the Service (on Kubernetes 1.33+ it prints a deprecation warning; `kubectl get endpointslices -l kubernetes.io/service-name=web` is the new form).
- `kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://web`: temporary Pod inside the cluster; `web` is resolved by cluster DNS (Domain Name System) to the ClusterIP; `--rm` deletes the Pod afterwards.

**Verify:** `kubectl get endpoints web` → `10.244.0.11:80,10.244.0.12:80,10.244.0.13:80` (matches `get pods -o wide`); the busybox `wget` prints the "Welcome to nginx!" HTML.

**🧠 In simple words:** Pod IPs change every time a Pod is recreated, so clients should not use them. A ClusterIP Service gives one fixed IP and DNS name (`web`, or `web.default.svc.cluster.local` from another namespace) and load-balances to whichever Pods currently match the label. In short: label the Pods → Service selects that label → endpoints list the Pod IPs.

### Lab 2 · Create a NodePort Service and reach the app through the node's port
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3     # skip if it exists from Lab 1
kubectl expose deployment web --name=web-np --type=NodePort --port=80 --target-port=80
kubectl get svc web-np                    # TYPE NodePort, PORT(S) 80:3xxxx/TCP
kubectl get endpoints web-np              # Pod IPs :80
kubectl get nodes -o wide                 # INTERNAL-IP of a node, e.g. 192.168.49.2
NODE_PORT=$(kubectl get svc web-np -o jsonpath='{.spec.ports[0].nodePort}')
curl http://<NODE_IP>:$NODE_PORT          # minikube on Linux: curl http://$(minikube ip):$NODE_PORT
```
Manifest with a fixed node port (`kubectl apply -f web-np.yaml`, instead of the `expose` line):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-np
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```
**What each line does:**
- `--type=NodePort`: opens the same port (range 30000–32767) on EVERY node; traffic to `NodeIP:nodePort` → Service → a Pod. It also still gets a ClusterIP.
- `PORT(S) 80:30080/TCP`: left = Service port inside the cluster, right = node port from outside (TCP, Transmission Control Protocol).
- `nodePort: 30080`: choose the port yourself; leave it out and Kubernetes picks a random free one.
- `jsonpath='{.spec.ports[0].nodePort}'`: reads the assigned node port into a shell variable.
- `curl http://<NODE_IP>:<nodePort>`: proves access from outside the cluster.
- (Mac, minikube Docker driver: the node IP is not reachable from macOS, so run `minikube service web-np --url` and keep that terminal open; it prints `http://127.0.0.1:<port>` to curl. `kubectl port-forward svc/web-np 8080:80` also works, but it bypasses the node port.)

**Verify:** `kubectl get svc web-np` → `NodePort … 80:30080/TCP`; `curl http://192.168.49.2:30080` (or the `minikube service … --url` address) → "Welcome to nginx!".

**🧠 In simple words:** ClusterIP only works inside the cluster; NodePort additionally opens a high port on every node so someone outside can reach the app with `node-IP:node-port`. It is the simple way to test external access without a cloud load balancer. In short: type NodePort → read the 3xxxx port → curl NodeIP:port.

### Lab 3 · Service port and targetPort differ; prove traffic reaches the container port
**Answer (what to write in the exam):** the app listens on **8080**, clients use Service port **80**.
```bash
kubectl apply -f hello.yaml
kubectl rollout status deployment/hello
kubectl get svc hello                      # PORT(S) 80/TCP
kubectl get endpoints hello                # PodIP:8080  <- the targetPort
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://hello        # :80 works
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 http://hello:8080   # fails
```
`hello.yaml` (imperative alternative for the Service: `kubectl expose deployment hello --port=80 --target-port=8080`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: echo
          image: kicbase/echo-server:1.0
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 8080
```
**What each line does:**
- `image: kicbase/echo-server:1.0` + `containerPort: 8080`: a small HTTP (HyperText Transfer Protocol) server that listens on 8080 and replies with the request it received.
- `port: 80`: the port clients use on the Service (ClusterIP / DNS name `hello`).
- `targetPort: 8080`: the port on the Pod the Service forwards to; it must equal the port the app really listens on.
- `get endpoints hello`: shows `PodIP:8080`, the proof that the Service targets the container port.
- `wget http://hello` (port 80) → reply "Request served by hello-…"; `wget http://hello:8080` fails, because the Service itself only listens on 80.

**Verify:** `kubectl get svc hello` → `80/TCP`; `kubectl get endpoints hello` → `10.244.0.x:8080,10.244.0.y:8080`; busybox `wget -qO- http://hello` → "Request served by hello-xxxxx". (Other common pair: nginx with `port: 8080`, `targetPort: 80`.)

**🧠 In simple words:** `port` is the door on the Service, `targetPort` is the door on the container, and kube-proxy forwards from one to the other, so they do not have to be the same number. If `targetPort` is wrong, the Service still exists but requests fail (connection refused). In short: clients hit Service:80 → kube-proxy → Pod:8080, checked with `get endpoints` and a busybox `wget`.

---

## Session 12 · Ingress, ConfigMaps & Secrets

### Lab 1 · Deploy an app and route traffic to the right Service by host and URL path with an Ingress
**Answer (what to write in the exam):**
```bash
minikube addons enable ingress                       # install the NGINX Ingress controller
kubectl create deployment web --image=nginx:1.27
kubectl expose deployment web --port=80
kubectl create deployment api --image=kicbase/echo-server:1.0
kubectl expose deployment api --port=80 --target-port=8080
kubectl apply -f shop-ingress.yaml
kubectl get ingress shop-ingress                     # wait until ADDRESS is filled
curl -H "Host: shop.example.com" http://$(minikube ip)/      # nginx welcome page (web)
curl -H "Host: shop.example.com" http://$(minikube ip)/api   # "Request served by api-..." (api)
```
`shop-ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```
**What each line does:**
- `minikube addons enable ingress`: an Ingress object is only a set of rules; this installs the **controller** (an NGINX proxy Pod) that reads the rules and actually forwards traffic. No controller = rules do nothing.
- `create deployment` + `expose`: two apps, each behind its own ClusterIP Service (`--target-port=8080` because echo-server listens on 8080).
- `apiVersion: networking.k8s.io/v1` / `kind: Ingress`: the current Ingress API.
- `ingressClassName: nginx`: "this rule is for the NGINX controller".
- `host: shop.example.com`: only requests whose HTTP (HyperText Transfer Protocol) `Host` header is this name match.
- `path: /api` + `pathType: Prefix`: any URL (Uniform Resource Locator) path starting with `/api` goes to Service `api`; everything else (`/`) goes to `web`. The longest matching prefix wins.
- `curl -H "Host: ..."`: fakes the domain name, so you don't need a real DNS (Domain Name System) record.
**Verify:** `kubectl describe ingress shop-ingress` → Rules list `shop.example.com /  web:80` and `/api  api:80`; the two `curl` calls return different apps.
(Mac with Docker driver: run `minikube tunnel` in a second terminal and curl `http://127.0.0.1/` instead of `$(minikube ip)`.)

**🧠 In simple words:** An Ingress is one front door (a single IP address) for many Services. The controller reads the host name and the URL path of every request and sends it to the matching Service. In short: Ingress = HTTP routing rules, the controller = the thing that obeys them.

### Lab 2 · Create a ConfigMap, inject it into a Deployment as environment variables, verify inside the Pod
**Answer (what to write in the exam):**
```bash
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod
kubectl get configmap app-config -o yaml             # see the key/value data
kubectl create deployment web --image=nginx:1.27 --dry-run=client -o yaml > web.yaml
# edit web.yaml: add the envFrom block under the container (shown below)
kubectl apply -f web.yaml
kubectl rollout status deployment/web
kubectl exec deploy/web -- env | grep APP_
```
`web.yaml` (final):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          envFrom:
            - configMapRef:
                name: app-config
```
**What each line does:**
- `create configmap ... --from-literal=KEY=value`: stores plain, non-secret settings as key/value pairs in the cluster.
- `--dry-run=client -o yaml > web.yaml`: generates a Deployment file without creating anything, so you don't type it from scratch.
- `envFrom: - configMapRef: name: app-config`: every key in the ConfigMap becomes an environment variable in the container (`APP_COLOR`, `APP_MODE`).
- `kubectl exec deploy/web -- env`: runs `env` inside a Pod of the Deployment; `grep APP_` filters our variables.
- (One variable only: use `env: - name: APP_COLOR  valueFrom: configMapKeyRef: {name: app-config, key: APP_COLOR}`.)
**Verify:** `kubectl exec deploy/web -- env | grep APP_` → `APP_COLOR=blue` and `APP_MODE=prod`.
Note: env vars are read only when the container starts, so after changing the ConfigMap run `kubectl rollout restart deployment/web`.

**🧠 In simple words:** A ConfigMap keeps settings outside the image, so the same image runs in dev and prod with different values. Kubernetes copies those values into the container as environment variables when it starts. In short: settings live in the ConfigMap, the Pod reads them as env vars.

### Lab 3 · Create a Secret for a credential and consume it in a Pod without hard-coding it in the manifest
**Answer (what to write in the exam):**
```bash
kubectl create secret generic db-secret --from-literal=DB_PASSWORD='ChangeMe123'
kubectl get secret db-secret -o yaml                 # value is base64, not plain text
kubectl apply -f db-client.yaml
kubectl rollout status deployment/db-client
kubectl exec deploy/db-client -- printenv DB_PASSWORD
```
`db-client.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db-client
  template:
    metadata:
      labels:
        app: db-client
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "sleep 3600"]
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
```
**What each line does:**
- `create secret generic db-secret --from-literal=...`: stores the credential in a Secret object (only in the cluster, never in the YAML file you commit). In real use prefer `--from-file=DB_PASSWORD=./pass.txt` so the password is not left in shell history.
- `secretKeyRef: name: db-secret key: DB_PASSWORD`: "fill env var `DB_PASSWORD` from key `DB_PASSWORD` of Secret `db-secret`". The manifest holds only a reference, not the password.
- `printenv DB_PASSWORD`: proves the running container received the value.
- `get secret -o yaml`: shows `DB_PASSWORD: Q2hhbmdlTWUxMjM=`; that is base64 **encoding**, not encryption (anyone can decode it), so limit access with RBAC (Role-Based Access Control).
**Verify:** `kubectl exec deploy/db-client -- printenv DB_PASSWORD` → `ChangeMe123`; `grep -i password db-client.yaml` shows only the reference names, no value.
(Decode check: `kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d`.)

**🧠 In simple words:** The password is stored once in a Secret, and the Deployment only says "take it from that Secret". So the file you share or put in Git never contains the real credential. In short: Secret holds the value, `secretKeyRef` points to it.

---

## Session 13 · Storage, HPA & Probes

### Lab 1 · Create a PVC, mount it in a Pod, write a file, recreate the Pod, prove the file survived
**Answer (what to write in the exam):**
```bash
kubectl apply -f data-pvc.yaml -f data-pod.yaml
kubectl get pvc data-pvc                              # STATUS Bound
kubectl wait --for=condition=Ready pod/data-pod
kubectl exec data-pod -- sh -c 'echo hello-from-pvc > /data/note.txt'
kubectl delete pod data-pod                           # Pod gone, PVC stays
kubectl apply -f data-pod.yaml                        # brand-new Pod, same PVC
kubectl wait --for=condition=Ready pod/data-pod
kubectl exec data-pod -- cat /data/note.txt           # hello-from-pvc
```
`data-pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```
`data-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```
**What each line does:**
- PVC (PersistentVolumeClaim): a request "give me 100Mi of disk". No `storageClassName` → the cluster's **default StorageClass** (minikube: `standard`) creates a PV (PersistentVolume) automatically and binds it.
- `ReadWriteOnce` (RWO): the volume can be mounted read-write by one node at a time.
- `volumes: persistentVolumeClaim: claimName` + `volumeMounts: mountPath: /data`: attach the claimed disk inside the container at `/data`.
- `kubectl wait --for=condition=Ready`: blocks until the Pod is running.
- `delete pod` then `apply` again: the container filesystem is thrown away, but `/data` lives on the PV, not in the container.
**Verify:** `kubectl get pvc,pv` → PVC `Bound` to a PV; the last `cat` prints `hello-from-pvc` from the new Pod.
(If the PVC stays `Pending`, there is no default StorageClass: check `kubectl get storageclass`.)

**🧠 In simple words:** Anything a container writes to its own filesystem dies with the Pod. A PVC gives the Pod a separate disk that outlives it, so a new Pod mounting the same claim sees the old data. In short: PVC = storage that survives Pod restarts and deletes.

### Lab 2 · Add readiness and liveness probes and show how Kubernetes reacts when each one fails
**Answer (what to write in the exam):**
```bash
kubectl apply -f web-probes.yaml && kubectl expose deployment web --port=80
kubectl get pods -l app=web                          # READY 1/1, RESTARTS 0
# A) app NOT READY: readiness path now returns 404
kubectl exec deploy/web -- rm /usr/share/nginx/html/index.html
kubectl get pods -l app=web -w                       # READY 0/1, RESTARTS stays 0
kubectl get endpoints web                            # ENDPOINTS <none>: no traffic sent
kubectl exec deploy/web -- sh -c 'echo ok > /usr/share/nginx/html/index.html'   # back to 1/1
# B) app UNHEALTHY: liveness path now returns 404
kubectl exec deploy/web -- rm /usr/share/nginx/html/50x.html
kubectl get pods -l app=web -w                       # RESTARTS 0 -> 1, then 1/1 again
kubectl describe pod -l app=web | grep -iE 'unhealthy|restarted'
```
`web-probes.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /index.html
              port: 80
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /50x.html
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
```
**What each line does:**
- `readinessProbe httpGet`: every 5 s the kubelet calls `GET /index.html`; 200-399 = ready. Fail → Pod marked **not ready** and **removed from Service endpoints**, but **not restarted**.
- `livenessProbe httpGet`: checks `/50x.html` (a file shipped in the nginx image). 3 failures in a row (`failureThreshold`) → kubelet **kills and restarts** the container; `RESTARTS` goes up.
- `initialDelaySeconds`: wait before the first check so a starting app is not killed.
- Real apps use endpoints like `/ready` and `/healthz`; two nginx files are used here only so each probe can be broken separately.
- After the restart the container starts fresh from the image, so both files are back and the Pod is healthy.
**Verify:** Events show `Readiness probe failed: HTTP probe failed with statuscode: 404` (case A) and `Liveness probe failed ... Container nginx failed liveness probe, will be restarted` (case B).

**🧠 In simple words:** Readiness answers "can this Pod take traffic right now?"; if not, the Service just stops sending it requests. Liveness answers "is this Pod stuck or broken?"; if so, Kubernetes restarts it. In short: readiness failure = removed from traffic, liveness failure = restart.

### Lab 3 · Configure an HPA on a Deployment with CPU requests, generate load, watch it scale
**Answer (what to write in the exam):**
```bash
minikube addons enable metrics-server                # prerequisite 1: metrics
kubectl top pods                                     # works after ~1 min = metrics ready
kubectl apply -f web-hpa-app.yaml                    # Deployment (with CPU request) + Service
kubectl autoscale deployment web --cpu-percent=50 --min=1 --max=5
kubectl get hpa web                                  # TARGETS cpu: 0%/50%
kubectl run load --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://web; done"
kubectl get hpa web -w                               # TARGETS > 50%, REPLICAS 1 -> 2 ... 5
kubectl get deployment web                           # more Pods running
kubectl delete pod load                              # stop load; scales back to 1 after ~5 min
```
`web-hpa-app.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m
            limits:
              cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```
**What each line does:**
- HPA (Horizontal Pod Autoscaler): adds or removes Pod replicas to keep average CPU (Central Processing Unit) usage near a target.
- Prerequisites: (1) **metrics-server** running (HPA reads CPU from it; without it TARGETS shows `<unknown>`), (2) **CPU `requests`** set on the container, because "50%" means 50% of the requested `200m` (0.2 CPU core), (3) room on the node for extra Pods.
- `registry.k8s.io/hpa-example`: Kubernetes' demo PHP page that does CPU-heavy work on every request, so load is easy to create.
- `kubectl autoscale ... --cpu-percent=50 --min=1 --max=5`: creates the HPA (newest kubectl warns this flag is deprecated and also accepts `--cpu=50%`).
- `kubectl run load ... while true; do wget ...`: a busybox Pod that calls the Service non-stop.
- `kubectl get hpa -w`: watch live; HPA re-checks about every 15 s; scale-down waits ~5 min to avoid flapping.
**Verify:** `kubectl get hpa web` → e.g. `cpu: 180%/50%  MINPODS 1  MAXPODS 5  REPLICAS 4`; `kubectl describe hpa web` shows `New size: N; reason: cpu resource utilization above target`.
(Same HPA as YAML: `apiVersion: autoscaling/v2`, `scaleTargetRef: {kind: Deployment, name: web}`, `minReplicas: 1`, `maxReplicas: 5`, metric `cpu` with `averageUtilization: 50`.)

**🧠 In simple words:** The HPA watches how busy the Pods are compared to the CPU they asked for. When the busybox loop overloads them, it adds Pods up to 5; when the load stops, it slowly removes them. In short: metrics-server + CPU requests + HPA = automatic scaling on load.

---

## Session 14 · Kubernetes troubleshooting

### Lab 1 · Deploy with a bad image, diagnose ImagePullBackOff (get, describe, events), fix the image
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:doesnotexist   # broken on purpose
kubectl get pods -l app=web                  # STATUS ErrImagePull -> ImagePullBackOff, READY 0/1
kubectl describe pod -l app=web              # Events: Failed to pull image "nginx:doesnotexist": ... not found
kubectl get events --sort-by=.lastTimestamp  # newest events last: Pulling -> Failed -> BackOff
kubectl set image deployment/web nginx=nginx:1.27          # container "nginx" gets a real tag
kubectl rollout status deployment/web        # successfully rolled out
kubectl get pods -l app=web                  # Running, READY 1/1
```
**What each line does:**
- `create deployment ... --image=nginx:doesnotexist`: the tag does not exist on Docker Hub, so the node cannot download the image.
- `get pods`: quick status. `ErrImagePull` = first pull failed; `ImagePullBackOff` = Kubernetes is now waiting longer and longer (back-off) between retries.
- `describe pod`: the **Events** section at the bottom gives the exact reason (`manifest unknown` / `not found` = wrong name or tag; `unauthorized` / `pull access denied` = private registry needs an `imagePullSecret`).
- `get events --sort-by=.lastTimestamp`: all recent cluster events in time order (pull errors, scheduling errors).
- `set image deployment/web nginx=nginx:1.27`: `<container name>=<new image>`; `kubectl create deployment` names the container after the image (`nginx`). This triggers a new rollout.
- `rollout status`: waits until the new Pods are up.
**Verify:** `kubectl get pods -l app=web` → `Running 1/1`; `kubectl describe pod -l app=web | grep Image:` → `nginx:1.27`.
(In a manifest-based setup, fix the `image:` line in the YAML and `kubectl apply -f` again.)

**🧠 In simple words:** The Pod could not start because the node could not download the image. `describe` and `events` tell you why (here: the tag does not exist), and `set image` points the Deployment to a real image, which starts fresh Pods. In short: ImagePullBackOff = wrong or private image; read the Events, fix the image name.

### Lab 2 · Diagnose a CrashLoopBackOff with logs and describe, fix the config, verify recovery
**Answer (what to write in the exam):**
```bash
kubectl apply -f api.yaml
kubectl get pods -l app=api                  # STATUS CrashLoopBackOff, RESTARTS 3, 4, 5...
kubectl logs deploy/api --previous           # ERROR: DB_HOST is not set
kubectl describe pod -l app=api              # Last State: Terminated, Reason: Error, Exit Code: 1
                                             # Events: Back-off restarting failed container
kubectl set env deployment/api DB_HOST=db.default.svc.cluster.local   # add the missing setting
kubectl rollout status deployment/api
kubectl get pods -l app=api                  # Running 1/1, RESTARTS 0 (new Pod)
kubectl logs deploy/api                      # Started, DB=db.default.svc.cluster.local
```
`api.yaml` (broken: needs `DB_HOST` but none is given):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: busybox:1.36
          command: ["sh", "-c"]
          args:
            - 'if [ -z "$DB_HOST" ]; then echo "ERROR: DB_HOST is not set"; exit 1; fi; echo "Started, DB=$DB_HOST"; sleep 3600'
```
**What each line does:**
- CrashLoopBackOff: the container starts, exits with an error, is restarted, crashes again; Kubernetes waits longer between each retry.
- `logs --previous` (`-p`): shows logs of the **last crashed** container (the current one may have no output yet). This is where the root cause usually is.
- `describe pod`: `Last State: Terminated` with the **Exit Code**: `1` = app error, `137` = killed (often `OOMKilled`, out of memory: raise memory limit), `127` = command not found.
- `set env deployment/api DB_HOST=...`: adds the missing environment variable; the Deployment rolls out new Pods with it. (Better long-term: add `env:` to `api.yaml` and `kubectl apply -f api.yaml`.)
**Verify:** `kubectl get pods -l app=api` → `Running`, `RESTARTS 0`, and stays that way; `kubectl logs deploy/api` → `Started, DB=...`.

**🧠 In simple words:** The app kept crashing on startup because a required setting was missing. The previous container's logs said exactly what was wrong and `describe` confirmed it exited with code 1; adding the setting made the new Pods stay up. In short: CrashLoopBackOff = app keeps exiting; read `logs --previous` and the exit code, fix the cause, re-check.

### Lab 3 · Create a Service with a wrong selector, find the problem via labels and endpoints, fix it
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=2   # Pods get label app=web
kubectl apply -f web-svc.yaml                # Service selects app=webapp (typo)
kubectl get endpoints web-svc                # ENDPOINTS <none>   <- the symptom
kubectl describe svc web-svc                 # Selector: app=webapp, Endpoints: (empty)
kubectl get pods --show-labels               # LABELS app=web,...
kubectl get pods -l app=webapp               # No resources found -> selector matches nothing
kubectl edit svc web-svc                     # change selector to app: web, save
kubectl get endpoints web-svc                # 10.244.0.5:80,10.244.0.6:80
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://web-svc
```
`web-svc.yaml` (broken; the fix is `app: web`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
```
**What each line does:**
- A Service finds its Pods **only by labels**: `selector: app: webapp` means "send traffic to Pods labelled `app=webapp`". None exist, so the Service has no endpoints and requests fail.
- `get endpoints web-svc`: the list of Pod IP:port pairs behind the Service; `<none>` is the classic sign of a selector mismatch (or of Pods not being Ready).
- `describe svc`: shows the Service's `Selector`, `TargetPort` and `Endpoints` side by side.
- `get pods --show-labels` / `get pods -l app=webapp`: compare the real Pod labels with what the selector asks for.
- `kubectl edit svc web-svc`: opens the live object in an editor; changing the selector to `app: web` fixes it immediately. (Same fix: correct the YAML and `kubectl apply -f web-svc.yaml`, or `kubectl patch svc web-svc -p '{"spec":{"selector":{"app":"web"}}}'`.)
- Also check `targetPort` equals the container's port (80 for nginx); a wrong port gives endpoints but connection errors.
**Verify:** `kubectl get endpoints web-svc` → two Pod IPs on port 80; the `wget` from the temporary busybox Pod prints the nginx welcome HTML.

**🧠 In simple words:** The Service was looking for Pods with a label that no Pod had, so it pointed at nothing. Comparing the Service selector with the Pods' real labels showed the typo, and fixing the selector filled the endpoints. In short: empty endpoints = selector does not match Pod labels (or Pods not ready).

---

## Session 15 · Helm

> Helm = the package manager for Kubernetes (K8s). A **chart** = a folder of K8s YAML templates + default values. A **release** = one installed copy of a chart in the cluster. Every install/upgrade/rollback creates a new numbered **revision**.
> (Mac: `brew install helm`; use minikube as the cluster. Commands are identical.)

### Lab 1 · Create a Helm chart for a web app and explain Chart.yaml, values.yaml, templates/
**Answer (what to write in the exam):**
```bash
helm create web                  # scaffolds a ready-made nginx chart in folder ./web
ls -R web                        # (or: tree web) show the layout
cat web/Chart.yaml web/values.yaml
helm lint ./web                  # check the chart for mistakes
helm template web ./web          # render the final YAML locally, installs nothing
```
Layout and what each part is for:
```
web/
├── Chart.yaml          # chart metadata: name, chart version, app version
├── values.yaml         # DEFAULT settings that fill the template blanks
├── charts/             # sub-charts this chart depends on (empty)
└── templates/          # K8s YAML with {{ .Values.x }} blanks
    ├── deployment.yaml, service.yaml, serviceaccount.yaml, ingress.yaml, hpa.yaml
    ├── _helpers.tpl     # reusable snippets (names, labels); "_" = not rendered as a file
    ├── NOTES.txt        # text printed after install (how to reach the app)
    └── tests/test-connection.yaml   # run with: helm test web
```
`web/Chart.yaml`:
```yaml
apiVersion: v2            # chart format for Helm 3/4
name: web                 # chart name
description: A Helm chart for Kubernetes
type: application
version: 0.1.0            # CHART version: bump when templates/values change
appVersion: "1.16.0"      # version of the APP inside; default image tag
```
`web/values.yaml` (key defaults):
```yaml
replicaCount: 1
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""                 # empty means "use appVersion"
service:
  type: ClusterIP
  port: 80
```
`templates/deployment.yaml` (excerpt showing the blanks):
```yaml
  replicas: {{ .Values.replicaCount }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```
**What each line does:**
- `helm create web`: makes a working starter chart (Deployment + Service for nginx) so you don't write YAML from zero.
- `Chart.yaml`: the chart's ID card. `version` = chart version, `appVersion` = app version (they are different numbers), `apiVersion: v2` = Helm 3+ format.
- `values.yaml`: the default "settings"; users override them with `--set` or `-f my-values.yaml`.
- `templates/`: K8s manifests where `{{ .Values.x }}` is replaced by a value; `{{ .Chart.x }}` / `{{ .Release.Name }}` come from Chart.yaml and the release name.
- `helm lint`: checks syntax and structure. `helm template`: prints the rendered YAML without touching the cluster.

**Verify:** `helm lint ./web` → `1 chart(s) linted, 0 chart(s) failed`; `helm template web ./web | grep -E 'kind:|image:'` → `kind: Service`, `kind: Deployment`, `image: "nginx:1.16.0"`.

**🧠 In simple words:** A chart is a reusable template for an app's K8s objects. Chart.yaml describes the chart, values.yaml holds the default settings, and templates/ holds the YAML with blanks that Helm fills from those values. In short: templates + values = the final K8s YAML, and Chart.yaml says what and which version.

### Lab 2 · Deploy the chart with custom image, replicas and Service port; verify rendered and installed resources
**Answer (what to write in the exam):**
```bash
# 1) In templates/deployment.yaml change  containerPort: {{ .Values.service.port }}
#    to  containerPort: {{ .Values.containerPort }}  and add  containerPort: 80  to values.yaml
helm lint ./web
helm template web ./web --set image.tag=1.27 --set replicaCount=3 --set service.port=8080 \
  | grep -E 'replicas:|image:|port:'                  # check the rendered YAML first
helm install web ./web --set image.tag=1.27 --set replicaCount=3 --set service.port=8080
helm list                                             # release "web", REVISION 1, deployed
helm get values web                                   # only the values YOU overrode
helm get manifest web                                 # exact YAML Helm sent to the cluster
kubectl get all -l app.kubernetes.io/instance=web     # 3 pods, deploy, rs, svc
kubectl rollout status deployment/web
kubectl get endpoints web                             # 3 pod IPs on port 80
kubectl port-forward svc/web 8080:8080                # then in another tab: curl localhost:8080
```
Same thing with a values file (cleaner for many settings): `helm install web ./web -f prod-values.yaml`
```yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.27"
service:
  port: 8080
```
**What each line does:**
- Step 1 (gotcha): the starter chart reuses `service.port` as the container port, but nginx listens on 80; with 8080 the probes fail and pods restart. A separate `containerPort` value fixes it. The Service then maps port 8080 → `targetPort: http` (80).
- `--set key=value`: overrides one value from values.yaml (`image.tag` = nested key). `-f file.yaml` overrides many at once.
- `helm install <release> <chart>`: renders the templates and creates the objects; this is REVISION 1.
- `helm list`: shows releases (name, namespace, revision, status, chart, app version).
- `helm get values` / `helm get manifest`: what was set / what was actually applied.
- `-l app.kubernetes.io/instance=web`: label every chart object gets, so this lists everything from the release.

**Verify:** `helm list` → `web  default  1  deployed  web-0.1.0  1.16.0`; `kubectl get deploy web` → `READY 3/3`; `kubectl get svc web` → `PORT(S) 8080/TCP`; `helm get values web` → `USER-SUPPLIED VALUES:` listing image.tag 1.27, replicaCount 3, service.port 8080.

**🧠 In simple words:** The same chart can be installed with different settings without editing its files: you pass the image tag, replica count and port at install time. `helm template` shows the YAML before installing, and `helm get manifest` plus `kubectl get all` prove what is really running. In short: `--set`/`-f` customise, `template` previews, `install` deploys, `get`/`kubectl` verify.

### Lab 3 · Upgrade a release, inspect history, roll back, verify final status
**Answer (what to write in the exam):**
```bash
helm upgrade web ./web --reuse-values --set image.tag=1.28   # REVISION 2
kubectl rollout status deployment/web
helm history web                                              # 1 superseded, 2 deployed
helm rollback web 1                                           # go back to rev 1 → creates REVISION 3
kubectl rollout status deployment/web
helm history web                                              # 3 deployed "Rollback to 1"
helm status web                                               # STATUS: deployed, REVISION: 3
helm get values web                                           # image.tag is 1.27 again
kubectl get deploy web -o jsonpath='{.spec.template.spec.containers[0].image}'   # nginx:1.27
```
Expected `helm history web` after the rollback:
```
REVISION  STATUS      CHART      APP VERSION  DESCRIPTION
1         superseded  web-0.1.0  1.16.0       Install complete
2         superseded  web-0.1.0  1.16.0       Upgrade complete
3         deployed    web-0.1.0  1.16.0       Rollback to 1
```
**What each line does:**
- `helm upgrade <release> <chart>`: applies new values/chart version to the existing release; makes a new revision. `--reuse-values` keeps the old `--set` values (without it, replicas and port reset to the chart defaults).
- `helm history web`: table of all revisions with status and description.
- `helm rollback web 1`: re-applies revision 1's config. It does NOT delete revision 2; it writes a NEW revision 3 that copies revision 1. (`helm rollback web` with no number = the previous revision.)
- `helm status web`: current state, revision number, last deploy time, and NOTES.txt.
- `helm get values` + `kubectl get deploy … jsonpath`: prove the image really went back to 1.27.

**Verify:** `helm status web` → `STATUS: deployed`, `REVISION: 3`; `kubectl get pods -l app.kubernetes.io/instance=web` → 3 Running pods with image `nginx:1.27`.

**🧠 In simple words:** Helm keeps a numbered record of every change to a release. An upgrade adds a revision; if it goes wrong, `helm rollback` puts back an older revision's settings as a brand-new revision, so history is never lost. In short: upgrade → history → rollback (new revision number) → status to confirm.

---

## Session 16 · CI/CD & GitHub Actions

> GitHub Actions = GitHub's built-in CI/CD (Continuous Integration / Continuous Delivery). A **workflow** is a YAML file in `.github/workflows/`; it has triggers (`on:`), **jobs** (run on a **runner** VM chosen by `runs-on:`), and each job has **steps** (`uses:` = run a ready-made action, `run:` = run shell commands).
> Example app used below: a tiny Node.js app with `package.json` scripts `"build": "mkdir -p dist && cp index.js sum.js dist/"` and `"test": "node --test"`, plus a committed `package-lock.json`.

### Lab 1 · Workflow on push: checkout, install dependencies, build
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

### Lab 2 · Add a test step; show a failing test fails the run and a fixed test passes
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

### Lab 3 · Upload a build artifact downloadable from the Actions run
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

---

## Session 17 · Complete CI/CD & DevSecOps

> DevSecOps = adding security checks into every stage of the CI/CD (Continuous Integration / Continuous Delivery) pipeline ("shift left" = check early). All workflows below live in `.github/workflows/` of a repo that has a `Dockerfile` and a Node.js app with `package-lock.json`.

### Lab 1 · Pipeline that builds a Docker image and pushes it to a registry using repository secrets
**Answer (what to write in the exam):**
```bash
# 1) Docker Hub → Account settings → Personal access tokens → create token (Read & Write)
# 2) GitHub repo → Settings → Secrets and variables → Actions → New repository secret
gh secret set DOCKERHUB_USERNAME          # (CLI way) paste your Docker Hub username
gh secret set DOCKERHUB_TOKEN             # paste the access token (never your password)
# 3) add .github/workflows/docker-publish.yml (below), then:
git add .github/workflows/docker-publish.yml && git commit -m "ci: build and push image" && git push
gh run watch
docker pull <your-dockerhub-user>/web:<commit-sha>    # proves the image is in the registry
```
`.github/workflows/docker-publish.yml`:
```yaml
name: Docker publish

on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  build-push:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/web:${{ github.sha }}
```
**What each line does:**
- `${{ secrets.DOCKERHUB_TOKEN }}`: reads an encrypted repository secret at run time; GitHub masks it as `***` in logs, and it is never in the YAML or Git history.
- `permissions: contents: read`: least privilege; the built-in `GITHUB_TOKEN` can only read the repo.
- `docker/login-action@v3`: runs `docker login` with the secrets.
- `docker/build-push-action@v6`: builds from the `Dockerfile` in `context: .`; `push: true` uploads it; `tags:` = image name, tagged with the commit SHA so every build is traceable.
- GHCR (GitHub Container Registry) instead: `registry: ghcr.io`, `username: ${{ github.actor }}`, `password: ${{ secrets.GITHUB_TOKEN }}`, add `packages: write` under `permissions:`, tag `ghcr.io/<owner>/web:<sha>`.

**Verify:** run is ✅, log shows `Login Succeeded` and `pushing ... done` with the username shown as `***`; Docker Hub → repository `web` shows a tag equal to the commit SHA.

**🧠 In simple words:** On every push GitHub builds the Docker image and uploads it to Docker Hub. The login details are stored as encrypted repository secrets and injected only while the job runs, so no password is ever written in the code. In short: secrets in Settings → `${{ secrets.X }}` in YAML → login → build → push.

### Lab 2 · Add SAST, dependency scanning and secret scanning; make the security check block later stages
**Answer (what to write in the exam):**
```bash
# 1) add .github/workflows/secure-ci.yml (below) and push
git add . && git commit -m "ci: add security job" && git push
gh run watch                                  # security ✅ → build runs
# 2) GitHub → Settings → Branches → Add branch protection rule (or Rules → Rulesets) for "main":
#    tick "Require a pull request before merging" + "Require status checks to pass",
#    search and add the check "security" → Save
# 3) test it: open a PR that adds a vulnerable package (e.g. npm install lodash@4.17.15)
gh pr create --fill
gh pr checks                                  # security ❌ fail, build - skipped, merge blocked
```
`.github/workflows/secure-ci.yml`:
```yaml
name: Secure CI

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

permissions:
  contents: read

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code (full history for secret scan)
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: SAST with Semgrep
        run: docker run --rm -v "$PWD:/src" semgrep/semgrep semgrep scan --config auto --error

      - name: Dependency scan with npm audit
        run: npm audit --audit-level=high

      - name: Secret scan with gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  build:
    needs: security
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Build image
        run: docker build -t web:${{ github.sha }} .
```
**What each line does:**
- SAST (Static Application Security Testing) = Semgrep reads the source code for insecure patterns without running it; `--error` makes it exit 1 if it finds anything. (Alternatives: CodeQL, or `bandit -r .` for Python.)
- Dependency scan = `npm audit --audit-level=high` checks `package-lock.json` against known CVEs (Common Vulnerabilities and Exposures) and fails on HIGH or CRITICAL. (Python: `pip-audit -r requirements.txt`; any language: `trivy fs --exit-code 1 .`)
- Secret scan = gitleaks searches the code and Git history (`fetch-depth: 0`) for passwords, keys and tokens.
- `needs: security`: `build` waits for `security` and is **skipped** if it fails, so nothing later runs.
- Branch protection "Require status checks to pass" + check `security`: GitHub refuses to merge a PR into main until that job is green.

**Verify:** a PR with `lodash@4.17.15` (or a committed fake token) shows `security` ❌, `build` skipped, and the merge button says "Required statuses must pass"; after removing it the checks are ✅ and merge is allowed.

**🧠 In simple words:** The security job checks three things on every push and PR: your own code (SAST), the libraries you use (dependency scan), and leaked passwords or keys (secret scan). Because the build job `needs` it and branch protection requires it, a failed check stops the pipeline and blocks the merge. In short: security job first, everything else `needs:` it, and main only accepts green PRs.

### Lab 3 · Container image scanning with a gate that blocks deployment above a severity threshold
**Answer (what to write in the exam):**
```bash
# Threshold agreed: no HIGH or CRITICAL vulnerability that has a fix available.
# Same check by hand on a lab machine (exit code 1 = gate closed):
docker build -t web:test .
trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed web:test
echo $?                                        # 1 = vulnerabilities found, 0 = clean
# In CI: add .github/workflows/image-gate.yml (below), push, then:
gh run watch                                   # scan ❌ → push + deploy skipped
```
`.github/workflows/image-gate.yml`:
```yaml
name: Image scan gate

on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  build-scan-push:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Build image (local only)
        run: docker build -t web:${{ github.sha }} .

      - name: Trivy scan (gate)
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: web:${{ github.sha }}
          severity: CRITICAL,HIGH
          ignore-unfixed: true
          exit-code: '1'

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Push image (only if scan passed)
        run: |
          docker tag web:${{ github.sha }} ${{ secrets.DOCKERHUB_USERNAME }}/web:${{ github.sha }}
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/web:${{ github.sha }}

  deploy:
    needs: build-scan-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy
        run: echo "Deploying web:${{ github.sha }} (e.g. helm upgrade / kubectl set image)"
```
**What each line does:**
- `trivy image`: lists the OS and library packages inside the image and matches them against known CVEs.
- `severity: CRITICAL,HIGH` (`--severity`): the agreed threshold; only these levels count.
- `exit-code: '1'` (`--exit-code 1`): fail the step if any such CVE is found (default is 0 = report only, never blocks).
- `ignore-unfixed: true` (`--ignore-unfixed`): skip CVEs with no patch yet, so the gate is actionable.
- Scan runs BEFORE login/push, so a vulnerable image never reaches the registry; `deploy` has `needs: build-scan-push`, so it is skipped when the scan fails.
- `environment: production`: optional extra gate; add "Required reviewers" to it in Settings → Environments.
- Pin actions to a fixed version (or commit SHA) so a moved tag cannot change what runs.

**Verify:** with an old base image (e.g. `FROM node:14`) the log shows a table of HIGH/CRITICAL CVEs, the Trivy step is ❌, push and `deploy` show "skipped"; switch to a current base (e.g. `FROM node:22-alpine`) → `Total: 0 (HIGH: 0, CRITICAL: 0)` → push and deploy ✅.

**🧠 In simple words:** Trivy looks inside the built image for software with known security holes. If it finds any HIGH or CRITICAL ones that can be fixed, it exits with code 1, which fails the job, and because deploy `needs` that job, nothing is pushed or deployed. In short: build → scan with a severity threshold → only a clean image is pushed and deployed.

---

## Session 18 · Terraform & IaC

IaC = Infrastructure as Code (servers, networks and files described in code files instead of clicked by hand). Terraform files use HCL (HashiCorp Configuration Language) and end in `.tf`. These labs use the `local` provider (writes files on your machine) and the `random` provider (generates random values), so they work anywhere with no cloud account.

### Lab 1 · Declare a provider, create a small resource, run fmt / init / validate / plan, explain the output
**Answer (what to write in the exam):**
```bash
mkdir tf-lab && cd tf-lab       # create main.tf (below) in this folder
terraform fmt                   # tidy formatting
terraform init                  # download the local + random providers
terraform validate              # check syntax and references
terraform plan                  # preview only: nothing is created yet
```
`main.tf`
```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

provider "local" {}
provider "random" {}

resource "random_pet" "name" {
  length = 2
}

resource "local_file" "hello" {
  filename = "${path.module}/hello.txt"
  content  = "Hello from ${random_pet.name.id}\n"
}
```
**What each line does:**
- `required_providers`: which plugins (providers) this code needs, where to download them from, and which versions are allowed (`~> 2.5` = 2.5 or newer, but below 3.0).
- `provider "local" {}` / `provider "random" {}`: turn the providers on (these two need no settings).
- `resource "random_pet" "name"`: creates a random name such as `happy-tiger`; the full address is `random_pet.name`.
- `resource "local_file" "hello"`: writes `hello.txt`. Using `${random_pet.name.id}` makes it depend on the pet, so Terraform creates the pet first.
- `terraform fmt`: rewrites `.tf` files in standard style (prints the names of the files it changed).
- `terraform init`: downloads the providers into `.terraform/` and writes `.terraform.lock.hcl`. Run it first, and again whenever you add a provider.
- `terraform validate`: checks syntax, types and references without contacting anything.
- `terraform plan`: compares your code with the state file (empty at first) and prints what it *would* do.

**Verify / read the output:**
- `validate` → `Success! The configuration is valid.`
- `plan` → `# local_file.hello will be created` and `# random_pet.name will be created`, each with a `+` in front.
  - `+ create` = new resource. Other symbols: `~` update in place, `-/+` destroy and re-create, `-` destroy.
  - `(known after apply)` = the value only exists after creation (for example, the random name).
  - Last line: `Plan: 2 to add, 0 to change, 0 to destroy.` = 2 new resources, nothing modified or deleted.
  - The note "You didn't use the -out option" means you can run `terraform plan -out=tfplan` and then `terraform apply tfplan` to apply exactly that plan.

**🧠 In simple words:** You describe what you want (a random name and a file) in `main.tf`. `init` downloads the plugins, `fmt` and `validate` check the code, and `plan` shows a dry-run list of changes without touching anything. In short: write `.tf`, then run fmt → init → validate → plan, and read the `+` lines and the "Plan: X to add" summary.

### Lab 2 · Use variables and outputs, pass values through a .tfvars file, print an output after apply
**Answer (what to write in the exam):**
```bash
terraform fmt && terraform init && terraform validate
terraform plan  -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"    # type: yes
terraform output                                 # list all outputs
terraform output -raw file_path                  # just the value, good for scripts
cat shop-test.txt
```
`variables.tf`
```hcl
variable "app_name" {
  description = "Name of the app"
  type        = string
}

variable "environment" {
  description = "dev, test or prod"
  type        = string
  default     = "dev"
}
```
`main.tf`
```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

resource "local_file" "config" {
  filename = "${path.module}/${var.app_name}-${var.environment}.txt"
  content  = "app=${var.app_name}\nenv=${var.environment}\n"
}
```
`outputs.tf`
```hcl
output "file_path" {
  description = "Where the config file was written"
  value       = local_file.config.filename
}
```
`terraform.tfvars`
```hcl
app_name    = "shop"
environment = "test"
```
**What each line does:**
- `variable "app_name"`: declares an input. It has no `default`, so it must be given a value (or Terraform asks for it).
- `var.app_name`: how the code reads a variable's value.
- `terraform.tfvars`: the values file. Terraform loads a file with exactly this name automatically. You need `-var-file=` for any other name, such as `-var-file=prod.tfvars`.
- Precedence (later wins): `default` → `terraform.tfvars` → `*.auto.tfvars` → `-var-file` / `-var` on the command line.
- `output "file_path"`: prints a useful value after apply. Other tools can read it with `terraform output`.

**Verify:** `apply` ends with `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.` and then `Outputs: file_path = "./shop-test.txt"`. `cat shop-test.txt` → `app=shop` / `env=test`.

**🧠 In simple words:** Variables make one piece of code reusable: the same `main.tf` can build dev, test or prod just by changing the `.tfvars` file. Outputs print the important results (like a file path or a server IP) after apply. In short: declare inputs in variables.tf, set them in a .tfvars file, and read results with `terraform output`.

### Lab 3 · Modify a resource, inspect the plan before applying, explain state, and keep secrets and state out of Git
**Answer (what to write in the exam):**
```bash
terraform apply                    # first build: random_pet.name + local_file.hello
# edit main.tf: change the file content (below)
terraform plan                     # ALWAYS read the plan before applying
terraform apply                    # type yes only if the plan is what you expect
terraform state list               # resources Terraform is tracking
terraform state show local_file.hello
git status                         # state and .terraform/ must NOT be listed
```
Change in `main.tf`:
```hcl
resource "local_file" "hello" {
  filename = "${path.module}/hello.txt"
  content  = "Hi again from ${random_pet.name.id}\n"   # was "Hello from ..."
}
```
`.gitignore`
```gitignore
.terraform/
*.tfstate*
*.tfvars
tfplan
crash.log
```
**What the plan shows:**
- `-/+ resource "local_file" "hello"` with `~ content = "Hello…" -> "Hi again…" # forces replacement` → `Plan: 1 to add, 0 to change, 1 to destroy.` The provider cannot edit this attribute in place, so Terraform deletes the file and creates it again.
- `~ update in-place`: shown when an attribute can be changed without re-creating (for example, changing `tags` on a cloud virtual machine) → `Plan: 0 to add, 1 to change, 0 to destroy.`
- `random_pet.name` is not listed, because its config did not change.

**How state maps config to real infrastructure:**
- `terraform.tfstate` is a JSON (JavaScript Object Notation) file. For each resource address in the code (`local_file.hello`), it stores the real object's ID and attributes.
- `plan` = read the real objects (refresh) + compare them with your code + compare them with the state, then print the difference. Without state, Terraform would not know that it already created something.
- Don't commit state: it can hold secrets in plain text (passwords, keys), and two people with different local copies would overwrite each other. Teams keep state in a remote backend with locking (for example, an S3 bucket with locking, or Terraform Cloud).
- `*.tfvars` is ignored only when it holds secrets. Commit `.terraform.lock.hcl`, because it pins provider versions.

**Verify:** `terraform state list` → `local_file.hello` and `random_pet.name`. `git check-ignore -v terraform.tfstate` → prints the `.gitignore` rule that hides it.

**🧠 In simple words:** When you edit code, Terraform compares it with its state file and the real world, then shows a plan: `~` means "edit it", `-/+` means "delete and rebuild it". Reading the plan first stops surprise deletions. In short: edit → plan → read the symbols → apply, and keep `*.tfstate*`, `.terraform/` and secret `.tfvars` files out of Git.

---

## Session 19 · Cloud & Terraform in action

Acronyms: VPC = Virtual Private Cloud (your own private network inside AWS, Amazon Web Services). CIDR = Classless Inter-Domain Routing (an IP range written like `10.0.0.0/16`). IGW = Internet Gateway. NAT = Network Address Translation. SG = Security Group (a firewall around a server). EC2 = Elastic Compute Cloud (AWS virtual machines). AMI = Amazon Machine Image (the operating-system image a VM boots from). AZ = Availability Zone (one data-centre location inside a region). Azure uses the names VNet and NSG (Network Security Group) for the same ideas.

### Lab 1 · Design a VPC with public and private subnets, route tables and security rules; draw it and explain the traffic flow
**Answer (model diagram):**
```
                     Internet (users, admin laptop)
                                 |
                     [ Internet Gateway (IGW) ]
                                 |
+---------------------- VPC 10.0.0.0/16 --------------------------+
|                                                                 |
|  PUBLIC subnet 10.0.1.0/24       PRIVATE subnet 10.0.2.0/24     |
|  +-------------------------+     +-------------------------+    |
|  | web server   [SG-web]   |---->| app / DB    [SG-app]    |    |
|  | NAT gateway (Elastic IP)|<----| (no public IP)          |    |
|  +-------------------------+     +-------------------------+    |
|  Route table "public"            Route table "private"          |
|   10.0.0.0/16 -> local            10.0.0.0/16 -> local          |
|   0.0.0.0/0   -> IGW              0.0.0.0/0   -> NAT gateway    |
+-----------------------------------------------------------------+
```
**Security rules:**
| Security group | Inbound allowed | Outbound |
|---|---|---|
| SG-web (public subnet) | TCP 80 and 443 from `0.0.0.0/0`. TCP 22 (SSH, Secure Shell) **only** from the admin's IP, e.g. `203.0.113.10/32` | all |
| SG-app (private subnet) | app port 8080 **only** from SG-web (a security group is the source, not an IP range) | all (leaves through the NAT gateway) |

**Traffic flow:**
1. **User → web:** internet → IGW → public route table → web server. SG-web allows port 443.
2. **Web → app:** web server → app on 8080 through the `local` route (traffic stays inside the VPC). SG-app allows it only because the source is SG-web.
3. **Private server → internet (updates):** app → private route table `0.0.0.0/0` → NAT gateway in the public subnet → IGW → internet. Replies come back the same way.
4. **Blocked:** the internet cannot start a connection to the private subnet (the servers have no public IP, and NAT only lets out connections that started inside). SSH from any IP except the admin `/32` is dropped by SG-web.
- Security groups are *stateful*: reply traffic is allowed automatically. NACLs (Network Access Control Lists) are optional, *stateless* filters at the subnet level.
- A subnet is "public" only because its route table sends `0.0.0.0/0` to an IGW.

**🧠 In simple words:** The VPC is your private network. The public subnet holds things the internet must reach, and the private subnet hides the app and database. Route tables decide where traffic goes (IGW for public, NAT for private outbound only), and security groups decide which ports are allowed. In short: public = route to IGW, private = route to NAT, and SGs open only the ports you need.

### Lab 2 · Use Terraform to build a small VPC and one cheap VM in a sandbox account; plan before apply, then destroy
**Answer (what to write in the exam):**
```bash
aws configure --profile sandbox        # keys are saved in ~/.aws, never in .tf files
export AWS_PROFILE=sandbox
terraform fmt && terraform init && terraform validate
terraform plan -var="admin_cidr=203.0.113.10/32" -out=tfplan
terraform apply tfplan                 # applies exactly the reviewed plan
terraform output public_ip
curl http://$(terraform output -raw public_ip)
terraform destroy -var="admin_cidr=203.0.113.10/32"   # type yes
```
`main.tf`
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region # credentials come from `aws configure` / env vars, never from this file
}

variable "region" {
  type    = string
  default = "ap-south-1"
}

variable "admin_cidr" {
  type = string # e.g. "203.0.113.10/32"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "lab-vpc" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  tags                    = { Name = "lab-public" }
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  tags       = { Name = "lab-private" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "web" {
  name   = "lab-web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023*-x86_64"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.al2023.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  tags                   = { Name = "lab-web" }
  user_data              = <<-EOT
    #!/bin/bash
    dnf install -y nginx
    systemctl enable --now nginx
  EOT
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```
**What each line does:**
- `provider "aws"`: chooses the region. Credentials come from `aws configure` or environment variables, so they are never written in code.
- `aws_vpc` / `aws_subnet`: the network (`/16` = 65,536 addresses) and two `/24` slices (256 addresses each). `map_public_ip_on_launch` gives VMs in the public subnet a public IP.
- `aws_internet_gateway` + `aws_route_table` (`0.0.0.0/0 → igw`) + `aws_route_table_association`: these three make the subnet public. The private subnet has no such route. (A NAT gateway is skipped in a sandbox because it is billed per hour.)
- `aws_security_group`: allows 80 and 443 from anywhere, and SSH only from `var.admin_cidr`. `egress` allows all outbound traffic (Terraform removes AWS's default egress rule, so you must add it).
- `data "aws_ami"`: looks up the newest Amazon Linux 2023 image. `t3.micro` is a free-tier-sized VM. `user_data` installs nginx (a web server) at first boot.
- `plan -out=tfplan` then `apply tfplan`: review first, then apply exactly what you reviewed. `destroy` deletes everything in the state, so nothing keeps costing money.

**Verify:** the plan ends with `Plan: 8 to add, 0 to change, 0 to destroy.`, apply with `Apply complete! Resources: 8 added`, `curl` returns the nginx welcome page, and destroy ends with `Destroy complete! Resources: 8 destroyed.`

**🧠 In simple words:** One `main.tf` describes the whole network and the VM, so anyone can rebuild the same thing with one command. Planning first shows exactly what will be created, and destroy cleans up so the sandbox bill stays at zero. In short: write HCL → init → plan → apply → test → destroy, with credentials kept outside the code.

### Lab 3 · Security group that allows only the app port; prove SSH is not open to the whole internet
**Answer (what to write in the exam):**
```bash
MYIP=$(curl -s https://checkip.amazonaws.com)          # your public IP
terraform plan  -var="admin_cidr=${MYIP}/32" -out=tfplan
terraform apply tfplan
aws ec2 describe-security-groups --filters Name=group-name,Values=lab-web-sg \
  --query 'SecurityGroups[0].IpPermissions[].{port:FromPort,from:IpRanges[0].CidrIp}' --output table
curl -I http://<public_ip>             # app port works from anywhere
nc -zv -w 5 <public_ip> 22             # from YOUR IP: succeeded
nc -zv -w 5 <public_ip> 22             # from another network (phone hotspot): timed out
terraform plan -var="admin_cidr=0.0.0.0/0"   # rejected by the validation rule
```
Security rules in `main.tf`:
```hcl
variable "admin_cidr" {
  description = "Your own public IP as /32, e.g. 203.0.113.10/32"
  type        = string
  validation {
    condition     = endswith(var.admin_cidr, "/32")
    error_message = "admin_cidr must be ONE IP (/32), never 0.0.0.0/0."
  }
}

resource "aws_security_group" "web" {
  name   = "lab-web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "SSH only from admin IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```
**What each line does:**
- Ingress on port 80 from `0.0.0.0/0`: the only public port, because it is the application port. Add a 443 block the same way if the app uses HTTPS.
- Ingress on port 22 with `[var.admin_cidr]`: SSH is allowed from one IP only (`/32` = exactly one address). Every port not listed is denied by default.
- `validation { … }`: Terraform refuses any value that is not a single `/32`, so `0.0.0.0/0` (the whole internet) can never be applied by mistake.
- `describe-security-groups --query …`: prints the real rules that AWS stored, as evidence.
- `nc -zv -w 5 host 22`: tries a TCP connection with a 5-second timeout. "succeeded" means open; "timed out" means blocked.

**Verify:** the table shows `80 → 0.0.0.0/0` and `22 → <your IP>/32` (never `22 → 0.0.0.0/0`). `curl -I` → `HTTP/1.1 200 OK`. Port 22 from another network times out. The last plan fails with `Invalid value for variable … never 0.0.0.0/0`.

**🧠 In simple words:** A security group is an allow-list: anything not listed is blocked. Opening only port 80 to everyone and port 22 to your single IP means attackers scanning the internet cannot even try to log in over SSH. In short: app port open to `0.0.0.0/0`, SSH only from `your-ip/32`, proven with `describe-security-groups` and a blocked `nc` test.

---

## Session 20 · Monitoring, Observability & GitOps

Acronyms: PromQL = Prometheus Query Language. RED = Rate, Errors, Duration (the three standard panels for any service). p95 = 95th percentile. CRD = Custom Resource Definition (a new object type added to Kubernetes, such as Argo CD's `Application`). UI = User Interface. CLI = Command-Line Interface.

### Lab 1 · Expose app metrics, make Prometheus scrape them, check the target is UP, name two metrics
**Answer (what to write in the exam):**
```bash
python3 -m venv venv && . venv/bin/activate && pip install prometheus-client
python3 app.py &                                     # metrics at :8000/metrics
curl -s localhost:8000/metrics | grep http_requests_total
docker run -d --name prometheus -p 9090:9090 \
  --add-host=host.docker.internal:host-gateway \
  -v "$(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml" \
  prom/prometheus
# Browser: http://localhost:9090 → Status → Targets  (both jobs must show UP)
curl -s 'localhost:9090/api/v1/query?query=up'
```
`app.py`
```python
from prometheus_client import Counter, Histogram, start_http_server
import random, time

REQUESTS = Counter("http_requests_total", "Total HTTP requests", ["status"])
LATENCY = Histogram("http_request_duration_seconds", "Request latency in seconds")

start_http_server(8000)          # serves http://localhost:8000/metrics
while True:                      # fake traffic: one request at a time
    with LATENCY.time():
        time.sleep(random.random() / 5)
    REQUESTS.labels(status=random.choice(["200", "200", "200", "500"])).inc()
```
`prometheus.yml`
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: myapp
    scrape_interval: 5s
    metrics_path: /metrics
    static_configs:
      - targets: ["host.docker.internal:8000"]
```
**What each line does:**
- `Counter` / `Histogram`: the app records a request count (labelled by HTTP status) and the time each request took. `start_http_server(8000)` publishes these numbers as text at `/metrics`.
- `scrape_configs`: the list of things Prometheus pulls metrics from. `job_name` is the label it gives them. `scrape_interval` sets how often it pulls (5s here; otherwise the global 15s). `static_configs.targets` lists the `host:port` to scrape.
- `--add-host=host.docker.internal:host-gateway`: lets the Prometheus container reach the app running on the host machine (needed on Linux; Docker Desktop on Mac has it built in).
- `-v …prometheus.yml:/etc/prometheus/prometheus.yml`: mounts your config file over the default one.

**Verify:** Status → Targets shows `myapp` and `prometheus` with State **UP**. In the Graph tab, `up{job="myapp"}` → `1` (0 would mean the scrape failed).
**Two or more metrics:**
- `http_requests_total{status="200"}`: a counter of requests served (it only goes up).
- `http_request_duration_seconds_bucket`: a latency histogram (used for p95).
- `process_cpu_seconds_total{job="prometheus"}`: CPU (Central Processing Unit) seconds the process has used. (Mac: the Python app shows `process_*` metrics only on Linux, so read this one from the `prometheus` job.)

**🧠 In simple words:** The app shows its numbers on a `/metrics` web page, and Prometheus visits that page every few seconds and stores each value with a timestamp. The Targets page and the `up` metric prove the scraping works. In short: expose `/metrics`, list it under `scrape_configs`, and check that Targets says UP.

### Lab 2 · Grafana dashboard with request rate, error and latency panels; explain each panel
**Answer (what to write in the exam):**
```bash
docker network create monitoring
docker network connect monitoring prometheus
docker run -d --name grafana --network monitoring -p 3000:3000 grafana/grafana
# Browser: http://localhost:3000 (first login admin/admin, then set a new password)
# Connections → Data sources → Add → Prometheus → URL http://prometheus:9090 → Save & test
# Dashboards → New → Add visualization → Prometheus → paste PromQL → set unit → Apply
# Repeat for the 3 panels below → Save dashboard ("App RED")
```
| Panel (Time series) | PromQL | Unit | What it indicates |
|---|---|---|---|
| Request rate | `sum(rate(http_requests_total[5m]))` | req/s | Traffic: how many requests per second the app is serving. |
| Error rate | `sum(rate(http_requests_total{status=~"5.."}[5m]))` | req/s | Server errors (5xx) per second. It should stay near 0. |
| Error % (optional) | `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))` | percent (0.0-1.0) | The fraction of requests that fail. |
| Latency p95 | `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))` | seconds | 95% of requests finished faster than this value. Shows slowness that users feel. |

**What each part means:**
- `rate(x[5m])`: the per-second increase of a counter, averaged over the last 5 minutes (it handles counter resets).
- `sum(...)`: adds all label series (all statuses or instances) into one line.
- `{status=~"5.."}`: a regular-expression match, meaning any status from 500 to 599.
- `sum by (le)`: keeps the histogram's `le` ("less than or equal") bucket label, which `histogram_quantile` needs to work out p95.
- Data source URL `http://prometheus:9090`: both containers are on the `monitoring` network, so Grafana reaches Prometheus by its container name.
- If the supplied environment has no app, use Prometheus's own metrics: `prometheus_http_requests_total` (its status label is `code`) and `prometheus_http_request_duration_seconds_bucket`.

**Verify:** "Save & test" → `Successfully queried the Prometheus API.` With the app from Lab 1, the panels show about 10 req/s, about 25% errors and a p95 of about 0.2 s.

**🧠 In simple words:** Grafana does not store data. It asks Prometheus with PromQL and draws the answers as graphs. Rate, errors and latency (RED) answer "how busy is it, how often does it fail, how slow is it", which is enough to spot most problems. In short: add Prometheus as a data source and build three panels: `rate()` for traffic, a `5..` filter for errors, and `histogram_quantile(0.95, …)` for latency.

### Lab 3 · GitOps with Argo CD: change the replica count in Git and see the cluster reconcile
**Answer (what to write in the exam):**
```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
kubectl -n argocd port-forward svc/argocd-server 8080:443 &        # UI: https://localhost:8080
kubectl apply -f argocd-app.yaml                   # register the app with Argo CD
kubectl get deploy web -n web                      # READY 2/2
# In the Git repo, edit k8s/deployment.yaml: replicas: 2 -> 4
git add k8s/deployment.yaml && git commit -m "Scale web to 4 replicas" && git push origin main
argocd app get web                                 # or watch the UI
kubectl get deploy web -n web                      # READY 4/4
```
`argocd-app.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-user>/gitops-demo.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: web
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```
`k8s/deployment.yaml` (in the Git repo)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```
**What each line does:**
- `apply --server-side`: installs Argo CD (its CRDs are too big for a normal client-side apply). The `argocd-initial-admin-secret` holds the first `admin` password.
- `repoURL` / `targetRevision` / `path`: which Git repo, branch and folder hold the desired state. For a private repo, add a token under Settings → Repositories in the UI, never in this YAML.
- `destination`: which cluster (`https://kubernetes.default.svc` = the cluster Argo CD runs in) and which namespace to deploy into. `CreateNamespace=true` creates `web` if it is missing.
- `automated`: sync automatically when Git changes. `prune: true` deletes objects that were removed from Git. `selfHeal: true` undoes manual changes made in the cluster (drift).
- `git commit` + `push`: the only way to change the app. Argo CD polls Git about every 3 minutes (run `argocd app sync web` or click Refresh to sync sooner).

**Verify:** `argocd app get web` → `Sync Status: Synced to main (<commit>)`, `Health Status: Healthy`, and the UI shows the app green with **Synced + Healthy**. `kubectl get deploy web -n web` → `READY 4/4`. Drift test: `kubectl scale deploy web -n web --replicas=1` → selfHeal sets it back to 4 within seconds.

**🧠 In simple words:** In GitOps, Git holds the desired state and Argo CD keeps comparing the cluster with Git, fixing any difference. You never run `kubectl apply` for the app yourself: you change a YAML file, commit and push, and Argo CD rolls it out. In short: Application YAML points Argo CD at a repo folder; edit `replicas` → commit → push → Argo CD syncs → `kubectl get deploy` shows the new count.

---
