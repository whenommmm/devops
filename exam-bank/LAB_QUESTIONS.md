# 🧪 Lab questions · all 57, in session order

**How to use:** read the steps, write your commands in the box, tick ☑ when done, then compare with [LAB_ANSWERS.md](LAB_ANSWERS.md) (same lab number).

> 💻 On your Mac: run `echo "setopt interactivecomments" >> ~/.zshrc && source ~/.zshrc` once, so `# comments` don't break pasted commands. Port 8080 is taken on your Mac, so use 9080.

## 📋 Overview

| Session | Lab 1 | Lab 2 | Lab 3 |
|---|---|---|---|
| **1** DevOps intro & roadmap | [Draw a CI/CD pipeline](#lab-1-1) | [Monolith vs microservices for an online shop](#lab-1-2) | [A DevOps roadmap for a team that deploys by hand](#lab-1-3) |
| **2** Linux | [Files and folders: create, list, copy, move, search, delete](#lab-2-1) | [Users, groups and folder permissions](#lab-2-2) | [Check a service, read its logs, fix a failure](#lab-2-3) |
| **3** Shell scripting | [Script: count files and subfolders in a directory](#lab-3-1) | [Script: is this file readable, writable, executable?](#lab-3-2) | [Script: a reusable system-health report](#lab-3-3) |
| **4** Networking | [Your IP address, subnet, gateway and open ports](#lab-4-1) | [Troubleshoot a web service, layer by layer](#lab-4-2) | [DNS lookup, then check HTTP and HTTPS](#lab-4-3) |
| **5** Git & GitHub | [New repository → 3 commits → GitHub](#lab-5-1) | [Feature branch → Pull Request → merge](#lab-5-2) | [Create and resolve a merge conflict](#lab-5-3) |
| **6–7** Docker & Dockerfile | [Run nginx and publish a port](#lab-67-1) | [Your own image: Dockerfile for an HTML page](#lab-67-2) | [.dockerignore + an environment variable](#lab-67-3) |
| **8** Docker networking, volumes & Compose | [Two containers talking by name](#lab-8-1) | [Database data that survives the container](#lab-8-2) | [Docker Compose: web + database](#lab-8-3) |
| **9** Kubernetes fundamentals | [Explore a Kubernetes cluster](#lab-9-1) | [Deploy a web app and inspect it](#lab-9-2) | [Draw the cluster architecture](#lab-9-3) |
| **10** Pods, ReplicaSets & Deployments | [An nginx Pod: status, labels, IP, logs](#lab-10-1) | [Scale a Deployment: 2 → 5 → 2](#lab-10-2) | [Rolling update to v2, then roll back](#lab-10-3) |
| **11** Kubernetes Services | [ClusterIP Service in front of several Pods](#lab-11-1) | [NodePort Service: reach the app from outside](#lab-11-2) | [Service port ≠ targetPort](#lab-11-3) |
| **12** Ingress, ConfigMaps & Secrets | [Ingress: route by host name and URL path](#lab-12-1) | [ConfigMap → environment variables](#lab-12-2) | [Secret → used by a Pod (no hard-coding)](#lab-12-3) |
| **13** Storage, HPA & Probes | [PVC: data that survives a new Pod](#lab-13-1) | [Readiness and liveness probes in action](#lab-13-2) | [HPA: scale on CPU under load](#lab-13-3) |
| **14** Kubernetes troubleshooting | [Fix an ImagePullBackOff](#lab-14-1) | [Fix a CrashLoopBackOff](#lab-14-2) | [Fix a Service with the wrong selector](#lab-14-3) |
| **15** Helm | [Create a Helm chart and explain its parts](#lab-15-1) | [Install the chart with your own values](#lab-15-2) | [Upgrade, history, rollback](#lab-15-3) |
| **16** CI/CD & GitHub Actions | [A workflow that builds on every push](#lab-16-1) | [Add tests: red on failure, green when fixed](#lab-16-2) | [Upload a build artifact](#lab-16-3) |
| **17** Complete CI/CD & DevSecOps | [Build and push an image using secrets](#lab-17-1) | [Security scans that block the pipeline](#lab-17-2) | [Image scan with a severity gate](#lab-17-3) |
| **18** Terraform & IaC | [First Terraform config: fmt → init → validate → plan](#lab-18-1) | [Variables, a .tfvars file, and outputs](#lab-18-2) | [Change a resource, read the plan, understand state](#lab-18-3) |
| **19** Cloud & Terraform in action | [Design a cloud network (diagram)](#lab-19-1) | [Build the network + one small server with Terraform](#lab-19-2) | [Security group: only the app port is open](#lab-19-3) |
| **20** Monitoring, Observability & GitOps | [Prometheus scrapes your app](#lab-20-1) | [Grafana dashboard: rate, errors, latency](#lab-20-2) | [GitOps with Argo CD: change replicas in Git](#lab-20-3) |

*Short on time? Do the command labs first (Sessions 6–18) and leave the diagram/essay ones (1.1–1.3, 9.3, 19.1) for last.*

---

## Session 1 · DevOps intro & roadmap

<a id="lab-1-1"></a>
### 🧪 Lab 1.1 · Draw a CI/CD pipeline  ·  ⏱ ~10 min

🎯 **Goal:** Show how code travels from a commit to a running, monitored app.

**Steps**
1. Pick a simple app (e.g. an online shop) and start from **a developer pushing a commit**.
2. Draw the stages as boxes with arrows: **Build → Test → Package → Deploy → Monitor**.
3. Under each box, write one tool (e.g. GitHub Actions, pytest, Docker, Kubernetes, Prometheus).
4. Add what happens on failure (the pipeline stops, the developer is told) and a **feedback arrow** from Monitor back to the developer.

✅ **Done when:** All 5 stages are labelled in the right order, each has a tool, and there's a failure path and a feedback loop.

💡 **Hint:** Commit → CI (build + test) → image in a registry → deploy → dashboards/alerts.

📄 *Exam wording:* "Given a sample application scenario, draw a simple CI/CD workflow from code commit to deployment. Label the build, test, package, deploy, and monitoring stages."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-1-2"></a>
### 🧪 Lab 1.2 · Monolith vs microservices for an online shop  ·  ⏱ ~10 min

🎯 **Goal:** Compare the two designs and judge microservices fairly.

**Steps**
1. Describe the **monolith**: one app containing catalog, cart, payments and users, with one database.
2. Describe the **microservices** version: each of those is its own small service with its own database, talking over the network.
3. Make a small comparison (deploying, scaling, what happens when one part fails).
4. Write exactly **2 operational benefits** and **2 challenges** of microservices, each with a one-line reason.

✅ **Done when:** Both designs are described (a sketch helps), and you have 2 benefits + 2 challenges with reasons.

💡 **Hint:** Benefits: deploy/scale one service alone, a failure stays contained. Challenges: network calls and debugging across services, more things to run and monitor.

📄 *Exam wording:* "Compare a monolithic application and a microservices application for a provided online shopping use case. Identify two operational benefits and two challenges of microservices."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-1-3"></a>
### 🧪 Lab 1.3 · A DevOps roadmap for a team that deploys by hand  ·  ⏱ ~10 min

🎯 **Goal:** Plan the order in which a manual team should adopt DevOps practices.

**Steps**
1. Write the starting point: code is copied to servers by hand, with no tests.
2. Put these 5 steps in priority order: **version control, automated tests, containerization, CI/CD, monitoring**.
3. Give each step a one-line reason (why it comes at that point).
4. Add how you'd know it's working (e.g. deploys per week, failed deploys).

✅ **Done when:** Five steps in a sensible order, each with a reason, fitting on one page.

💡 **Hint:** Each step depends on the one before it: you can't automate tests without Git, or do CI/CD without tests and containers.

📄 *Exam wording:* "Create a one-page DevOps roadmap for a small team that currently deploys manually. Prioritize version control, automated tests, containerization, CI/CD, and monitoring, with a short reason for each."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 2 · Linux

<a id="lab-2-1"></a>
### 🧪 Lab 2.1 · Files and folders: create, list, copy, move, search, delete  ·  ⏱ ~8 min

🎯 **Goal:** Practise every basic file command on a small project.

**Steps**
1. Create a folder `project` with three subfolders: `app`, `logs`, `config`.
2. Put a few sample files in them (one log file should contain the word `ERROR`).
3. **List** everything, including hidden files and details; then list every subfolder.
4. **Copy** a file, then copy a whole folder.
5. **Move** a file into another folder.
6. **Search** for the word `ERROR` inside files, then find all `.conf` files by name.
7. **Delete** one file and one folder.

✅ **Done when:** `ls -R` shows your tree, the search prints the `ERROR` line with its file name, and the deleted items are gone.

💡 **Hint:** `mkdir -p`, `echo … >`, `touch`, `ls -la`, `ls -R`, `cp`, `cp -r`, `mv`, `grep -rn`, `find -name`, `rm`, `rm -r`.

📄 *Exam wording:* "Create a project directory with app, logs, and config subdirectories. Create sample files and demonstrate listing, copying, moving, searching, and deleting files."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-2-2"></a>
### 🧪 Lab 2.2 · Users, groups and folder permissions  ·  ⏱ ~10 min

🎯 **Goal:** Create a user and a team group, and give the team a private folder.

**Steps**
1. Create a group called `devops`.
2. Create a user called `examuser` (the password must be asked for, never typed into the command).
3. Add `examuser` to `devops`, and check it worked.
4. Create a project folder; make **examuser** its owner and **devops** its group.
5. Set permissions: owner and group get full access, everyone else gets nothing.
6. Prove examuser can create a file inside.

✅ **Done when:** `id examuser` lists `devops`, `ls -ld` shows `drwxrwx---  examuser  devops`, and the test file is created.

💡 **Hint:** macOS: `dseditgroup`, `sysadminctl`, `chown`, `chmod 770`, `sudo -u`. Linux: `groupadd`, `useradd -m`, `usermod -aG`.

📄 *Exam wording:* "Create a user named examuser and a group named devops. Add the user to the group and configure suitable ownership and permissions on a project directory."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-2-3"></a>
### 🧪 Lab 2.3 · Check a service, read its logs, fix a failure  ·  ⏱ ~12 min

🎯 **Goal:** Diagnose a broken background service the way an admin would.

**Steps**
1. Start a service (e.g. nginx) and check its **status**.
2. **Simulate a problem**: add one invalid line to its config, then restart it. It should fail.
3. Check the status again: what does it say now?
4. Read the service's **logs** and its own config test to find the exact error.
5. Fix the cause, restart, and prove it works (port listening + a test request).

✅ **Done when:** You can write the symptom, the evidence (the log line), the cause and the fix, and the final status is running.

💡 **Hint:** macOS: `brew services list/restart`, `nginx -t`, `tail error.log`, `lsof`. Linux: `systemctl status`, `journalctl -u`, `ss -tulpn`.

📄 *Exam wording:* "Check the status of a Linux service, inspect its logs, and document the commands used to diagnose a simulated service problem."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 3 · Shell scripting

<a id="lab-3-1"></a>
### 🧪 Lab 3.1 · Script: count files and subfolders in a directory  ·  ⏱ ~10 min

🎯 **Goal:** Write a script that takes a folder path and reports what's inside.

**Steps**
1. Start with a shebang and read the folder path from the **first argument**.
2. If no argument is given, print a usage message and exit with code 1.
3. If the path isn't a directory, print an error and exit with code 2.
4. Otherwise print the number of **files** and the number of **subdirectories**.

✅ **Done when:** `./count.sh /some/dir` prints two numbers; running it with no argument or a bad path prints a clear message, and `echo $?` shows 1 or 2.

💡 **Hint:** `$#`, `$1`, `[ -d "$1" ]`, `find "$1" -maxdepth 1 -type f | wc -l`, `exit`.

📄 *Exam wording:* "Write a shell script that accepts a directory path as an argument and displays the number of files and subdirectories. Handle a missing or invalid argument."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-3-2"></a>
### 🧪 Lab 3.2 · Script: is this file readable, writable, executable?  ·  ⏱ ~8 min

🎯 **Goal:** Check a file's permissions and report each one clearly.

**Steps**
1. Take the file name as the first argument (usage message if it's missing).
2. If the file doesn't exist, say so and exit with an error code.
3. Otherwise print one line each: readable YES/NO, writable YES/NO, executable YES/NO.

✅ **Done when:** Running it on a file, then again after `chmod -w` or `chmod +x`, shows the answers change.

💡 **Hint:** `[ -e file ]`, `[ -r file ]`, `[ -w file ]`, `[ -x file ]`, `if … then … else … fi`.

📄 *Exam wording:* "Write a script that checks whether a supplied file exists and is readable, writable, and executable. Print a clear result for each check."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-3-3"></a>
### 🧪 Lab 3.3 · Script: a reusable system-health report  ·  ⏱ ~15 min

🎯 **Goal:** Combine variables, conditions, loops and functions into one useful script.

**Steps**
1. Use a **variable** for a disk-usage alert threshold (e.g. 80%), changeable by an argument.
2. Write a **function** for each check: disk usage, memory, uptime.
3. Use a **loop** to go through each disk and an **if** to warn when one is over the threshold.
4. Print a final HEALTHY/WARNING line and exit with 0 or a non-zero code.

✅ **Done when:** It prints three sections (disk, memory, uptime) and a result line, and `echo $?` matches the result.

💡 **Hint:** `df -h`, `uptime`, memory: macOS `top -l 1 | grep PhysMem` (Linux `free -h`), `name() { … }`, `for`, `while read`.

📄 *Exam wording:* "Write a reusable system-health script using variables, conditions, loops, and functions. Report disk usage, memory, and uptime."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 4 · Networking

<a id="lab-4-1"></a>
### 🧪 Lab 4.1 · Your IP address, subnet, gateway and open ports  ·  ⏱ ~8 min

🎯 **Goal:** Find out who your machine is on the network, and explain each result.

**Steps**
1. Show your machine's **IP address**.
2. Show the **subnet** (mask or `/prefix`) and work out the network range.
3. Show the **default route** (gateway).
4. List the **listening ports** and which programs own them.
5. Under each command, write one line saying what the output means.

✅ **Done when:** You can point to your IP, subnet, gateway and at least one listening port, and explain each in plain words.

💡 **Hint:** macOS: `ipconfig getifaddr en0`, `ifconfig en0`, `route -n get default`, `lsof -iTCP -sTCP:LISTEN -P -n`. Linux: `ip a`, `ip route`, `ss -tulpn`.

📄 *Exam wording:* "Use Linux commands to display the machine IP address, subnet information, default route, and listening ports. Include the command and a brief interpretation for each."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-4-2"></a>
### 🧪 Lab 4.2 · Troubleshoot a web service, layer by layer  ·  ⏱ ~12 min

🎯 **Goal:** Work out whether a failure is the network, a closed port, or the app itself.

**Steps**
1. Start a small test web server (e.g. Python's) on a free port.
2. Check **reachability** with `ping`.
3. Check the **port**: is something listening, and can you connect?
4. Check **HTTP** with `curl` and read the status code.
5. Simulate each failure (an unreachable IP, the server stopped, a missing page) and write which layer each one is.

✅ **Done when:** For each simulated failure you can name the layer and quote the message that proves it.

💡 **Hint:** `ping -c 4`, `lsof -nP -iTCP:<port>` / `nc -zv` (Linux `ss -tulpn`), `curl -v`, `curl -I`, status codes 200 / 404 / 5xx.

📄 *Exam wording:* "Use ping, ss, and curl to troubleshoot a web service. Determine whether a simulated failure is caused by reachability, a closed port, or an HTTP response."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-4-3"></a>
### 🧪 Lab 4.3 · DNS lookup, then check HTTP and HTTPS  ·  ⏱ ~8 min

🎯 **Goal:** Turn a name into an IP, then prove the website answers.

**Steps**
1. Resolve a domain (e.g. example.com) and note the IP address and record type.
2. Request the site over **HTTP** and over **HTTPS** and note each status code.
3. If you get a redirect (301/302), follow it and note the final status.

✅ **Done when:** You have an IP from DNS and a status code for both HTTP and HTTPS.

💡 **Hint:** `nslookup`, `dig`, `dig +short`, `curl -I`, `curl -IL` (follow redirects).

📄 *Exam wording:* "Resolve a domain using nslookup or dig, then use curl to verify HTTP/HTTPS connectivity. Record the DNS result and HTTP status."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 5 · Git & GitHub

<a id="lab-5-1"></a>
### 🧪 Lab 5.1 · New repository → 3 commits → GitHub  ·  ⏱ ~10 min

🎯 **Goal:** Put a small project under Git and publish it.

**Steps**
1. Create a folder with a tiny app file and turn it into a Git repository.
2. Add a `README.md` and commit it.
3. Make **two more meaningful commits** (e.g. add the app, then add a `.gitignore`).
4. Create an empty repository on GitHub, connect it as the remote, and push `main`.

✅ **Done when:** GitHub shows your README and at least 3 commits with clear messages.

💡 **Hint:** `git init`, `git add`, `git commit -m`, `git branch -M main`, `git remote add origin`, `git push -u origin main`.

📄 *Exam wording:* "Create a Git repository for a small application, add a README, make at least three meaningful commits, and push it to GitHub."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-5-2"></a>
### 🧪 Lab 5.2 · Feature branch → Pull Request → merge  ·  ⏱ ~10 min

🎯 **Goal:** Make a change the team way: on a branch, reviewed, then merged.

**Steps**
1. Create a feature branch and switch to it.
2. Make a change and commit it.
3. Push the branch to GitHub.
4. Open a **Pull Request** into `main` and review the diff.
5. Merge it, then update your local `main`.

✅ **Done when:** The PR shows as merged on GitHub, and `git log` on `main` contains your change.

💡 **Hint:** `git switch -c`, `git push -u origin <branch>`, `gh pr create`, `git diff main...feature`, `gh pr merge`, `git pull`.

📄 *Exam wording:* "Create a feature branch, implement a change, push the branch, and merge it into main through a Pull Request after reviewing the diff."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-5-3"></a>
### 🧪 Lab 5.3 · Create and resolve a merge conflict  ·  ⏱ ~10 min

🎯 **Goal:** Cause a conflict on purpose and resolve it correctly.

**Steps**
1. On `main`, commit a file with one line of text.
2. On a new branch, change **that same line** and commit.
3. Back on `main`, change the same line differently and commit.
4. Merge the branch: Git reports a CONFLICT.
5. Open the file, choose the final text, remove the conflict markers, then finish the merge.
6. Show the history as a graph.

✅ **Done when:** The merge is committed, the file has no `<<<<<<<` markers, and the graph shows two lines joining.

💡 **Hint:** `git switch -c`, `git merge`, `git status`, edit the file, `git add`, `git commit`, `git log --oneline --graph --all`.

📄 *Exam wording:* "Create a merge conflict by editing the same line on two branches. Resolve it, commit the resolution, and verify the final history."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 6–7 · Docker & Dockerfile

<a id="lab-67-1"></a>
### 🧪 Lab 6–7.1 · Run nginx and publish a port  ·  ⏱ ~6 min

🎯 **Goal:** Run a web server in a container and reach it from your machine.

**Steps**
1. Run an nginx container in the background, mapping **host 8080 → container 80** (on your Mac use 9080).
2. Check it's running.
3. Request the page with `curl`.
4. Read the container's logs.

✅ **Done when:** `curl` shows the "Welcome to nginx!" page, and the logs show your GET request.

💡 **Hint:** `docker run -d --name … -p HOST:CONTAINER`, `docker ps`, `curl`, `docker logs`.

📄 *Exam wording:* "Run an Nginx container with host port 8080 mapped to container port 80. Verify access using curl and inspect the container logs."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-67-2"></a>
### 🧪 Lab 6–7.2 · Your own image: Dockerfile for an HTML page  ·  ⏱ ~10 min

🎯 **Goal:** Package a web page into your own image.

**Steps**
1. Create an `index.html`.
2. Write a Dockerfile using **FROM, WORKDIR, COPY, CMD**.
3. Build it with a name and tag.
4. Run it with a port mapping and check your page appears.

✅ **Done when:** `curl` shows **your** HTML, not the default nginx page.

💡 **Hint:** FROM `nginx:alpine`, WORKDIR `/usr/share/nginx/html`, `docker build -t name:tag .`, `docker run -d -p`.

📄 *Exam wording:* "Create a Dockerfile for a simple HTML page using FROM, WORKDIR, COPY, and CMD. Build the image, run it, and verify the page."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-67-3"></a>
### 🧪 Lab 6–7.3 · .dockerignore + an environment variable  ·  ⏱ ~10 min

🎯 **Goal:** Keep junk and secrets out of the image, and pass in a setting.

**Steps**
1. Add a `.dockerignore` that excludes things like `.git`, `.env` and `*.log`.
2. Add an `ENV` instruction (e.g. an app version) to the Dockerfile.
3. Rebuild and run the image.
4. Prove the variable is set inside the container and the ignored files are missing.
5. Write one line explaining each new instruction.

✅ **Done when:** `docker exec … env` shows your variable, and the ignored files aren't in the image.

💡 **Hint:** `.dockerignore`, `ENV KEY=value`, `docker exec <c> env`, `docker run -e KEY=other` (overrides it).

📄 *Exam wording:* "Add a .dockerignore file and an environment variable to the Docker build. Rebuild and run the image, then explain the purpose of each added instruction."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 8 · Docker networking, volumes & Compose

<a id="lab-8-1"></a>
### 🧪 Lab 8.1 · Two containers talking by name  ·  ⏱ ~8 min

🎯 **Goal:** Show that containers on a custom network can find each other by name.

**Steps**
1. Create a custom (user-defined) bridge network.
2. Run two containers on it (e.g. a web server and a small client).
3. From the client, reach the web server **by its container name**.

✅ **Done when:** `ping` or `wget` from the client to the server's name works.

💡 **Hint:** `docker network create`, `docker run --network`, `docker exec … ping`, `wget -qO- http://<name>`.

📄 *Exam wording:* "Create a custom Docker network and run two containers on it. Demonstrate name-based communication between the containers."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-8-2"></a>
### 🧪 Lab 8.2 · Database data that survives the container  ·  ⏱ ~10 min

🎯 **Goal:** Prove a named volume keeps data after the container is deleted.

**Steps**
1. Run a database container (e.g. Postgres) with a **named volume** on its data folder.
2. Create a table and insert a row.
3. Delete the container completely.
4. Start a new container using the **same volume**, and read the row back.

✅ **Done when:** The row is still there after the container was removed and recreated.

💡 **Hint:** `-v name:/var/lib/postgresql/data`, `-e POSTGRES_PASSWORD`, `docker exec … psql -c`, `docker rm -f`.

📄 *Exam wording:* "Run a database container with a named volume, create sample data, remove and recreate the container, and verify that the data persists."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-8-3"></a>
### 🧪 Lab 8.3 · Docker Compose: web + database  ·  ⏱ ~12 min

🎯 **Goal:** Run a two-service app from one file.

**Steps**
1. Write a `compose.yaml` with a **web** service and a **db** service (database data in a named volume, password from an `.env` file).
2. Start the stack in the background and check both are up.
3. Prove the web container can reach the database by its service name.
4. Read the logs.
5. Shut everything down.

✅ **Done when:** Both services are Up, web reaches `db` by name, and `down` removes the containers.

💡 **Hint:** `services:`, `depends_on`, `volumes:`, `${DB_PASSWORD}`, `docker compose up -d / ps / logs / down`.

📄 *Exam wording:* "Write a compose.yaml file for a web application and database. Start the stack, verify service communication, inspect logs, and shut it down."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 9 · Kubernetes fundamentals

<a id="lab-9-1"></a>
### 🧪 Lab 9.1 · Explore a Kubernetes cluster  ·  ⏱ ~8 min

🎯 **Goal:** Connect to a cluster and describe what's in it.

**Steps**
1. Check which cluster you're connected to.
2. List the **nodes** (with extra details).
3. List the **namespaces**.
4. List the resources across all namespaces, including the system Pods.
5. Under each command, write what it shows.

✅ **Done when:** You can name your node(s), the namespaces, and the main system Pods, with one line of meaning per command.

💡 **Hint:** `kubectl config current-context`, `cluster-info`, `get nodes -o wide`, `get ns`, `get all -A`, `get pods -n kube-system`.

📄 *Exam wording:* "Connect to a Kubernetes cluster and inspect nodes, namespaces, and cluster resources using kubectl. Record what each command shows."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-9-2"></a>
### 🧪 Lab 9.2 · Deploy a web app and inspect it  ·  ⏱ ~8 min

🎯 **Goal:** Run an app and check it reaches a healthy state.

**Steps**
1. Create a Deployment for a simple web image (e.g. nginx).
2. Wait for the rollout to finish.
3. Inspect it with **get**, **describe** and **logs**.
4. Confirm the Pods are Running and Ready.

✅ **Done when:** Pods show `1/1 Running`, `describe` Events are all Normal, and the logs show the app started.

💡 **Hint:** `kubectl create deployment`, `rollout status`, `get pods -o wide`, `describe pod`, `logs`.

📄 *Exam wording:* "Deploy a simple web application and inspect it using kubectl get, describe, and logs. Confirm that the workload reaches a healthy state."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-9-3"></a>
### 🧪 Lab 9.3 · Draw the cluster architecture  ·  ⏱ ~10 min

🎯 **Goal:** Explain what each part of Kubernetes does.

**Steps**
1. Draw the **control plane** (API Server, Scheduler, Controller Manager, etcd) and at least one **worker node** (kubelet, kube-proxy, container runtime).
2. Show **kubectl** outside, talking to the API Server.
3. Write one line per component saying its job.
4. Optional: trace what happens after `kubectl apply`.

✅ **Done when:** Every component is placed correctly with a one-line role, and only the API Server talks to etcd.

💡 **Hint:** kubectl → API Server → etcd (stores state) → Scheduler (picks a node) → kubelet (starts containers).

📄 *Exam wording:* "Identify the role of the API Server, Scheduler, Controller Manager, etcd, kubelet, and kubectl in a short cluster architecture diagram."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 10 · Pods, ReplicaSets & Deployments

<a id="lab-10-1"></a>
### 🧪 Lab 10.1 · An nginx Pod: status, labels, IP, logs  ·  ⏱ ~6 min

🎯 **Goal:** Create one Pod and read everything about it.

**Steps**
1. Create a Pod running nginx, with a label.
2. Show its **status** and **IP address**.
3. Show its **labels**.
4. Read its details and its **logs**.

✅ **Done when:** You can quote the Pod's IP, its label, `Running`, and a log line.

💡 **Hint:** `kubectl run --labels`, `get pod -o wide --show-labels`, `describe pod`, `logs`.

📄 *Exam wording:* "Create a Pod running Nginx and inspect its status, labels, IP address, and logs using kubectl."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-10-2"></a>
### 🧪 Lab 10.2 · Scale a Deployment: 2 → 5 → 2  ·  ⏱ ~6 min

🎯 **Goal:** Change the number of replicas and watch Kubernetes follow.

**Steps**
1. Create a Deployment with **2** replicas.
2. Scale it to **5** and verify.
3. Scale it back to **2** and verify.

✅ **Done when:** READY shows 2/2, then 5/5, then 2/2.

💡 **Hint:** `kubectl create deployment --replicas`, `kubectl scale --replicas`, `kubectl get deploy`, `get pods`.

📄 *Exam wording:* "Create a Deployment with two replicas, scale it to five, and then scale it back to two. Verify the replica count at each step."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-10-3"></a>
### 🧪 Lab 10.3 · Rolling update to v2, then roll back  ·  ⏱ ~8 min

🎯 **Goal:** Update an app with no downtime, then undo the update.

**Steps**
1. Deploy **version 1** of an app (e.g. `nginx:1.26`).
2. Update it to **version 2** (e.g. `nginx:1.27`) and watch the rollout.
3. Check the rollout history.
4. Roll back to version 1 and confirm the image.

✅ **Done when:** The image changes to v2 and then back to v1, with the history showing both revisions.

💡 **Hint:** `kubectl set image`, `rollout status`, `rollout history`, `rollout undo`.

📄 *Exam wording:* "Deploy version 1 of an application, perform a rolling update to version 2, verify the rollout, and roll back to version 1."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 11 · Kubernetes Services

<a id="lab-11-1"></a>
### 🧪 Lab 11.1 · ClusterIP Service in front of several Pods  ·  ⏱ ~8 min

🎯 **Goal:** Give a group of Pods one stable address inside the cluster.

**Steps**
1. Create a Deployment with 3 replicas.
2. Create a **ClusterIP** Service whose selector matches the Pods' label.
3. Check the Service's **endpoints** (one per Pod).
4. Call the Service by name from a temporary Pod.

✅ **Done when:** `get endpoints` lists 3 Pod IPs, and the test Pod gets the web page from `http://<service-name>`.

💡 **Hint:** `kubectl expose deployment --port`, `get svc`, `get endpoints`, `kubectl run tmp --rm -it --image=busybox -- wget -qO- http://web`.

📄 *Exam wording:* "Deploy an application with multiple replicas and create a ClusterIP Service that selects the correct Pods. Verify its endpoints."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-11-2"></a>
### 🧪 Lab 11.2 · NodePort Service: reach the app from outside  ·  ⏱ ~8 min

🎯 **Goal:** Open the app on a port of the node.

**Steps**
1. Create a **NodePort** Service for your Deployment.
2. Find which node port (30000–32767) it was given.
3. Open the app through **NodeIP:nodePort**.

✅ **Done when:** A request to the node's IP and port returns the app.

💡 **Hint:** `kubectl expose --type=NodePort`, `get svc` (PORT column `80:3XXXX`), `minikube service <name> --url` on minikube.

📄 *Exam wording:* "Create a NodePort Service and demonstrate access to the application through the exposed node port."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-11-3"></a>
### 🧪 Lab 11.3 · Service port ≠ targetPort  ·  ⏱ ~10 min

🎯 **Goal:** Show that the Service's port and the container's port can differ.

**Steps**
1. Run an app whose container listens on one port (e.g. 8080).
2. Create a Service with **port 80** and **targetPort 8080**.
3. Check the endpoints show the Pod IP **with the target port**.
4. Call the Service on port 80 and confirm the app answers.

✅ **Done when:** Endpoints show `PodIP:8080`, and `http://<service>` (port 80) gets a reply.

💡 **Hint:** `port:` = the Service's port, `targetPort:` = where the app listens, `--port=80 --target-port=8080`.

📄 *Exam wording:* "Configure a Service whose port and targetPort differ. Verify that traffic reaches the intended container port."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 12 · Ingress, ConfigMaps & Secrets

<a id="lab-12-1"></a>
### 🧪 Lab 12.1 · Ingress: route by host name and URL path  ·  ⏱ ~12 min

🎯 **Goal:** Send different URLs to different Services through one entry point.

**Steps**
1. Make sure an **Ingress controller** is running (on minikube: the ingress addon).
2. Deploy two apps, each with a Service (e.g. `web` and `api`).
3. Write an Ingress: one host, with `/` going to `web` and `/api` going to `api`.
4. Test both paths with `curl`, setting the Host header.

✅ **Done when:** `/` is answered by web and `/api` by api; a wrong host gives 404.

💡 **Hint:** `ingressClassName: nginx`, `rules → host → http.paths → path / pathType / backend.service`, `curl -H "Host: …"`.

📄 *Exam wording:* "Deploy an application and configure an Ingress rule that routes requests by host or URL path to the correct Service."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-12-2"></a>
### 🧪 Lab 12.2 · ConfigMap → environment variables  ·  ⏱ ~8 min

🎯 **Goal:** Give an app its settings without rebuilding the image.

**Steps**
1. Create a ConfigMap with two keys (e.g. `APP_COLOR`, `APP_MODE`).
2. Load **all its keys** as environment variables in a Deployment.
3. Check the values **inside a running Pod**.

✅ **Done when:** `env` inside the Pod shows both keys with the right values.

💡 **Hint:** `kubectl create configmap --from-literal`, `envFrom: - configMapRef:`, `kubectl exec deploy/<name> -- env`.

📄 *Exam wording:* "Create a ConfigMap and inject its values into a Deployment as environment variables. Verify the values inside a running Pod."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-12-3"></a>
### 🧪 Lab 12.3 · Secret → used by a Pod (no hard-coding)  ·  ⏱ ~8 min

🎯 **Goal:** Pass a password to an app without writing it in the manifest.

**Steps**
1. Create a Secret holding a sample credential with `kubectl`, not in a YAML file.
2. In the Pod/Deployment, read it as an environment variable **by reference**.
3. Check the app sees the value, and that the manifest only names the Secret.

✅ **Done when:** The value appears inside the Pod, and nowhere in your YAML.

💡 **Hint:** `kubectl create secret generic --from-literal`, `valueFrom: secretKeyRef: name / key`, `kubectl exec … -- printenv`.

📄 *Exam wording:* "Create a Kubernetes Secret for a sample credential and consume it from a Pod without hard-coding the credential in the Deployment manifest."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 13 · Storage, HPA & Probes

<a id="lab-13-1"></a>
### 🧪 Lab 13.1 · PVC: data that survives a new Pod  ·  ⏱ ~10 min

🎯 **Goal:** Prove storage outlives the Pod that wrote to it.

**Steps**
1. Create a **PersistentVolumeClaim** (e.g. 100Mi, ReadWriteOnce).
2. Run a Pod that mounts it (e.g. at `/data`).
3. Write a file into `/data`.
4. Delete the Pod, create it again, and read the file.

✅ **Done when:** The PVC is `Bound`, and the file is still there in the new Pod.

💡 **Hint:** `kind: PersistentVolumeClaim`, `volumes: persistentVolumeClaim: claimName`, `volumeMounts`, `kubectl exec … cat`.

📄 *Exam wording:* "Create a PersistentVolumeClaim, mount it into a Pod, write a file, recreate the Pod, and verify that the file remains."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-13-2"></a>
### 🧪 Lab 13.2 · Readiness and liveness probes in action  ·  ⏱ ~12 min

🎯 **Goal:** See how Kubernetes treats "not ready" versus "broken".

**Steps**
1. Add a **readiness** probe and a **liveness** probe (HTTP checks) to a web app.
2. Make readiness fail: the Pod should go **0/1** and leave the Service's endpoints, without restarting.
3. Make liveness fail: the container should be **restarted** (RESTARTS goes up).
4. Write down what you saw for each.

✅ **Done when:** Readiness failure → 0/1 and no endpoint; liveness failure → RESTARTS increases.

💡 **Hint:** `readinessProbe` / `livenessProbe` with `httpGet: path, port`, `kubectl get pods -w`, `get endpoints`, `describe pod`.

📄 *Exam wording:* "Configure readiness and liveness probes for a sample application. Demonstrate how Kubernetes reacts when the application is not ready or becomes unhealthy."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-13-3"></a>
### 🧪 Lab 13.3 · HPA: scale on CPU under load  ·  ⏱ ~15 min

🎯 **Goal:** Make Kubernetes add Pods automatically when CPU is busy.

**Steps**
1. List the **prerequisites** (Metrics Server installed, CPU requests set on the Pods).
2. Create a Deployment with a CPU **request** and a Service.
3. Create an **HPA** (e.g. target 50% CPU, 1–5 replicas).
4. Generate load from another Pod and watch the replica count rise.
5. Stop the load and note that it scales down slowly.

✅ **Done when:** `kubectl get hpa` shows real CPU numbers (not `<unknown>`) and REPLICAS goes above 1 under load.

💡 **Hint:** `minikube addons enable metrics-server`, `resources.requests.cpu`, `kubectl autoscale --cpu-percent=50 --min --max`, a `while true; do wget …; done` loop, `get hpa -w`.

📄 *Exam wording:* "Configure an HPA for a Deployment with CPU requests defined. Generate load and observe replica scaling; document any cluster prerequisites."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 14 · Kubernetes troubleshooting

<a id="lab-14-1"></a>
### 🧪 Lab 14.1 · Fix an ImagePullBackOff  ·  ⏱ ~8 min

🎯 **Goal:** Diagnose why an image won't download, then fix it.

**Steps**
1. Deploy something with an **invalid image** (e.g. a tag that doesn't exist).
2. Check the Pod's status.
3. Find the exact error with **describe** and the **events**.
4. Fix the image and confirm the Pod runs.

✅ **Done when:** You can quote the pull error, and after the fix the Pod is `Running`.

💡 **Hint:** `kubectl get pods`, `describe pod` (Events), `get events --sort-by=.lastTimestamp`, `kubectl set image`.

📄 *Exam wording:* "Deploy a workload with an invalid image. Diagnose ImagePullBackOff using kubectl get, describe, and events, then correct the image."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-14-2"></a>
### 🧪 Lab 14.2 · Fix a CrashLoopBackOff  ·  ⏱ ~10 min

🎯 **Goal:** Find why a container keeps crashing and fix the cause.

**Steps**
1. Deploy an app that crashes on start (e.g. a missing environment variable).
2. Read the logs of the **crashed** container.
3. Use **describe** to read the Last State and exit code.
4. Fix the configuration and confirm it stays Running.

✅ **Done when:** You can name the root cause, and after the fix RESTARTS stops increasing.

💡 **Hint:** `kubectl logs --previous`, `describe pod` (Last State, Exit Code), `kubectl set env`, exit codes 1 / 127 / 137.

📄 *Exam wording:* "Diagnose a CrashLoopBackOff workload using logs and describe. Identify the root cause, fix the configuration, and verify recovery."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-14-3"></a>
### 🧪 Lab 14.3 · Fix a Service with the wrong selector  ·  ⏱ ~8 min

🎯 **Goal:** Find why a Service sends traffic nowhere, then fix it.

**Steps**
1. Create a Service whose selector **doesn't match** the Pods' labels.
2. Show that its endpoints are **empty**.
3. Compare the Service's selector with the Pods' labels.
4. Fix the selector and confirm endpoints appear and requests work.

✅ **Done when:** Endpoints go from `<none>` to Pod IPs, and a test request succeeds.

💡 **Hint:** `kubectl get endpoints`, `describe svc` (Selector), `get pods --show-labels`, `kubectl edit svc`.

📄 *Exam wording:* "Create a Service with an incorrect selector. Use labels, selectors, endpoints, and Service configuration to identify and fix the issue."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 15 · Helm

<a id="lab-15-1"></a>
### 🧪 Lab 15.1 · Create a Helm chart and explain its parts  ·  ⏱ ~10 min

🎯 **Goal:** Build a chart and say what each main file is for.

**Steps**
1. Create a starter chart for a web app.
2. Look at its structure.
3. Explain **Chart.yaml**, **values.yaml** and the **templates/** folder in one or two lines each.
4. Check the chart is valid.

✅ **Done when:** You can explain the three parts, and `helm lint` reports 0 charts failed.

💡 **Hint:** `helm create`, Chart.yaml (name, version, appVersion), values.yaml (defaults), templates/ (YAML with `{{ .Values… }}`), `helm lint`.

📄 *Exam wording:* "Create a Helm chart for a simple web application. Explain the purpose of Chart.yaml, values.yaml, and the templates directory."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-15-2"></a>
### 🧪 Lab 15.2 · Install the chart with your own values  ·  ⏱ ~10 min

🎯 **Goal:** Deploy the chart with a custom image, replica count and Service port.

**Steps**
1. Render the chart first to preview the YAML with your values.
2. Install it as a release, setting the **image tag**, **replica count** and **Service port**.
3. Verify the release and the Kubernetes objects it created.

✅ **Done when:** The rendered YAML and the live objects both show your values.

💡 **Hint:** `helm template`, `helm install <release> <chart> --set image.tag=… --set replicaCount=… --set service.port=…`, `helm list`, `helm get values`, `kubectl get all`.

📄 *Exam wording:* "Deploy the chart with configurable image, replica count, and Service port values. Verify the rendered and installed Kubernetes resources."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-15-3"></a>
### 🧪 Lab 15.3 · Upgrade, history, rollback  ·  ⏱ ~10 min

🎯 **Goal:** Change a release, look at its history, and undo the change.

**Steps**
1. **Upgrade** the release with a change (e.g. a new image tag).
2. Show the release's **revision history**.
3. **Roll back** to the previous revision.
4. Check the final status, and notice the revision number.

✅ **Done when:** History shows the upgrade and the rollback, and `helm status` says deployed.

💡 **Hint:** `helm upgrade`, `helm history`, `helm rollback <release> <revision>`, `helm status`. A rollback creates a **new** revision number.

📄 *Exam wording:* "Upgrade a Helm release, inspect its revision history, and roll back to the previous revision. Verify the final release status."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 16 · CI/CD & GitHub Actions

<a id="lab-16-1"></a>
### 🧪 Lab 16.1 · A workflow that builds on every push  ·  ⏱ ~10 min

🎯 **Goal:** Automate the build of your app on GitHub.

**Steps**
1. Create a workflow file in the correct folder.
2. Trigger it on **push**.
3. Steps: check out the code, set up the language, **install dependencies**, **build**.
4. Push and watch the run.

✅ **Done when:** The Actions tab shows a green ✅ run with every step passing.

💡 **Hint:** `.github/workflows/ci.yml`, `on: push`, `runs-on: ubuntu-latest`, `uses: actions/checkout@v4`, `run:`.

📄 *Exam wording:* "Create a GitHub Actions workflow triggered on push that checks out the repository, installs dependencies, and builds the application."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-16-2"></a>
### 🧪 Lab 16.2 · Add tests: red on failure, green when fixed  ·  ⏱ ~10 min

🎯 **Goal:** Show that a failing test stops the pipeline.

**Steps**
1. Add a **test** step to the workflow.
2. Push a deliberately **failing** test and see the run turn red ❌.
3. Fix the test, push again, and see it turn green ✅.

✅ **Done when:** Two runs: one red (on the test step) and one green.

💡 **Hint:** A test command like `npm test` / `pytest`; any non-zero exit code fails the step; `gh run view --log-failed`.

📄 *Exam wording:* "Add an automated test step. Demonstrate that a failing test causes the workflow to fail and that a corrected test passes."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-16-3"></a>
### 🧪 Lab 16.3 · Upload a build artifact  ·  ⏱ ~8 min

🎯 **Goal:** Keep the build output so it can be downloaded from the run.

**Steps**
1. After the build, add a step that **uploads** the build folder as an artifact with a name.
2. Push, open the run's page, and find the artifact.
3. Download it.

✅ **Done when:** The run page lists your artifact, and the download contains your build files.

💡 **Hint:** `uses: actions/upload-artifact@v4`, `with: name / path`, `gh run download`.

📄 *Exam wording:* "Extend the workflow to upload a build artifact and make it downloadable from the Actions run."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 17 · Complete CI/CD & DevSecOps

<a id="lab-17-1"></a>
### 🧪 Lab 17.1 · Build and push an image using secrets  ·  ⏱ ~12 min

🎯 **Goal:** Publish a Docker image from CI without putting a password in the code.

**Steps**
1. Store the registry username and token as **repository secrets**.
2. In a workflow: check out, **log in** to the registry using those secrets, **build** the image, **push** it.
3. Tag the image with the commit SHA.

✅ **Done when:** The image appears in the registry, and no credential is written in the YAML.

💡 **Hint:** `${{ secrets.NAME }}`, `docker/login-action`, `docker/build-push-action`, `${{ github.sha }}`.

📄 *Exam wording:* "Create a CI/CD pipeline that builds a Docker image and pushes it to a container registry using repository secrets rather than hard-coded credentials."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-17-2"></a>
### 🧪 Lab 17.2 · Security scans that block the pipeline  ·  ⏱ ~12 min

🎯 **Goal:** Add three kinds of scan and make a failure stop everything after it.

**Steps**
1. Add a **SAST** scan (the code itself).
2. Add **dependency scanning** (the libraries you use).
3. Add **secret scanning** (leaked passwords/keys).
4. Make the build/deploy jobs **depend on** the security job, and make it a required check.

✅ **Done when:** When a scan fails, the later jobs are skipped and a PR can't be merged.

💡 **Hint:** Semgrep / CodeQL / Bandit (SAST), `npm audit` / `pip-audit` (dependencies), gitleaks (secrets), `needs:`, branch protection → required status checks.

📄 *Exam wording:* "Add SAST, dependency scanning, and secret scanning to a pipeline. Configure a required security check to block later stages when it fails."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-17-3"></a>
### 🧪 Lab 17.3 · Image scan with a severity gate  ·  ⏱ ~10 min

🎯 **Goal:** Stop a deployment when the image has serious vulnerabilities.

**Steps**
1. Scan the built image for vulnerabilities.
2. Set the **threshold** (e.g. block on HIGH and CRITICAL).
3. Make the scan **fail the job** above the threshold.
4. Make the deploy job depend on the scan.

✅ **Done when:** A vulnerable image stops the pipeline before deploy; a clean one goes through.

💡 **Hint:** `trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed <image>`, `needs:`.

📄 *Exam wording:* "Add container image vulnerability scanning and implement a gate that prevents deployment when the agreed severity threshold is exceeded."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 18 · Terraform & IaC

<a id="lab-18-1"></a>
### 🧪 Lab 18.1 · First Terraform config: fmt → init → validate → plan  ·  ⏱ ~10 min

🎯 **Goal:** Write a small configuration and read what Terraform plans to do.

**Steps**
1. Write a config that declares a **provider** and creates one small resource (e.g. a local file).
2. Run **fmt**, **init**, **validate** and **plan**.
3. Explain the plan's output: the symbols and the summary line.

✅ **Done when:** `validate` says the configuration is valid, and you can explain `+ create` and `Plan: X to add…`.

💡 **Hint:** `terraform { required_providers }`, `resource "local_file" …`, `terraform fmt / init / validate / plan`.

📄 *Exam wording:* "Write Terraform configuration that declares a provider and creates a small supported resource. Run fmt, init, validate, and plan, then explain the output."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-18-2"></a>
### 🧪 Lab 18.2 · Variables, a .tfvars file, and outputs  ·  ⏱ ~10 min

🎯 **Goal:** Make the config reusable and print a useful result.

**Steps**
1. Add a **variable** and use it in a resource.
2. Give it a value from a **`.tfvars`** file.
3. Add an **output** that prints something useful.
4. Apply and read the output.

✅ **Done when:** After `apply`, `terraform output` prints your value, which came from the .tfvars file.

💡 **Hint:** `variable "name" { }`, `var.name`, `terraform.tfvars`, `-var-file=`, `output "x" { value = … }`, `terraform output`.

📄 *Exam wording:* "Use variables and outputs in a Terraform configuration. Pass a variable through a .tfvars file and print a useful output after apply."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-18-3"></a>
### 🧪 Lab 18.3 · Change a resource, read the plan, understand state  ·  ⏱ ~10 min

🎯 **Goal:** See how Terraform compares your code with what it built.

**Steps**
1. Change one setting in an existing resource.
2. Run **plan** before applying and read the symbol (`~` update or `-/+` replace).
3. Explain how the **state file** maps your code to the real resource.
4. Add a `.gitignore` so state and secrets are never committed.

✅ **Done when:** You can explain the plan's symbol and what `terraform.tfstate` is, and the `.gitignore` covers state, `.terraform/` and secret tfvars.

💡 **Hint:** `terraform plan`, `~` / `-/+` / `# forces replacement`, `terraform state list`, `.gitignore`: `*.tfstate*`, `.terraform/`.

📄 *Exam wording:* "Modify a resource configuration, inspect the plan before applying, and explain how Terraform state maps configuration to real infrastructure. Do not commit secrets or state files."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 19 · Cloud & Terraform in action

<a id="lab-19-1"></a>
### 🧪 Lab 19.1 · Design a cloud network (diagram)  ·  ⏱ ~12 min

🎯 **Goal:** Lay out a VPC with public and private parts and explain the traffic.

**Steps**
1. Draw a **VPC** with an address range (e.g. 10.0.0.0/16).
2. Add a **public** subnet and a **private** subnet.
3. Add the **route tables**: public → internet gateway, private → NAT gateway.
4. Add **security rules** (what each server allows in).
5. Describe the traffic flow: user → web server → database, and the database's way out.

✅ **Done when:** The diagram has every part labelled, and the traffic description matches it.

💡 **Hint:** VPC, subnet, route table, internet gateway, NAT gateway, security group, `0.0.0.0/0`.

📄 *Exam wording:* "Design a cloud network with a VPC/VNet, public and private subnets, route tables, and security rules. Submit a diagram and explain the traffic flow."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-19-2"></a>
### 🧪 Lab 19.2 · Build the network + one small server with Terraform  ·  ⏱ ~15 min

🎯 **Goal:** Provision real cloud infrastructure safely, then remove it.

**Steps**
1. Write Terraform for a VPC, a subnet, an internet gateway, a route table (and its association) and a security group.
2. Add **one low-cost server** (e.g. t3.micro).
3. Run **plan** and read it **before** apply.
4. Apply, check it works, then **destroy** everything.

✅ **Done when:** Plan shows what will be created, apply succeeds, and destroy removes it all.

💡 **Hint:** `aws_vpc`, `aws_subnet`, `aws_internet_gateway`, `aws_route_table` + `_association`, `aws_security_group`, `aws_instance`, `terraform plan / apply / destroy`.

📄 *Exam wording:* "Use Terraform to provision a small cloud network and one low-cost compute resource in a sandbox account. Run plan before apply and destroy the lab resources afterward."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-19-3"></a>
### 🧪 Lab 19.3 · Security group: only the app port is open  ·  ⏱ ~10 min

🎯 **Goal:** Allow just the web port and keep admin access closed to the internet.

**Steps**
1. Allow the app port (e.g. 80) from anywhere.
2. Allow **SSH (22)** only from **one** admin IP (a `/32`), never `0.0.0.0/0`.
3. Show evidence: the rules, the app working, and SSH blocked from elsewhere.

✅ **Done when:** The rules list shows 80 open and 22 restricted to one IP, and the app responds.

💡 **Hint:** `ingress { from_port, to_port, protocol, cidr_blocks }`, `var.admin_cidr`, `curl -I`, `nc -zv <ip> 22`.

📄 *Exam wording:* "Configure a security group/firewall rule that permits only the required application port. Demonstrate that SSH/admin access is not open to the entire internet."

✍️ **Your answer:**

```bash



```

- [ ] Done

---

## Session 20 · Monitoring, Observability & GitOps

<a id="lab-20-1"></a>
### 🧪 Lab 20.1 · Prometheus scrapes your app  ·  ⏱ ~12 min

🎯 **Goal:** Collect metrics from an app and check they arrive.

**Steps**
1. Run an app that exposes a `/metrics` page.
2. Configure Prometheus to **scrape** it (a job and a target).
3. Check the target is **UP**.
4. Name and query at least **two metrics**.

✅ **Done when:** The Targets page shows UP, and `up` returns 1 for your job.

💡 **Hint:** `prometheus.yml` → `scrape_configs: job_name, static_configs: targets`, Status → Targets, PromQL `up`.

📄 *Exam wording:* "Expose application metrics and configure Prometheus to scrape them. Verify the target is healthy and identify at least two metrics."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-20-2"></a>
### 🧪 Lab 20.2 · Grafana dashboard: rate, errors, latency  ·  ⏱ ~12 min

🎯 **Goal:** Show the app's health on one dashboard.

**Steps**
1. Connect Grafana to Prometheus as a **data source**.
2. Make three panels: **request rate**, **error rate**, **latency**.
3. Write one line for each panel saying what it tells you.

✅ **Done when:** Three panels show data, each with a one-line meaning.

💡 **Hint:** `rate(http_requests_total[5m])`, a `status=~"5.."` filter for errors, `histogram_quantile(0.95, …)` for latency.

📄 *Exam wording:* "Create a Grafana dashboard with panels for request rate, errors, and latency, or use the metrics available in the supplied environment. Explain what each panel indicates."

✍️ **Your answer:**

```bash



```

- [ ] Done

<a id="lab-20-3"></a>
### 🧪 Lab 20.3 · GitOps with Argo CD: change replicas in Git  ·  ⏱ ~12 min

🎯 **Goal:** Change the app by changing Git, and watch the cluster follow.

**Steps**
1. Create an Argo CD **Application** pointing at a Git repo folder with your manifests.
2. Turn on automated sync.
3. Change `replicas` in Git, then commit and push.
4. Verify Argo CD syncs and the cluster now runs the new count.

✅ **Done when:** Argo shows Synced + Healthy, and `kubectl get deploy` shows the new replica count.

💡 **Hint:** Application YAML: `source.repoURL / path / targetRevision`, `destination`, `syncPolicy.automated`; `argocd app get`; `kubectl get deploy`.

📄 *Exam wording:* "Configure a basic GitOps deployment using Argo CD or a supplied GitOps environment. Change the desired replica count in Git and verify reconciliation in the cluster."

✍️ **Your answer:**

```bash



```

- [ ] Done

