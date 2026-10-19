# Labs · Sessions 6–7: Docker basics (images, containers, Dockerfile)

## Lab 1 · Run Nginx on host port 8080 → container port 80, curl it, read its logs
**Answer (what to write in the exam):**
```bash
docker pull nginx                              # download the image (optional: run pulls it anyway)
docker run -d --name web -p 8080:80 nginx      # start the container in the background
docker ps                                      # confirm it is "Up" with 0.0.0.0:8080->80/tcp
curl http://localhost:8080                     # fetch the page through the mapped port
docker logs web                                # see Nginx's access log (your curl request)
docker logs -f --tail 5 web                    # optional: follow live, last 5 lines (Ctrl+C to stop)
docker rm -f web                               # clean up: stop + delete the container
```
(On your Mac use 9080: `-p 9080:80` and `curl http://localhost:9080`, because 8080 is taken.)

**What each line does:**
- `docker pull nginx`: downloads the `nginx:latest` image from Docker Hub (the default registry).
- `docker run -d --name web -p 8080:80 nginx`: creates and starts a container. `-d` = detached (runs in background), `--name web` = a fixed name so later commands can use it, `-p HOST:CONTAINER` = publish port: traffic to host port 8080 goes to container port 80 (where Nginx listens).
- `docker ps`: lists running containers; the PORTS column proves the mapping.
- `curl http://localhost:8080`: curl (Client URL) sends an HTTP (HyperText Transfer Protocol) request from the host.
- `docker logs web`: prints what the container wrote to stdout/stderr (standard output / standard error). Nginx sends its access log there. `-f` = follow, `--tail N` = only the last N lines.
- `docker rm -f web`: `-f` force-removes a running container (stop + delete in one step).

**Verify:** `curl` prints HTML containing `<title>Welcome to nginx!</title>`; `docker logs web` shows a line like `"GET / HTTP/1.1" 200 615 "-" "curl/8.x"`.

**🧠 In simple words:** We start the official Nginx web server inside a container and "punch a hole" from the host's port 8080 to the container's port 80, so a request to localhost:8080 reaches Nginx. Then we prove it works with curl and read the container's output with `docker logs`, which shows our request was served. In short: `-p 8080:80` = host port first, container port second; `docker logs` = the container's stdout/stderr.

## Lab 2 · Dockerfile (FROM, WORKDIR, COPY, CMD) for an HTML page → build, run, verify
**Answer (what to write in the exam):**
```bash
mkdir mysite && cd mysite
echo '<h1>Hello from my Docker image</h1>' > index.html
# create the Dockerfile shown below (e.g. with nano Dockerfile / vi Dockerfile)
docker build -t mysite:1.0 .                         # build the image from ./Dockerfile
docker images mysite                                 # image is listed with tag 1.0
docker run -d --name mysite -p 8080:80 mysite:1.0    # run it
curl http://localhost:8080                           # → <h1>Hello from my Docker image</h1>
```
(On your Mac use 9080: `-p 9080:80` and `curl http://localhost:9080`.)

`Dockerfile`:
```dockerfile
FROM nginx:alpine
WORKDIR /usr/share/nginx/html
COPY index.html .
CMD ["nginx", "-g", "daemon off;"]
```

**What each line does:**
- `FROM nginx:alpine`: base image = Nginx on Alpine Linux (small). Every Dockerfile starts with FROM.
- `WORKDIR /usr/share/nginx/html`: sets (and creates if missing) the working directory inside the image; later COPY/RUN/CMD use it. This is the folder Nginx serves HTML (HyperText Markup Language) files from.
- `COPY index.html .`: copies `index.html` from the build context (your folder) into the WORKDIR (`.`), replacing Nginx's default page.
- `CMD ["nginx", "-g", "daemon off;"]`: default command when a container starts. `daemon off;` keeps Nginx in the foreground; if the main process exits, the container stops.
- `docker build -t mysite:1.0 .`: `-t` = name:tag for the image, `.` = build context (the folder sent to the builder, which also holds the Dockerfile).
- `docker run -d --name mysite -p 8080:80 mysite:1.0`: run your own image exactly like Lab 1.

**Verify:** `curl http://localhost:8080` prints `<h1>Hello from my Docker image</h1>`; `docker exec mysite ls /usr/share/nginx/html` lists `index.html`.

**🧠 In simple words:** A Dockerfile is a recipe: start from a ready Nginx image, go to the folder Nginx serves pages from, put our page there, and say which command to run at start. `docker build` turns the recipe into an image, and `docker run` starts a container from it, which we test with curl. In short: Dockerfile → `docker build` → image → `docker run` → container.

## Lab 3 · Add a .dockerignore and an environment variable, rebuild, run, explain
**Answer (what to write in the exam):**
```bash
cd mysite
touch .env debug.log                       # junk files that must NOT go into the image
printf '.git\n.env\n*.log\nDockerfile\n.dockerignore\n' > .dockerignore
# update the Dockerfile as shown below
docker build --build-arg APP_VERSION=2.0 -t mysite:2.0 .
docker rm -f mysite                        # remove the old container (name is reused)
docker run -d --name mysite -p 8080:80 mysite:2.0
docker exec mysite env | grep APP_         # → APP_ENV=production, APP_VERSION=2.0
docker exec mysite ls -a /usr/share/nginx/html   # no .env, no debug.log, no Dockerfile
docker run -d --name mysite-stg -p 8081:80 -e APP_ENV=staging mysite:2.0   # override at run time
```
(On your Mac use 9080: `-p 9080:80`.)

`.dockerignore`:
```text
.git
.env
*.log
Dockerfile
.dockerignore
```

`Dockerfile`:
```dockerfile
FROM nginx:alpine
ARG APP_VERSION=1.0
ENV APP_ENV=production
ENV APP_VERSION=${APP_VERSION}
WORKDIR /usr/share/nginx/html
COPY . .
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**What each added part does:**
- `.dockerignore`: lists paths left OUT of the build context, so `COPY . .` never copies them. Gives smaller, faster builds and keeps secrets (`.env`) and junk (`*.log`, `.git`) out of the image. (Docker still reads the Dockerfile itself; it just is not copied in.)
- `ARG APP_VERSION=1.0`: a build-time variable with a default; changed with `--build-arg APP_VERSION=2.0`. It exists only during the build.
- `ENV APP_ENV=production` / `ENV APP_VERSION=${APP_VERSION}`: environment variables saved in the image, so every container gets them at run time (the second line copies the ARG value into the image).
- `COPY . .`: copies the whole folder minus the ignored paths.
- `EXPOSE 80`: documents the port the app listens on (it does not publish it; `-p` does).
- `-e APP_ENV=staging`: overrides an ENV value for one container without rebuilding.

**Verify:** `docker exec mysite env | grep APP_` → `APP_ENV=production` and `APP_VERSION=2.0`; `ls -a` shows only `.`, `..`, `index.html` and `50x.html` (no `.env`, no `debug.log`). `docker exec mysite-stg env | grep APP_ENV` → `APP_ENV=staging`.

**🧠 In simple words:** `.dockerignore` works like `.gitignore` for builds: it stops secrets and junk files from ever being sent into the image. ENV bakes a setting into the image that the app can read at run time, ARG is a knob you turn only while building, and `-e` lets you change a setting per container without rebuilding. In short: `.dockerignore` = keep files out; ARG = build-time value; ENV = run-time value baked in; `-e` = override at run.
