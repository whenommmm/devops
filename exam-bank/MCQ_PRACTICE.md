# 📝 MCQ practice · 190 questions, answers after each session

*Do a session's 10 questions (write your letters down), then open **✅ Answers** right below them and check. Numbers = session.question.*

**Sessions:** [1](#s1) · [2](#s2) · [3](#s3) · [4](#s4) · [5](#s5) · [6–7](#s67) · [8](#s8) · [9](#s9) · [10](#s10) · [11](#s11) · [12](#s12) · [13](#s13) · [14](#s14) · [15](#s15) · [16](#s16) · [17](#s17) · [18](#s18) · [19](#s19) · [20](#s20)

---

<a id="s1"></a>
## Session 1 · DevOps intro & roadmap

**1.1** What is the primary objective of DevOps?  
A. Faster, reliable delivery through collaboration and automation  
B. Replace developers with servers  
C. Avoid version control  
D. Eliminate all testing

**1.2** What does CI mean?  
A. Central Installation  
B. Continuous Integration  
C. Cloud Inspection  
D. Code Isolation

**1.3** Which practice helps catch defects early?  
A. Manual deployment only  
B. Disabling logs  
C. Skipping code reviews  
D. Automated tests in CI

**1.4** What is infrastructure as code?  
A. Keeping server settings undocumented  
B. Writing only frontend code  
C. Managing infrastructure using versioned configuration  
D. Purchasing hardware manually

**1.5** Which tool is commonly used for source control?  
A. Prometheus  
B. Nginx  
C. Git  
D. Grafana

**1.6** What does a CI/CD pipeline automate?  
A. Build, test and delivery steps  
B. Employee attendance  
C. Domain purchases  
D. Keyboard configuration

**1.7** Why are containers useful?  
A. They package applications with dependencies  
B. They remove the need for networks  
C. They guarantee zero bugs  
D. They replace all databases

**1.8** What is monitoring used for?  
A. Formatting YAML  
B. Compiling source code  
C. Tracking system health and behavior  
D. Creating Git branches

**1.9** Which mindset is central to DevOps?  
A. Shared ownership across development and operations  
B. Only manual changes  
C. Separate teams with no communication  
D. No production feedback

**1.10** Which is a common DevOps lifecycle activity?  
A. Continuous feedback  
B. Disabling security scans  
C. Ignoring incidents  
D. One-time deployment forever

<details>
<summary><b>✅ Answers · Session 1</b> (answer all 10 first, then click)</summary>

**Key:** 1-A · 2-B · 3-D · 4-C · 5-C · 6-A · 7-A · 8-C · 9-A · 10-A

**1.1** ✅ **A. Faster, reliable delivery through collaboration and automation**  
💡 DevOps joins development and operations so software ships often and safely; collaboration and automation are how it gets there.

**1.2** ✅ **B. Continuous Integration**  
💡 CI = Continuous Integration: developers merge small changes often, and every merge is automatically built and tested.

**1.3** ✅ **D. Automated tests in CI**  
💡 Tests that run automatically on every push catch a bug minutes after it's written, long before production.

**1.4** ✅ **C. Managing infrastructure using versioned configuration**  
💡 IaC = Infrastructure as Code: servers and networks are described in files kept in Git, so they can be reviewed, versioned and rebuilt identically.

**1.5** ✅ **C. Git**  
💡 Git is the standard version-control (source-control) tool; it records every change to the code.

**1.6** ✅ **A. Build, test and delivery steps**  
💡 A pipeline automates the build → test → delivery steps that would otherwise be done by hand on every change.

**1.7** ✅ **A. They package applications with dependencies**  
💡 A container bundles the app with its libraries and settings, so it runs the same on any machine.

**1.8** ✅ **C. Tracking system health and behavior**  
💡 Monitoring tracks health and behaviour (CPU, errors, response time) so problems are noticed and alerted on quickly.

**1.9** ✅ **A. Shared ownership across development and operations**  
💡 DevOps culture = shared ownership: developers and operations are responsible for the software together, instead of 'throwing it over the wall'.

**1.10** ✅ **A. Continuous feedback**  
💡 Monitoring and user feedback flow back into planning; the DevOps lifecycle is a loop (Plan → Code → Build → Test → Release → Deploy → Operate → Monitor → Plan).

</details>

---

<a id="s2"></a>
## Session 2 · Linux

**2.1** Which command prints the working directory?  
A. mv  
B. ls  
C. cd  
D. pwd

**2.2** Which command lists files including hidden files?  
A. pwd -a  
B. rm -a  
C. ls -a  
D. mkdir -a

**2.3** What does chmod change?  
A. File permissions  
B. Hostname  
C. File ownership only  
D. Network interface

**2.4** Which command searches text in files?  
A. pwd  
B. touch  
C. mv  
D. grep

**2.5** Which command displays a file's contents?  
A. kill  
B. rmdir  
C. mkdir  
D. cat

**2.6** What does the pipe operator do?  
A. Redirects all input to a file  
B. Runs commands only as root  
C. Deletes temporary files  
D. Passes one command's output to another

**2.7** Which command shows running processes?  
A. cp  
B. ls  
C. ps  
D. mkdir

**2.8** Which command changes the current directory?  
A. cd  
B. echo  
C. pwd  
D. cat

**2.9** What does sudo typically allow?  
A. Run commands with elevated privileges  
B. Download a container image  
C. Create a symbolic link  
D. Show DNS records

**2.10** Which command displays disk space usage by filesystem?  
A. du -a  
B. uname -a  
C. free -m  
D. df -h

<details>
<summary><b>✅ Answers · Session 2</b> (answer all 10 first, then click)</summary>

**Key:** 1-D · 2-C · 3-A · 4-D · 5-D · 6-D · 7-C · 8-A · 9-A · 10-D

**2.1** ✅ **D. pwd**  
💡 pwd = print working directory: shows the full path of the folder you are in.

**2.2** ✅ **C. ls -a**  
💡 ls -a = list all, including hidden files whose names start with a dot (e.g. .bashrc).

**2.3** ✅ **A. File permissions**  
💡 chmod = change mode: changes read/write/execute permissions, e.g. chmod 755 file (rwx r-x r-x).

**2.4** ✅ **D. grep**  
💡 grep searches for a text pattern inside files and prints the matching lines (grep error app.log).

**2.5** ✅ **D. cat**  
💡 cat (concatenate) prints a file's contents to the screen.

**2.6** ✅ **D. Passes one command's output to another**  
💡 The pipe | sends one command's output into the next command as input (ps aux | grep nginx).

**2.7** ✅ **C. ps**  
💡 ps = process status: lists running processes; ps aux shows every process.

**2.8** ✅ **A. cd**  
💡 cd = change directory; cd .. goes up one level, cd ~ goes home.

**2.9** ✅ **A. Run commands with elevated privileges**  
💡 sudo = 'superuser do': runs one command with root (administrator) privileges.

**2.10** ✅ **D. df -h**  
💡 df -h = disk free, in human-readable sizes, for each filesystem. (du shows how big a folder is.)

</details>

---

<a id="s3"></a>
## Session 3 · Shell scripting

**3.1** Which line commonly selects Bash as script interpreter?  
A. #bash/start  
B. //bin/bash  
C. #!/bin/bash  
D. #!/usr/git

**3.2** How do you reference the first positional argument in Bash?  
A. $0  
B. $#  
C. $?  
D. $1

**3.3** Which operator tests whether a regular file exists in Bash?  
A. [ -p file ]  
B. [ -d file ]  
C. [ -f file ]  
D. [ -z file ]

**3.4** What does $? contain immediately after a command?  
A. Exit status of the previous command  
B. Current username  
C. Number of script lines  
D. Current process ID

**3.5** Which loop iterates over a sequence of values?  
A. exit  
B. for  
C. case  
D. if

**3.6** How do you make a script executable?  
A. grep +x script.sh  
B. pwd script.sh  
C. chmod +x script.sh  
D. cat +x script.sh

**3.7** Which construct makes a conditional decision?  
A. for ... done  
B. echo ... print  
C. if ... then ... fi  
D. read ... exit

**3.8** How do you assign a variable in Bash?  
A. $NAME=value  
B. NAME=value  
C. set NAME := value  
D. NAME = value

**3.9** Which command reads input from the terminal?  
A. touch  
B. echo  
C. printf  
D. read

**3.10** What does && mean between two commands?  
A. Run the second only if first fails  
B. Run the second only if the first succeeds  
C. Always run both in parallel  
D. Append output to a file

<details>
<summary><b>✅ Answers · Session 3</b> (answer all 10 first, then click)</summary>

**Key:** 1-C · 2-D · 3-C · 4-A · 5-B · 6-C · 7-C · 8-B · 9-D · 10-B

**3.1** ✅ **C. #!/bin/bash**  
💡 #!/bin/bash (the shebang) on line 1 tells the system which interpreter should run the script.

**3.2** ✅ **D. $1**  
💡 $1 is the first argument given to the script ($0 = the script's name, $# = how many arguments).

**3.3** ✅ **C. [ -f file ]**  
💡 [ -f file ] is true when the path exists and is a regular file (-d = a directory, -e = exists at all).

**3.4** ✅ **A. Exit status of the previous command**  
💡 $? holds the exit status of the last command: 0 = success, anything else = failure.

**3.5** ✅ **B. for**  
💡 for loops over a list of values: for f in *.log; do echo $f; done.

**3.6** ✅ **C. chmod +x script.sh**  
💡 chmod +x adds execute permission, so you can run the script as ./script.sh.

**3.7** ✅ **C. if ... then ... fi**  
💡 if … then … fi makes a decision; fi ('if' backwards) closes the block.

**3.8** ✅ **B. NAME=value**  
💡 NAME=value assigns a variable, with no spaces around the =; read it back with $NAME.

**3.9** ✅ **D. read**  
💡 read waits for the user to type a line and stores it in a variable (read name).

**3.10** ✅ **B. Run the second only if the first succeeds**  
💡 a && b runs b only if a succeeded (exit 0); a || b runs b only if a failed.

</details>

---

<a id="s4"></a>
## Session 4 · Networking

**4.1** What does DNS primarily do?  
A. Resolve names to IP addresses  
B. Build application binaries  
C. Schedule containers  
D. Encrypt disk volumes

**4.2** What does an IP address identify?  
A. A Docker image tag  
B. A CPU instruction  
C. A network interface on an IP network  
D. A Git repository

**4.3** Which protocol provides reliable ordered delivery?  
A. ICMP  
B. ARP  
C. TCP  
D. UDP

**4.4** Which command can test basic ICMP reachability?  
A. ping  
B. git status  
C. chmod  
D. docker build

**4.5** What does HTTP status 404 mean?  
A. Authentication succeeded  
B. Resource not found  
C. Successful request  
D. Server started

**4.6** What is the usual default port for HTTPS?  
A. 22  
B. 53  
C. 443  
D. 80

**4.7** Which tool can make HTTP requests from a terminal?  
A. tar  
B. mkdir  
C. grep  
D. curl

**4.8** What does a subnet mask/prefix indicate?  
A. A Docker volume size  
B. A password length  
C. The network portion of an IP address  
D. A DNS TTL only

**4.9** What is the role of a default gateway?  
A. Forward traffic to other networks  
B. Compile Java programs  
C. Start systemd services  
D. Resolve Git merge conflicts

**4.10** Which protocol is commonly used for secure remote shell access?  
A. DHCP  
B. FTP  
C. SSH  
D. SMTP

<details>
<summary><b>✅ Answers · Session 4</b> (answer all 10 first, then click)</summary>

**Key:** 1-A · 2-C · 3-C · 4-A · 5-B · 6-C · 7-D · 8-C · 9-A · 10-C

**4.1** ✅ **A. Resolve names to IP addresses**  
💡 DNS = Domain Name System: turns names like google.com into IP addresses.

**4.2** ✅ **C. A network interface on an IP network**  
💡 An IP (Internet Protocol) address identifies one network interface, i.e. one connection of a machine, on an IP network.

**4.3** ✅ **C. TCP**  
💡 TCP = Transmission Control Protocol: connection-based, resends lost data and keeps it in order. UDP is faster but guarantees nothing.

**4.4** ✅ **A. ping**  
💡 ping sends ICMP (Internet Control Message Protocol) echo requests to check whether a host can be reached.

**4.5** ✅ **B. Resource not found**  
💡 404 Not Found: the server answered, but the requested path doesn't exist (4xx = a client-side error).

**4.6** ✅ **C. 443**  
💡 HTTPS uses port 443 by default (HTTP 80, SSH 22, DNS 53).

**4.7** ✅ **D. curl**  
💡 curl sends HTTP requests from the terminal (curl -I shows only the response headers).

**4.8** ✅ **C. The network portion of an IP address**  
💡 A subnet mask or prefix (/24 = 255.255.255.0) says which part of an IP address is the network and which part is the host.

**4.9** ✅ **A. Forward traffic to other networks**  
💡 The default gateway (usually the router) forwards traffic that is going to other networks, such as the internet.

**4.10** ✅ **C. SSH**  
💡 SSH = Secure Shell: encrypted remote login, on port 22.

</details>

---

<a id="s5"></a>
## Session 5 · Git & GitHub

**5.1** Which command initializes a local Git repository?  
A. git init  
B. git boot  
C. git create  
D. git start

**5.2** Which command stages a file for commit?  
A. git fetch  
B. git log  
C. git add  
D. git push

**5.3** Which command records staged changes?  
A. git status  
B. git diff  
C. git commit  
D. git clone

**5.4** Which command shows the current working tree state?  
A. git status  
B. git branch -d  
C. git gc  
D. git tag -d

**5.5** Which command creates a new branch?  
A. git reset feature  
B. git branch feature  
C. git commit feature  
D. git push feature

**5.6** Which command integrates another branch into the current branch?  
A. git fetch  
B. git clean  
C. git merge  
D. git stash

**5.7** What does git clone do?  
A. Deletes a remote branch  
B. Reverts the operating system  
C. Renames a commit  
D. Copies a repository to a local directory

**5.8** What is a pull request typically used for?  
A. Building a Docker layer  
B. Starting a Kubernetes pod  
C. Reviewing and discussing changes before merge  
D. Provisioning a VPC

**5.9** What does .gitignore control?  
A. Which untracked paths Git should ignore  
B. Who can access GitHub  
C. The commit author  
D. The default remote URL

**5.10** Which command uploads local commits to a remote?  
A. git diff  
B. git pull  
C. git push  
D. git show

<details>
<summary><b>✅ Answers · Session 5</b> (answer all 10 first, then click)</summary>

**Key:** 1-A · 2-C · 3-C · 4-A · 5-B · 6-C · 7-D · 8-C · 9-A · 10-C

**5.1** ✅ **A. git init**  
💡 git init creates a new repository (the hidden .git folder) in the current folder.

**5.2** ✅ **C. git add**  
💡 git add puts changes into the staging area, ready for the next commit.

**5.3** ✅ **C. git commit**  
💡 git commit saves the staged changes as a snapshot, with a message (git commit -m "…").

**5.4** ✅ **A. git status**  
💡 git status shows which files are modified, staged or untracked (git diff shows the changed lines themselves).

**5.5** ✅ **B. git branch feature**  
💡 git branch feature creates a branch but stays on the current one (git switch -c feature creates it and switches to it).

**5.6** ✅ **C. git merge**  
💡 git merge feature brings that branch's commits into the branch you are on.

**5.7** ✅ **D. Copies a repository to a local directory**  
💡 git clone copies a remote repository, with its full history, into a new local folder.

**5.8** ✅ **C. Reviewing and discussing changes before merge**  
💡 A pull request (PR) proposes merging a branch, so teammates can review and discuss the changes first.

**5.9** ✅ **A. Which untracked paths Git should ignore**  
💡 .gitignore lists untracked files and folders Git should ignore (node_modules/, .env); files that are already tracked are not affected.

**5.10** ✅ **C. git push**  
💡 git push uploads your local commits to the remote repository (e.g. GitHub).

</details>

---

<a id="s67"></a>
## Session 6–7 · Docker & Dockerfile

**6–7.1** What is a Docker image?  
A. A running virtual machine only  
B. A Git branch  
C. A DNS record  
D. A read-only template for creating containers

**6–7.2** Which command builds an image from a Dockerfile?  
A. docker start  
B. docker logs  
C. docker build  
D. docker exec

**6–7.3** Which Dockerfile instruction sets the base image?  
A. COPY  
B. CMD  
C. EXPOSE  
D. FROM

**6–7.4** Which Dockerfile instruction copies files into an image?  
A. USER  
B. WORKDIR  
C. COPY  
D. RUN

**6–7.5** Which command lists running containers?  
A. docker images  
B. docker ps  
C. docker build  
D. docker pull

**6–7.6** What does docker run -p 8080:80 map?  
A. A volume to port 80  
B. Host port 80 to container port 8080  
C. Host port 8080 to container port 80  
D. Two host ports together

**6–7.7** What does docker logs show?  
A. Container standard output and error  
B. Registry credentials  
C. Git commit history  
D. Host kernel source

**6–7.8** Which command executes a command inside a running container?  
A. docker exec  
B. docker save  
C. docker tag  
D. docker push

**6–7.9** What is the purpose of .dockerignore?  
A. Block network access  
B. Ignore runtime errors  
C. Exclude paths from the build context  
D. Hide containers from ps

**6–7.10** Which Dockerfile instruction defines a default command?  
A. LABEL  
B. ARG  
C. EXPOSE  
D. CMD

<details>
<summary><b>✅ Answers · Session 6–7</b> (answer all 10 first, then click)</summary>

**Key:** 1-D · 2-C · 3-D · 4-C · 5-B · 6-C · 7-A · 8-A · 9-C · 10-D

**6–7.1** ✅ **D. A read-only template for creating containers**  
💡 An image is a read-only template (the app plus its dependencies, in layers); a container is a running instance of it.

**6–7.2** ✅ **C. docker build**  
💡 docker build -t name . builds an image from the Dockerfile in the current folder (the . is the build context).

**6–7.3** ✅ **D. FROM**  
💡 FROM sets the base image and must be the first instruction (FROM python:3.12-slim).

**6–7.4** ✅ **C. COPY**  
💡 COPY copies files from the build context into the image (COPY . /app).

**6–7.5** ✅ **B. docker ps**  
💡 docker ps lists running containers; docker ps -a also shows stopped ones.

**6–7.6** ✅ **C. Host port 8080 to container port 80**  
💡 -p 8080:80 means host port 8080 → container port 80 (host first, container second).

**6–7.7** ✅ **A. Container standard output and error**  
💡 docker logs shows what the container's main process wrote to stdout (standard output) and stderr (standard error).

**6–7.8** ✅ **A. docker exec**  
💡 docker exec runs a command inside a running container (docker exec -it web sh opens a shell).

**6–7.9** ✅ **C. Exclude paths from the build context**  
💡 .dockerignore keeps files (.git, node_modules, .env) out of the build context, so builds are faster and secrets stay out of images.

**6–7.10** ✅ **D. CMD**  
💡 CMD sets the default command a container runs when it starts; arguments after the image name in docker run replace it.

</details>

---

<a id="s8"></a>
## Session 8 · Docker networking, volumes & Compose

**8.1** Why use Docker volumes?  
A. Increase CPU frequency  
B. Disable networking  
C. Persist data beyond a container lifecycle  
D. Replace container images

**8.2** Which command lists Docker volumes?  
A. docker volumes show  
B. docker storage ps  
C. docker volume ls  
D. docker volume ps

**8.3** What is a bind mount?  
A. A remote Git branch  
B. A Kubernetes namespace  
C. A host path mounted inside a container  
D. An image registry

**8.4** Which Docker network driver is the typical default for standalone containers?  
A. overlay  
B. macvtap only  
C. host-only VM  
D. bridge

**8.5** What does a user-defined bridge network provide?  
A. Host kernel upgrades  
B. Container-to-container name resolution  
C. Automatic cloud load balancing  
D. Persistent image layers

**8.6** Which flag publishes a container port?  
A. -w  
B. -e  
C. -v  
D. -p

**8.7** Which command creates a Docker network?  
A. docker net init  
B. docker bridge build  
C. docker network create  
D. docker network push

**8.8** What is Docker Compose mainly for?  
A. Replacing source control  
B. Defining and running multi-container applications  
C. Managing Linux users  
D. Creating DNS root servers

**8.9** Which file is commonly used by Docker Compose?  
A. Chart.yaml  
B. terraform.tfstate  
C. Jenkinsfile  
D. compose.yaml

**8.10** Which command starts services from a Compose file?  
A. docker compose attach-all  
B. docker compose init-only  
C. docker compose up  
D. docker compose commit

<details>
<summary><b>✅ Answers · Session 8</b> (answer all 10 first, then click)</summary>

**Key:** 1-C · 2-C · 3-C · 4-D · 5-B · 6-D · 7-C · 8-B · 9-D · 10-C

**8.1** ✅ **C. Persist data beyond a container lifecycle**  
💡 A volume keeps data outside the container's writable layer, so the data survives removing the container.

**8.2** ✅ **C. docker volume ls**  
💡 docker volume ls lists volumes (docker volume inspect shows details such as where the data lives).

**8.3** ✅ **C. A host path mounted inside a container**  
💡 A bind mount maps a folder on the host into the container (-v /host/path:/container/path).

**8.4** ✅ **D. bridge**  
💡 bridge is the default network driver for standalone containers.

**8.5** ✅ **B. Container-to-container name resolution**  
💡 A user-defined bridge network has built-in DNS, so containers reach each other by name (the default bridge doesn't do this).

**8.6** ✅ **D. -p**  
💡 -p (--publish) publishes a container port on the host so outside traffic can reach it.

**8.7** ✅ **C. docker network create**  
💡 docker network create mynet creates a network; connect containers with --network mynet.

**8.8** ✅ **B. Defining and running multi-container applications**  
💡 Compose defines and runs a multi-container application (e.g. web + database) from one YAML file.

**8.9** ✅ **D. compose.yaml**  
💡 compose.yaml is Compose's file (the older name docker-compose.yml still works).

**8.10** ✅ **C. docker compose up**  
💡 docker compose up (add -d for background) starts every service; docker compose down stops and removes them.

</details>

---

<a id="s9"></a>
## Session 9 · Kubernetes fundamentals

**9.1** What is Kubernetes primarily used for?  
A. Replacing Git history  
B. Editing photos  
C. Compiling Linux kernels  
D. Orchestrating containerized applications

**9.2** What is a Kubernetes Pod?  
A. A Terraform provider  
B. A physical data center  
C. Smallest deployable unit containing one or more containers  
D. A Docker image registry

**9.3** Which component stores Kubernetes cluster state?  
A. containerd  
B. CoreDNS  
C. etcd  
D. kube-proxy

**9.4** Which component schedules Pods onto nodes?  
A. kube-scheduler  
B. kube-proxy  
C. kubectl  
D. kubelet

**9.5** What does the kubelet do?  
A. Stores all cluster state  
B. Builds Helm charts  
C. Approves pull requests  
D. Ensures Pod containers run on a node

**9.6** What is kubectl?  
A. A Linux filesystem  
B. A CLI for interacting with the Kubernetes API  
C. A DNS protocol  
D. A container image format

**9.7** Which command lists Pods?  
A. kubectl show nodes  
B. kubectl list images  
C. kubectl build pods  
D. kubectl get pods

**9.8** What is a Kubernetes namespace?  
A. A Dockerfile instruction  
B. A logical grouping for cluster resources  
C. A container image tag  
D. A dedicated physical server

**9.9** Which component exposes the Kubernetes API?  
A. kubelet  
B. kube-apiserver  
C. CoreDNS  
D. kube-proxy

**9.10** What is a worker node?  
A. A DNS resolver record  
B. A GitHub workflow file  
C. A Helm repository  
D. A machine that runs application Pods

<details>
<summary><b>✅ Answers · Session 9</b> (answer all 10 first, then click)</summary>

**Key:** 1-D · 2-C · 3-C · 4-A · 5-D · 6-B · 7-D · 8-B · 9-B · 10-D

**9.1** ✅ **D. Orchestrating containerized applications**  
💡 Kubernetes orchestrates containers: it places them on machines, restarts failed ones, scales them and updates them.

**9.2** ✅ **C. Smallest deployable unit containing one or more containers**  
💡 A Pod is the smallest deployable unit: one or more containers that share one IP address and storage.

**9.3** ✅ **C. etcd**  
💡 etcd is the key-value database that stores the whole cluster's state; only the API server talks to it.

**9.4** ✅ **A. kube-scheduler**  
💡 kube-scheduler picks a node for each new Pod, based on free resources and constraints.

**9.5** ✅ **D. Ensures Pod containers run on a node**  
💡 The kubelet is the agent on every node that makes sure the Pod's containers are running.

**9.6** ✅ **B. A CLI for interacting with the Kubernetes API**  
💡 kubectl is the command-line tool that sends your requests to the Kubernetes API server.

**9.7** ✅ **D. kubectl get pods**  
💡 kubectl get pods lists the Pods in the current namespace (add -A for all namespaces).

**9.8** ✅ **B. A logical grouping for cluster resources**  
💡 A namespace is a logical group, like a folder, for resources inside one cluster (default, kube-system…).

**9.9** ✅ **B. kube-apiserver**  
💡 kube-apiserver exposes the Kubernetes API; every kubectl command and every component goes through it.

**9.10** ✅ **D. A machine that runs application Pods**  
💡 A worker node is a machine that runs the application Pods (it has the kubelet, kube-proxy and a container runtime).

</details>

---

<a id="s10"></a>
## Session 10 · Pods, ReplicaSets & Deployments

**10.1** What is the purpose of a Deployment?  
A. Store cluster state  
B. Manage desired replicas and rolling updates  
C. Expose a public DNS zone  
D. Create container images

**10.2** What does a ReplicaSet ensure?  
A. A desired number of matching Pods are running  
B. Nodes never fail  
C. Every Pod has a public IP  
D. All images are signed

**10.3** Which field commonly sets Deployment replica count?  
A. status.desiredNodes  
B. spec.ports  
C. metadata.replicas  
D. spec.replicas

**10.4** What does a Pod label help with?  
A. Allocating disk partitions  
B. Installing Helm  
C. Selecting and organizing resources  
D. Encrypting image layers

**10.5** What is a label selector used for?  
A. Selecting Git commits by date  
B. Matching resources with specified labels  
C. Choosing Docker build stages  
D. Filtering shell history

**10.6** Which command scales a Deployment?  
A. kubectl duplicate image  
B. kubectl increase node  
C. kubectl scale deployment  
D. kubectl resize pod

**10.7** What happens when a Deployment Pod crashes?  
A. A controller works to restore desired state  
B. Git automatically rolls back  
C. Kubernetes deletes the Deployment permanently  
D. The cluster always stops

**10.8** Which command views rollout status?  
A. kubectl get dns  
B. kubectl rollout status deployment/name  
C. kubectl helm status  
D. kubectl logs nodes

**10.9** What does a rolling update do?  
A. Gradually replace old Pods with new ones  
B. Change namespace ownership  
C. Restart the control plane  
D. Delete all nodes first

**10.10** Which resource usually creates and manages ReplicaSets?  
A. Secret  
B. PersistentVolume  
C. ConfigMap  
D. Deployment

<details>
<summary><b>✅ Answers · Session 10</b> (answer all 10 first, then click)</summary>

**Key:** 1-B · 2-A · 3-D · 4-C · 5-B · 6-C · 7-A · 8-B · 9-A · 10-D

**10.1** ✅ **B. Manage desired replicas and rolling updates**  
💡 A Deployment keeps the desired number of replicas running and performs rolling updates and rollbacks.

**10.2** ✅ **A. A desired number of matching Pods are running**  
💡 A ReplicaSet makes sure the desired number of Pods matching its selector are running.

**10.3** ✅ **D. spec.replicas**  
💡 spec.replicas sets how many Pods the Deployment wants.

**10.4** ✅ **C. Selecting and organizing resources**  
💡 Labels are key=value tags (app=web) used to select and organise resources.

**10.5** ✅ **B. Matching resources with specified labels**  
💡 A label selector matches resources by their labels; Services and ReplicaSets use one to find their Pods.

**10.6** ✅ **C. kubectl scale deployment**  
💡 kubectl scale deployment web --replicas=5 changes the number of replicas.

**10.7** ✅ **A. A controller works to restore desired state**  
💡 The controller sees fewer Pods than desired and creates a replacement: this is self-healing.

**10.8** ✅ **B. kubectl rollout status deployment/name**  
💡 kubectl rollout status deployment/web waits and reports whether the rollout has finished.

**10.9** ✅ **A. Gradually replace old Pods with new ones**  
💡 A rolling update replaces old Pods with new ones a few at a time, so the app stays available.

**10.10** ✅ **D. Deployment**  
💡 A Deployment creates and manages ReplicaSets (one per version), and each ReplicaSet manages the Pods.

</details>

---

<a id="s11"></a>
## Session 11 · Kubernetes Services

**11.1** Why are Kubernetes Services used?  
A. Provide stable access to a set of Pods  
B. Compile application code  
C. Build container images  
D. Store Git history

**11.2** Which Service type is typically reachable only within the cluster?  
A. ClusterIP  
B. NodePort  
C. LoadBalancer  
D. ExternalName only

**11.3** Which Service type exposes a port on each node?  
A. NodePort  
B. ExternalName  
C. Headless only  
D. ClusterIP

**11.4** Which Service type commonly provisions an external load balancer on supported clouds?  
A. LoadBalancer  
B. ClusterIP  
C. ExternalName  
D. Headless

**11.5** How does a Service identify backend Pods?  
A. Label selector  
B. Node hostname only  
C. Git branch name  
D. Docker image digest

**11.6** What does a Service port targetPort refer to?  
A. SSH port of the node  
B. Port on a Git remote  
C. Port on backend Pods  
D. Port of the API server

**11.7** What is the role of kube-proxy in many clusters?  
A. Build Dockerfiles  
B. Schedule Pods  
C. Store secrets in etcd  
D. Implement Service traffic routing rules

**11.8** Which DNS name form can identify a Service across namespaces?  
A. pod.image.registry.local  
B. service.cluster.node.local  
C. service.namespace.svc.cluster.local  
D. namespace.service.github.io

**11.9** What does a headless Service commonly set?  
A. replicas: 0  
B. type: NodePort  
C. hostNetwork: true  
D. clusterIP: None

**11.10** Which command lists Services?  
A. kubectl build svc  
B. kubectl init service  
C. kubectl get svc  
D. kubectl get images

<details>
<summary><b>✅ Answers · Session 11</b> (answer all 10 first, then click)</summary>

**Key:** 1-A · 2-A · 3-A · 4-A · 5-A · 6-C · 7-D · 8-C · 9-D · 10-C

**11.1** ✅ **A. Provide stable access to a set of Pods**  
💡 Pod IPs change whenever Pods are replaced; a Service gives one stable name and IP that load-balances across the matching Pods.

**11.2** ✅ **A. ClusterIP**  
💡 ClusterIP (the default type) is reachable only from inside the cluster.

**11.3** ✅ **A. NodePort**  
💡 NodePort opens the same port (30000–32767) on every node, so the app can be reached at <NodeIP>:<nodePort>.

**11.4** ✅ **A. LoadBalancer**  
💡 LoadBalancer asks the cloud provider for an external load balancer with a public IP.

**11.5** ✅ **A. Label selector**  
💡 A Service finds its Pods with a label selector; matching Pods that are Ready become its endpoints.

**11.6** ✅ **C. Port on backend Pods**  
💡 targetPort is the port on the Pods where the app actually listens; port is the Service's own port.

**11.7** ✅ **D. Implement Service traffic routing rules**  
💡 kube-proxy runs on every node and writes the routing rules (iptables/IPVS) that send Service traffic to the Pods.

**11.8** ✅ **C. service.namespace.svc.cluster.local**  
💡 <service>.<namespace>.svc.cluster.local is the full DNS name; a short name like web only works within the same namespace.

**11.9** ✅ **D. clusterIP: None**  
💡 A headless Service sets clusterIP: None, so DNS returns the individual Pod IPs (used with StatefulSets).

**11.10** ✅ **C. kubectl get svc**  
💡 kubectl get svc (short for services) lists Services with their type, cluster IP and ports.

</details>

---

<a id="s12"></a>
## Session 12 · Ingress, ConfigMaps & Secrets

**12.1** What does Ingress commonly define?  
A. HTTP/HTTPS routing rules to Services  
B. Git branch protections  
C. Node CPU architecture  
D. Container build steps

**12.2** What is needed to implement Ingress rules?  
A. A Git tag  
B. A Terraform lock file  
C. A Docker registry only  
D. An Ingress controller

**12.3** What is a ConfigMap intended to store?  
A. Persistent disk blocks  
B. Passwords by default  
C. Container images  
D. Non-sensitive configuration data

**12.4** What is a Secret intended to store?  
A. Sensitive configuration values  
B. Container images  
C. Cluster logs only  
D. Helm binaries

**12.5** How can a ConfigMap be consumed by a Pod?  
A. Node kernel flags only  
B. Docker build cache only  
C. Git commit messages only  
D. Environment variables or mounted files

**12.6** What does a path rule in Ingress match?  
A. Container filesystem path  
B. The HTTP request path  
C. Host disk mount path  
D. Git remote path

**12.7** What does TLS termination usually provide at Ingress?  
A. Automatic Pod autoscaling  
B. Persistent volumes  
C. Build artifact signing  
D. HTTPS encryption for client connections

**12.8** Which command lists ConfigMaps?  
A. kubectl get configmaps  
B. kubectl get secrets-only  
C. kubectl get networks  
D. kubectl get charts

**12.9** Are Kubernetes Secret values encrypted merely because YAML uses base64?  
A. Yes, base64 is strong encryption  
B. No, base64 is encoding rather than encryption  
C. Yes, they are always hashed  
D. No, Secrets cannot contain strings

**12.10** Which Kubernetes resource commonly holds a TLS certificate and key for Ingress?  
A. ConfigMap  
B. Deployment  
C. Secret  
D. ServiceAccount only

<details>
<summary><b>✅ Answers · Session 12</b> (answer all 10 first, then click)</summary>

**Key:** 1-A · 2-D · 3-D · 4-A · 5-D · 6-B · 7-D · 8-A · 9-B · 10-C

**12.1** ✅ **A. HTTP/HTTPS routing rules to Services**  
💡 An Ingress holds HTTP/HTTPS routing rules (by host name and URL path) that send requests to Services.

**12.2** ✅ **D. An Ingress controller**  
💡 Ingress rules do nothing until an Ingress controller (such as NGINX) runs in the cluster to apply them.

**12.3** ✅ **D. Non-sensitive configuration data**  
💡 A ConfigMap holds non-secret settings as key–value pairs, kept outside the image.

**12.4** ✅ **A. Sensitive configuration values**  
💡 A Secret holds sensitive values: passwords, tokens, keys, certificates.

**12.5** ✅ **D. Environment variables or mounted files**  
💡 A Pod uses a ConfigMap as environment variables (env / envFrom) or as files from a mounted volume.

**12.6** ✅ **B. The HTTP request path**  
💡 A path rule matches the URL path of the HTTP request (e.g. /api), not a folder on disk.

**12.7** ✅ **D. HTTPS encryption for client connections**  
💡 TLS termination: the Ingress controller holds the certificate and handles HTTPS with the client, then forwards plain HTTP to the Pods.

**12.8** ✅ **A. kubectl get configmaps**  
💡 kubectl get configmaps (short: cm) lists ConfigMaps.

**12.9** ✅ **B. No, base64 is encoding rather than encryption**  
💡 base64 is encoding: anyone can reverse it with base64 -d, so it is not encryption.

**12.10** ✅ **C. Secret**  
💡 A Secret of type kubernetes.io/tls holds tls.crt and tls.key; the Ingress names it in tls.secretName.

</details>

---

<a id="s13"></a>
## Session 13 · Storage, HPA & Probes

**13.1** What is a PersistentVolumeClaim (PVC)?  
A. A request for persistent storage  
B. A network firewall  
C. A container image  
D. A Git tag

**13.2** What is a PersistentVolume (PV)?  
A. An Ingress controller  
B. A cluster storage resource  
C. A Deployment revision  
D. A DNS zone

**13.3** What does a StorageClass help define?  
A. Which Git branch is default  
B. Which HTTP path is matched  
C. Which Dockerfile is used  
D. How dynamic storage provisioning occurs

**13.4** What does HPA stand for?  
A. HTTP Proxy Adapter  
B. Helm Package Archive  
C. Horizontal Pod Autoscaler  
D. Host Port Allocator

**13.5** What does an HPA commonly adjust?  
A. Container image contents  
B. Number of Pod replicas  
C. Git history  
D. Node operating system

**13.6** What does a readiness probe determine?  
A. Whether a container image exists  
B. Whether a container should receive traffic  
C. Whether Git is clean  
D. Whether a node is a control plane

**13.7** What does a liveness probe help detect?  
A. Whether a Service has a public IP  
B. Whether a volume is encrypted  
C. Whether Terraform is initialized  
D. When a container should be restarted

**13.8** What does a startup probe help with?  
A. Allowing slow-starting apps time to initialize  
B. Provisioning a cloud account  
C. Creating a Docker network  
D. Checking Git credentials

**13.9** What can happen if a Pod's PVC remains unbound?  
A. The image is rebuilt automatically  
B. The Pod may remain Pending  
C. The Pod is always Running  
D. The Deployment is deleted

**13.10** Which command lists PVCs?  
A. kubectl get hpa-only  
B. kubectl get routes  
C. kubectl get mounts  
D. kubectl get pvc

<details>
<summary><b>✅ Answers · Session 13</b> (answer all 10 first, then click)</summary>

**Key:** 1-A · 2-B · 3-D · 4-C · 5-B · 6-B · 7-D · 8-A · 9-B · 10-D

**13.1** ✅ **A. A request for persistent storage**  
💡 A PVC (PersistentVolumeClaim) is a Pod's request for storage: a size and an access mode.

**13.2** ✅ **B. A cluster storage resource**  
💡 A PV (PersistentVolume) is an actual piece of storage in the cluster; a PVC binds to one.

**13.3** ✅ **D. How dynamic storage provisioning occurs**  
💡 A StorageClass describes how to create storage automatically (dynamic provisioning) when a PVC asks for it.

**13.4** ✅ **C. Horizontal Pod Autoscaler**  
💡 HPA = Horizontal Pod Autoscaler ('horizontal' = more or fewer Pods, not bigger Pods).

**13.5** ✅ **B. Number of Pod replicas**  
💡 An HPA changes the number of Pod replicas based on metrics such as CPU use.

**13.6** ✅ **B. Whether a container should receive traffic**  
💡 Readiness decides whether a Pod gets traffic; when it fails, the Pod is removed from the Service's endpoints but not restarted.

**13.7** ✅ **D. When a container should be restarted**  
💡 Liveness detects a stuck container; when it fails, the kubelet restarts the container.

**13.8** ✅ **A. Allowing slow-starting apps time to initialize**  
💡 A startup probe gives slow-starting apps time to start; liveness and readiness wait until it passes.

**13.9** ✅ **B. The Pod may remain Pending**  
💡 A Pod that needs a PVC which can't be bound cannot be scheduled, so it stays Pending.

**13.10** ✅ **D. kubectl get pvc**  
💡 kubectl get pvc lists claims and their status (Bound or Pending).

</details>

---

<a id="s14"></a>
## Session 14 · Kubernetes troubleshooting

**14.1** Which command gives a quick overview of Pod status?  
A. kubectl get pods  
B. kubectl publish  
C. kubectl init  
D. kubectl build pods

**14.2** Which command displays detailed Pod events and conditions?  
A. kubectl top namespace  
B. kubectl apply -k  
C. kubectl delete pod  
D. kubectl describe pod

**14.3** Which command reads a container's logs?  
A. kubectl version  
B. kubectl create  
C. kubectl label  
D. kubectl logs

**14.4** Which command runs a command inside a running container?  
A. kubectl exec  
B. kubectl config view  
C. kubectl explain  
D. kubectl diff

**14.5** What often causes ImagePullBackOff?  
A. An oversized log file only  
B. A healthy application  
C. A missing readiness probe only  
D. An image cannot be downloaded

**14.6** What often causes CrashLoopBackOff?  
A. A stable Service IP  
B. A container repeatedly exits or crashes  
C. A successful rollout  
D. A valid PVC

**14.7** What is a common reason for a Pod to remain Pending?  
A. Successful readiness probe  
B. Correct image tag  
C. Healthy DNS  
D. Insufficient schedulable resources

**14.8** Where can you see recent scheduling or pull errors?  
A. Kubernetes Events  
B. Git tags  
C. Docker Hub stars  
D. Helm templates only

**14.9** What should you check if a Service has no endpoints?  
A. Only the Git commit message  
B. Whether its selector matches ready Pods  
C. Only the cluster timezone  
D. Only Docker build cache

**14.10** What is the final step after fixing an incident?  
A. Delete all logs  
B. Disable probes  
C. Verify the workload behaves correctly  
D. Skip testing

<details>
<summary><b>✅ Answers · Session 14</b> (answer all 10 first, then click)</summary>

**Key:** 1-A · 2-D · 3-D · 4-A · 5-D · 6-B · 7-D · 8-A · 9-B · 10-C

**14.1** ✅ **A. kubectl get pods**  
💡 kubectl get pods gives the quick overview: READY, STATUS, RESTARTS and AGE.

**14.2** ✅ **D. kubectl describe pod**  
💡 kubectl describe pod shows the details: conditions, Last State, exit codes and, at the bottom, the Events.

**14.3** ✅ **D. kubectl logs**  
💡 kubectl logs reads what a container printed (add --previous for the copy that crashed).

**14.4** ✅ **A. kubectl exec**  
💡 kubectl exec runs a command inside a running container (kubectl exec -it <pod> -- sh).

**14.5** ✅ **D. An image cannot be downloaded**  
💡 ImagePullBackOff means the image can't be downloaded: wrong name or tag, or a private registry without a login.

**14.6** ✅ **B. A container repeatedly exits or crashes**  
💡 CrashLoopBackOff means the container keeps exiting and Kubernetes keeps restarting it, waiting longer each time.

**14.7** ✅ **D. Insufficient schedulable resources**  
💡 Pending usually means no node has enough unreserved CPU/memory for the Pod's requests (or a selector, taint or PVC blocks it).

**14.8** ✅ **A. Kubernetes Events**  
💡 Events (bottom of describe, or kubectl get events) record scheduling failures and image-pull errors.

**14.9** ✅ **B. Whether its selector matches ready Pods**  
💡 No endpoints → check that the Service's selector matches the Pods' labels and that those Pods are Ready.

**14.10** ✅ **C. Verify the workload behaves correctly**  
💡 After any fix, verify that the workload really works: Running and Ready, endpoints present, a test request succeeds.

</details>

---

<a id="s15"></a>
## Session 15 · Helm

**15.1** What is Helm?  
A. A package manager for Kubernetes  
B. A container runtime  
C. A network protocol  
D. A cloud hypervisor

**15.2** What is a Helm chart?  
A. A persistent disk  
B. A running Pod  
C. A Git branch  
D. A package of Kubernetes templates and metadata

**15.3** What is Chart.yaml used for?  
A. Pod runtime logs  
B. Ingress certificates  
C. Chart metadata such as name and version  
D. Terraform state

**15.4** What is values.yaml used for?  
A. Default configurable values for templates  
B. Git merge history  
C. Container registry authentication only  
D. Linux process IDs

**15.5** Which command creates a starter chart?  
A. helm pod create  
B. helm create  
C. helm chart start  
D. helm init cluster

**15.6** Which command installs a chart release?  
A. helm install  
B. helm push pod  
C. helm apply  
D. helm deploy node

**15.7** Which command updates an installed release?  
A. helm restart node  
B. helm rerun  
C. helm upgrade  
D. helm patch image

**15.8** Which command returns to an earlier release revision?  
A. helm revert image  
B. helm reset cluster  
C. helm rollback  
D. helm undo commit

**15.9** Which flag can override a value on the command line?  
A. --copy  
B. --set  
C. --port-forward  
D. --replace-node

**15.10** Which command renders templates without installing?  
A. helm template  
B. helm render-pod  
C. helm build-image  
D. helm dryrun-cluster

<details>
<summary><b>✅ Answers · Session 15</b> (answer all 10 first, then click)</summary>

**Key:** 1-A · 2-D · 3-C · 4-A · 5-B · 6-A · 7-C · 8-C · 9-B · 10-A

**15.1** ✅ **A. A package manager for Kubernetes**  
💡 Helm is a package manager for Kubernetes, like apt or pip but for apps on a cluster.

**15.2** ✅ **D. A package of Kubernetes templates and metadata**  
💡 A chart is a package: Kubernetes YAML templates plus default values plus metadata.

**15.3** ✅ **C. Chart metadata such as name and version**  
💡 Chart.yaml holds metadata: name, version (of the chart), appVersion (of the app), apiVersion v2.

**15.4** ✅ **A. Default configurable values for templates**  
💡 values.yaml holds the default values that fill the templates' blanks; override them with -f or --set.

**15.5** ✅ **B. helm create**  
💡 helm create <name> scaffolds a starter chart folder.

**15.6** ✅ **A. helm install**  
💡 helm install <release> <chart> installs a chart as a named release.

**15.7** ✅ **C. helm upgrade**  
💡 helm upgrade changes an installed release (new values or a new chart) and records a new revision.

**15.8** ✅ **C. helm rollback**  
💡 helm rollback <release> <revision> returns to an earlier revision (recorded as a new revision number).

**15.9** ✅ **B. --set**  
💡 --set key=value overrides a value on the command line; it beats values.yaml and -f files.

**15.10** ✅ **A. helm template**  
💡 helm template renders the final YAML on your machine without installing anything.

</details>

---

<a id="s16"></a>
## Session 16 · CI/CD & GitHub Actions

**16.1** Where are GitHub Actions workflow files normally stored?  
A. .github/workflows/  
B. .kube/actions/  
C. .git/actions/  
D. .docker/workflows/

**16.2** Which YAML key specifies workflow triggers?  
A. jobs  
B. env  
C. on  
D. steps

**16.3** What is a job in GitHub Actions?  
A. A Docker volume  
B. A Kubernetes Service  
C. A group of steps run on a runner  
D. A Git branch

**16.4** What does runs-on specify?  
A. Runner environment for a job  
B. Git default branch  
C. Kubernetes namespace  
D. Target Docker registry

**16.5** Which action commonly checks out repository code?  
A. actions/checkout  
B. actions/setup-dockerhub  
C. github/create-branch  
D. runner/load-repo

**16.6** What does a step's run field do?  
A. Executes shell commands  
B. Defines a Docker image digest  
C. Schedules a Kubernetes node  
D. Creates a cloud VPC automatically

**16.7** What does a step's uses field do?  
A. Creates a new repository  
B. Writes Terraform state  
C. Invokes a reusable action  
D. Starts an SSH daemon

**16.8** How should sensitive tokens be supplied to workflows?  
A. GitHub Actions secrets  
B. Committed to README  
C. Hardcoded in YAML  
D. Printed to logs

**16.9** What does needs do between jobs?  
A. Set Git author  
B. Select a Dockerfile stage  
C. Define container port mapping  
D. Declare job dependencies

**16.10** Which trigger runs a workflow on code pushes?  
A. push  
B. docker-build  
C. commit-only  
D. upload

<details>
<summary><b>✅ Answers · Session 16</b> (answer all 10 first, then click)</summary>

**Key:** 1-A · 2-C · 3-C · 4-A · 5-A · 6-A · 7-C · 8-A · 9-D · 10-A

**16.1** ✅ **A. .github/workflows/**  
💡 Workflow files must live in .github/workflows/ (exact folder name); anywhere else they are ignored.

**16.2** ✅ **C. on**  
💡 on: lists the triggers (push, pull_request, schedule, workflow_dispatch).

**16.3** ✅ **C. A group of steps run on a runner**  
💡 A job is a group of steps that run on one runner; jobs run in parallel unless needs: orders them.

**16.4** ✅ **A. Runner environment for a job**  
💡 runs-on chooses the runner, the machine the job runs on (e.g. ubuntu-latest).

**16.5** ✅ **A. actions/checkout**  
💡 actions/checkout copies the repository's code onto the runner; it's almost always step 1.

**16.6** ✅ **A. Executes shell commands**  
💡 run: executes shell commands on the runner.

**16.7** ✅ **C. Invokes a reusable action**  
💡 uses: runs a ready-made reusable action, written owner/repo@version.

**16.8** ✅ **A. GitHub Actions secrets**  
💡 Store tokens as GitHub Actions secrets and read them as ${{ secrets.NAME }}; they appear as *** in logs.

**16.9** ✅ **D. Declare job dependencies**  
💡 needs: makes a job wait for (depend on) other jobs; if they fail, it is skipped.

**16.10** ✅ **A. push**  
💡 on: push runs the workflow whenever commits are pushed.

</details>

---

<a id="s17"></a>
## Session 17 · Complete CI/CD & DevSecOps

**17.1** What is DevSecOps?  
A. Avoiding dependency updates  
B. Replacing CI with manual reviews  
C. Running security only after release  
D. Integrating security throughout development and delivery

**17.2** What does SAST analyze?  
A. Source code or binaries without executing the application  
B. Only production traffic  
C. Only DNS responses  
D. Only Kubernetes events

**17.3** What does DAST test?  
A. Only Git commit messages  
B. Only Docker layers  
C. A running application from the outside  
D. Only static source code

**17.4** What does dependency scanning identify?  
A. Known vulnerabilities in third-party packages  
B. Unused shell aliases  
C. Missing Git branches  
D. Slow DNS queries

**17.5** What does container image scanning inspect?  
A. Only Kubernetes Service names  
B. Image packages and known vulnerabilities  
C. Only developer laptops  
D. Only branch protections

**17.6** What is the principle of least privilege?  
A. Disable authentication  
B. Grant only permissions required  
C. Share one token everywhere  
D. Grant administrator rights to all

**17.7** Why should secrets not be committed to Git?  
A. They may be exposed through repository history  
B. They break YAML parsing  
C. Git cannot store text  
D. They prevent all merges

**17.8** What is an SBOM?  
A. Software Bill of Materials listing components  
B. Secure Backup Operating Mode  
C. Source Branch Ownership Matrix  
D. Server Build Output Map

**17.9** What is a common purpose of a security gate in CI?  
A. Disable audit logs  
B. Block promotion when policy checks fail  
C. Allow all vulnerabilities  
D. Skip all unit tests

**17.10** What does shift-left security mean?  
A. Run security checks earlier in development  
B. Move production servers physically  
C. Only scan after deployment  
D. Remove review from pipelines

<details>
<summary><b>✅ Answers · Session 17</b> (answer all 10 first, then click)</summary>

**Key:** 1-D · 2-A · 3-C · 4-A · 5-B · 6-B · 7-A · 8-A · 9-B · 10-A

**17.1** ✅ **D. Integrating security throughout development and delivery**  
💡 DevSecOps builds security into every stage of development and delivery, not just a check at the end.

**17.2** ✅ **A. Source code or binaries without executing the application**  
💡 SAST (Static Application Security Testing) analyses source code without running it (white-box).

**17.3** ✅ **C. A running application from the outside**  
💡 DAST (Dynamic Application Security Testing) attacks the running app from the outside (black-box).

**17.4** ✅ **A. Known vulnerabilities in third-party packages**  
💡 Dependency scanning (SCA, Software Composition Analysis) finds known vulnerabilities (CVEs) in third-party packages.

**17.5** ✅ **B. Image packages and known vulnerabilities**  
💡 Image scanning (e.g. Trivy) checks the image's OS packages and libraries for known vulnerabilities.

**17.6** ✅ **B. Grant only permissions required**  
💡 Least privilege: give each user or program only the permissions it needs, nothing more.

**17.7** ✅ **A. They may be exposed through repository history**  
💡 Git keeps every commit forever, so a committed secret stays in the history even after deletion; it must be revoked.

**17.8** ✅ **A. Software Bill of Materials listing components**  
💡 SBOM = Software Bill of Materials: a list of every component, and its version, inside the software.

**17.9** ✅ **B. Block promotion when policy checks fail**  
💡 A security gate stops the pipeline (blocks promotion) when a check fails, e.g. a scanner exiting with code 1.

**17.10** ✅ **A. Run security checks earlier in development**  
💡 Shift-left = run security checks early (on commit or PR), where problems are cheapest to fix.

</details>

---

<a id="s18"></a>
## Session 18 · Terraform & IaC

**18.1** What is Terraform used for?  
A. Writing application unit tests  
B. Serving static HTML only  
C. Monitoring CPU usage only  
D. Provisioning and managing infrastructure as code

**18.2** Which command initializes providers and backend?  
A. terraform output  
B. terraform destroy  
C. terraform apply  
D. terraform init

**18.3** Which command previews infrastructure changes?  
A. terraform show-only  
B. terraform plan  
C. terraform state rm  
D. terraform fmt

**18.4** Which command applies the planned changes?  
A. terraform fmt  
B. terraform validate  
C. terraform apply  
D. terraform graph

**18.5** What does Terraform state track?  
A. Only shell command history  
B. Only application logs  
C. Mappings between configuration and managed resources  
D. Only Docker images

**18.6** What is a Terraform provider?  
A. A Kubernetes Pod  
B. Plugin for interacting with an API or platform  
C. A Linux package manager  
D. A Git commit

**18.7** What is a Terraform module?  
A. Reusable group of Terraform configuration  
B. A DNS record type  
C. A container image  
D. An HTTP load balancer

**18.8** Which command checks Terraform configuration syntax and consistency?  
A. terraform destroy  
B. terraform output  
C. terraform import  
D. terraform validate

**18.9** What is a variable in Terraform?  
A. An automatically encrypted password  
B. An input used to parameterize configuration  
C. A Helm release  
D. A running cloud VM

**18.10** Why use a remote backend with locking?  
A. Share state safely and reduce concurrent update conflicts  
B. Run Pods without nodes  
C. Replace all version control  
D. Eliminate the need for credentials

<details>
<summary><b>✅ Answers · Session 18</b> (answer all 10 first, then click)</summary>

**Key:** 1-D · 2-D · 3-B · 4-C · 5-C · 6-B · 7-A · 8-D · 9-B · 10-A

**18.1** ✅ **D. Provisioning and managing infrastructure as code**  
💡 Terraform provisions and manages infrastructure from code files (IaC, Infrastructure as Code).

**18.2** ✅ **D. terraform init**  
💡 terraform init downloads the providers and sets up the backend (where state is kept); always run it first.

**18.3** ✅ **B. terraform plan**  
💡 terraform plan previews what will be created, changed or destroyed, without changing anything.

**18.4** ✅ **C. terraform apply**  
💡 terraform apply carries out the planned changes (it asks you to type yes).

**18.5** ✅ **C. Mappings between configuration and managed resources**  
💡 The state file (terraform.tfstate) maps each resource in your code to the real object it created.

**18.6** ✅ **B. Plugin for interacting with an API or platform**  
💡 A provider is a plugin that talks to one platform's API (aws, azurerm, google, local…).

**18.7** ✅ **A. Reusable group of Terraform configuration**  
💡 A module is a reusable group of Terraform configuration: a folder of .tf files that you call from other code.

**18.8** ✅ **D. terraform validate**  
💡 terraform validate checks syntax and internal consistency, without contacting the cloud.

**18.9** ✅ **B. An input used to parameterize configuration**  
💡 A variable is an input (variable "region" {…}) that parameterises the configuration.

**18.10** ✅ **A. Share state safely and reduce concurrent update conflicts**  
💡 A remote backend (e.g. S3) lets the team share one state file; locking stops two applies running at the same time.

</details>

---

<a id="s19"></a>
## Session 19 · Cloud & Terraform in action

**19.1** What does IaaS provide?  
A. Virtualized infrastructure such as compute and networks  
B. Only email templates  
C. Only finished end-user applications  
D. Only source code hosting

**19.2** What does PaaS primarily provide?  
A. Raw CPU chips only  
B. Only Git branches  
C. Managed platform for deploying applications  
D. Only DNS records

**19.3** What does SaaS typically deliver?  
A. An unmanaged hypervisor  
B. A local kernel module  
C. Ready-to-use software over a network  
D. A raw storage controller

**19.4** What is a cloud region?  
A. A geographic area containing cloud infrastructure  
B. A container tag  
C. A single application process  
D. A Git repository

**19.5** What is an availability zone?  
A. An isolated infrastructure location within a region  
B. A Kubernetes label only  
C. A Terraform variable  
D. A GitHub runner type

**19.6** What is a VPC?  
A. A logically isolated virtual network  
B. A container image  
C. A managed source code editor  
D. A version control protocol

**19.7** What is a subnet?  
A. A Git branch  
B. A CI pipeline stage  
C. An IP address range within a network  
D. A Docker build layer

**19.8** What does a route table define?  
A. Where network traffic should be forwarded  
B. Terraform module version  
C. Container restart count  
D. Application test coverage

**19.9** What does a security group commonly control?  
A. CPU instruction set  
B. YAML indentation  
C. Git merge policy  
D. Allowed inbound and outbound network traffic

**19.10** Why combine Terraform with cloud services?  
A. Reproducible, versioned infrastructure provisioning  
B. To compile frontend code  
C. To remove the need for networks  
D. To avoid reviewing changes

<details>
<summary><b>✅ Answers · Session 19</b> (answer all 10 first, then click)</summary>

**Key:** 1-A · 2-C · 3-C · 4-A · 5-A · 6-A · 7-C · 8-A · 9-D · 10-A

**19.1** ✅ **A. Virtualized infrastructure such as compute and networks**  
💡 IaaS (Infrastructure as a Service) gives you virtual machines, networks and storage; you manage the OS and everything above it (e.g. AWS EC2).

**19.2** ✅ **C. Managed platform for deploying applications**  
💡 PaaS (Platform as a Service) gives a managed platform; you only deploy your code (e.g. Heroku, Elastic Beanstalk).

**19.3** ✅ **C. Ready-to-use software over a network**  
💡 SaaS (Software as a Service) is ready-to-use software over the internet (e.g. Gmail).

**19.4** ✅ **A. A geographic area containing cloud infrastructure**  
💡 A region is a geographic area (e.g. ap-south-1, Mumbai) that contains several data centres.

**19.5** ✅ **A. An isolated infrastructure location within a region**  
💡 An AZ (Availability Zone) is an isolated data-centre location inside a region; using two or more gives high availability.

**19.6** ✅ **A. A logically isolated virtual network**  
💡 A VPC (Virtual Private Cloud) is your own isolated virtual network in the cloud (Azure calls it a VNet).

**19.7** ✅ **C. An IP address range within a network**  
💡 A subnet is a smaller range of IP addresses (e.g. 10.0.1.0/24) inside the VPC, placed in one AZ.

**19.8** ✅ **A. Where network traffic should be forwarded**  
💡 A route table says where traffic is sent; a route 0.0.0.0/0 → internet gateway makes a subnet public.

**19.9** ✅ **D. Allowed inbound and outbound network traffic**  
💡 A security group is a stateful virtual firewall listing the inbound and outbound traffic allowed for an instance.

**19.10** ✅ **A. Reproducible, versioned infrastructure provisioning**  
💡 Terraform + cloud = reproducible, version-controlled infrastructure that can be recreated identically.

</details>

---

<a id="s20"></a>
## Session 20 · Monitoring, Observability & GitOps

**20.1** What is a metric?  
A. A YAML parser  
B. A complete container image  
C. A Git branch  
D. A numeric measurement over time

**20.2** What is a log?  
A. A cloud region  
B. A persistent volume claim  
C. A DNS subnet  
D. A timestamped record of events

**20.3** What is a distributed trace?  
A. A view of a request across services  
B. A Docker image tag  
C. A Git commit hash only  
D. A shell variable

**20.4** What is Prometheus commonly used for?  
A. Building Docker images  
B. Provisioning VPCs  
C. Collecting and querying time-series metrics  
D. Hosting Git repositories

**20.5** What is Grafana commonly used for?  
A. Visualizing metrics and creating dashboards  
B. Building Helm charts  
C. Scanning source code  
D. Scheduling Kubernetes Pods

**20.6** What does observability help engineers do?  
A. Avoid all incident response  
B. Infer internal system behavior from telemetry  
C. Disable logs  
D. Replace application testing

**20.7** What is GitOps?  
A. Replacing Kubernetes with Git  
B. Storing passwords in plaintext  
C. Running all changes manually  
D. Managing desired system state through version-controlled Git

**20.8** In GitOps, what is the source of truth?  
A. An untracked server file  
B. A developer's local terminal  
C. A random running Pod  
D. The desired configuration stored in Git

**20.9** What is Argo CD commonly used for?  
A. Reconciling Kubernetes deployments with Git  
B. Creating Docker base images  
C. Resolving DNS on a laptop  
D. Compiling Java source code

**20.10** What does configuration drift mean?  
A. Actual state differs from declared desired state  
B. All Pods have identical images  
C. Every Git branch is merged  
D. Metrics are stored in a database

<details>
<summary><b>✅ Answers · Session 20</b> (answer all 10 first, then click)</summary>

**Key:** 1-D · 2-D · 3-A · 4-C · 5-A · 6-B · 7-D · 8-D · 9-A · 10-A

**20.1** ✅ **D. A numeric measurement over time**  
💡 A metric is a number measured over time (CPU %, requests per second).

**20.2** ✅ **D. A timestamped record of events**  
💡 A log is a timestamped record of an event: one line per thing that happened.

**20.3** ✅ **A. A view of a request across services**  
💡 A distributed trace follows one request across many services and shows the time spent in each.

**20.4** ✅ **C. Collecting and querying time-series metrics**  
💡 Prometheus scrapes (pulls) metrics from targets and stores them as time series; you query them with PromQL.

**20.5** ✅ **A. Visualizing metrics and creating dashboards**  
💡 Grafana visualises data in dashboards and panels, using sources like Prometheus; it stores no metrics itself.

**20.6** ✅ **B. Infer internal system behavior from telemetry**  
💡 Observability = being able to understand what's happening inside a system from its telemetry (metrics, logs, traces).

**20.7** ✅ **D. Managing desired system state through version-controlled Git**  
💡 GitOps keeps the desired state in Git, and an agent (e.g. Argo CD) makes the cluster match it.

**20.8** ✅ **D. The desired configuration stored in Git**  
💡 In GitOps, the desired configuration stored in Git is the single source of truth.

**20.9** ✅ **A. Reconciling Kubernetes deployments with Git**  
💡 Argo CD reconciles (syncs) the cluster with the manifests stored in Git.

**20.10** ✅ **A. Actual state differs from declared desired state**  
💡 Drift = the real state differs from the declared desired state, e.g. after a manual kubectl change.

</details>

