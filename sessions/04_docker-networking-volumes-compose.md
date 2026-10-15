# Session 04 · Docker III: volumes, networking, YAML & Compose (Oct 12, Session 2)

**Time:** ~90–100 min · **Concepts:** 42 → 45 · **Answers:** `MY_ANSWERS.md` or say "Compose questions" · **Answer key:** `04_docker-networking-volumes-compose_ANSWERS.md`

**Before you start:** `colima status`. Reminder: **never remove the `minikube` or `localstack` containers.**

---

## 🔁 Warm-up

**W1.** A Dockerfile has `ENTRYPOINT ["echo", "hi"]`. You run `docker run img there`. What's printed?
- A) `hi`
- B) `there`
- C) `hi there`
- D) An error

**W2.** You build an image, change one line in `requirements.txt`, and build again. The Dockerfile copies `requirements.txt`, runs `pip install`, then copies the code. Which steps run again?
- A) Only the final COPY of the code
- B) Only the FROM line at the very top again
- C) Nothing, because Docker caches every layer
- D) The requirements COPY and all after it

**W3.** `DIR="my files"`. Which command treats it as **one** folder name?
- A) `ls "$DIR"`
- B) `ls $DIR`
- C) `ls '$DIR'`
- D) `ls ${DIR}`

---

## Concept 42 · Keeping data: volumes and bind mounts

**Words you'll learn:** persistent, ephemeral, data loss, volume, named volume, bind mount, mount, tmpfs, `-v`, read-only (`:ro`).

### Step 1: the problem
From Session 02: files a container creates live in its **writable layer**, which is **deleted when the container is removed**.

That's fine for a web server. But a **database** (a program that stores data in an organised way, like users and orders) running in a container would **lose all its data** whenever its container is replaced. And in DevOps, containers get replaced **all the time**: new versions, crashes, moving to other machines.

Two useful words:
- **Ephemeral** = **short-lived, temporary**. A container's writable layer is ephemeral.
- **Persistent** = **lasting**, surviving restarts and replacements. Databases need persistent storage.

### Step 2: mounting (the core idea)
To **mount** = **attach a storage location to a folder**, so whatever is written into that folder actually goes to the attached storage.

**Analogy:** plugging a USB drive into a computer. It appears as a folder. Files you save into that folder go **onto the drive**, so if the computer is thrown away, the drive and your files survive.

Docker lets you mount storage **into a folder inside a container**. The container thinks it's writing to its own folder, but the data really goes **outside** the container, where it survives.

### Step 3: the three kinds

**1. Volume** (also called a **named volume**)
- Storage **created and managed by Docker** itself. You give it a name; Docker decides where it physically lives.
- It **isn't tied to any container**: remove the container, and the volume (and the data) stays.
- **Best for:** databases and any data the app must keep.
```bash
docker volume create dbdata                         # make a volume
docker run -v dbdata:/var/lib/postgresql/data …     # mount it into the container's data folder
docker volume ls                                    # list volumes
docker volume rm dbdata                             # delete it (and its data!)
```

**2. Bind mount**
- **A specific folder from your Mac** mounted into the container. You choose the exact path.
- Changes show up **instantly in both directions**: edit a file on your Mac, and the container sees it right away.
- **Best for:** development, where you edit code on your Mac and the container runs it live.
```bash
docker run -v /Users/you/site:/usr/share/nginx/html …
```

**3. tmpfs mount** (short)
- Storage **in memory (RAM) only**. It disappears when the container stops. It's fast, and used for temporary or sensitive scratch files.

### Step 4: how Docker tells them apart with `-v`
Format: **`-v SOURCE:PATH-INSIDE-CONTAINER`**
- If SOURCE is **just a name** (`dbdata`), it's a **volume**.
- If SOURCE is a **path** (starts with `/` or `~` or `./`), it's a **bind mount**.

Add **`:ro`** (**read-only**) at the end to stop the container changing the files: `-v ~/site:/usr/share/nginx/html:ro`.

| | **Volume** | **Bind mount** |
|---|---|---|
| Who manages where it lives | Docker | You (a folder you pick) |
| `-v` source looks like | a name: `dbdata` | a path: `/Users/you/site` |
| Survives removing the container | ✅ Yes | ✅ Yes (it's just your folder) |
| Typical use | Database data in any environment | Live code editing during development |

### 🛠 Do (~8 min)

**A volume keeps data after containers are gone:**
```bash
docker volume create notes-data
docker run --rm -v notes-data:/data alpine sh -c 'echo "saved by container 1" > /data/note.txt'
docker run --rm -v notes-data:/data alpine cat /data/note.txt
```
*Line 2: a container writes a file into the volume, then **deletes itself** (`--rm`). Line 3: a **brand-new** container reads the file, and it's still there: `saved by container 1`.*
*(`sh -c '…'` = run this text as a shell command inside the container. Note the single quotes: the outer quotes decide!)*

**A bind mount shows your Mac's files live:**
```bash
mkdir -p ~/devops-practice/site
echo '<h1>Bind mount works</h1>' > ~/devops-practice/site/index.html
docker run -d --name site -p 8081:80 -v ~/devops-practice/site:/usr/share/nginx/html:ro nginx
curl http://localhost:8081
echo '<h1>Edited on my Mac!</h1>' > ~/devops-practice/site/index.html
curl http://localhost:8081
docker rm -f site
```
*The second curl shows the **edited** page, with no rebuild and no restart. nginx is serving files straight from your Mac folder.*

### ✅ Check 42

**42.1** A database container runs with **no** volume. You add data, run `docker rm -f db`, and start a fresh database container. Where's the data?
- A) Still there, restored from the old image
- B) Gone, with the removed writable layer
- C) Saved on Docker Hub until next pull
- D) Moved to /var/log inside the new one

**42.2** `-v /Users/vansh/code:/app` is a:
- A) Bind mount
- B) Named volume
- C) tmpfs mount
- D) Port mapping

---

## Concept 43 · Docker networking: how containers talk

**Words you'll learn:** network driver, bridge network, default bridge, user-defined bridge, Docker DNS, host network, none, overlay, container-to-container communication, publish (recap).

### Step 1: each container has its own network
From Session 02: **namespaces** give each container its **own private network**, with its **own IP address**, usually something like `172.17.0.2`.

From the networking session: `172.16.x.x` to `172.31.x.x` is a **private** range. So container IPs are private: reachable only inside Docker's networks, not from the internet or directly from your Mac.

### Step 2: network drivers (the kinds of Docker networks)
A **network driver** is **the type of network Docker creates**.

| Driver | What it is | When it's used |
|---|---|---|
| **bridge** (the default) | A **private virtual network inside the host**. Containers on it get private IPs and can talk to each other | Almost always, on one machine |
| **host** | The container uses the **host's network directly**: no separate IP, no `-p` needed | Rare. Max speed, but no network isolation |
| **none** | **No network at all** | Fully isolated jobs |
| **overlay** | One network **spanning several host machines** | Clusters (awareness only) |

**What "bridge" means:** in networking, a bridge **connects devices into one local network**, like a switch. Docker's bridge is a virtual one that connects containers.

### Step 3: the default bridge vs your own bridge (very important)
- If you don't choose a network, containers join the **default bridge** network (named `bridge`). There, containers can reach each other **only by IP address**. **Names don't work.**
- If you **create your own bridge network** (a **user-defined bridge**) and put containers on it, Docker gives that network **built-in DNS**: **containers can reach each other by container name.**

From the networking session: DNS turns **names into IP addresses**. Docker runs a small DNS for each user-defined network, so the name `db` turns into the `db` container's IP. Container IPs change whenever containers are recreated, so **using names is far more reliable**.

### Step 4: container-to-container vs outside-to-container
- **Container → container** (on the same user-defined network): connect directly to the **other container's name and its own port**, like `db:5432`. **No `-p` needed.**
- **Outside → container** (your Mac's browser, or users): needs **`-p HOST:CONTAINER`** to publish the port (Session 02).

⚠️ **Classic trap:** inside a container, **`localhost` means that container itself**, not your Mac and not other containers. (It's the loopback idea from the networking session.) A web container connecting to `localhost:5432` looks for a database **inside itself**, and fails.

```
                     Your Mac ──(-p 8080:5000)──┐
                                                ▼
┌──────────────── user-defined network "appnet" ─────────────────┐
│   [ web ]  ── connects to "db:5432" ──►  [ db ]                 │
│   (Docker DNS turns "db" into db's private IP; no -p needed)    │
└─────────────────────────────────────────────────────────────────┘
```

**Commands:**
```bash
docker network create appnet                 # make a user-defined bridge
docker run -d --name db --network appnet …   # attach containers with --network
docker network ls                            # list networks
docker network inspect appnet                # details: subnet, which containers are attached
docker network rm appnet                     # delete it
```

### 🛠 Do (~6 min)
```bash
docker network create appnet
docker run -d --name api --network appnet nginx
docker run --rm --network appnet alpine wget -qO- http://api | head -4
```
*A temporary alpine container on the **same** network fetches `http://api` **by name**, and you see nginx's HTML. (`wget -qO-` = download quietly and print to the screen. It's alpine's built-in alternative to curl.)*

```bash
docker run --rm alpine wget -qO- http://api
```
*This one is on the **default** network (no `--network`), so the name means nothing there: `wget: bad address 'api'`. **Names only work on user-defined networks.***

```bash
docker network inspect appnet | grep -i subnet
docker rm -f api && docker network rm appnet
```
*The subnet is something like `172.18.0.0/16`: a private range, written in CIDR from the networking session.*

### ✅ Check 43

**43.1** Containers `web` and `db` are on the user-defined network `appnet`. What address should `web` use to reach db's database on port 5432?
- A) `localhost:5432`
- B) `127.0.0.1:5432`
- C) `appnet:5432`
- D) `db:5432`

**43.2** Two containers on the same user-defined network need to talk to each other. Do they need `-p`?
- A) No, -p is only for reaching them from outside
- B) Yes, every container port must be published
- C) Only if the traffic between them uses UDP
- D) Only if one of the two runs a database

---

## Concept 44 · YAML: the format for config files

**Words you'll learn:** YAML, key, value, key–value pair, map (dictionary), list, nesting, indentation, comment, JSON.

### Step 1: why
From here on, almost **every DevOps tool is configured with YAML files**: Docker Compose (next concept), Kubernetes, GitHub Actions, Helm. Reading and fixing YAML is a **core exam skill**, and indentation bugs are favourite debugging questions.

### Step 2: what YAML is
**YAML** stands for **"YAML Ain't Markup Language"**. (A recursive joke: the name refers to itself. It originally meant "Yet Another Markup Language".)

> **YAML is a human-friendly text format for writing structured information**: settings, lists, and nested groups.

### Step 3: the building blocks

**1. Key–value pairs.** A **key** is a label, and a **value** is its data. Separated by **a colon and a space**:
```yaml
name: web
port: 8080
debug: true
```

**2. Maps (nesting with indentation).** A **map** (also called a **dictionary**) is a group of key–value pairs under one key. You show that something belongs inside something else by **indenting** it (adding spaces at the start of the line):
```yaml
database:
  host: db
  port: 5432
```
Here `host` and `port` belong **inside** `database`.

**3. Lists.** A **list** is several items in order. Each item starts with **`- `** (a dash and a space):
```yaml
ports:
  - "8080:80"
  - "8443:443"
```

**4. A list of maps** (very common in Kubernetes):
```yaml
containers:
  - name: web
    image: nginx
  - name: cache
    image: redis
```
Each `- ` starts a new item, and each item has its own keys.

**5. Comments** start with `#`, and YAML ignores them.

### Step 4: the rules that cause most bugs
1. **Indent with spaces only, never Tab characters.** Tabs cause errors.
2. **Be consistent.** Items at the same level must line up exactly (2 spaces per level is the usual style).
3. **Always put a space after the colon:** `port: 80`, not `port:80`.
4. **Quote values that could be misread:**
   - `"8080:80"`, because a colon inside a value can confuse YAML
   - `"yes"` / `"no"` / `"on"`, which YAML may otherwise turn into true/false
5. **A dash means a list item.** Mixing up a list and a map changes the meaning completely.

### Step 5: YAML vs JSON (awareness)
**JSON** (**JavaScript Object Notation**) is another data format, with braces and brackets. The same data in both:
```yaml
web:                       # YAML
  ports:
    - 80
```
```json
{ "web": { "ports": [80] } }
```
YAML uses **indentation** where JSON uses `{ }` and `[ ]`.

### ✅ Check 44

**44.1 🐞** Find the bug in this Compose snippet (describe it in one line):
```yaml
services:
  web:
   image: nginx
    ports:
      - "8080:80"
```

**44.2** Which one is a **list** of two ports?
- A) `ports: 8080, 9090` written on a single line
- B) `ports = [8080 9090]` written on a single line
- C) `ports:`, then `- 8080` and `- 9090` on own lines
- D) `- ports: 8080 9090` with a dash in front

---

## Concept 45 · Docker Compose: running a multi-container app with one file

**Words you'll learn:** multi-container application, Docker Compose, compose file, service, `services`, `image`, `build`, `ports`, `environment`, `volumes`, `depends_on`, project, `docker compose up/down/ps/logs`, Redis.

### Step 1: the problem
Real apps are rarely one container. A typical **multi-container application** might be:
- a **web** app (your code)
- a **database**
- a **cache**

Starting them by hand means long `docker run` commands with networks, volumes, ports, and environment variables, typed correctly and in the right order, every time. It's easy to get wrong, and hard to share with teammates.

### Step 2: Docker Compose
> **Docker Compose** is a tool where you **describe all of an app's containers in one YAML file**, then **start or stop them all with one command**.

- The file is usually called **`compose.yaml`** (older name: `docker-compose.yml`).
- The command is **`docker compose …`**. Older systems used `docker-compose` with a hyphen. Same idea.
- Each container in the file is called a **service**.

**Analogy:** instead of phoning the caterer, the band, and the decorator separately with instructions every time, you write **one event plan** and hand it over: "make it happen".

### Step 3: anatomy of a compose file
```yaml
services:                      # the list of containers (services) in this app
  web:                         # service name: also its DNS name on the network!
    build: .                   # build an image from the Dockerfile in this folder
    ports:
      - "8080:5000"            # publish: Mac 8080 → container 5000
    environment:
      - REDIS_HOST=cache       # env var: tells the app where Redis is (by service name)
    depends_on:
      - cache                  # start "cache" before "web"
  cache:
    image: redis:7-alpine      # use a ready-made image from Docker Hub
    volumes:
      - redis-data:/data       # named volume, so the data survives
volumes:
  redis-data:                  # declare the named volume
```

| Key | Meaning |
|---|---|
| `services` | The containers that make up the app |
| `image` | Use an existing image |
| `build` | Build an image from a Dockerfile (path to the folder) |
| `ports` | `-p` mappings, `"HOST:CONTAINER"` |
| `environment` | `-e` environment variables |
| `volumes` | `-v` mounts (named volumes must also be listed under the top-level `volumes:`) |
| `depends_on` | **Start order**: start these services first |

**Two automatic behaviours:**
1. **Compose creates a network for the app automatically**, and every service joins it. On that network, **each service name works as a DNS name**. That's why `REDIS_HOST=cache` works: `cache` is the Redis service's name. (Concept 43's user-defined-network DNS, done for you.)
2. All of an app's containers, networks, and volumes are grouped as one **project**, named after the folder by default.

⚠️ **The `depends_on` trap:** it only waits for the other container to **start**, **not** for the program inside to be **ready**. A database can take several seconds to get ready after its container starts, so a web app that connects immediately may fail at first.

### Step 4: the commands

| Command | What it does |
|---|---|
| `docker compose up -d` | Build (if needed), create, and start **everything**, in the background |
| `docker compose ps` | List this app's containers |
| `docker compose logs -f` | Show all services' logs, live (`-f` = follow) |
| `docker compose exec web sh` | Open a shell inside the `web` service |
| `docker compose stop` | Stop everything (containers kept) |
| `docker compose down` | Stop **and remove** the containers and the network. **Volumes are kept** |
| `docker compose down -v` | Same, **plus delete the volumes**, so ⚠️ **the data is lost** |
| `docker compose up -d --build` | Rebuild images first (after you change code) |

**Dockerfile vs Compose:** a **Dockerfile** describes how to **build ONE image**. A **compose file** describes how to **run SEVERAL containers together**. They work together: a compose service can `build:` from a Dockerfile.

**Compose vs Kubernetes:** Compose runs containers on **one machine**. To run them across **many machines**, with automatic healing and scaling, you need **Kubernetes**, which starts tomorrow. Many Compose ideas (services, ports, env vars, volumes) reappear there.

### Step 5: Redis (the extra service in the exercise)
**Redis** is a very fast **in-memory data store**: it keeps data in RAM for speed, and can also save it to disk. It's often used as a **cache** (a short-term memory, from the DNS concept) or as a counter. We'll use it to count page visits.

### 🛠 Do (~15 min): a two-container app

```bash
mkdir -p ~/devops-practice/counter && cd ~/devops-practice/counter
code app.py
```
Paste and save:
```python
import os
from flask import Flask
import redis                                   # library for talking to Redis

app = Flask(__name__)
r = redis.Redis(host=os.environ.get("REDIS_HOST", "localhost"), port=6379)

@app.route("/")
def home():
    count = r.incr("visits")                   # add 1 to "visits" in Redis, and get the new number
    return f"Hello! This page has been visited {count} times.\n"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```
```bash
code requirements.txt
```
```
flask==3.0.3
redis==5.0.8
```
```bash
code Dockerfile
```
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```
```bash
code compose.yaml
```
Paste the compose file from **Step 3** (the whole thing, starting at `services:`), and save.

**Run the whole app:**
```bash
docker compose up -d
docker compose ps
curl http://localhost:8080
curl http://localhost:8080
curl http://localhost:8080
```
*The count goes up: 1, 2, 3. The web container reached Redis by the **name** `cache`.*

**Prove the volume keeps data:**
```bash
docker compose down          # removes the containers + network (volume kept)
docker compose up -d
curl http://localhost:8080   # continues from 4, because the data survived in the volume
docker compose logs web | tail -5
docker compose down -v       # removes the volume too, so the count is reset next time
```

### ✅ Check 45

**45.1** In the compose file, the web app reaches Redis using the hostname `cache`. Why does that work?
- A) `cache` was written into /etc/hosts on the Mac
- B) Redis registers the name `cache` on the internet
- C) The `ports:` line creates the name `cache`
- D) Compose's network resolves service names

**45.2** What does `docker compose down -v` remove?
- A) Only the containers, nothing else
- B) Containers and network, keeping volumes
- C) Containers, network, and named volumes
- D) The images that the services were built from

---

---

## ➕ Small but important (containers, volumes, networks, Compose): read before the quiz

| Item | What it is |
|---|---|
| **Restart policies** | `docker run --restart …` decides whether Docker restarts a container automatically: **`no`** (default), **`on-failure`** (only if it exits with a non-zero code), **`always`**, **`unless-stopped`** (always, except after you stopped it yourself). In Compose: `restart: unless-stopped` |
| **Resource limits** | `docker run --memory 512m --cpus 1.5 …` caps memory and CPU (this is **cgroups**, from Session 02). Going over the memory limit gets the container **killed** (OOM) |
| `docker cp` | Copy files between your Mac and a container: `docker cp web:/etc/nginx/nginx.conf .` |
| `docker inspect` | Every detail of a container: its IP, mounts, environment variables, restart policy |
| `docker volume inspect` | Where a volume lives and its details |
| **Anonymous volume** | A volume with no name (Docker makes up an ID). It's hard to find again, which is why named volumes are preferred |
| `-p 127.0.0.1:8080:80` | Publish the port **only on your Mac's loopback**, so other devices on the network can't reach it (networking session!) |
| `docker network connect appnet web` | Attach an existing container to another network |
| **`docker0`** | The name of the Linux network interface behind Docker's default bridge network |
| **`env_file:` / `.env`** | Compose can read environment variables from a file (`env_file: .env`) instead of listing them. Keep `.env` out of Git |
| `depends_on` + `condition: service_healthy` | Makes Compose wait until the other service's **healthcheck passes**, not just until it starts. This fixes the readiness trap |
| `docker compose config` | Prints the final, fully-filled-in compose file. Great for catching YAML mistakes |
| `docker compose up --scale web=3` | Run 3 copies of a service (the ports must not clash) |

**Quick checks:** (1) Which restart policy brings a container back after a crash, but not after you deliberately stopped it? (2) How do you make Compose wait until the database is actually ready?
*Answers:* (1) `unless-stopped` (`on-failure` also restarts after crashes, but only when the exit code is non-zero). (2) Give the db service a `healthcheck`, and use `depends_on` with `condition: service_healthy`.

---

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory

**Q1.** In a compose file, `web` has `depends_on: [db]`. On `docker compose up`, web crashes with "connection refused" to db, but works if restarted 10 seconds later. Why?
- A) depends_on waits for start, not readiness
- B) web and db were put on separate networks
- C) db needs `-p 5432:5432` to accept web
- D) The volume for db was never declared

**Q2.** Which `-v` creates a **named volume**?
- A) `-v /data:/var/lib/mysql`
- B) `-v dbdata:/var/lib/mysql`
- C) `-v ./data:/var/lib/mysql`
- D) `-v ~/data:/var/lib/mysql`

**Q3.** Inside the `web` container, the app tries to connect to Redis at `localhost:6379`. Redis runs in the `cache` service on the same Compose network. It fails. What's the fix?
- A) Connect to `cache:6379` instead
- B) Publish Redis with `-p 6379:6379`
- C) Put Redis on the host network only
- D) Restart Colima and try again

**Q4.** Two services in one compose file both have `ports: - "8080:80"`. What happens on `docker compose up`?
- A) Both start and share port 8080
- B) Compose moves one to port 8081
- C) Neither service is allowed to start
- D) The second fails: 8080 taken

**L1.** Explain volume vs bind mount in contrast form, with one typical use for each.

**L2.** Write the full form of YAML, and two indentation rules.
