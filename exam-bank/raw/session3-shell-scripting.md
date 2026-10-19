# Labs · Session 3 · Shell Scripting

Exit-code rule used in all three scripts: `0` = success, `1` = called wrongly (usage error), `2` = bad input or a problem found. Error messages go to stderr with `>&2`.

## Lab 1 · Script that takes a directory path and prints how many files and subdirectories it has; handle a missing/invalid argument
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

## Lab 2 · Script that checks a supplied file exists and whether it is readable, writable and executable
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

## Lab 3 · Reusable system-health script (variables, conditions, loops, functions) reporting disk, memory and uptime
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
