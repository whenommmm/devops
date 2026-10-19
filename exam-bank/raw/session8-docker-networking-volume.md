# Labs · Session 8: Docker networking, volumes and Compose

## Lab 1 · Custom network + two containers that talk to each other by name
**Answer (what to write in the exam):**
```bash
docker network create appnet                                    # user-defined bridge network
docker run -d --name web    --network appnet nginx              # container 1: web server
docker run -d --name client --network appnet alpine sleep 3600  # container 2: a client that stays up
docker network ls                                               # appnet is listed (driver: bridge)
docker network inspect appnet                                   # "Containers": web and client, with IPs
docker exec client ping -c 2 web                                # name "web" resolves to its IP
docker exec client wget -qO- http://web                         # fetch Nginx's page BY NAME
docker rm -f web client && docker network rm appnet             # clean up
```

**What each line does:**
- `docker network create appnet`: makes a new network; default driver = `bridge` (a private virtual switch on the host). A user-defined bridge gives built-in DNS (Domain Name System): containers find each other by container name.
- `--network appnet`: attaches the container to that network instead of the default `bridge`.
- `alpine sleep 3600`: tiny Linux image; `sleep` keeps it running for 1 hour so we can `exec` into it.
- `docker network inspect appnet`: shows the subnet and which containers are attached, with their IPs.
- `docker exec client ping -c 2 web`: runs ping inside `client`; `-c 2` = send 2 packets. Success proves name → IP resolution.
- `docker exec client wget -qO- http://web`: HTTP (HyperText Transfer Protocol) request to the `web` container on port 80; `-q` quiet, `-O-` print the page to the screen.
- No `-p` is needed: containers on the same network reach each other's ports directly; `-p` is only for access from the host.

**Verify:** ping shows `PING web (172.18.0.2)` and `0% packet loss`; wget prints HTML containing `Welcome to nginx!`.

**🧠 In simple words:** We build a private network and put two containers on it. Because it is a user-defined bridge network, Docker runs a small DNS server for it, so `client` can reach the other container just by typing `web` instead of an IP address that may change. (On the default `bridge` network, names do NOT resolve; only IPs work.) In short: `docker network create` + `--network` on both containers = they talk by name.

## Lab 2 · Database with a named volume: add data, delete + recreate container, data survives
**Answer (what to write in the exam):**
```bash
docker volume create dbdata                       # named volume (optional: -v would create it)
docker run -d --name db -e POSTGRES_PASSWORD="$DB_PASSWORD" \
  -v dbdata:/var/lib/postgresql/data postgres:16-alpine
sleep 8                                           # wait until Postgres is ready
docker exec db psql -U postgres -c "CREATE TABLE notes(msg text); INSERT INTO notes VALUES ('hello');"
docker rm -f db                                   # delete the container
docker volume ls                                  # dbdata is STILL there
docker run -d --name db -e POSTGRES_PASSWORD="$DB_PASSWORD" \
  -v dbdata:/var/lib/postgresql/data postgres:16-alpine   # brand-new container, same volume
sleep 5
docker exec db psql -U postgres -c "SELECT * FROM notes;"   # → hello
```
- Set the password once in the shell first (not saved in any file you submit): `export DB_PASSWORD=pass`.
- Better wait than `sleep`: `until docker exec db pg_isready -U postgres -h localhost; do sleep 1; done` (`-h localhost` checks over TCP (Transmission Control Protocol, a real network connection), which only opens when the real server is up).

Same thing as the one-liner already verified on your Mac:
```bash
docker run -d --name db -e POSTGRES_PASSWORD=pass -v dbdata:/var/lib/postgresql/data postgres:16-alpine && sleep 8 && docker exec db psql -U postgres -c "CREATE TABLE notes(msg text); INSERT INTO notes VALUES ('hello');" && docker rm -f db && docker run -d --name db -e POSTGRES_PASSWORD=pass -v dbdata:/var/lib/postgresql/data postgres:16-alpine && sleep 5 && docker exec db psql -U postgres -c "SELECT * FROM notes;"
```

**What each line does:**
- `docker volume create dbdata`: a named volume = storage managed by Docker that lives outside any container.
- `-e POSTGRES_PASSWORD=...`: the Postgres image refuses to start without it (it sets the `postgres` superuser password).
- `-v dbdata:/var/lib/postgresql/data`: mounts the volume on Postgres's data directory, so the database files are written into the volume, not into the container's own layer.
- `sleep 8`: Postgres needs a few seconds to initialise on first start; `psql` before that fails ("is the server running?").
- `docker exec db psql -U postgres -c "..."`: psql = PostgreSQL's command-line client; `-U postgres` = user, `-c` = run this SQL (Structured Query Language) and exit.
- `docker rm -f db`: deletes the container. Volumes are NOT deleted unless you use `docker rm -v` or `docker volume rm`.
- The second `docker run`: a fresh container, same volume, so it finds the existing database (no re-initialisation).

**Verify:** the last command prints:
```text
  msg
-------
 hello
(1 row)
```
`docker volume inspect dbdata` shows its `Mountpoint`. Clean up: `docker rm -f db && docker volume rm dbdata`.

**🧠 In simple words:** A container's own files disappear when the container is deleted, so we keep the database files in a named volume instead. We create a table, throw the container away, start a brand-new one pointing at the same volume, and the row is still there. In short: data lives in the volume, not the container, so deleting the container does not delete the data.

## Lab 3 · compose.yaml for web + database: start, check they talk, logs, shut down
**Answer (what to write in the exam):**
```bash
mkdir webapp && cd webapp
echo 'DB_PASSWORD=change-me' > .env        # secret lives here, not in compose.yaml
echo '.env' >> .gitignore                  # never commit it
# create compose.yaml (below)
docker compose config                      # check the file + see ${DB_PASSWORD} filled in
docker compose up -d                       # create network + volume, start db then web
docker compose ps                          # both services "running"; web shows 0.0.0.0:8080->80
curl http://localhost:8080                 # web answers from the host
docker compose exec web ping -c 2 db       # web reaches db BY SERVICE NAME
docker compose exec db pg_isready -U postgres     # → accepting connections
docker compose logs db                     # logs of one service (logs -f = follow)
docker compose down                        # stop + remove containers and network (volume kept)
```
(On your Mac use 9080: `"9080:80"` in compose.yaml and `curl http://localhost:9080`.)

`compose.yaml`:
```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      - db
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - dbdata:/var/lib/postgresql/data

volumes:
  dbdata:
```

**What each line does:**
- `services:` → `web`, `db`: each service becomes a container; the service name is also its DNS (Domain Name System) name on the network Compose creates (`<folder>_default`).
- `ports: "8080:80"`: publish host port 8080 → container port 80 (same as `-p`). `db` has no `ports`: only other services need to reach it.
- `depends_on: - db`: start `db` before `web`. It waits for the container to START, not for Postgres to be READY (for that, add a `healthcheck` + `condition: service_healthy`).
- `POSTGRES_PASSWORD: ${DB_PASSWORD}`: Compose reads `DB_PASSWORD` from the `.env` file in the same folder, so no password is hard-coded in the YAML (YAML Ain't Markup Language) file.
- `volumes: - dbdata:/var/lib/postgresql/data` + top-level `volumes: dbdata:`: a named volume (created as `<folder>_dbdata`) so data survives `down`/`up`.
- `docker compose up -d` / `ps` / `exec` / `logs` / `down`: start in background / list / run a command in a service / show output / tear down. `down -v` also deletes the volume.

**Verify:** `docker compose ps` shows `web` and `db` Up; ping prints `0% packet loss`; `pg_isready` prints `/var/run/postgresql:5432 - accepting connections`; after `down`, `docker compose ps` is empty but `docker volume ls` still lists `webapp_dbdata`.

**🧠 In simple words:** Instead of typing several long `docker run` commands, we describe the whole app (a web server, a database, their network and storage) in one `compose.yaml`, and Compose builds all of it with `up` and removes it with `down`. Compose puts both services on one network where they find each other by service name, and the password comes from `.env` so it is never written in the config. In short: `compose.yaml` = the app's blueprint; `up -d` to start, `logs` to debug, `down` to stop.
