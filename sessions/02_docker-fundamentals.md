# Session 02 · Docker I, from absolute zero (Oct 11, Session 2)

**Time:** ~90–100 min · **Concepts:** 34 → 37 · **Answers go in:** `MY_ANSWERS.md` (or say "Docker questions" to Claude) · **Answer key:** `02_docker-fundamentals_ANSWERS.md`

**How to read this:** every new word is in **bold** and explained right where it first appears. Every command is broken into its parts, and you're shown roughly what you should see. Go slowly, and type every command yourself.

---

## 🔁 Warm-up (from earlier sessions)

**W1.** Which Git command shows the changes you've made but **haven't staged yet**?
- A) `git status`
- B) `git diff`
- C) `git log`
- D) `git diff --staged`

**W2.** An app on your laptop listens on `127.0.0.1:3000`. Your phone, on the same Wi-Fi, opens `http://<laptop-ip>:3000`. What happens?
- A) It works, because they're on the same Wi-Fi
- B) It fails, because 127.0.0.1 accepts connections only from the laptop itself
- C) It fails with a DNS error, because no domain name was used
- D) It returns 404, because the path is missing

**W3.** What does this print?
```bash
f() { echo hi; return 3; }
f
echo $?
```
- A) `hi`, then `0`
- B) `hi`, then `3`
- C) Only `3`
- D) `hi`, then `1`

---

## Before you start: the tools on your Mac, explained

You'll type `docker …` commands. Here's what's installed and why, in plain words.

- **Homebrew** (`brew`): an app installer for the Terminal, from the Linux session (like `apt` on Linux). It's how these tools were installed.
- **Docker** (the `docker` command): the tool this session is about. You'll learn what it is in Concept 35.
- **Colima:** Docker needs **Linux** to work (Concept 34 explains why), and your Mac runs **macOS**, not Linux. Colima solves this by running a **small, hidden Linux computer inside your Mac** (a "virtual machine", explained in Concept 34). Docker does its work inside that hidden Linux computer. You never see it; you just type `docker` commands in your normal Terminal.
  - The hidden Linux is **Ubuntu**, a popular version of Linux.
  - Colima is a free alternative to the "Docker Desktop" app you may see mentioned in tutorials. It does the same job.

**Step 1:** check that Colima is running:
```bash
colima status
```
**You should see** something like:
```
INFO colima is running using macOS Virtualization.Framework
INFO runtime: docker
```
If it says it's **not running**, type `colima start` and wait about a minute.

> ⚠️ **Don't delete two things you'll see later.** When you list containers, you'll notice ones called **`minikube`** (your Kubernetes, needed from Oct 13) and **`localstack`** (a pretend Amazon cloud, useful for Terraform). Leave them alone. Only remove containers you created in this session.

---

## Concept 34 · What a container is, and how it differs from a virtual machine

**Words you'll learn:** dependency, environment, container, isolation, host, kernel (recap), namespace, cgroup, virtual machine (VM), guest OS, host OS, hypervisor.

### Step 1: the problem, as a story
Riya writes a small web app on her laptop. To run, the app **needs** certain things installed:
- Python version 3.12
- a library called Flask (a **library** is ready-made code that other programs use)
- a settings file

Anything an app needs in order to run is called a **dependency**.

Riya's laptop has all the dependencies, so the app works. Then she copies the app to a **server** (a computer that serves users, from Session 1). The server has **Python 3.8** and **no Flask**, so the app **crashes**. That's the **"it works on my machine"** problem from Day 1.

Two more headaches:
- The same server runs another app that **needs Python 3.8**. Installing 3.12 for Riya's app might break that one.
- Every new server needs someone to repeat a long installation checklist by hand. It's slow, and mistakes happen.

**The full set of things around an app** (the OS, the installed versions, the settings) is called its **environment**. The problem is that **every environment is slightly different.**

### Step 2: the idea that fixes it
What if Riya could **pack her app together with its own copy of Python 3.12, Flask, and the settings file into one sealed box**, and that box ran **exactly the same** on every computer?

**That box is a container.**

> **A container is a package containing an app plus everything that app needs to run. It runs separated from everything else on the computer.**

**Analogy: a shipping container.** Before shipping containers, every port loaded boxes, barrels, and sacks differently, which was slow and messy. Then everyone agreed on **one standard metal box**. Any ship, truck, or crane can move it, whatever's inside. Software containers do the same for apps: one standard package that any computer with Docker can run.

### Step 3: "runs separated": isolation
**Isolation** means **being kept apart so you can't see or disturb others**. Each container:
- has **its own files**, so it can't see the files of other containers or your Mac
- has **its own list of running programs**
- has **its own network address**

It's as if each container believes it's the only thing on the computer.

The computer that runs the containers is called the **host**. (Like a party host: it hosts the guests.)

### Step 4: how isolation works (a peek under the hood)
Remember **processes** (Session 1)? A process is a running program. **A container is really just a process that has been walled off.** The walls come from two features built into the **Linux kernel**. Recap: the kernel is the core part of the operating system that controls the hardware.

1. **Namespaces** decide **what a process can see**. The kernel gives the container's process its own private "view": its own list of processes, its own network, its own files. **Analogy:** blinkers on a horse, which limit what it sees.
2. **cgroups** (short for **control groups**) decide **how much a process can use**: how much CPU and memory. That way, one greedy container can't slow down all the others. **Analogy:** a data limit on a phone plan.

Both are **Linux** features. That's why Docker needs Linux, and why your Mac needs Colima's hidden Linux.

### Step 5: the older way, virtual machines
Before containers became popular, the usual way to separate apps was a **virtual machine**.

> A **virtual machine (VM)** is **a whole computer imitated by software**: pretend CPU, pretend memory, pretend disk, and a **complete operating system of its own**.

- The operating system inside a VM is called the **guest OS** (it's a "guest" on the real machine).
- The real computer's own operating system is the **host OS**.
- The software that creates and runs VMs is the **hypervisor**. It divides the real hardware between the VMs. Examples: VMware, VirtualBox.

**Each VM has its own full operating system, including its own kernel.** That's the big difference.

### Step 6: side by side
```
        VIRTUAL MACHINES                             CONTAINERS
 ┌─────────┐ ┌─────────┐ ┌─────────┐      ┌─────────┐ ┌─────────┐ ┌─────────┐
 │  App A  │ │  App B  │ │  App C  │      │  App A  │ │  App B  │ │  App C  │
 │  + libs │ │  + libs │ │  + libs │      │  + libs │ │  + libs │ │  + libs │
 │ GUEST OS│ │ GUEST OS│ │ GUEST OS│      └─────────┘ └─────────┘ └─────────┘
 │(+kernel)│ │(+kernel)│ │(+kernel)│      ┌─────────────────────────────────┐
 └─────────┘ └─────────┘ └─────────┘      │   Docker                        │
 ┌─────────────────────────────────┐      ├─────────────────────────────────┤
 │   Hypervisor                    │      │   Host OS with ONE shared kernel│
 ├─────────────────────────────────┤      ├─────────────────────────────────┤
 │   Real hardware                 │      │   Real hardware                 │
 └─────────────────────────────────┘      └─────────────────────────────────┘
```

| | **Virtual machine** | **Container** |
|---|---|---|
| What's inside | A whole operating system + the app | Only the app + its dependencies |
| Kernel | Each VM has **its own** | All containers **share the host's one kernel** |
| Size | Big: gigabytes (GB) | Small: megabytes (MB). 1 GB = 1000 MB |
| Start-up time | Minutes (it has to boot a whole OS, like turning on a PC) | Seconds (it just starts one process) |
| Separation strength | **Stronger** (fully separate kernels) | Good, but **weaker** (they share a kernel) |
| How many fit on one computer | A few | Dozens or hundreds |

**Analogy: houses vs flats.**
- **VMs are separate houses.** Each has its own foundation, walls, plumbing, and electricity. Very private, but expensive and slow to build.
- **Containers are flats in one building.** Each flat is private and locked, but they **share** the building's foundation and plumbing (the kernel). Cheaper and quicker, but a crack in the shared foundation affects every flat.

### Step 7: your Mac uses both
The containers you'll run need a Linux kernel to share. Your Mac's kernel isn't Linux. So **Colima runs one small Linux VM**, and **all your containers run inside it, sharing its Linux kernel.** That's VMs and containers working together.

### 🛠 Do (1 min)
```bash
docker info | grep -i "operating system"
```
**What each part means:**
- `docker info` prints lots of details about your Docker setup.
- `|` (a pipe, from the Linux session) sends that output into the next command.
- `grep -i "operating system"` keeps only the line containing "operating system". `-i` = ignore upper/lower case.

**You should see:** `Operating System: Ubuntu 24.04…`. **Ubuntu**, even though you're on macOS. That line proves your containers live inside Colima's hidden Linux VM.

### ✅ Check 34

**34.1** Why do containers start in seconds while VMs take minutes?
- A) Containers are stored on faster disks
- B) Containers don't boot a whole OS; they share the host's kernel
- C) VMs have to download their OS every time they start
- D) Containers skip running the app's code until needed

**34.2** Which is a **disadvantage** of containers compared with VMs?
- A) They're larger in size
- B) They start more slowly
- C) Isolation is weaker, because all containers share one kernel
- D) They can't run on Linux

---

## Concept 35 · What Docker is, and how its parts work together

**Words you'll learn:** Docker, client, daemon (recap), CLI, API, REST API, registry, Docker Hub, repository, tag, `latest`, pull, push, official image.

### Step 1: what Docker is
**Docker** is a **tool (a set of programs) for building, sharing, and running containers.** It was released in 2013 and made containers easy enough that everyone started using them. When people say "containers", they very often mean Docker containers.

### Step 2: Docker is split into two programs
From Session 1: a **client** asks, and a **server** answers. Docker works exactly that way.

**1. The Docker client: what you type.**
The `docker` command in your Terminal is the **client**. It's a **CLI**: **Command-Line Interface**, meaning a program you control by typing commands instead of clicking buttons.

The client **does no real work itself**. It just **passes your request** to the second program.

**2. The Docker daemon: the worker.**
The **Docker daemon**, whose program name is **`dockerd`** (Docker daemon), is a **daemon**. From the Linux session: a program that runs **in the background all the time**, providing a service. It does **all the real work**: downloading, building, starting and stopping containers, connecting them to networks. On your Mac, it runs **inside Colima's Linux VM**.

**How they talk: an API.**
An **API** (**Application Programming Interface**) is **an agreed list of requests one program can send to another, and the answers it will get back**. Think of a restaurant menu: it tells you exactly what you can order and what you'll receive.

Docker's API is a **REST API** (**Representational State Transfer**). That's a style of API that works using **ordinary HTTP requests**, the same HTTP from the networking session (GET, POST, and so on).

So when you type `docker ps`, the client sends an HTTP-style request to the daemon ("list the containers"), and the daemon answers.

**Analogy: a restaurant.**
- **You** = the person typing.
- **The waiter** = the **Docker client**: takes your order, carries it to the kitchen.
- **The kitchen** = the **Docker daemon**: actually cooks.
- **The menu** = the **API**: what you're allowed to order.
- **The supplier's warehouse** = the **registry** (next step): where ingredients come from.

*(Deeper inside, the daemon uses two smaller programs, **containerd** and **runc**, to actually start containers. You only need to know their names.)*

### Step 3: where containers come from: registries
To run a container, Docker needs an **image**: the packed box from Concept 34 (full explanation in Concept 36). Images are stored and shared in a **registry**.

> A **registry** is **a server (a website for programs) that stores images**, so people can download and share them, like an app store for containers.

- **Docker Hub** (`hub.docker.com`) is the **default** registry. Docker uses it unless you say otherwise.
- **GHCR** (**GitHub Container Registry**, at `ghcr.io`) is GitHub's registry.
- Cloud companies have their own too, like Amazon's **ECR** (**Elastic Container Registry**).

Two words for moving images:
- **pull** = **download** an image from a registry to your machine
- **push** = **upload** an image from your machine to a registry

*(The same words as Git's pull/push, and a similar idea.)*

### Step 4: how an image's name works
Inside a registry, images are organised like this:
- A **repository** is **one image name**, holding all its versions. For example, the `nginx` repository. (Not quite the same as a Git repository, but a similar idea: one project, many versions.)
- A **tag** is the **version label**, written after a colon: `nginx:1.27`. If you don't write a tag, Docker uses **`latest`**.

```
 ghcr.io / myteam / api : 2.1
 └──┬──┘   └─────┬────┘   └┬┘
 registry   repository    tag
 (where)    (which image) (which version)
```
For Docker Hub, the registry part is usually left out. `nginx:1.27` is short for `docker.io/library/nginx:1.27`.

> ⚠️ **`latest` doesn't mean "newest".** It's just the tag that's used when no tag is given. Whoever publishes the image decides what `latest` points to.

**Official images** are images on Docker Hub that are maintained by Docker or by the software's own makers: `nginx`, `python`, `redis`, `ubuntu`, and so on. They're the trusted ones to start from.

*(**nginx**, pronounced "engine-x", is the popular web server program from the Linux session. You'll run it soon.)*

### Step 5: what really happens when you type `docker run nginx`
```
1. You type: docker run nginx
2. The CLIENT sends the request to the DAEMON (through the API)
3. The DAEMON checks: "Do I already have the nginx image on this machine?"
4. If not → it PULLS (downloads) nginx:latest from Docker Hub
5. The DAEMON creates a container from the image
6. The DAEMON starts the container
```

### Step 6: an error you'll definitely meet
```
Cannot connect to the Docker daemon at unix:///…/docker.sock. Is the docker daemon running?
```
**Meaning:** the **client** worked (it printed this message), but there was **no daemon to talk to**. On your Mac, that means **Colima is stopped**. Fix: `colima start`.

*(The `docker.sock` part is a "socket file": the doorway the client uses to reach the daemon on the same machine.)*

### 🛠 Do (3 min)
```bash
docker version
```
**You should see two sections:**
```
Client: Docker Engine - Community        ← the client: your docker command, on macOS
 Version:  29.x
 OS/Arch:  darwin/arm64                   ← darwin = macOS, arm64 = your Apple-chip processor
Server: Docker Engine - Community        ← the daemon, running inside Colima's Linux
 Engine:
  Version: 29.x
```
Two separate programs, client and server, exactly as described.

```bash
docker run hello-world
```
**Read the output carefully. It describes the 6 steps above:**
```
Unable to find image 'hello-world:latest' locally    ← step 3: not on this machine
latest: Pulling from library/hello-world             ← step 4: downloading from Docker Hub
Status: Downloaded newer image for hello-world:latest
Hello from Docker!                                    ← steps 5–6: the container ran and printed this
This message shows that your installation appears to be working correctly.
...
```
Run `docker run hello-world` **a second time**. Notice the "Pulling" lines are gone: the image is already on your machine.

### ✅ Check 35

**35.1** `docker ps` fails with `Cannot connect to the Docker daemon. Is the docker daemon running?` What's the most likely fix on your Mac?
- A) Reinstall the `docker` command with Homebrew
- B) Start the daemon with `colima start`
- C) Log in to Docker Hub first
- D) Run `docker pull` before `docker ps`

**35.2** In `ghcr.io/myteam/api:2.1`, what is `2.1`?
- A) The registry
- B) The repository
- C) The tag
- D) The container ID

---

## Concept 36 · Images vs containers

**Words you'll learn:** image, read-only, layer, instance, writable layer.

### Step 1: what an image is
> An **image** is **the packed, sealed box**: a file (really a bundle of files) containing everything needed to run an app.

It contains:
- a small set of operating-system files (enough for the app; **not** a kernel, since containers share the host's)
- the language runtime (for example, Python itself)
- the libraries
- the app's code
- a note saying **which command to run when it starts**

An image is **read-only**: it **can't be changed** once built. To change it, you build a new image.

Images are built in **layers**, like transparent sheets stacked on top of each other, each adding some files. For example: a base OS layer, then a Python layer, then a Flask layer, then your code layer. If two images use the same base layer, Docker stores that layer **only once**. (More on layers in Session 03.)

### Step 2: what a container is
> A **container** is **a running (or stopped) copy of an image**. The proper word is an **instance** of the image.

When Docker creates a container from an image, it adds a thin **writable layer** on top: a private scratch area where that container can create and change files. **The image underneath stays untouched.**

### Step 3: you already know this pattern
From Session 1:
- A **program** is a file sitting on disk, doing nothing.
- A **process** is that program **running**.
- One program can run as **many processes** at once (like your browser's 20+ processes).

It's exactly the same here:

| Session 1 idea | Docker idea |
|---|---|
| Program (file on disk) | **Image** |
| Process (the program running) | **Container** |
| One program → many processes | One image → many containers |

**Another analogy:** an image is a **recipe**, and a container is a **dish cooked from it**. You can cook many dishes from one recipe, and spilling sauce on one dish doesn't change the recipe.

### Step 4: ⚠️ things you do inside a container are temporary
Files you create inside a container go into **its writable layer**. When you **delete the container** (`docker rm`, coming up), that layer is deleted too, **along with your files**. The image never changed, so a new container from the same image starts fresh and clean.

That's why databases running in containers need **volumes** (Session 04) to keep their data safe.

### Step 5: commands for images

| Command | What it does |
|---|---|
| `docker images` | List the images stored on this machine |
| `docker pull nginx:1.27` | Download an image without running it |
| `docker rmi nginx:1.27` | Delete an image (`rmi` = **r**e**m**ove **i**mage). You have to delete its containers first |
| `docker image inspect nginx` | Show every detail about an image |

**What `docker images` shows on your Docker version:**
```
IMAGE              ID             DISK USAGE   CONTENT SIZE
hello-world:latest a1b2c3d4e5f6   20kB         3kB
nginx:latest       …              ~190MB       ~50MB
```
- **IMAGE** = repository:tag
- **ID** = the image's unique ID
- **DISK USAGE** = how much space it takes once unpacked
- **CONTENT SIZE** = the download size

(You'll also see other images already on your Mac, like `busybox` and `amazon/aws-cli`. They were pulled when the tools were set up.)

### ✅ Check 36

**36.1** You run `docker run -d nginx` three times. What do you now have?
- A) 3 images and 1 container
- B) 1 image and 3 containers
- C) 3 images and 3 containers
- D) 1 image and 1 container

**36.2** (one line) You create a file inside a running container, then `docker rm` the container. What happens to the file, and why?

---

## Concept 37 · A container's life, and the commands to control it

**Words you'll learn:** lifecycle, state, created, running, paused, stopped/exited, removed, flag, detached, publish a port, port mapping, main process (PID 1), exit code (recap), logs, exec, interactive, terminal.

### Step 1: the lifecycle
A **lifecycle** is **the stages something goes through from birth to end**. A container passes through these **states** (a state is the condition it's in right now):

```
                docker start            docker stop            docker rm
   CREATED  ──────────────►  RUNNING  ───────────►  STOPPED  ───────────►  REMOVED
 (made, but                 (its program          (program ended;          (deleted
  not started)               is working)           container still          completely)
                              │   ▲                 exists. Docker
                 docker pause │   │ docker unpause  calls this "Exited")
                              ▼   │
                             PAUSED (frozen in place)
```
**`docker run` = create + start in one go** (and it pulls the image first, if needed). It's what you'll use most.

**Important:** a **stopped** container **still exists**. It's just not running. It only disappears when you **remove** it.

### Step 2: the commands, one by one

| Command | What it does |
|---|---|
| `docker run <image>` | Create and start a new container from an image |
| `docker ps` | List **running** containers (`ps` = process status, like the Linux command) |
| `docker ps -a` | List **all** containers, including stopped ones (`-a` = all) |
| `docker stop <name>` | Stop a running container politely (Step 5) |
| `docker start <name>` | Start a stopped container again |
| `docker restart <name>` | Stop, then start |
| `docker rm <name>` | **Remove** (delete) a stopped container. `docker rm -f <name>` forces it, even if running |
| `docker logs <name>` | Show everything the app inside has printed. Add `-f` to **follow** new lines live, like `tail -f` |
| `docker exec -it <name> sh` | Run a command **inside** a running container. Here it runs `sh` (a shell), so you get a prompt inside it |
| `docker inspect <name>` | Every detail: settings, IP address, state |
| `docker stats` | Live CPU and memory use of each container (Ctrl + C to quit) |

### Step 3: flags for `docker run`
A **flag** (also called an **option**, from the Linux session) is an extra setting added to a command, starting with `-`. These are the important ones for `docker run`:

| Flag | Full meaning | What it does |
|---|---|---|
| `-d` | **detached** | Run the container **in the background**, so your terminal is free again. Without it, the container's output fills your terminal until it stops |
| `--name web` | name | Give the container a name you choose (`web`). Without it, Docker makes up a random name like `jolly_turing` |
| `-p 8080:80` | **publish** a port | Connect **your Mac's port 8080** to **the container's port 80**. Format: **MAC-PORT : CONTAINER-PORT** |
| `-e APP_ENV=prod` | **environment** variable | Set an environment variable inside the container (from the env-var concept) |
| `--rm` | remove | Delete the container automatically as soon as it stops |
| `-it` | **interactive** + **terminal** | `-i` keeps the input open so you can type. `-t` gives it a proper terminal screen. Use both when you want to type inside the container |

### Step 4: why `-p` is needed (port mapping)
From the networking session: a **port** is a door number on a computer that leads to one specific program.

Remember **namespaces**: each container has **its own private network**, separate from your Mac's. So when nginx inside a container listens on **port 80**, that's port 80 **inside the container's private network**. Your Mac and browser **can't reach it**.

`-p 8080:80` **publishes** the port. It tells Docker: *"Anything arriving at **my Mac's** door 8080, pass it through to **the container's** door 80."* This is called **port mapping**.

```
 Your browser → localhost:8080 ──(Docker forwards)──► container's port 80 → nginx
```
**Analogy:** a hotel. The container is a guest in room 80, and guests can't be reached directly from outside. `-p 8080:80` tells reception: *"Calls to extension 8080 should be put through to room 80."*

### Step 5: stopping, and signals again
From the Linux session: a **signal** is a message the OS sends to a process. **SIGTERM** means "please stop", and **SIGKILL** means "stop NOW".
- `docker stop` sends **SIGTERM** first, giving the app a chance to finish and clean up. If the app is still running after **10 seconds**, Docker sends **SIGKILL**.
- `docker kill` sends **SIGKILL** immediately: forced, no clean-up.

### Step 6: why some containers stop by themselves
Every container has one **main process**: the program the image was told to start. Inside the container, that process has **PID 1**. (From the Linux session: a PID is a process ID, and PID 1 is the "first" process.)

> **A container lives exactly as long as its main process.** When the main process ends, the container stops.

- The **nginx** image's main process is a **web server**, which runs forever, waiting for visitors. So the container stays **Up**.
- The **ubuntu** image's main process is just a **shell** (bash), which waits for someone to type. If you start it without a terminal attached, nobody can type, so the shell **ends immediately**, and the container stops straight away. That's normal, not an error.
- `docker run -it ubuntu bash` keeps it alive, because now **you** are attached and typing.

**How it ended: exit codes (from shell scripting).** `docker ps -a` shows the STATUS:
- **`Exited (0)`** = the main process finished **successfully**
- **`Exited (1)`** (or any number other than 0) = it **failed**

When a container won't stay running, **the first thing to check is `docker logs <name>`**. The app usually printed its error there.

### 🛠 Do (~12 min). Type each line, then read the explanation beneath it

```bash
docker run -d -p 8080:80 --name web nginx
```
*Create and start a container from the `nginx` image, in the background (`-d`), named `web`, with Mac port 8080 connected to container port 80. It prints a long ID.*

```bash
docker ps
```
*You should see a row for `web`. Look at the columns: **CONTAINER ID**, **IMAGE** (nginx), **STATUS** (`Up … seconds`), **PORTS** (`0.0.0.0:8080->80/tcp`, meaning Mac port 8080 → container port 80), **NAMES** (web). (You'll also see `minikube` and `localstack`. Leave them alone.)*

```bash
curl -I http://localhost:8080
```
*`curl -I` asks for just the response headers (networking session). You should see `HTTP/1.1 200 OK` and `Server: nginx`. **You just got a webpage from inside a container.** Also open `http://localhost:8080` in your browser and you'll see "Welcome to nginx!"*

```bash
docker logs web
```
*nginx has written a line for each request, including your curl and browser visits.*

```bash
docker exec -it web sh
```
*You're now **inside** the container. The prompt changes to `#`. Try:*
```bash
ls /usr/share/nginx/html          # the website's files live here inside the container
cat /etc/os-release | head -2     # the container's own mini operating system (Debian)
exit                              # leave the container (it keeps running)
```

```bash
docker stop web
docker ps
docker ps -a
```
*After stop, `web` disappears from `docker ps` (running only), but `docker ps -a` still shows it, with STATUS `Exited (0)`. Stopped, not deleted.*

```bash
docker start web
curl -I http://localhost:8080
```
*It's back, the same container.*

```bash
docker rm -f web
```
*Delete it (`-f` = force, even though it's running). Check `docker ps -a`: it's gone.*

```bash
docker run ubuntu
docker ps -a | head -3
```
*`docker run ubuntu` returns instantly with no output. `docker ps -a` shows an ubuntu container with `Exited (0)`: its main process (a shell with nobody typing) finished immediately.*

```bash
docker images
```
*You now have `hello-world`, `nginx`, and `ubuntu` images (plus the ones that were already there).*

### ✅ Check 37

**37.1** You run `docker run -d -p 3000:80 nginx`. Which URL opens the site on your Mac?
- A) `http://localhost:80`
- B) `http://localhost:3000`
- C) `http://nginx:3000`
- D) `http://localhost:8080`

**37.2** `docker run ubuntu` returns immediately and `docker ps` shows nothing. Why?
- A) The ubuntu image is broken
- B) The container's main process finished, so the container stopped
- C) Ubuntu containers need a `-p` flag to stay running
- D) The Docker daemon crashed

**37.3** (one line) What's the difference between `docker stop` and `docker kill`?

---

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory

**Q1.** Which statement is accurate?
- A) Containers share the host's kernel; each VM runs a full guest OS with its own kernel
- B) Each container includes its own kernel, which is why containers are small
- C) VMs share the host's kernel, while containers run on a hypervisor
- D) Containers and VMs both need a hypervisor in order to start

**Q2.** You run `docker run -d --name app -p 8080:80 nginx`, then `docker stop app`, then the **same** `docker run -d --name app -p 8080:80 nginx` again. What happens on the last command?
- A) A second container starts, also on port 8080
- B) It fails, because the name `app` is still taken by the stopped container
- C) The stopped container is restarted instead
- D) It fails, because port 8080 is still held by the stopped container

**Q3.** `docker exec -it web sh` fails with `container … is not running`. What's needed first?
- A) `docker pull web`
- B) `docker rm web`
- C) Add `-p 8080:80` to the exec command
- D) `docker start web`

**Q4.** `docker ps -a` shows a container with status `Exited (1)`. What does that tell you, and what's the best first command?
- A) It finished successfully, so run `docker start`
- B) Its main process failed, so check `docker logs <name>`
- C) It's paused, so run `docker unpause`
- D) Its image is missing, so run `docker pull`

**L1.** Write the full forms of **CLI** and **VM**, and say what the `-d` flag does.

**L2.** Explain image vs container in one contrast sentence, using the program/process idea.

---

### 📖 Words from this session (also added to `notes/GLOSSARY.md`)
container · image · layer · writable layer · instance · VM · guest OS · host OS · hypervisor · namespace · cgroup · Docker client · Docker daemon (dockerd) · CLI · API · REST API · registry · Docker Hub · GHCR · repository · tag · `latest` · pull · push · official image · lifecycle · detached · port mapping · PID 1 · Colima · Ubuntu
