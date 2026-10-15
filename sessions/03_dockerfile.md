# Session 03 · Docker II: building your own images with a Dockerfile (Oct 12, Session 1)

**Time:** ~90–100 min · **Concepts:** 38 → 41 · **Answers:** `MY_ANSWERS.md` or say "Dockerfile questions" · **Answer key:** `03_dockerfile_ANSWERS.md`

**Before you start:** `colima status` (if stopped: `colima start`). Keep VS Code open. You'll write files.

---

## 🔁 Warm-up

**W1.** What does this print?
```bash
check() { [ -d /nope ]; }
check
echo $?
```
- A) `0`
- B) `1`
- C) Nothing at all
- D) `/nope`

**W2.** You changed `app.py` and haven't staged it. Which command shows the **exact lines** you changed?
- A) `git status`
- B) `git log -p`
- C) `git diff`
- D) `git show`

**W3.** You ran `docker stop api` an hour ago. Now `docker run --name api nginx` fails. Why?
- A) A stopped container named api still exists
- B) The nginx image was deleted when api stopped
- C) Docker blocks reusing a name for one hour
- D) The api container is still holding port 80

---

## Concept 38 · What a Dockerfile is, layers, and your first image

**Words you'll learn:** Dockerfile, instruction, build, base image, build context, layer, build cache, `FROM`, `WORKDIR`, `COPY`, `RUN`, `ENV`, `EXPOSE`, `-t` (tag), Flask.

### Step 1: the problem
In Session 02 you **used** other people's images (`nginx`, `ubuntu`). But your company's app isn't on Docker Hub. **You need to make an image of your own app.**

You could install things into a container by hand and save it, but nobody could repeat your exact steps, and you'd forget them too. We want the steps **written down, so they can be repeated exactly**, the same way a shell script writes down commands.

### Step 2: the Dockerfile
> A **Dockerfile** is a **plain text file of step-by-step instructions** that tells Docker how to **build** an image.

- **Build** = turning those instructions into a finished image.
- The file is literally named **`Dockerfile`**: capital D, **no** extension (not `.txt`).
- Each line starts with an **instruction**, a keyword in CAPITALS like `FROM` or `COPY`, followed by its details.

**Analogy:** a Dockerfile is a **recipe card**. `docker build` is **cooking** it. The **image** is the finished dish, ready to be served (run as containers) as many times as you like.

### Step 3: the core instructions

| Instruction | What it does | Example |
|---|---|---|
| **`FROM`** | Which existing image to **start from**, called the **base image**. Must be the first instruction | `FROM python:3.12-slim` |
| **`WORKDIR`** | Set the **working folder** inside the image (created if missing). Later instructions run there. Like `cd` | `WORKDIR /app` |
| **`COPY`** | **Copy files from your Mac** (the build context, Step 5) **into the image** | `COPY app.py .` (`.` = into the current WORKDIR) |
| **`RUN`** | **Run a command while building** the image, usually to install things | `RUN pip install flask` |
| **`ENV`** | Set an **environment variable** inside the image. It's also available when containers run | `ENV APP_ENV=docker` |
| **`EXPOSE`** | **Note down** which port the app listens on. ⚠️ It does **not** open or publish the port. Only `-p` does that | `EXPOSE 5000` |
| **`CMD`** | The **default command to run when a container starts** (Concept 39 goes deeper) | `CMD ["python", "app.py"]` |

**Base image explained:** you almost never start from nothing. `python:3.12-slim` is an official image that already has a small Linux and Python 3.12 installed. You add your app on top. `slim` means a smaller version with fewer extras.

### Step 4: layers and the build cache (why line order matters)
From Session 02: images are built in **layers**, stacked like transparent sheets.

**Each instruction that changes files (`COPY`, `RUN`, `ADD`) creates one new layer.**

When you build again, Docker checks each instruction from the top: *"Has this instruction, and the files it uses, changed since last time?"*
- **No change** → Docker **reuses** the layer it saved last time. This is the **build cache**, and it prints `CACHED`. Fast.
- **Changed** → Docker rebuilds that layer **and every layer after it** (later layers sit on top of it, so they must be rebuilt too).

**So: put steps that rarely change near the top, and steps that change often (your code) near the bottom.**

**Analogy:** stacking plates. If you need to change the 2nd plate from the bottom, every plate above it has to come off and go back on. The bottom plates you don't touch stay put.

**The classic pattern:**
```dockerfile
COPY requirements.txt .                  # the list of libraries (rarely changes)
RUN pip install -r requirements.txt      # slow install, which stays CACHED as long as the list doesn't change
COPY app.py .                            # your code (changes all the time) LAST
```
If you `COPY` everything **before** installing, then every tiny code change would make Docker reinstall all the libraries.

### Step 5: building
```bash
docker build -t hello-app:1.0 .
```
- `docker build`: build an image from a Dockerfile.
- `-t hello-app:1.0`: **t**ag it: give the image a name (`hello-app`) and a version tag (`1.0`).
- `.` (the dot at the end): the **build context**, meaning **the folder whose files Docker can use** in `COPY`. `.` = the current folder. Docker sends this folder to the daemon (remember, the daemon does the work), and the Dockerfile in it is used.

### Step 6: Flask (the tiny app we'll package)
**Flask** is a small Python **library** for making web apps. A few lines of Python give you a web server that answers HTTP requests. You don't need to know Python well. Just read the comments.

### 🛠 Do (~15 min): package your first app

```bash
mkdir -p ~/devops-practice/hello-app && cd ~/devops-practice/hello-app
code app.py
```
Paste into `app.py` and **save (Cmd + S)**:
```python
from flask import Flask      # use the Flask library
import os                    # lets Python read environment variables

app = Flask(__name__)        # create the web app

@app.route("/")              # when someone requests the path "/"...
def home():
    env = os.environ.get("APP_ENV", "dev")       # ...read APP_ENV (default "dev")
    return f"Hello from the {env} container!\n"  # ...and send back this text

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)   # listen on ALL addresses, port 5000
```
⚠️ **Why `host="0.0.0.0"`?** From the networking session: `127.0.0.1` accepts connections **only from the same machine**. Inside a container, "the same machine" means **the container itself**. Docker's port forwarding comes from **outside** the container, so an app listening on `127.0.0.1` can't be reached even with `-p`. `0.0.0.0` = accept from anywhere. **This is one of the most common Docker bugs.**

```bash
code requirements.txt
```
Paste and save (this is the list of Python libraries the app needs):
```
flask==3.0.3
```

```bash
code Dockerfile
```
Paste and save (no file extension!):
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
ENV APP_ENV=docker
EXPOSE 5000
CMD ["python", "app.py"]
```
*(`pip` is Python's package installer, like `brew`/`apt` but for Python libraries. `--no-cache-dir` = don't keep downloaded installer files, which keeps the image smaller.)*

**Build it:**
```bash
docker build -t hello-app:1.0 .
```
*You'll see numbered steps like `[1/5] FROM …`, `[3/5] RUN pip install …`. The first build downloads things and takes a minute.*

**Run it:**
```bash
docker run -d -p 8080:5000 --name hello hello-app:1.0
curl http://localhost:8080
```
*You should see: `Hello from the docker container!` Your Mac's port **8080** → the container's port **5000**. The text says "docker" because of the `ENV` line.*

**Override the environment variable at run time:**
```bash
docker rm -f hello
docker run -d -p 8080:5000 -e APP_ENV=production --name hello hello-app:1.0
curl http://localhost:8080
```
*Now it says `production`, from the same image. That's the env-var idea from the Linux session, in Docker.*

**Watch the cache:** change the text in `app.py` (e.g., "Hello" → "Hi"), save, and rebuild:
```bash
docker build -t hello-app:1.1 .
```
*Look for **`CACHED`** next to the `pip install` step. Only the `COPY app.py` step onwards re-ran.*

### ✅ Check 38

**38.1** A Dockerfile has `EXPOSE 5000`. You run `docker run -d myapp` (no `-p`). Can your Mac's browser open `localhost:5000`?
- A) Yes, EXPOSE publishes it on the Mac
- B) No, EXPOSE only documents the port
- C) Yes, once the container is restarted
- D) No, port 5000 only works with --rm

**38.2** You'll edit `app.py` often. Which order makes rebuilds fastest?
- A) `COPY . .` → `RUN pip install -r requirements.txt`
- B) `COPY requirements.txt .` → `RUN pip install …` → `COPY . .`
- C) `RUN pip install …` → `COPY requirements.txt .` → `COPY . .`
- D) Any order, since Docker caches every layer anyway

---

## Concept 39 · CMD vs ENTRYPOINT, RUN vs CMD, COPY vs ADD, ARG vs ENV

**Words you'll learn:** `CMD`, `ENTRYPOINT`, override, exec form, shell form, `ADD`, `ARG`, build-time, run-time.

### Step 1: RUN vs CMD (the most confused pair)
- **`RUN`** happens **during the build**, once. Its result (installed software, created files) is **saved into the image**.
- **`CMD`** happens **every time a container starts**. It's the command the container runs.

**Analogy:** `RUN` = cooking the meal (done once, in the kitchen). `CMD` = "serve it hot when ordered" (done every time someone orders).

### Step 2: CMD vs ENTRYPOINT
Both decide **what runs when a container starts**. The difference is **how easily you can change it** when you type `docker run`.

- **`CMD`** = the **default** command. If you type a command after the image name, it **replaces** CMD completely.
- **`ENTRYPOINT`** = the **fixed** main program. Anything you type after the image name is **added to it as extra arguments**. It doesn't replace it. (To replace it, you'd need the special `--entrypoint` flag.)

**To override** means to replace a default with your own choice.

| Dockerfile has | You type | What actually runs |
|---|---|---|
| `CMD ["echo", "hello"]` | `docker run img` | `echo hello` |
| `CMD ["echo", "hello"]` | `docker run img ls` | `ls` (CMD **replaced**) |
| `ENTRYPOINT ["echo"]` | `docker run img hi` | `echo hi` (**added**) |
| `ENTRYPOINT ["ping"]` + `CMD ["localhost"]` | `docker run img` | `ping localhost` (CMD = default arguments) |
| `ENTRYPOINT ["ping"]` + `CMD ["localhost"]` | `docker run img google.com` | `ping google.com` (only the CMD part replaced) |

**Analogy:** ENTRYPOINT is a **coffee machine** that always makes coffee, and CMD is the **default size, "medium"**. If you ask for "large", you get large coffee. You can't turn the coffee machine into a juicer just by asking.

If a Dockerfile has several CMD lines, **only the last one counts**.

### Step 3: exec form vs shell form
You can write CMD and ENTRYPOINT two ways:
- **Exec form** (✅ recommended): a list in square brackets with double quotes: `CMD ["python", "app.py"]`. Docker runs `python` **directly** as the main process (**PID 1**).
- **Shell form:** a plain sentence: `CMD python app.py`. Docker runs it **through a shell** (`/bin/sh -c "python app.py"`), so the **shell** becomes PID 1 instead of your app.

**Why it matters:** `docker stop` sends **SIGTERM** to PID 1 (Session 02). With shell form, the shell may **not pass SIGTERM on** to your app, so the app never hears "please stop". Docker waits the full **10 seconds**, then **SIGKILLs** it, with no clean shutdown. **Use exec form.**

### Step 4: COPY vs ADD
- **`COPY`** copies files from your build context into the image. Simple and predictable.
- **`ADD`** does the same, **plus** two extras: it can **download from a URL**, and it **automatically unpacks** `.tar` archive files.

**Rule:** use **`COPY`** unless you specifically need one of ADD's extras. Surprise behaviour is bad.

*(A **tar archive** is many files packed into one file, like a zip.)*

### Step 5: ARG vs ENV
- **`ARG`** = a variable that exists **only during the build** (**build-time**). You set it with `docker build --build-arg VERSION=1.2 .`. It's **gone** in the running container.
- **`ENV`** = a variable that is **baked into the image** and exists in **running containers** too (**run-time**).

⚠️ **Never put passwords or secret keys in ARG or ENV.** Anyone with the image can read them with `docker history` or `docker inspect`. (This is a DevSecOps idea. Secrets are passed in safely at run time; you'll learn how in Kubernetes Secrets.)

### 🛠 Do (5 min)
```bash
cd ~/devops-practice/hello-app
mkdir pinger && cd pinger
code Dockerfile
```
Paste and save:
```dockerfile
FROM alpine:3.20
ENTRYPOINT ["ping", "-c", "2"]
CMD ["localhost"]
```
*(`alpine` is a tiny Linux image, about 5 MB. `ping -c 2` = ping twice, then stop.)*
```bash
docker build -t pinger .
docker run --rm pinger              # → ping -c 2 localhost
docker run --rm pinger google.com   # → ping -c 2 google.com  (CMD replaced, ENTRYPOINT kept)
```

### ✅ Check 39

**39.1** A Dockerfile has `ENTRYPOINT ["ping"]` and `CMD ["localhost"]`. You run `docker run myping google.com`. What runs?
- A) `ping localhost`
- B) `ping google.com`
- C) `google.com` on its own
- D) `ping localhost google.com`

**39.2** Which statement is correct?
- A) RUN executes each time a container starts
- B) CMD executes once, while the image is built
- C) RUN runs at build; CMD runs at start
- D) RUN and CMD both execute only during the build

---

## Concept 40 · .dockerignore and multi-stage builds

**Words you'll learn:** `.dockerignore`, build context (recap), secret, multi-stage build, stage, builder, `AS`, `COPY --from`, Node.js, npm, compile.

### Step 1: .dockerignore
Remember `.gitignore` (Git session): a list of files Git should ignore. **`.dockerignore`** is the same idea for Docker builds:

> **`.dockerignore`** is a file listing what should **NOT** be sent in the build context, so it can never end up in the image.

**Why it matters:**
- **Speed:** the whole build-context folder is sent to the daemon before building. Huge folders (like `node_modules`, `.git`) make every build slower.
- **Size:** `COPY . .` copies everything that was sent, junk included.
- **Security:** a **secret** is a private value like a password or key. Files like `.env` hold secrets, and you don't want them copied into an image that others might download.

Example `.dockerignore`:
```
.git
.env
*.log
__pycache__
node_modules
```
*(`*.log` = every file ending in `.log`. `*` is the wildcard from the shell-loops session.)*

### Step 2: the problem multi-stage builds solve
To **build** some apps, you need heavy tools. To **run** them, you only need the finished result.

**Example:** a website built with **Node.js** (a program that runs JavaScript outside a browser) uses **npm** (**Node Package Manager**, which installs JavaScript libraries) and a build step that **compiles** (converts and packs) the source code into a few small, ready-to-serve files.
- Tools needed to **build**: Node.js + npm + hundreds of libraries, roughly **1 GB**.
- What's needed to **run**: just those few finished files plus a small web server (nginx).

If one image does both, you ship **1 GB of build tools nobody needs at run time**. That's bigger, slower to download, and every extra tool is a potential security hole.

### Step 3: multi-stage builds
> A **multi-stage build** is a Dockerfile with **several `FROM` lines**. Each `FROM` starts a new **stage**. An early stage builds the app; the **final stage copies only the finished result** out of it. **Only the final stage becomes the image.**

```dockerfile
# ---------- Stage 1: "build" (has all the heavy tools) ----------
FROM node:20 AS build              # AS build = give this stage a name
WORKDIR /app
COPY package*.json ./              # the list of JavaScript libraries
RUN npm ci                         # install them
COPY . .
RUN npm run build                  # compile the site into the /app/dist folder

# ---------- Stage 2: the final image (small) ----------
FROM nginx:alpine                  # a tiny web server, about 50 MB
COPY --from=build /app/dist /usr/share/nginx/html   # take ONLY the finished files from stage 1
```
- `AS build` names the first stage. (The first stage is often called the **builder**.)
- `COPY --from=build …` = copy files **from that earlier stage**, not from your Mac.
- The final image contains nginx + your finished files. **No Node.js, no npm, none of the 1 GB.**

**Analogy:** a cake. You need a big kitchen, mixing bowls, and an oven to **make** it, but you **deliver** just the cake in a box. You don't ship the whole kitchen to the customer.

### ✅ Check 40

**40.1** What's the main benefit of a multi-stage build?
- A) Builds run in parallel on several machines
- B) The final image leaves out the build tools
- C) Containers start up in two separate stages
- D) Base images update themselves automatically

**40.2** (one line) Name two things you'd put in `.dockerignore`, and why.

---

## Concept 41 · Making images small and safe, tagging, and pushing

**Words you'll learn:** image optimisation, attack surface, vulnerability, alpine, slim, non-root user, `USER`, `docker history`, push, `docker tag`, `docker login`.

### Step 1: why small images matter
**Image optimisation** = making images **smaller, faster to build, and safer**.
- **Faster deploys:** servers download smaller images faster.
- **Less storage** used everywhere.
- **Safer:** a **vulnerability** is a known weakness that attackers can use. Every extra program in an image could contain one. The total of everything an attacker could target is called the **attack surface**. Fewer programs = a smaller attack surface.

### Step 2: the techniques

| Technique | Example | Why it helps |
|---|---|---|
| **Small base image** | `python:3.12-slim` instead of `python:3.12`; or **`alpine`** variants | Fewer pre-installed programs (alpine Linux is ~5 MB) |
| **Clean up in the same RUN** | `RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*` | Leftover download files are deleted **in the same layer**, so they never get saved |
| **Don't keep installer caches** | `pip install --no-cache-dir …` | Same idea, for Python |
| **Good line order** | requirements before code | Faster rebuilds (cache) |
| **`.dockerignore`** | ignore `.git`, logs | Less junk copied in |
| **Multi-stage builds** | build in stage 1, copy the result | No build tools in the final image |
| **Run as a non-root user** | `USER appuser` | Safety (Step 3) |
| **Specific tags, not `latest`** | `FROM python:3.12-slim` | The same image every time, with no surprise changes |

⚠️ **Common mistake:** splitting one RUN into many RUN lines **doesn't** make an image smaller. Every RUN adds another layer, and deleting a file in a *later* layer doesn't remove it from the earlier layer where it was saved.

### Step 3: run as a non-root user (security)
By default, the main process inside a container runs as **root**, the all-powerful admin user from the Linux session. If an attacker breaks into your app, they'd have root powers inside the container.

The **`USER`** instruction switches to a normal user, so a break-in gets far fewer powers (**least privilege**, from the Linux session):
```dockerfile
RUN useradd --create-home appuser     # create a normal user
USER appuser                          # everything after this (and the app) runs as appuser
```

### Step 4: seeing image sizes and layers
- `docker images`: the DISK USAGE column shows each image's size.
- `docker history hello-app:1.0`: lists every **layer**, which instruction made it, and its size. Great for finding what makes an image big.

### Step 5: tagging and pushing to a registry (sharing your image)
To share your image (with teammates, or with servers that will run it), you **push** it to a **registry** (Session 02). First, name it with the registry's address, then log in, then push:
```bash
docker tag hello-app:1.0 ghcr.io/<your-github-username>/hello-app:1.0   # give it a second name that includes the registry
docker login ghcr.io                                                    # sign in to the registry
docker push ghcr.io/<your-github-username>/hello-app:1.0                # upload it
```
- **`docker tag`** doesn't copy anything. It adds **another name** pointing to the same image.
- In CI/CD (Oct 17), pipelines do exactly this automatically after every code change.

*(Optional, if you want to try GHCR: `gh auth refresh -s write:packages`, then `gh auth token | docker login ghcr.io -u $(gh api user -q .login) --password-stdin`. Not required for the exam.)*

### 🛠 Do (3 min)
```bash
docker images | grep -E "hello-app|python"
docker history hello-app:1.0
```
*Find the biggest layer in the history. It's usually the base image or the `pip install` step.*

### ✅ Check 41

**41.1** Which change does **NOT** make the final image smaller?
- A) Using `python:3.12-slim` instead of `python:3.12`
- B) Adding `--no-cache-dir` to the `pip install` line
- C) Splitting one long RUN into five RUN lines
- D) Moving the build steps into a separate stage

**41.2** Why add `USER appuser` to a Dockerfile?
- A) It makes the container start noticeably faster
- B) A break-in gets far fewer powers than root
- C) Docker Hub rejects images that run as root
- D) It shrinks the image by removing root's files

---

---

## ➕ Small but important (Dockerfile & images): read before the quiz

| Item | What it is |
|---|---|
| **`HEALTHCHECK`** | Dockerfile instruction that tells Docker how to test whether the app inside is **healthy**, e.g. `HEALTHCHECK CMD curl -f http://localhost:5000/ \|\| exit 1`. `docker ps` then shows `(healthy)` or `(unhealthy)`. Exit code 0 = healthy. *(You saw `Up 4 days (healthy)` on your localstack container!)* Kubernetes' **probes** (Oct 15) are the same idea |
| **`LABEL`** | Adds information (metadata) to an image, e.g. `LABEL maintainer="me@x.com" version="1.0"` |
| **`VOLUME`** | Marks a folder inside the image as a place for persistent data. Docker attaches an anonymous volume there |
| **Image digest** | A unique fingerprint of an exact image: `nginx@sha256:3f1c…`. Unlike a tag (which can be moved to point at a different image), **a digest never changes**. Used when you need exactly the same image every time |
| `docker save` / `docker load` | Export an image to a `.tar` file / import it back (moving images without a registry) |
| `docker build --no-cache` | Ignore the cache and rebuild every layer |
| `docker build -f other.Dockerfile .` | Use a Dockerfile with a different name or location |
| `docker image prune` / `docker system prune` | Delete unused images / unused containers, networks, and images to free disk space. ⚠️ Read the prompt before you confirm |
| **Distroless / scratch** | Minimal base images. `scratch` = completely empty; "distroless" = only the app's runtime, no shell. The smallest and safest options |

**Quick checks:** (1) What does `docker ps` show for a container whose HEALTHCHECK keeps failing? (2) Why might a team deploy `nginx@sha256:…` instead of `nginx:1.27`?
*Answers:* (1) `(unhealthy)` next to the status. (2) A digest always means the exact same image, while a tag can be changed to point at a different one.

---

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory

**Q1.** This Dockerfile was built once. You then change **only a comment** in `app.py` and build again. What happens?
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```
- A) Everything is cached, so the build is instant
- B) `COPY . .` and every step after it run again
- C) Only the `CMD` step runs again
- D) Only the `FROM` step downloads again

**Q2.** A Dockerfile ends with `CMD python app.py`. Every `docker stop` takes exactly 10 seconds. What's the most likely reason?
- A) The image is too large to stop quickly
- B) The Dockerfile has no EXPOSE instruction
- C) PID 1 is a shell that drops SIGTERM
- D) The project folder has no .dockerignore

**Q3.** A team needs the database password while building their image. Which line leaves **the password itself** readable by anyone who runs `docker inspect` on the final image?
- A) `ENV DB_PASS=s3cr3t`
- B) `ARG DB_PASS` (set with `--build-arg`)
- C) `LABEL owner=platform-team`
- D) `EXPOSE 5432/tcp`

**Q4.** A server has an old image tagged `myapp:latest`. You've just pushed a new image tagged `myapp:2.0`. On the server, you run `docker run myapp`. What runs?
- A) `myapp:2.0`, as it's the newest image
- B) An error, as `latest` is unclear
- C) Both images, one after the other
- D) Whatever `myapp:latest` points to

**L1.** Explain `RUN` vs `CMD` in contrast form ("RUN…, whereas CMD…").

**L2.** Why must a Flask app inside a container listen on `0.0.0.0` rather than `127.0.0.1`?
