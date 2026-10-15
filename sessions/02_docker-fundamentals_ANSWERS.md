# Answer key · Session 02 · Docker I

⚠️ Only open after writing your answers in `MY_ANSWERS.md`.

---

## Warm-up

**W1.** Which command shows changes you haven't staged yet?
- A) `git status`
- B) `git diff`
- C) `git log`
- D) `git diff --staged`

**Correct: B.** `git diff` shows the line-by-line changes that aren't staged. A only **lists** which files changed, not the changes themselves. D shows changes that **are** staged. C shows history.

---

**W2.** An app listens on `127.0.0.1:3000`. Your phone opens `http://<laptop-ip>:3000`. What happens?
- A) It works, because they're on the same Wi-Fi
- B) It fails, because 127.0.0.1 accepts connections only from the laptop itself
- C) It fails with a DNS error, because no domain name was used
- D) It returns 404, because the path is missing

**Correct: B.** 127.0.0.1 is the **loopback** address, which only accepts connections from the same machine. The phone's request arrives on the laptop's Wi-Fi address and is refused. The fix is to listen on `0.0.0.0`. C: using an IP directly involves **no DNS**. D: a 404 would need the app to actually answer.

---

**W3.** `f() { echo hi; return 3; }; f; echo $?` prints?
- A) `hi`, then `0`
- B) `hi`, then `3`
- C) Only `3`
- D) `hi`, then `1`

**Correct: B.** The function prints `hi`, and `return 3` sets its exit code to 3. `$?` reads the last command's exit code, which is the function call: 3. A is the trap: thinking echo's success sets it.

---

## Check 34

**34.1** Why do containers start in seconds and VMs in minutes?
- A) Containers are stored on faster disks
- B) Containers don't boot a whole OS; they share the host's kernel
- C) VMs have to download their OS every time they start
- D) Containers skip running the app's code until needed

**Correct: B.** A VM has to boot an entire operating system. A container just starts a process on an already-running kernel.

---

**34.2** A disadvantage of containers vs VMs?
- A) They're larger in size
- B) They start more slowly
- C) Isolation is weaker, because all containers share one kernel
- D) They can't run on Linux

**Correct: C.** Sharing the kernel makes containers light, but a kernel-level problem or attack can affect all of them. A and B are **advantages** of containers, stated backwards. D: containers **need** Linux.

---

## Check 35

**35.1** `Cannot connect to the Docker daemon`. The fix on your Mac?
- A) Reinstall the `docker` command with Homebrew
- B) Start the daemon with `colima start`
- C) Log in to Docker Hub first
- D) Run `docker pull` before `docker ps`

**Correct: B.** The **client** works (it printed the error), but the **daemon** isn't running. On your setup, Colima runs the daemon. C and D both still need a running daemon.

---

**35.2** In `ghcr.io/myteam/api:2.1`, what is `2.1`?
- A) The registry
- B) The repository
- C) The tag
- D) The container ID

**Correct: C.** The part after `:` is the **tag** (version). `ghcr.io` = registry, `myteam/api` = repository.

---

## Check 36

**36.1** `docker run -d nginx` three times gives:
- A) 3 images and 1 container
- B) 1 image and 3 containers
- C) 3 images and 3 containers
- D) 1 image and 1 container

**Correct: B.** The image is downloaded **once** and reused. Each `run` creates a **new container**. It's like starting one program three times: 1 program, 3 processes.

---

**36.2** A file created inside a container, then `docker rm`?
**Model answer:** *"It's deleted, because it lived in the container's writable layer, which is removed along with the container. The image never changed. To keep data, use a volume."*

---

## Check 37

**37.1** `docker run -d -p 3000:80 nginx`. Which URL?
- A) `http://localhost:80`
- B) `http://localhost:3000`
- C) `http://nginx:3000`
- D) `http://localhost:8080`

**Correct: B.** `-p HOST:CONTAINER`: your Mac's **3000** forwards to the container's 80. A uses the container's port, which isn't published on the Mac. C: `nginx` isn't a name your Mac knows. D is a different port.

---

**37.2** `docker run ubuntu` returns immediately. Why?
- A) The ubuntu image is broken
- B) The container's main process finished, so the container stopped
- C) Ubuntu containers need a `-p` flag to stay running
- D) The Docker daemon crashed

**Correct: B.** A container lives only as long as its main process (PID 1). Ubuntu's default shell had nothing to do without a terminal, so it exited with `Exited (0)`, meaning success. C: `-p` is about networking, not staying alive.

---

**37.3** `docker stop` vs `docker kill`?
**Model answer:** *"`docker stop` sends SIGTERM so the app can shut down cleanly, then SIGKILL after 10 seconds if it hasn't, whereas `docker kill` sends SIGKILL immediately, with no clean-up."*

---

## Session quiz

**Q1.** Which is accurate?
- A) Containers share the host's kernel; each VM runs a full guest OS with its own kernel
- B) Each container includes its own kernel, which is why containers are small
- C) VMs share the host's kernel, while containers run on a hypervisor
- D) Containers and VMs both need a hypervisor in order to start

**Correct: A.** B is backwards: containers have **no** kernel of their own, which is exactly why they're small. C swaps the two. D: containers need a container engine, not a hypervisor. (Your Mac uses a VM only because macOS isn't Linux.)

---

**Q2.** `run --name app` → `stop app` → the same `run --name app …` again?
- A) A second container starts, also on port 8080
- B) It fails, because the name `app` is still taken by the stopped container
- C) The stopped container is restarted instead
- D) It fails, because port 8080 is still held by the stopped container

**Correct: B.** A **stopped** container still **exists** (`docker ps -a`), and names must be unique, so you get `Conflict. The container name "/app" is already in use`. D is the clever trap: a stopped container **releases** its port, so the port isn't the problem. Fix: `docker start app`, or `docker rm app` first.

---

**Q3.** `docker exec` says the container isn't running. What's needed first?
- A) `docker pull web`
- B) `docker rm web`
- C) Add `-p 8080:80` to the exec command
- D) `docker start web`

**Correct: D.** `exec` runs a command **inside a running** container, so you have to start it first. B would delete it. A downloads an image (and `web` is a container name, not an image).

---

**Q4.** `Exited (1)`. What does it mean, and what's the best first command?
- A) It finished successfully, so run `docker start`
- B) Its main process failed, so check `docker logs <name>`
- C) It's paused, so run `docker unpause`
- D) Its image is missing, so run `docker pull`

**Correct: B.** A non-zero exit code means failure (shell-scripting exit codes again). The app's error message will be in its logs. A is what `Exited (0)` means. C: paused containers show `Paused`, not `Exited`.

---

**L1.** Full forms + `-d`
**Model answer:** *"CLI = Command-Line Interface. VM = Virtual Machine. `-d` = detached: run the container in the background and return the terminal."*

---

**L2.** Image vs container
**Model answer:** *"An image is like a program file: a read-only package sitting on disk, whereas a container is like a process: a running instance of that image with its own writable layer, and one image can run as many containers."*
