# DevOps Glossary: Full Forms

New acronyms are added as they're taught. Grouped by topic.

## General / DevOps
| Short | Full form | One-line meaning |
|---|---|---|
| DevOps | **Dev**elopment + **Op**eration**s** | Culture + practices uniting the two teams, with automation |
| OS | Operating System | Manages hardware and runs programs (macOS, Linux, Windows) |
| CPU | Central Processing Unit | The chip that runs instructions |
| RAM | Random Access Memory | Fast, temporary working memory |
| PID | Process ID(entifier) | The number the OS gives each running process |
| SDLC | Software Development Life Cycle | Requirements → design → build → test → deploy → maintain |
| CI | Continuous Integration | Merge often; every merge is built and tested automatically |
| CD | Continuous Delivery / Continuous Deployment | Always release-ready (human approves) / released automatically |
| CALMS | Culture, Automation, Lean, Measurement, Sharing | The five pillars of DevOps |
| DORA | DevOps Research and Assessment | Group behind the 4 DevOps performance metrics |
| MTTR | Mean Time To Restore (Recovery) | Average time to recover from a failure |
| IaC | Infrastructure as Code | Servers/networks defined in code files |
| API | Application Programming Interface | The agreed requests a program accepts and the responses it gives |
| SOA | Service-Oriented Architecture | Older cousin of microservices |
| QA | Quality Assurance | Testing |
| prod | production | The live environment real users use |

## Linux & shell
| Short | Full form | One-line meaning |
|---|---|---|
| bash | **B**ourne **A**gain **SH**ell | Most common shell on Linux servers |
| zsh | Z shell | Your Mac's default shell |
| sh | shell | The original, basic shell |
| pwd | print working directory | Where am I? |
| ls | list | List folder contents |
| cd | change directory | Move into a folder |
| mkdir | make directory | Create a folder |
| cp / mv / rm | copy / move / remove | File operations |
| cat | concatenate | Print (and join) files |
| grep | global regular expression print | Search text for matching lines |
| wc | word count | Count lines (`-l`), words, characters |
| chmod | change mode | Change permissions |
| chown | change owner | Change a file's owner |
| sudo | superuser do | Run one command as root |
| umask | user (file-creation) mask | Which permissions new files/folders don't get |
| ps | process status | List processes |
| SIGTERM | signal: terminate | Polite "please stop" (kill default, 15) |
| SIGKILL | signal: kill | Forced stop, can't be ignored (kill -9) |
| SIGINT | signal: interrupt | What Ctrl + C sends |
| apt | Advanced Package Tool | Package manager on Ubuntu/Debian |
| yum / dnf | Yellowdog Updater, Modified / Dandified YUM | Package managers on Red Hat family |
| apk | Alpine Package Keeper | Package manager on Alpine |
| systemd / systemctl | system daemon / system control | Linux service manager (PID 1) / its command |
| journalctl | journal control | Read systemd logs |
| cron / crontab | from *chronos* (Greek: time) / cron table | Scheduler daemon / its schedule file |
| PATH | (not an acronym) | List of folders the shell searches for commands |
| FHS | Filesystem Hierarchy Standard | The standard Linux folder layout (/etc, /var/log, …) |

## Networking
| Short | Full form | One-line meaning |
|---|---|---|
| IP | Internet Protocol | Rules for addressing and delivering data between machines |
| IPv4 / IPv6 | IP version 4 / version 6 | 4-number addresses / newer long addresses |
| CIDR | Classless Inter-Domain Routing | `/24`-style notation for address ranges |
| TCP | **Transmission Control Protocol** | Reliable: connects first, guarantees delivery and order |
| UDP | **User Datagram Protocol** | Fast: no connection, no guarantees ("datagram" = one self-contained packet) |
| SYN / ACK | synchronize / acknowledge | Messages in TCP's 3-way handshake |
| DNS | Domain Name System | Turns names into IP addresses |
| URL | Uniform Resource Locator | A full web address |
| TLD | Top-Level Domain | The ending: .com, .org, .in |
| TTL | Time To Live | How long an answer may be cached |
| A / AAAA | Address record / "quad-A" (IPv6 address) record | Name → IPv4 / name → IPv6 |
| CNAME | Canonical Name record | Name → another name (alias) |
| MX | Mail Exchange record | Domain → its mail server |
| HTTP | HyperText Transfer Protocol | Rules for web requests and responses |
| HTTPS | HTTP Secure | HTTP inside TLS encryption |
| HTML | HyperText Markup Language | The language web pages are written in |
| TLS | Transport Layer Security | Standard for encrypting connections |
| SSL | Secure Sockets Layer | Old name for TLS |
| CA | Certificate Authority | Trusted organisation that signs certificates |
| SSH | Secure Shell | Encrypted remote login to servers (port 22) |
| curl | client URL | Command-line tool for making web requests |
| dig | domain information groper | DNS lookup tool |
| nslookup | name server lookup | Older DNS lookup tool |
| lsof | list open files | Shows which process uses a file or port |
| NAT | Network Address Translation | Many private devices sharing one public IP (awareness only) |

## Git
| Short | Full form | One-line meaning |
|---|---|---|
| VCS | Version Control System | Records every change to files over time |
| repo | repository | A folder Git is tracking (history lives in `.git`) |
| HEAD | (not an acronym) | "Where you are now": the current commit/branch |
| PR | Pull Request | A request on GitHub to merge one branch into another, with review |
| SVN | Subversion | An older, centralised version control system |
| gh | GitHub CLI | GitHub's command-line tool |

## Docker
| Short | Full form | One-line meaning |
|---|---|---|
| VM | Virtual Machine | A whole computer imitated in software, with its own OS and kernel |
| OS (guest / host) | Operating System | Guest = the OS inside a VM; host = the real machine's OS |
| cgroups | control groups | Linux feature limiting how much CPU/memory a process can use |
| namespaces | (not an acronym) | Linux feature limiting what a process can see (own processes, network, files) |
| dockerd | Docker daemon | The background service that does Docker's real work |
| CLI | Command-Line Interface | A program you use by typing commands (`docker`, `git`, `kubectl`) |
| API | Application Programming Interface | The agreed requests one program can send another |
| REST | Representational State Transfer | A style of API that uses ordinary HTTP requests |
| GHCR | GitHub Container Registry | GitHub's image registry (`ghcr.io`) |
| ECR | Elastic Container Registry | Amazon's image registry |
| PID 1 | Process ID 1 | A container's main process; when it ends, the container stops |
| `-d` / `-p` / `-e` / `-it` | detached / publish / environment / interactive + terminal | Common `docker run` flags |
| rmi | remove image | `docker rmi` deletes an image |
| Colima | (name: "Containers on Lima") | Runs Docker inside a small Linux VM on your Mac |

## Common container paths (memorise for "write the command" questions)
| Image | Mount to | What lives there |
|---|---|---|
| nginx | `/usr/share/nginx/html` | Website files it serves |
| nginx | `/etc/nginx/conf.d/` (or `/etc/nginx/nginx.conf`) | Its configuration |
| postgres | `/var/lib/postgresql/data` | Database data |
| mysql | `/var/lib/mysql` | Database data |
| mongo | `/data/db` | Database data |
| redis | `/data` | Saved data |
| your own app | whatever `WORKDIR` says (e.g. `/app`) | Your code |
