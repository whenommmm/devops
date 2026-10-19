# Writing a Dockerfile · everything you need (both templates tested on this Mac)

## The 8 instructions
| Instruction | What it does | Example |
|---|---|---|
| `FROM` | Base image to start from (**always first**) | `FROM python:3.12-slim` |
| `WORKDIR` | Go into a folder inside the image (creates it) | `WORKDIR /app` |
| `COPY` | Copy files from your folder into the image | `COPY . .` |
| `RUN` | Run a command **while building** (install things) | `RUN pip install -r requirements.txt` |
| `ENV` | Environment variable inside the container | `ENV APP_ENV=production` |
| `EXPOSE` | Documents the app's port (`-p` actually opens it) | `EXPOSE 5000` |
| `CMD` | Command run **when the container starts** | `CMD ["python", "app.py"]` |
| `ARG` | Variable for **build time only** (`--build-arg`) | `ARG VERSION=1.0` |

**Order:** FROM → WORKDIR → COPY → RUN → (ENV / EXPOSE) → CMD · *"From Where? Copy, Run, then Command."*

## Template A · static HTML page
```dockerfile
FROM nginx:alpine
WORKDIR /usr/share/nginx/html
COPY index.html .
CMD ["nginx", "-g", "daemon off;"]
```
Remember the path `/usr/share/nginx/html` (where nginx serves pages from). `daemon off;` keeps nginx in the foreground so the container keeps running.

## Template B · Python app
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV APP_ENV=production
EXPOSE 5000
CMD ["python", "app.py"]
```
Copy `requirements.txt` and install **before** `COPY . .` so code edits don't re-run the slow install (layer cache). The app must listen on `0.0.0.0`.

## .dockerignore (next to the Dockerfile)
```
.git
.env
__pycache__
*.log
```

## Build → run → verify
```bash
docker build -t myapp:1.0 .
docker run -d --name myapp -p 9080:5000 myapp:1.0
curl http://localhost:9080
```
(nginx template: `-p 9080:80`. On this Mac use 9080; in the exam 8080 is fine.)

## Exam traps
| Trap | Right answer |
|---|---|
| RUN vs CMD | RUN = during build; CMD = when the container starts |
| CMD vs ENTRYPOINT | Args after the image name replace CMD but are added to ENTRYPOINT |
| `EXPOSE` without `-p` | Not reachable from the host; EXPOSE only documents |
| Shell-form `CMD python app.py` | Works, but `docker stop` waits 10 s; prefer exec form `["python","app.py"]` |
| Smaller image | `-slim`/`alpine`, `--no-cache-dir`, multi-stage, `.dockerignore` |
| ENV vs ARG | ENV stays in the image; ARG only during the build |

---

# Likely exam questions about Dockerfiles, with answers

## 1. MCQs from the question bank (certain)
| Question | Answer |
|---|---|
| Which command builds an image from a Dockerfile? | **docker build** |
| Which instruction sets the base image? | **FROM** |
| Which instruction copies files into an image? | **COPY** |
| What is the purpose of .dockerignore? | **Exclude paths from the build context** |
| Which instruction defines a default command? | **CMD** |

## 2. Differences (short answers)
- **RUN vs CMD:** RUN executes once while the image is built and saves the result (e.g. installing packages), whereas CMD is the command run each time a container starts.
- **CMD vs ENTRYPOINT:** CMD is a default that `docker run` arguments replace, whereas ENTRYPOINT is the fixed main command and arguments are added to it.
- **COPY vs ADD:** COPY only copies local files; ADD can also fetch URLs and unpack `.tar` files. Prefer COPY.
- **ENV vs ARG:** ENV stays in the image and is visible to the app; ARG exists only during the build (`--build-arg`).
- **EXPOSE:** only documents the port; `-p host:container` actually publishes it.
- **Layers / cache:** each instruction is a cached layer; copy `requirements.txt` and install before `COPY . .` so code edits don't re-run the install.
- **Multi-stage build:** several `FROM` stages; build with full tools, then copy only the output into a small final image → smaller, safer image.
- **Smaller images:** slim/alpine base, multi-stage, `.dockerignore`, `--no-cache-dir`, combine `RUN`s.
- **USER:** run as a non-root user (least privilege).

## 3. Writing tasks
HTML page: see Template A above. With ENV + .dockerignore:
```dockerfile
FROM nginx:alpine
WORKDIR /usr/share/nginx/html
ENV APP_VERSION=1.0
COPY . .
CMD ["nginx", "-g", "daemon off;"]
```
Multi-stage:
```dockerfile
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

## 4. Find the bug
| Symptom / line | Fix |
|---|---|
| `COPY` before `FROM` | `FROM` must be first |
| `RUN python app.py` as last line | Use `CMD ["python", "app.py"]` (RUN is build-time) |
| `EXPOSE 5000`, no `-p`, curl fails | `docker run -p 8080:5000 …` |
| App listens on 127.0.0.1 | Listen on `0.0.0.0` |
| pip install re-runs on every code edit | Copy requirements + install before `COPY . .` |
| `.env` ends up in the image | Add `.env` to `.dockerignore`; never put secrets in `ENV` |
| `docker stop` takes 10 s | Exec form `CMD ["python", "app.py"]` |
| `COPY requirements.txt .` file not found | Not in the build context or listed in `.dockerignore` |

## 5. Predict the output
- `ENTRYPOINT ["echo","hi"]` + `docker run img there` → **hi there**
- `ENTRYPOINT ["ping"]` + `CMD ["localhost"]` + `docker run img google.com` → **ping google.com**
- `CMD ["echo","hello"]` + `docker run img date` → runs **date**
- Edit `app.py` with `COPY . .` before the install → **`COPY . .` and every later step rebuild**

## 6. Write the command
| Task | Command |
|---|---|
| Build `web:1.0` from this folder | `docker build -t web:1.0 .` |
| Build with an argument | `docker build --build-arg VERSION=2.0 -t web:2.0 .` |
| Show the layers | `docker history web:1.0` |
| Override ENV at run time | `docker run -e APP_ENV=dev web:1.0` |
| Run another command instead of CMD | `docker run web:1.0 sh` (`--entrypoint sh` replaces ENTRYPOINT) |
