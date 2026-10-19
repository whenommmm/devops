# MCQ bank · all 190 questions, in session order

*Each question shows its options, then the answer and why. Tip: cover the ✅ line, answer, then check.*

**Sessions:** [1](#session-1)  · [2](#session-2)  · [3](#session-3)  · [4](#session-4)  · [5](#session-5)  · [6–7](#session-67)  · [8](#session-8)  · [9](#session-9)  · [10](#session-10)  · [11](#session-11)  · [12](#session-12)  · [13](#session-13)  · [14](#session-14)  · [15](#session-15)  · [16](#session-16)  · [17](#session-17)  · [18](#session-18)  · [19](#session-19)  · [20](#session-20)

<a id="session-1"></a>
## Session 1 · DevOps intro & roadmap

**Q1. What is the primary objective of DevOps?**

A. Faster, reliable delivery through collaboration and automation  
B. Replace developers with servers  
C. Avoid version control  
D. Eliminate all testing

✅ **A. Faster, reliable delivery through collaboration and automation**  
💡 DevOps joins development and operations so software ships often and safely; collaboration and automation are how it gets there.

**Q2. What does CI mean?**

A. Central Installation  
B. Continuous Integration  
C. Cloud Inspection  
D. Code Isolation

✅ **B. Continuous Integration**  
💡 CI = Continuous Integration: developers merge small changes often, and every merge is automatically built and tested.

**Q3. Which practice helps catch defects early?**

A. Manual deployment only  
B. Disabling logs  
C. Skipping code reviews  
D. Automated tests in CI

✅ **D. Automated tests in CI**  
💡 Tests that run automatically on every push catch a bug minutes after it's written, long before production.

**Q4. What is infrastructure as code?**

A. Keeping server settings undocumented  
B. Writing only frontend code  
C. Managing infrastructure using versioned configuration  
D. Purchasing hardware manually

✅ **C. Managing infrastructure using versioned configuration**  
💡 IaC = Infrastructure as Code: servers and networks are described in files kept in Git, so they can be reviewed, versioned and rebuilt identically.

**Q5. Which tool is commonly used for source control?**

A. Prometheus  
B. Nginx  
C. Git  
D. Grafana

✅ **C. Git**  
💡 Git is the standard version-control (source-control) tool; it records every change to the code.

**Q6. What does a CI/CD pipeline automate?**

A. Build, test and delivery steps  
B. Employee attendance  
C. Domain purchases  
D. Keyboard configuration

✅ **A. Build, test and delivery steps**  
💡 A pipeline automates the build → test → delivery steps that would otherwise be done by hand on every change.

**Q7. Why are containers useful?**

A. They package applications with dependencies  
B. They remove the need for networks  
C. They guarantee zero bugs  
D. They replace all databases

✅ **A. They package applications with dependencies**  
💡 A container bundles the app with its libraries and settings, so it runs the same on any machine.

**Q8. What is monitoring used for?**

A. Formatting YAML  
B. Compiling source code  
C. Tracking system health and behavior  
D. Creating Git branches

✅ **C. Tracking system health and behavior**  
💡 Monitoring tracks health and behaviour (CPU, errors, response time) so problems are noticed and alerted on quickly.

**Q9. Which mindset is central to DevOps?**

A. Shared ownership across development and operations  
B. Only manual changes  
C. Separate teams with no communication  
D. No production feedback

✅ **A. Shared ownership across development and operations**  
💡 DevOps culture = shared ownership: developers and operations are responsible for the software together, instead of 'throwing it over the wall'.

**Q10. Which is a common DevOps lifecycle activity?**

A. Continuous feedback  
B. Disabling security scans  
C. Ignoring incidents  
D. One-time deployment forever

✅ **A. Continuous feedback**  
💡 Monitoring and user feedback flow back into planning; the DevOps lifecycle is a loop (Plan → Code → Build → Test → Release → Deploy → Operate → Monitor → Plan).


<a id="session-2"></a>
## Session 2 · Linux

**Q1. Which command prints the working directory?**

A. mv  
B. ls  
C. cd  
D. pwd

✅ **D. pwd**  
💡 pwd = print working directory: shows the full path of the folder you are in.

**Q2. Which command lists files including hidden files?**

A. pwd -a  
B. rm -a  
C. ls -a  
D. mkdir -a

✅ **C. ls -a**  
💡 ls -a = list all, including hidden files whose names start with a dot (e.g. .bashrc).

**Q3. What does chmod change?**

A. File permissions  
B. Hostname  
C. File ownership only  
D. Network interface

✅ **A. File permissions**  
💡 chmod = change mode: changes read/write/execute permissions, e.g. chmod 755 file (rwx r-x r-x).

**Q4. Which command searches text in files?**

A. pwd  
B. touch  
C. mv  
D. grep

✅ **D. grep**  
💡 grep searches for a text pattern inside files and prints the matching lines (grep error app.log).

**Q5. Which command displays a file's contents?**

A. kill  
B. rmdir  
C. mkdir  
D. cat

✅ **D. cat**  
💡 cat (concatenate) prints a file's contents to the screen.

**Q6. What does the pipe operator do?**

A. Redirects all input to a file  
B. Runs commands only as root  
C. Deletes temporary files  
D. Passes one command's output to another

✅ **D. Passes one command's output to another**  
💡 The pipe | sends one command's output into the next command as input (ps aux | grep nginx).

**Q7. Which command shows running processes?**

A. cp  
B. ls  
C. ps  
D. mkdir

✅ **C. ps**  
💡 ps = process status: lists running processes; ps aux shows every process.

**Q8. Which command changes the current directory?**

A. cd  
B. echo  
C. pwd  
D. cat

✅ **A. cd**  
💡 cd = change directory; cd .. goes up one level, cd ~ goes home.

**Q9. What does sudo typically allow?**

A. Run commands with elevated privileges  
B. Download a container image  
C. Create a symbolic link  
D. Show DNS records

✅ **A. Run commands with elevated privileges**  
💡 sudo = 'superuser do': runs one command with root (administrator) privileges.

**Q10. Which command displays disk space usage by filesystem?**

A. du -a  
B. uname -a  
C. free -m  
D. df -h

✅ **D. df -h**  
💡 df -h = disk free, in human-readable sizes, for each filesystem. (du shows how big a folder is.)


<a id="session-3"></a>
## Session 3 · Shell scripting

**Q1. Which line commonly selects Bash as script interpreter?**

A. #bash/start  
B. //bin/bash  
C. #!/bin/bash  
D. #!/usr/git

✅ **C. #!/bin/bash**  
💡 #!/bin/bash (the shebang) on line 1 tells the system which interpreter should run the script.

**Q2. How do you reference the first positional argument in Bash?**

A. $0  
B. $#  
C. $?  
D. $1

✅ **D. $1**  
💡 $1 is the first argument given to the script ($0 = the script's name, $# = how many arguments).

**Q3. Which operator tests whether a regular file exists in Bash?**

A. [ -p file ]  
B. [ -d file ]  
C. [ -f file ]  
D. [ -z file ]

✅ **C. [ -f file ]**  
💡 [ -f file ] is true when the path exists and is a regular file (-d = a directory, -e = exists at all).

**Q4. What does $? contain immediately after a command?**

A. Exit status of the previous command  
B. Current username  
C. Number of script lines  
D. Current process ID

✅ **A. Exit status of the previous command**  
💡 $? holds the exit status of the last command: 0 = success, anything else = failure.

**Q5. Which loop iterates over a sequence of values?**

A. exit  
B. for  
C. case  
D. if

✅ **B. for**  
💡 for loops over a list of values: for f in *.log; do echo $f; done.

**Q6. How do you make a script executable?**

A. grep +x script.sh  
B. pwd script.sh  
C. chmod +x script.sh  
D. cat +x script.sh

✅ **C. chmod +x script.sh**  
💡 chmod +x adds execute permission, so you can run the script as ./script.sh.

**Q7. Which construct makes a conditional decision?**

A. for ... done  
B. echo ... print  
C. if ... then ... fi  
D. read ... exit

✅ **C. if ... then ... fi**  
💡 if … then … fi makes a decision; fi ('if' backwards) closes the block.

**Q8. How do you assign a variable in Bash?**

A. $NAME=value  
B. NAME=value  
C. set NAME := value  
D. NAME = value

✅ **B. NAME=value**  
💡 NAME=value assigns a variable, with no spaces around the =; read it back with $NAME.

**Q9. Which command reads input from the terminal?**

A. touch  
B. echo  
C. printf  
D. read

✅ **D. read**  
💡 read waits for the user to type a line and stores it in a variable (read name).

**Q10. What does && mean between two commands?**

A. Run the second only if first fails  
B. Run the second only if the first succeeds  
C. Always run both in parallel  
D. Append output to a file

✅ **B. Run the second only if the first succeeds**  
💡 a && b runs b only if a succeeded (exit 0); a || b runs b only if a failed.


<a id="session-4"></a>
## Session 4 · Networking

**Q1. What does DNS primarily do?**

A. Resolve names to IP addresses  
B. Build application binaries  
C. Schedule containers  
D. Encrypt disk volumes

✅ **A. Resolve names to IP addresses**  
💡 DNS = Domain Name System: turns names like google.com into IP addresses.

**Q2. What does an IP address identify?**

A. A Docker image tag  
B. A CPU instruction  
C. A network interface on an IP network  
D. A Git repository

✅ **C. A network interface on an IP network**  
💡 An IP (Internet Protocol) address identifies one network interface, i.e. one connection of a machine, on an IP network.

**Q3. Which protocol provides reliable ordered delivery?**

A. ICMP  
B. ARP  
C. TCP  
D. UDP

✅ **C. TCP**  
💡 TCP = Transmission Control Protocol: connection-based, resends lost data and keeps it in order. UDP is faster but guarantees nothing.

**Q4. Which command can test basic ICMP reachability?**

A. ping  
B. git status  
C. chmod  
D. docker build

✅ **A. ping**  
💡 ping sends ICMP (Internet Control Message Protocol) echo requests to check whether a host can be reached.

**Q5. What does HTTP status 404 mean?**

A. Authentication succeeded  
B. Resource not found  
C. Successful request  
D. Server started

✅ **B. Resource not found**  
💡 404 Not Found: the server answered, but the requested path doesn't exist (4xx = a client-side error).

**Q6. What is the usual default port for HTTPS?**

A. 22  
B. 53  
C. 443  
D. 80

✅ **C. 443**  
💡 HTTPS uses port 443 by default (HTTP 80, SSH 22, DNS 53).

**Q7. Which tool can make HTTP requests from a terminal?**

A. tar  
B. mkdir  
C. grep  
D. curl

✅ **D. curl**  
💡 curl sends HTTP requests from the terminal (curl -I shows only the response headers).

**Q8. What does a subnet mask/prefix indicate?**

A. A Docker volume size  
B. A password length  
C. The network portion of an IP address  
D. A DNS TTL only

✅ **C. The network portion of an IP address**  
💡 A subnet mask or prefix (/24 = 255.255.255.0) says which part of an IP address is the network and which part is the host.

**Q9. What is the role of a default gateway?**

A. Forward traffic to other networks  
B. Compile Java programs  
C. Start systemd services  
D. Resolve Git merge conflicts

✅ **A. Forward traffic to other networks**  
💡 The default gateway (usually the router) forwards traffic that is going to other networks, such as the internet.

**Q10. Which protocol is commonly used for secure remote shell access?**

A. DHCP  
B. FTP  
C. SSH  
D. SMTP

✅ **C. SSH**  
💡 SSH = Secure Shell: encrypted remote login, on port 22.


<a id="session-5"></a>
## Session 5 · Git & GitHub

**Q1. Which command initializes a local Git repository?**

A. git init  
B. git boot  
C. git create  
D. git start

✅ **A. git init**  
💡 git init creates a new repository (the hidden .git folder) in the current folder.

**Q2. Which command stages a file for commit?**

A. git fetch  
B. git log  
C. git add  
D. git push

✅ **C. git add**  
💡 git add puts changes into the staging area, ready for the next commit.

**Q3. Which command records staged changes?**

A. git status  
B. git diff  
C. git commit  
D. git clone

✅ **C. git commit**  
💡 git commit saves the staged changes as a snapshot, with a message (git commit -m "…").

**Q4. Which command shows the current working tree state?**

A. git status  
B. git branch -d  
C. git gc  
D. git tag -d

✅ **A. git status**  
💡 git status shows which files are modified, staged or untracked (git diff shows the changed lines themselves).

**Q5. Which command creates a new branch?**

A. git reset feature  
B. git branch feature  
C. git commit feature  
D. git push feature

✅ **B. git branch feature**  
💡 git branch feature creates a branch but stays on the current one (git switch -c feature creates it and switches to it).

**Q6. Which command integrates another branch into the current branch?**

A. git fetch  
B. git clean  
C. git merge  
D. git stash

✅ **C. git merge**  
💡 git merge feature brings that branch's commits into the branch you are on.

**Q7. What does git clone do?**

A. Deletes a remote branch  
B. Reverts the operating system  
C. Renames a commit  
D. Copies a repository to a local directory

✅ **D. Copies a repository to a local directory**  
💡 git clone copies a remote repository, with its full history, into a new local folder.

**Q8. What is a pull request typically used for?**

A. Building a Docker layer  
B. Starting a Kubernetes pod  
C. Reviewing and discussing changes before merge  
D. Provisioning a VPC

✅ **C. Reviewing and discussing changes before merge**  
💡 A pull request (PR) proposes merging a branch, so teammates can review and discuss the changes first.

**Q9. What does .gitignore control?**

A. Which untracked paths Git should ignore  
B. Who can access GitHub  
C. The commit author  
D. The default remote URL

✅ **A. Which untracked paths Git should ignore**  
💡 .gitignore lists untracked files and folders Git should ignore (node_modules/, .env); files that are already tracked are not affected.

**Q10. Which command uploads local commits to a remote?**

A. git diff  
B. git pull  
C. git push  
D. git show

✅ **C. git push**  
💡 git push uploads your local commits to the remote repository (e.g. GitHub).


<a id="session-67"></a>
## Session 6–7 · Docker & Dockerfile

**Q1. What is a Docker image?**

A. A running virtual machine only  
B. A Git branch  
C. A DNS record  
D. A read-only template for creating containers

✅ **D. A read-only template for creating containers**  
💡 An image is a read-only template (the app plus its dependencies, in layers); a container is a running instance of it.

**Q2. Which command builds an image from a Dockerfile?**

A. docker start  
B. docker logs  
C. docker build  
D. docker exec

✅ **C. docker build**  
💡 docker build -t name . builds an image from the Dockerfile in the current folder (the . is the build context).

**Q3. Which Dockerfile instruction sets the base image?**

A. COPY  
B. CMD  
C. EXPOSE  
D. FROM

✅ **D. FROM**  
💡 FROM sets the base image and must be the first instruction (FROM python:3.12-slim).

**Q4. Which Dockerfile instruction copies files into an image?**

A. USER  
B. WORKDIR  
C. COPY  
D. RUN

✅ **C. COPY**  
💡 COPY copies files from the build context into the image (COPY . /app).

**Q5. Which command lists running containers?**

A. docker images  
B. docker ps  
C. docker build  
D. docker pull

✅ **B. docker ps**  
💡 docker ps lists running containers; docker ps -a also shows stopped ones.

**Q6. What does docker run -p 8080:80 map?**

A. A volume to port 80  
B. Host port 80 to container port 8080  
C. Host port 8080 to container port 80  
D. Two host ports together

✅ **C. Host port 8080 to container port 80**  
💡 -p 8080:80 means host port 8080 → container port 80 (host first, container second).

**Q7. What does docker logs show?**

A. Container standard output and error  
B. Registry credentials  
C. Git commit history  
D. Host kernel source

✅ **A. Container standard output and error**  
💡 docker logs shows what the container's main process wrote to stdout (standard output) and stderr (standard error).

**Q8. Which command executes a command inside a running container?**

A. docker exec  
B. docker save  
C. docker tag  
D. docker push

✅ **A. docker exec**  
💡 docker exec runs a command inside a running container (docker exec -it web sh opens a shell).

**Q9. What is the purpose of .dockerignore?**

A. Block network access  
B. Ignore runtime errors  
C. Exclude paths from the build context  
D. Hide containers from ps

✅ **C. Exclude paths from the build context**  
💡 .dockerignore keeps files (.git, node_modules, .env) out of the build context, so builds are faster and secrets stay out of images.

**Q10. Which Dockerfile instruction defines a default command?**

A. LABEL  
B. ARG  
C. EXPOSE  
D. CMD

✅ **D. CMD**  
💡 CMD sets the default command a container runs when it starts; arguments after the image name in docker run replace it.


<a id="session-8"></a>
## Session 8 · Docker networking, volumes & Compose

**Q1. Why use Docker volumes?**

A. Increase CPU frequency  
B. Disable networking  
C. Persist data beyond a container lifecycle  
D. Replace container images

✅ **C. Persist data beyond a container lifecycle**  
💡 A volume keeps data outside the container's writable layer, so the data survives removing the container.

**Q2. Which command lists Docker volumes?**

A. docker volumes show  
B. docker storage ps  
C. docker volume ls  
D. docker volume ps

✅ **C. docker volume ls**  
💡 docker volume ls lists volumes (docker volume inspect shows details such as where the data lives).

**Q3. What is a bind mount?**

A. A remote Git branch  
B. A Kubernetes namespace  
C. A host path mounted inside a container  
D. An image registry

✅ **C. A host path mounted inside a container**  
💡 A bind mount maps a folder on the host into the container (-v /host/path:/container/path).

**Q4. Which Docker network driver is the typical default for standalone containers?**

A. overlay  
B. macvtap only  
C. host-only VM  
D. bridge

✅ **D. bridge**  
💡 bridge is the default network driver for standalone containers.

**Q5. What does a user-defined bridge network provide?**

A. Host kernel upgrades  
B. Container-to-container name resolution  
C. Automatic cloud load balancing  
D. Persistent image layers

✅ **B. Container-to-container name resolution**  
💡 A user-defined bridge network has built-in DNS, so containers reach each other by name (the default bridge doesn't do this).

**Q6. Which flag publishes a container port?**

A. -w  
B. -e  
C. -v  
D. -p

✅ **D. -p**  
💡 -p (--publish) publishes a container port on the host so outside traffic can reach it.

**Q7. Which command creates a Docker network?**

A. docker net init  
B. docker bridge build  
C. docker network create  
D. docker network push

✅ **C. docker network create**  
💡 docker network create mynet creates a network; connect containers with --network mynet.

**Q8. What is Docker Compose mainly for?**

A. Replacing source control  
B. Defining and running multi-container applications  
C. Managing Linux users  
D. Creating DNS root servers

✅ **B. Defining and running multi-container applications**  
💡 Compose defines and runs a multi-container application (e.g. web + database) from one YAML file.

**Q9. Which file is commonly used by Docker Compose?**

A. Chart.yaml  
B. terraform.tfstate  
C. Jenkinsfile  
D. compose.yaml

✅ **D. compose.yaml**  
💡 compose.yaml is Compose's file (the older name docker-compose.yml still works).

**Q10. Which command starts services from a Compose file?**

A. docker compose attach-all  
B. docker compose init-only  
C. docker compose up  
D. docker compose commit

✅ **C. docker compose up**  
💡 docker compose up (add -d for background) starts every service; docker compose down stops and removes them.


<a id="session-9"></a>
## Session 9 · Kubernetes fundamentals

**Q1. What is Kubernetes primarily used for?**

A. Replacing Git history  
B. Editing photos  
C. Compiling Linux kernels  
D. Orchestrating containerized applications

✅ **D. Orchestrating containerized applications**  
💡 Kubernetes orchestrates containers: it places them on machines, restarts failed ones, scales them and updates them.

**Q2. What is a Kubernetes Pod?**

A. A Terraform provider  
B. A physical data center  
C. Smallest deployable unit containing one or more containers  
D. A Docker image registry

✅ **C. Smallest deployable unit containing one or more containers**  
💡 A Pod is the smallest deployable unit: one or more containers that share one IP address and storage.

**Q3. Which component stores Kubernetes cluster state?**

A. containerd  
B. CoreDNS  
C. etcd  
D. kube-proxy

✅ **C. etcd**  
💡 etcd is the key-value database that stores the whole cluster's state; only the API server talks to it.

**Q4. Which component schedules Pods onto nodes?**

A. kube-scheduler  
B. kube-proxy  
C. kubectl  
D. kubelet

✅ **A. kube-scheduler**  
💡 kube-scheduler picks a node for each new Pod, based on free resources and constraints.

**Q5. What does the kubelet do?**

A. Stores all cluster state  
B. Builds Helm charts  
C. Approves pull requests  
D. Ensures Pod containers run on a node

✅ **D. Ensures Pod containers run on a node**  
💡 The kubelet is the agent on every node that makes sure the Pod's containers are running.

**Q6. What is kubectl?**

A. A Linux filesystem  
B. A CLI for interacting with the Kubernetes API  
C. A DNS protocol  
D. A container image format

✅ **B. A CLI for interacting with the Kubernetes API**  
💡 kubectl is the command-line tool that sends your requests to the Kubernetes API server.

**Q7. Which command lists Pods?**

A. kubectl show nodes  
B. kubectl list images  
C. kubectl build pods  
D. kubectl get pods

✅ **D. kubectl get pods**  
💡 kubectl get pods lists the Pods in the current namespace (add -A for all namespaces).

**Q8. What is a Kubernetes namespace?**

A. A Dockerfile instruction  
B. A logical grouping for cluster resources  
C. A container image tag  
D. A dedicated physical server

✅ **B. A logical grouping for cluster resources**  
💡 A namespace is a logical group, like a folder, for resources inside one cluster (default, kube-system…).

**Q9. Which component exposes the Kubernetes API?**

A. kubelet  
B. kube-apiserver  
C. CoreDNS  
D. kube-proxy

✅ **B. kube-apiserver**  
💡 kube-apiserver exposes the Kubernetes API; every kubectl command and every component goes through it.

**Q10. What is a worker node?**

A. A DNS resolver record  
B. A GitHub workflow file  
C. A Helm repository  
D. A machine that runs application Pods

✅ **D. A machine that runs application Pods**  
💡 A worker node is a machine that runs the application Pods (it has the kubelet, kube-proxy and a container runtime).


<a id="session-10"></a>
## Session 10 · Pods, ReplicaSets & Deployments

**Q1. What is the purpose of a Deployment?**

A. Store cluster state  
B. Manage desired replicas and rolling updates  
C. Expose a public DNS zone  
D. Create container images

✅ **B. Manage desired replicas and rolling updates**  
💡 A Deployment keeps the desired number of replicas running and performs rolling updates and rollbacks.

**Q2. What does a ReplicaSet ensure?**

A. A desired number of matching Pods are running  
B. Nodes never fail  
C. Every Pod has a public IP  
D. All images are signed

✅ **A. A desired number of matching Pods are running**  
💡 A ReplicaSet makes sure the desired number of Pods matching its selector are running.

**Q3. Which field commonly sets Deployment replica count?**

A. status.desiredNodes  
B. spec.ports  
C. metadata.replicas  
D. spec.replicas

✅ **D. spec.replicas**  
💡 spec.replicas sets how many Pods the Deployment wants.

**Q4. What does a Pod label help with?**

A. Allocating disk partitions  
B. Installing Helm  
C. Selecting and organizing resources  
D. Encrypting image layers

✅ **C. Selecting and organizing resources**  
💡 Labels are key=value tags (app=web) used to select and organise resources.

**Q5. What is a label selector used for?**

A. Selecting Git commits by date  
B. Matching resources with specified labels  
C. Choosing Docker build stages  
D. Filtering shell history

✅ **B. Matching resources with specified labels**  
💡 A label selector matches resources by their labels; Services and ReplicaSets use one to find their Pods.

**Q6. Which command scales a Deployment?**

A. kubectl duplicate image  
B. kubectl increase node  
C. kubectl scale deployment  
D. kubectl resize pod

✅ **C. kubectl scale deployment**  
💡 kubectl scale deployment web --replicas=5 changes the number of replicas.

**Q7. What happens when a Deployment Pod crashes?**

A. A controller works to restore desired state  
B. Git automatically rolls back  
C. Kubernetes deletes the Deployment permanently  
D. The cluster always stops

✅ **A. A controller works to restore desired state**  
💡 The controller sees fewer Pods than desired and creates a replacement: this is self-healing.

**Q8. Which command views rollout status?**

A. kubectl get dns  
B. kubectl rollout status deployment/name  
C. kubectl helm status  
D. kubectl logs nodes

✅ **B. kubectl rollout status deployment/name**  
💡 kubectl rollout status deployment/web waits and reports whether the rollout has finished.

**Q9. What does a rolling update do?**

A. Gradually replace old Pods with new ones  
B. Change namespace ownership  
C. Restart the control plane  
D. Delete all nodes first

✅ **A. Gradually replace old Pods with new ones**  
💡 A rolling update replaces old Pods with new ones a few at a time, so the app stays available.

**Q10. Which resource usually creates and manages ReplicaSets?**

A. Secret  
B. PersistentVolume  
C. ConfigMap  
D. Deployment

✅ **D. Deployment**  
💡 A Deployment creates and manages ReplicaSets (one per version), and each ReplicaSet manages the Pods.


<a id="session-11"></a>
## Session 11 · Kubernetes Services

**Q1. Why are Kubernetes Services used?**

A. Provide stable access to a set of Pods  
B. Compile application code  
C. Build container images  
D. Store Git history

✅ **A. Provide stable access to a set of Pods**  
💡 Pod IPs change whenever Pods are replaced; a Service gives one stable name and IP that load-balances across the matching Pods.

**Q2. Which Service type is typically reachable only within the cluster?**

A. ClusterIP  
B. NodePort  
C. LoadBalancer  
D. ExternalName only

✅ **A. ClusterIP**  
💡 ClusterIP (the default type) is reachable only from inside the cluster.

**Q3. Which Service type exposes a port on each node?**

A. NodePort  
B. ExternalName  
C. Headless only  
D. ClusterIP

✅ **A. NodePort**  
💡 NodePort opens the same port (30000–32767) on every node, so the app can be reached at <NodeIP>:<nodePort>.

**Q4. Which Service type commonly provisions an external load balancer on supported clouds?**

A. LoadBalancer  
B. ClusterIP  
C. ExternalName  
D. Headless

✅ **A. LoadBalancer**  
💡 LoadBalancer asks the cloud provider for an external load balancer with a public IP.

**Q5. How does a Service identify backend Pods?**

A. Label selector  
B. Node hostname only  
C. Git branch name  
D. Docker image digest

✅ **A. Label selector**  
💡 A Service finds its Pods with a label selector; matching Pods that are Ready become its endpoints.

**Q6. What does a Service port targetPort refer to?**

A. SSH port of the node  
B. Port on a Git remote  
C. Port on backend Pods  
D. Port of the API server

✅ **C. Port on backend Pods**  
💡 targetPort is the port on the Pods where the app actually listens; port is the Service's own port.

**Q7. What is the role of kube-proxy in many clusters?**

A. Build Dockerfiles  
B. Schedule Pods  
C. Store secrets in etcd  
D. Implement Service traffic routing rules

✅ **D. Implement Service traffic routing rules**  
💡 kube-proxy runs on every node and writes the routing rules (iptables/IPVS) that send Service traffic to the Pods.

**Q8. Which DNS name form can identify a Service across namespaces?**

A. pod.image.registry.local  
B. service.cluster.node.local  
C. service.namespace.svc.cluster.local  
D. namespace.service.github.io

✅ **C. service.namespace.svc.cluster.local**  
💡 <service>.<namespace>.svc.cluster.local is the full DNS name; a short name like web only works within the same namespace.

**Q9. What does a headless Service commonly set?**

A. replicas: 0  
B. type: NodePort  
C. hostNetwork: true  
D. clusterIP: None

✅ **D. clusterIP: None**  
💡 A headless Service sets clusterIP: None, so DNS returns the individual Pod IPs (used with StatefulSets).

**Q10. Which command lists Services?**

A. kubectl build svc  
B. kubectl init service  
C. kubectl get svc  
D. kubectl get images

✅ **C. kubectl get svc**  
💡 kubectl get svc (short for services) lists Services with their type, cluster IP and ports.


<a id="session-12"></a>
## Session 12 · Ingress, ConfigMaps & Secrets

**Q1. What does Ingress commonly define?**

A. HTTP/HTTPS routing rules to Services  
B. Git branch protections  
C. Node CPU architecture  
D. Container build steps

✅ **A. HTTP/HTTPS routing rules to Services**  
💡 An Ingress holds HTTP/HTTPS routing rules (by host name and URL path) that send requests to Services.

**Q2. What is needed to implement Ingress rules?**

A. A Git tag  
B. A Terraform lock file  
C. A Docker registry only  
D. An Ingress controller

✅ **D. An Ingress controller**  
💡 Ingress rules do nothing until an Ingress controller (such as NGINX) runs in the cluster to apply them.

**Q3. What is a ConfigMap intended to store?**

A. Persistent disk blocks  
B. Passwords by default  
C. Container images  
D. Non-sensitive configuration data

✅ **D. Non-sensitive configuration data**  
💡 A ConfigMap holds non-secret settings as key–value pairs, kept outside the image.

**Q4. What is a Secret intended to store?**

A. Sensitive configuration values  
B. Container images  
C. Cluster logs only  
D. Helm binaries

✅ **A. Sensitive configuration values**  
💡 A Secret holds sensitive values: passwords, tokens, keys, certificates.

**Q5. How can a ConfigMap be consumed by a Pod?**

A. Node kernel flags only  
B. Docker build cache only  
C. Git commit messages only  
D. Environment variables or mounted files

✅ **D. Environment variables or mounted files**  
💡 A Pod uses a ConfigMap as environment variables (env / envFrom) or as files from a mounted volume.

**Q6. What does a path rule in Ingress match?**

A. Container filesystem path  
B. The HTTP request path  
C. Host disk mount path  
D. Git remote path

✅ **B. The HTTP request path**  
💡 A path rule matches the URL path of the HTTP request (e.g. /api), not a folder on disk.

**Q7. What does TLS termination usually provide at Ingress?**

A. Automatic Pod autoscaling  
B. Persistent volumes  
C. Build artifact signing  
D. HTTPS encryption for client connections

✅ **D. HTTPS encryption for client connections**  
💡 TLS termination: the Ingress controller holds the certificate and handles HTTPS with the client, then forwards plain HTTP to the Pods.

**Q8. Which command lists ConfigMaps?**

A. kubectl get configmaps  
B. kubectl get secrets-only  
C. kubectl get networks  
D. kubectl get charts

✅ **A. kubectl get configmaps**  
💡 kubectl get configmaps (short: cm) lists ConfigMaps.

**Q9. Are Kubernetes Secret values encrypted merely because YAML uses base64?**

A. Yes, base64 is strong encryption  
B. No, base64 is encoding rather than encryption  
C. Yes, they are always hashed  
D. No, Secrets cannot contain strings

✅ **B. No, base64 is encoding rather than encryption**  
💡 base64 is encoding: anyone can reverse it with base64 -d, so it is not encryption.

**Q10. Which Kubernetes resource commonly holds a TLS certificate and key for Ingress?**

A. ConfigMap  
B. Deployment  
C. Secret  
D. ServiceAccount only

✅ **C. Secret**  
💡 A Secret of type kubernetes.io/tls holds tls.crt and tls.key; the Ingress names it in tls.secretName.


<a id="session-13"></a>
## Session 13 · Storage, HPA & Probes

**Q1. What is a PersistentVolumeClaim (PVC)?**

A. A request for persistent storage  
B. A network firewall  
C. A container image  
D. A Git tag

✅ **A. A request for persistent storage**  
💡 A PVC (PersistentVolumeClaim) is a Pod's request for storage: a size and an access mode.

**Q2. What is a PersistentVolume (PV)?**

A. An Ingress controller  
B. A cluster storage resource  
C. A Deployment revision  
D. A DNS zone

✅ **B. A cluster storage resource**  
💡 A PV (PersistentVolume) is an actual piece of storage in the cluster; a PVC binds to one.

**Q3. What does a StorageClass help define?**

A. Which Git branch is default  
B. Which HTTP path is matched  
C. Which Dockerfile is used  
D. How dynamic storage provisioning occurs

✅ **D. How dynamic storage provisioning occurs**  
💡 A StorageClass describes how to create storage automatically (dynamic provisioning) when a PVC asks for it.

**Q4. What does HPA stand for?**

A. HTTP Proxy Adapter  
B. Helm Package Archive  
C. Horizontal Pod Autoscaler  
D. Host Port Allocator

✅ **C. Horizontal Pod Autoscaler**  
💡 HPA = Horizontal Pod Autoscaler ('horizontal' = more or fewer Pods, not bigger Pods).

**Q5. What does an HPA commonly adjust?**

A. Container image contents  
B. Number of Pod replicas  
C. Git history  
D. Node operating system

✅ **B. Number of Pod replicas**  
💡 An HPA changes the number of Pod replicas based on metrics such as CPU use.

**Q6. What does a readiness probe determine?**

A. Whether a container image exists  
B. Whether a container should receive traffic  
C. Whether Git is clean  
D. Whether a node is a control plane

✅ **B. Whether a container should receive traffic**  
💡 Readiness decides whether a Pod gets traffic; when it fails, the Pod is removed from the Service's endpoints but not restarted.

**Q7. What does a liveness probe help detect?**

A. Whether a Service has a public IP  
B. Whether a volume is encrypted  
C. Whether Terraform is initialized  
D. When a container should be restarted

✅ **D. When a container should be restarted**  
💡 Liveness detects a stuck container; when it fails, the kubelet restarts the container.

**Q8. What does a startup probe help with?**

A. Allowing slow-starting apps time to initialize  
B. Provisioning a cloud account  
C. Creating a Docker network  
D. Checking Git credentials

✅ **A. Allowing slow-starting apps time to initialize**  
💡 A startup probe gives slow-starting apps time to start; liveness and readiness wait until it passes.

**Q9. What can happen if a Pod's PVC remains unbound?**

A. The image is rebuilt automatically  
B. The Pod may remain Pending  
C. The Pod is always Running  
D. The Deployment is deleted

✅ **B. The Pod may remain Pending**  
💡 A Pod that needs a PVC which can't be bound cannot be scheduled, so it stays Pending.

**Q10. Which command lists PVCs?**

A. kubectl get hpa-only  
B. kubectl get routes  
C. kubectl get mounts  
D. kubectl get pvc

✅ **D. kubectl get pvc**  
💡 kubectl get pvc lists claims and their status (Bound or Pending).


<a id="session-14"></a>
## Session 14 · Kubernetes troubleshooting

**Q1. Which command gives a quick overview of Pod status?**

A. kubectl get pods  
B. kubectl publish  
C. kubectl init  
D. kubectl build pods

✅ **A. kubectl get pods**  
💡 kubectl get pods gives the quick overview: READY, STATUS, RESTARTS and AGE.

**Q2. Which command displays detailed Pod events and conditions?**

A. kubectl top namespace  
B. kubectl apply -k  
C. kubectl delete pod  
D. kubectl describe pod

✅ **D. kubectl describe pod**  
💡 kubectl describe pod shows the details: conditions, Last State, exit codes and, at the bottom, the Events.

**Q3. Which command reads a container's logs?**

A. kubectl version  
B. kubectl create  
C. kubectl label  
D. kubectl logs

✅ **D. kubectl logs**  
💡 kubectl logs reads what a container printed (add --previous for the copy that crashed).

**Q4. Which command runs a command inside a running container?**

A. kubectl exec  
B. kubectl config view  
C. kubectl explain  
D. kubectl diff

✅ **A. kubectl exec**  
💡 kubectl exec runs a command inside a running container (kubectl exec -it <pod> -- sh).

**Q5. What often causes ImagePullBackOff?**

A. An oversized log file only  
B. A healthy application  
C. A missing readiness probe only  
D. An image cannot be downloaded

✅ **D. An image cannot be downloaded**  
💡 ImagePullBackOff means the image can't be downloaded: wrong name or tag, or a private registry without a login.

**Q6. What often causes CrashLoopBackOff?**

A. A stable Service IP  
B. A container repeatedly exits or crashes  
C. A successful rollout  
D. A valid PVC

✅ **B. A container repeatedly exits or crashes**  
💡 CrashLoopBackOff means the container keeps exiting and Kubernetes keeps restarting it, waiting longer each time.

**Q7. What is a common reason for a Pod to remain Pending?**

A. Successful readiness probe  
B. Correct image tag  
C. Healthy DNS  
D. Insufficient schedulable resources

✅ **D. Insufficient schedulable resources**  
💡 Pending usually means no node has enough unreserved CPU/memory for the Pod's requests (or a selector, taint or PVC blocks it).

**Q8. Where can you see recent scheduling or pull errors?**

A. Kubernetes Events  
B. Git tags  
C. Docker Hub stars  
D. Helm templates only

✅ **A. Kubernetes Events**  
💡 Events (bottom of describe, or kubectl get events) record scheduling failures and image-pull errors.

**Q9. What should you check if a Service has no endpoints?**

A. Only the Git commit message  
B. Whether its selector matches ready Pods  
C. Only the cluster timezone  
D. Only Docker build cache

✅ **B. Whether its selector matches ready Pods**  
💡 No endpoints → check that the Service's selector matches the Pods' labels and that those Pods are Ready.

**Q10. What is the final step after fixing an incident?**

A. Delete all logs  
B. Disable probes  
C. Verify the workload behaves correctly  
D. Skip testing

✅ **C. Verify the workload behaves correctly**  
💡 After any fix, verify that the workload really works: Running and Ready, endpoints present, a test request succeeds.


<a id="session-15"></a>
## Session 15 · Helm

**Q1. What is Helm?**

A. A package manager for Kubernetes  
B. A container runtime  
C. A network protocol  
D. A cloud hypervisor

✅ **A. A package manager for Kubernetes**  
💡 Helm is a package manager for Kubernetes, like apt or pip but for apps on a cluster.

**Q2. What is a Helm chart?**

A. A persistent disk  
B. A running Pod  
C. A Git branch  
D. A package of Kubernetes templates and metadata

✅ **D. A package of Kubernetes templates and metadata**  
💡 A chart is a package: Kubernetes YAML templates plus default values plus metadata.

**Q3. What is Chart.yaml used for?**

A. Pod runtime logs  
B. Ingress certificates  
C. Chart metadata such as name and version  
D. Terraform state

✅ **C. Chart metadata such as name and version**  
💡 Chart.yaml holds metadata: name, version (of the chart), appVersion (of the app), apiVersion v2.

**Q4. What is values.yaml used for?**

A. Default configurable values for templates  
B. Git merge history  
C. Container registry authentication only  
D. Linux process IDs

✅ **A. Default configurable values for templates**  
💡 values.yaml holds the default values that fill the templates' blanks; override them with -f or --set.

**Q5. Which command creates a starter chart?**

A. helm pod create  
B. helm create  
C. helm chart start  
D. helm init cluster

✅ **B. helm create**  
💡 helm create <name> scaffolds a starter chart folder.

**Q6. Which command installs a chart release?**

A. helm install  
B. helm push pod  
C. helm apply  
D. helm deploy node

✅ **A. helm install**  
💡 helm install <release> <chart> installs a chart as a named release.

**Q7. Which command updates an installed release?**

A. helm restart node  
B. helm rerun  
C. helm upgrade  
D. helm patch image

✅ **C. helm upgrade**  
💡 helm upgrade changes an installed release (new values or a new chart) and records a new revision.

**Q8. Which command returns to an earlier release revision?**

A. helm revert image  
B. helm reset cluster  
C. helm rollback  
D. helm undo commit

✅ **C. helm rollback**  
💡 helm rollback <release> <revision> returns to an earlier revision (recorded as a new revision number).

**Q9. Which flag can override a value on the command line?**

A. --copy  
B. --set  
C. --port-forward  
D. --replace-node

✅ **B. --set**  
💡 --set key=value overrides a value on the command line; it beats values.yaml and -f files.

**Q10. Which command renders templates without installing?**

A. helm template  
B. helm render-pod  
C. helm build-image  
D. helm dryrun-cluster

✅ **A. helm template**  
💡 helm template renders the final YAML on your machine without installing anything.


<a id="session-16"></a>
## Session 16 · CI/CD & GitHub Actions

**Q1. Where are GitHub Actions workflow files normally stored?**

A. .github/workflows/  
B. .kube/actions/  
C. .git/actions/  
D. .docker/workflows/

✅ **A. .github/workflows/**  
💡 Workflow files must live in .github/workflows/ (exact folder name); anywhere else they are ignored.

**Q2. Which YAML key specifies workflow triggers?**

A. jobs  
B. env  
C. on  
D. steps

✅ **C. on**  
💡 on: lists the triggers (push, pull_request, schedule, workflow_dispatch).

**Q3. What is a job in GitHub Actions?**

A. A Docker volume  
B. A Kubernetes Service  
C. A group of steps run on a runner  
D. A Git branch

✅ **C. A group of steps run on a runner**  
💡 A job is a group of steps that run on one runner; jobs run in parallel unless needs: orders them.

**Q4. What does runs-on specify?**

A. Runner environment for a job  
B. Git default branch  
C. Kubernetes namespace  
D. Target Docker registry

✅ **A. Runner environment for a job**  
💡 runs-on chooses the runner, the machine the job runs on (e.g. ubuntu-latest).

**Q5. Which action commonly checks out repository code?**

A. actions/checkout  
B. actions/setup-dockerhub  
C. github/create-branch  
D. runner/load-repo

✅ **A. actions/checkout**  
💡 actions/checkout copies the repository's code onto the runner; it's almost always step 1.

**Q6. What does a step's run field do?**

A. Executes shell commands  
B. Defines a Docker image digest  
C. Schedules a Kubernetes node  
D. Creates a cloud VPC automatically

✅ **A. Executes shell commands**  
💡 run: executes shell commands on the runner.

**Q7. What does a step's uses field do?**

A. Creates a new repository  
B. Writes Terraform state  
C. Invokes a reusable action  
D. Starts an SSH daemon

✅ **C. Invokes a reusable action**  
💡 uses: runs a ready-made reusable action, written owner/repo@version.

**Q8. How should sensitive tokens be supplied to workflows?**

A. GitHub Actions secrets  
B. Committed to README  
C. Hardcoded in YAML  
D. Printed to logs

✅ **A. GitHub Actions secrets**  
💡 Store tokens as GitHub Actions secrets and read them as ${{ secrets.NAME }}; they appear as *** in logs.

**Q9. What does needs do between jobs?**

A. Set Git author  
B. Select a Dockerfile stage  
C. Define container port mapping  
D. Declare job dependencies

✅ **D. Declare job dependencies**  
💡 needs: makes a job wait for (depend on) other jobs; if they fail, it is skipped.

**Q10. Which trigger runs a workflow on code pushes?**

A. push  
B. docker-build  
C. commit-only  
D. upload

✅ **A. push**  
💡 on: push runs the workflow whenever commits are pushed.


<a id="session-17"></a>
## Session 17 · Complete CI/CD & DevSecOps

**Q1. What is DevSecOps?**

A. Avoiding dependency updates  
B. Replacing CI with manual reviews  
C. Running security only after release  
D. Integrating security throughout development and delivery

✅ **D. Integrating security throughout development and delivery**  
💡 DevSecOps builds security into every stage of development and delivery, not just a check at the end.

**Q2. What does SAST analyze?**

A. Source code or binaries without executing the application  
B. Only production traffic  
C. Only DNS responses  
D. Only Kubernetes events

✅ **A. Source code or binaries without executing the application**  
💡 SAST (Static Application Security Testing) analyses source code without running it (white-box).

**Q3. What does DAST test?**

A. Only Git commit messages  
B. Only Docker layers  
C. A running application from the outside  
D. Only static source code

✅ **C. A running application from the outside**  
💡 DAST (Dynamic Application Security Testing) attacks the running app from the outside (black-box).

**Q4. What does dependency scanning identify?**

A. Known vulnerabilities in third-party packages  
B. Unused shell aliases  
C. Missing Git branches  
D. Slow DNS queries

✅ **A. Known vulnerabilities in third-party packages**  
💡 Dependency scanning (SCA, Software Composition Analysis) finds known vulnerabilities (CVEs) in third-party packages.

**Q5. What does container image scanning inspect?**

A. Only Kubernetes Service names  
B. Image packages and known vulnerabilities  
C. Only developer laptops  
D. Only branch protections

✅ **B. Image packages and known vulnerabilities**  
💡 Image scanning (e.g. Trivy) checks the image's OS packages and libraries for known vulnerabilities.

**Q6. What is the principle of least privilege?**

A. Disable authentication  
B. Grant only permissions required  
C. Share one token everywhere  
D. Grant administrator rights to all

✅ **B. Grant only permissions required**  
💡 Least privilege: give each user or program only the permissions it needs, nothing more.

**Q7. Why should secrets not be committed to Git?**

A. They may be exposed through repository history  
B. They break YAML parsing  
C. Git cannot store text  
D. They prevent all merges

✅ **A. They may be exposed through repository history**  
💡 Git keeps every commit forever, so a committed secret stays in the history even after deletion; it must be revoked.

**Q8. What is an SBOM?**

A. Software Bill of Materials listing components  
B. Secure Backup Operating Mode  
C. Source Branch Ownership Matrix  
D. Server Build Output Map

✅ **A. Software Bill of Materials listing components**  
💡 SBOM = Software Bill of Materials: a list of every component, and its version, inside the software.

**Q9. What is a common purpose of a security gate in CI?**

A. Disable audit logs  
B. Block promotion when policy checks fail  
C. Allow all vulnerabilities  
D. Skip all unit tests

✅ **B. Block promotion when policy checks fail**  
💡 A security gate stops the pipeline (blocks promotion) when a check fails, e.g. a scanner exiting with code 1.

**Q10. What does shift-left security mean?**

A. Run security checks earlier in development  
B. Move production servers physically  
C. Only scan after deployment  
D. Remove review from pipelines

✅ **A. Run security checks earlier in development**  
💡 Shift-left = run security checks early (on commit or PR), where problems are cheapest to fix.


<a id="session-18"></a>
## Session 18 · Terraform & IaC

**Q1. What is Terraform used for?**

A. Writing application unit tests  
B. Serving static HTML only  
C. Monitoring CPU usage only  
D. Provisioning and managing infrastructure as code

✅ **D. Provisioning and managing infrastructure as code**  
💡 Terraform provisions and manages infrastructure from code files (IaC, Infrastructure as Code).

**Q2. Which command initializes providers and backend?**

A. terraform output  
B. terraform destroy  
C. terraform apply  
D. terraform init

✅ **D. terraform init**  
💡 terraform init downloads the providers and sets up the backend (where state is kept); always run it first.

**Q3. Which command previews infrastructure changes?**

A. terraform show-only  
B. terraform plan  
C. terraform state rm  
D. terraform fmt

✅ **B. terraform plan**  
💡 terraform plan previews what will be created, changed or destroyed, without changing anything.

**Q4. Which command applies the planned changes?**

A. terraform fmt  
B. terraform validate  
C. terraform apply  
D. terraform graph

✅ **C. terraform apply**  
💡 terraform apply carries out the planned changes (it asks you to type yes).

**Q5. What does Terraform state track?**

A. Only shell command history  
B. Only application logs  
C. Mappings between configuration and managed resources  
D. Only Docker images

✅ **C. Mappings between configuration and managed resources**  
💡 The state file (terraform.tfstate) maps each resource in your code to the real object it created.

**Q6. What is a Terraform provider?**

A. A Kubernetes Pod  
B. Plugin for interacting with an API or platform  
C. A Linux package manager  
D. A Git commit

✅ **B. Plugin for interacting with an API or platform**  
💡 A provider is a plugin that talks to one platform's API (aws, azurerm, google, local…).

**Q7. What is a Terraform module?**

A. Reusable group of Terraform configuration  
B. A DNS record type  
C. A container image  
D. An HTTP load balancer

✅ **A. Reusable group of Terraform configuration**  
💡 A module is a reusable group of Terraform configuration: a folder of .tf files that you call from other code.

**Q8. Which command checks Terraform configuration syntax and consistency?**

A. terraform destroy  
B. terraform output  
C. terraform import  
D. terraform validate

✅ **D. terraform validate**  
💡 terraform validate checks syntax and internal consistency, without contacting the cloud.

**Q9. What is a variable in Terraform?**

A. An automatically encrypted password  
B. An input used to parameterize configuration  
C. A Helm release  
D. A running cloud VM

✅ **B. An input used to parameterize configuration**  
💡 A variable is an input (variable "region" {…}) that parameterises the configuration.

**Q10. Why use a remote backend with locking?**

A. Share state safely and reduce concurrent update conflicts  
B. Run Pods without nodes  
C. Replace all version control  
D. Eliminate the need for credentials

✅ **A. Share state safely and reduce concurrent update conflicts**  
💡 A remote backend (e.g. S3) lets the team share one state file; locking stops two applies running at the same time.


<a id="session-19"></a>
## Session 19 · Cloud & Terraform in action

**Q1. What does IaaS provide?**

A. Virtualized infrastructure such as compute and networks  
B. Only email templates  
C. Only finished end-user applications  
D. Only source code hosting

✅ **A. Virtualized infrastructure such as compute and networks**  
💡 IaaS (Infrastructure as a Service) gives you virtual machines, networks and storage; you manage the OS and everything above it (e.g. AWS EC2).

**Q2. What does PaaS primarily provide?**

A. Raw CPU chips only  
B. Only Git branches  
C. Managed platform for deploying applications  
D. Only DNS records

✅ **C. Managed platform for deploying applications**  
💡 PaaS (Platform as a Service) gives a managed platform; you only deploy your code (e.g. Heroku, Elastic Beanstalk).

**Q3. What does SaaS typically deliver?**

A. An unmanaged hypervisor  
B. A local kernel module  
C. Ready-to-use software over a network  
D. A raw storage controller

✅ **C. Ready-to-use software over a network**  
💡 SaaS (Software as a Service) is ready-to-use software over the internet (e.g. Gmail).

**Q4. What is a cloud region?**

A. A geographic area containing cloud infrastructure  
B. A container tag  
C. A single application process  
D. A Git repository

✅ **A. A geographic area containing cloud infrastructure**  
💡 A region is a geographic area (e.g. ap-south-1, Mumbai) that contains several data centres.

**Q5. What is an availability zone?**

A. An isolated infrastructure location within a region  
B. A Kubernetes label only  
C. A Terraform variable  
D. A GitHub runner type

✅ **A. An isolated infrastructure location within a region**  
💡 An AZ (Availability Zone) is an isolated data-centre location inside a region; using two or more gives high availability.

**Q6. What is a VPC?**

A. A logically isolated virtual network  
B. A container image  
C. A managed source code editor  
D. A version control protocol

✅ **A. A logically isolated virtual network**  
💡 A VPC (Virtual Private Cloud) is your own isolated virtual network in the cloud (Azure calls it a VNet).

**Q7. What is a subnet?**

A. A Git branch  
B. A CI pipeline stage  
C. An IP address range within a network  
D. A Docker build layer

✅ **C. An IP address range within a network**  
💡 A subnet is a smaller range of IP addresses (e.g. 10.0.1.0/24) inside the VPC, placed in one AZ.

**Q8. What does a route table define?**

A. Where network traffic should be forwarded  
B. Terraform module version  
C. Container restart count  
D. Application test coverage

✅ **A. Where network traffic should be forwarded**  
💡 A route table says where traffic is sent; a route 0.0.0.0/0 → internet gateway makes a subnet public.

**Q9. What does a security group commonly control?**

A. CPU instruction set  
B. YAML indentation  
C. Git merge policy  
D. Allowed inbound and outbound network traffic

✅ **D. Allowed inbound and outbound network traffic**  
💡 A security group is a stateful virtual firewall listing the inbound and outbound traffic allowed for an instance.

**Q10. Why combine Terraform with cloud services?**

A. Reproducible, versioned infrastructure provisioning  
B. To compile frontend code  
C. To remove the need for networks  
D. To avoid reviewing changes

✅ **A. Reproducible, versioned infrastructure provisioning**  
💡 Terraform + cloud = reproducible, version-controlled infrastructure that can be recreated identically.


<a id="session-20"></a>
## Session 20 · Monitoring, Observability & GitOps

**Q1. What is a metric?**

A. A YAML parser  
B. A complete container image  
C. A Git branch  
D. A numeric measurement over time

✅ **D. A numeric measurement over time**  
💡 A metric is a number measured over time (CPU %, requests per second).

**Q2. What is a log?**

A. A cloud region  
B. A persistent volume claim  
C. A DNS subnet  
D. A timestamped record of events

✅ **D. A timestamped record of events**  
💡 A log is a timestamped record of an event: one line per thing that happened.

**Q3. What is a distributed trace?**

A. A view of a request across services  
B. A Docker image tag  
C. A Git commit hash only  
D. A shell variable

✅ **A. A view of a request across services**  
💡 A distributed trace follows one request across many services and shows the time spent in each.

**Q4. What is Prometheus commonly used for?**

A. Building Docker images  
B. Provisioning VPCs  
C. Collecting and querying time-series metrics  
D. Hosting Git repositories

✅ **C. Collecting and querying time-series metrics**  
💡 Prometheus scrapes (pulls) metrics from targets and stores them as time series; you query them with PromQL.

**Q5. What is Grafana commonly used for?**

A. Visualizing metrics and creating dashboards  
B. Building Helm charts  
C. Scanning source code  
D. Scheduling Kubernetes Pods

✅ **A. Visualizing metrics and creating dashboards**  
💡 Grafana visualises data in dashboards and panels, using sources like Prometheus; it stores no metrics itself.

**Q6. What does observability help engineers do?**

A. Avoid all incident response  
B. Infer internal system behavior from telemetry  
C. Disable logs  
D. Replace application testing

✅ **B. Infer internal system behavior from telemetry**  
💡 Observability = being able to understand what's happening inside a system from its telemetry (metrics, logs, traces).

**Q7. What is GitOps?**

A. Replacing Kubernetes with Git  
B. Storing passwords in plaintext  
C. Running all changes manually  
D. Managing desired system state through version-controlled Git

✅ **D. Managing desired system state through version-controlled Git**  
💡 GitOps keeps the desired state in Git, and an agent (e.g. Argo CD) makes the cluster match it.

**Q8. In GitOps, what is the source of truth?**

A. An untracked server file  
B. A developer's local terminal  
C. A random running Pod  
D. The desired configuration stored in Git

✅ **D. The desired configuration stored in Git**  
💡 In GitOps, the desired configuration stored in Git is the single source of truth.

**Q9. What is Argo CD commonly used for?**

A. Reconciling Kubernetes deployments with Git  
B. Creating Docker base images  
C. Resolving DNS on a laptop  
D. Compiling Java source code

✅ **A. Reconciling Kubernetes deployments with Git**  
💡 Argo CD reconciles (syncs) the cluster with the manifests stored in Git.

**Q10. What does configuration drift mean?**

A. Actual state differs from declared desired state  
B. All Pods have identical images  
C. Every Git branch is merged  
D. Metrics are stored in a database

✅ **A. Actual state differs from declared desired state**  
💡 Drift = the real state differs from the declared desired state, e.g. after a manual kubectl change.

