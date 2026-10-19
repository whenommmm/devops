# Labs · Session 2 · Linux

## Lab 1 · Make a project folder (app, logs, config), then list, copy, move, search and delete files
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

## Lab 2 · Create user examuser + group devops, add the user to the group, set ownership and permissions on a project directory
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

## Lab 3 · Check a Linux service's status, read its logs, and diagnose a simulated failure
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
