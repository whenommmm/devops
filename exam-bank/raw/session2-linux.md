# Labs · Session 2 · Linux (answers use macOS commands)

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
**Answer (macOS):**
```bash
sudo dseditgroup -o create devops
sudo sysadminctl -addUser examuser -fullName "Exam User" -password -
sudo dseditgroup -o edit -a examuser -t user devops
id examuser
dseditgroup -o checkmember -m examuser devops
sudo mkdir -p /Users/Shared/project
sudo chown -R examuser:devops /Users/Shared/project
sudo chmod -R 770 /Users/Shared/project
ls -ld /Users/Shared/project
sudo -u examuser touch /Users/Shared/project/test.txt
```
**What each line does:**
- `dseditgroup -o create devops`: creates a group called `devops` (`-o` = operation). macOS keeps users and groups in its Directory Service, not in `/etc/group`.
- `sysadminctl -addUser examuser -fullName "Exam User" -password -`: creates the user with a home folder. `-password -` (a dash) makes it **ask** you for the password, so it is never written in the command.
- `dseditgroup -o edit -a examuser -t user devops`: **adds** (`-a`) the user (`-t user` = it's a user, not a group) to `devops`.
- `id examuser`: shows the user's ID and every group it belongs to; `devops` should be in the list.
- `dseditgroup -o checkmember -m examuser devops`: asks directly "is examuser in devops?" and prints `yes examuser is a member of devops`.
- `mkdir -p /Users/Shared/project`: a folder every account can reach (your own home folder is private to you).
- `chown -R examuser:devops …`: owner = examuser, group = devops; `-R` = everything inside too.
- `chmod -R 770 …`: owner rwx (7), group rwx (7), others nothing (0). r=4, w=2, x=1; on a folder, x means "can enter it".
- `sudo -u examuser touch …`: runs one command **as examuser** to prove the permissions work.

**Verify:** `ls -ld /Users/Shared/project` → `drwxrwx---  … examuser  devops … /Users/Shared/project`; the `touch` gives no error; `ls -l /Users/Shared/project` shows `test.txt` owned by examuser.

**Clean up afterwards** (it is a real login account on your Mac):
```bash
sudo rm -rf /Users/Shared/project
sudo sysadminctl -deleteUser examuser
sudo dseditgroup -o delete devops
```
*(Linux version, if the exam asks for Linux: `sudo groupadd devops` · `sudo useradd -m -s /bin/bash examuser` · `sudo passwd examuser` · `sudo usermod -aG devops examuser` · then the same `chown` / `chmod 770` / `id`.)*

**🧠 In simple words:** We create a team group and a user, put the user into the group, and hand a shared project folder to that user and group. Permission 770 lets the owner and the team read, write and enter the folder while everyone else is locked out, and running `touch` as examuser proves it works. In short: create group → create user → add user to group → chown → chmod → test.

## Lab 3 · Check a service's status, read its logs, and diagnose a simulated failure
**Service used:** nginx installed with Homebrew. On macOS, background services are run by **launchd**, and `brew services` is the easy front end to it (the Mac's version of `systemctl`).
**Answer (macOS):**
```bash
brew install nginx
sed -i '' 's/listen[[:space:]]*8080;/listen 8081;/' /opt/homebrew/etc/nginx/nginx.conf
brew services start nginx
brew services list
curl -I http://localhost:8081
echo "broken_line;" > /opt/homebrew/etc/nginx/servers/bad.conf
brew services restart nginx
brew services list
nginx -t
tail -n 20 /opt/homebrew/var/log/nginx/error.log
launchctl print gui/$(id -u)/homebrew.mxcl.nginx | grep -E "state|last exit"
rm /opt/homebrew/etc/nginx/servers/bad.conf
nginx -t
brew services restart nginx
lsof -nP -iTCP:8081 -sTCP:LISTEN
curl -I http://localhost:8081
```
**What each line does:**
- `brew install nginx`: installs the nginx web server with Homebrew.
- `sed -i '' 's/…8080;/listen 8081;/' …nginx.conf`: Homebrew's nginx listens on 8080, which your other cluster already uses, so this switches it to 8081. (`sed -i ''` = edit the file in place; the empty `''` is required on macOS.)
- `brew services start nginx`: starts nginx in the background through launchd (the Mac's service manager) and keeps it starting at login.
- `brew services list`: one line per service with its **Status**: `started` = running, `error` (with an exit code) = it failed, `none` = not running.
- `curl -I http://localhost:8081`: asks for only the response headers; a healthy server answers `HTTP/1.1 200 OK` and `Server: nginx/…`.
- `echo "broken_line;" > …/servers/bad.conf`: **simulates the problem**: nginx reads every file in `servers/`, and this line is not valid nginx config.
- `brew services restart nginx` then `brew services list`: the restart fails, and the Status column shows `error`.
- `nginx -t`: nginx's own config test. It names the exact file and line: `unknown directive "broken_line" in /opt/homebrew/etc/nginx/servers/bad.conf:1`.
- `tail -n 20 …/error.log`: the last 20 lines of nginx's own log file; the same `[emerg] unknown directive` line appears there. (`tail -f` follows it live.)
- `launchctl print gui/$(id -u)/homebrew.mxcl.nginx`: asks launchd directly about the service (`homebrew.mxcl.nginx` is its name); `state` and `last exit code` show whether it is running or why it stopped.
- `rm …/bad.conf` + `nginx -t`: **fix** the cause, then the test says `syntax is ok` and `test is successful`.
- `brew services restart nginx`: start it again.
- `lsof -nP -iTCP:8081 -sTCP:LISTEN`: shows the program listening on port 8081 (nginx). This is the Mac's version of `ss -tulpn`.

**Diagnosis notes to write:** Symptom: the site is down, and `brew services list` shows `error`. Evidence: `nginx -t` and `error.log` say `unknown directive "broken_line"`. Cause: a bad config line. Fix: remove it, `nginx -t` passes, restart. Proof: `brew services list` = `started`, port 8081 listening, `curl -I` = `200 OK`.

**Verify:** `brew services list` → `nginx  started`; `curl -I http://localhost:8081` → `HTTP/1.1 200 OK`.

**Clean up afterwards:** `brew services stop nginx` (and `brew uninstall nginx` if you don't need it).

*(Linux version, if the exam asks for Linux: `systemctl status nginx` · `sudo systemctl restart nginx` · `journalctl -u nginx -n 30` · `sudo nginx -t` · `/var/log/nginx/error.log` · `ss -tulpn | grep :80` · `sudo systemctl enable nginx`.)*

**🧠 In simple words:** On a Mac, launchd runs background services and `brew services` lets you start, stop and check them, like `systemctl` on Linux. We break nginx on purpose with a bad config line, see `error` in the service list, read the exact reason with `nginx -t` and the log file, fix it, and prove it is back with `lsof` and `curl`. In short: status → logs → test config → fix → restart → verify.
