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
