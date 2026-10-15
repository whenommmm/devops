# Answer key · Session 04 · Volumes, networking, YAML & Compose

⚠️ Only open after answering.

---

## Warm-up

**W1.** `ENTRYPOINT ["echo", "hi"]`, run as `docker run img there`
- A) `hi`
- B) `there`
- C) `hi there`
- D) An error

**Correct: C.** Arguments after the image name are **added** to ENTRYPOINT, so it runs `echo hi there`.

---

**W2.** You change `requirements.txt`. Which steps rerun?
- A) Only the final COPY of the code
- B) Only the FROM line at the very top again
- C) Nothing, because Docker caches every layer
- D) The requirements COPY and all after it

**Correct: D.** The first changed layer and **every layer after it** rebuild, so pip install runs again too. A forgets that the requirements step itself changed. C: the cache only helps for steps **above** the change.

---

**W3.** `DIR="my files"`. Which treats it as one folder name?
- A) `ls "$DIR"`
- B) `ls $DIR`
- C) `ls '$DIR'`
- D) `ls ${DIR}`

**Correct: A.** Only double quotes keep the space inside one word. B and D split it into `my` and `files` (braces don't protect spaces). C looks for a folder literally named `$DIR`.

---

## Check 42

**42.1** A database with no volume, `docker rm -f db`, then a new one. Where's the data?
- A) Still there, restored from the old image
- B) Gone, with the removed writable layer
- C) Saved on Docker Hub until next pull
- D) Moved to /var/log inside the new one

**Correct: B.** Without a volume, the data lived in the container's ephemeral writable layer. The image never contained it (A), and nothing is uploaded automatically (C).

---

**42.2** `-v /Users/vansh/code:/app` is a:
- A) Bind mount
- B) Named volume
- C) tmpfs mount
- D) Port mapping

**Correct: A.** The source is a **path** on the Mac, so it's a bind mount. A plain name (like `dbdata`) would be a named volume (B).

---

## Check 43

**43.1** `web` → `db` on user-defined network `appnet`. Which address?
- A) `localhost:5432`
- B) `127.0.0.1:5432`
- C) `appnet:5432`
- D) `db:5432`

**Correct: D.** Docker's DNS on user-defined networks turns the container name `db` into its IP. A and B both mean **web itself** (loopback). C is the network's name, not a container's.

---

**43.2** Two containers on the same user-defined network. Is `-p` needed?
- A) No, -p is only for reaching them from outside
- B) Yes, every container port must be published
- C) Only if the traffic between them uses UDP
- D) Only if one of the two runs a database

**Correct: A.** Containers on the same network reach each other directly on their own ports. `-p` publishes to the **host** (your Mac or users).

---

## Check 44

**44.1** The YAML bug
**Model answer:** *"Inconsistent indentation: `image` is indented 3 spaces but `ports` is indented 4, so they're not at the same level. Both must line up under `web` (e.g., 4 spaces each)."*

---

**44.2** Which is a list of two ports?
- A) `ports: 8080, 9090` written on a single line
- B) `ports = [8080 9090]` written on a single line
- C) `ports:`, then `- 8080` and `- 9090` on own lines
- D) `- ports: 8080 9090` with a dash in front

**Correct: C.** List items each start with `- ` on their own line. A is one text value. B isn't YAML syntax. D is a list containing one map whose value is a single piece of text.

---

## Check 45

**45.1** Why does the hostname `cache` work?
- A) `cache` was written into /etc/hosts on the Mac
- B) Redis registers the name `cache` on the internet
- C) The `ports:` line creates the name `cache`
- D) Compose's network resolves service names

**Correct: D.** Compose creates a network where Docker's DNS resolves each **service name** to its container. The Mac's `/etc/hosts` (A) isn't used inside containers. `ports:` (C) only publishes to the host.

---

**45.2** `docker compose down -v` removes:
- A) Only the containers, nothing else
- B) Containers and network, keeping volumes
- C) Containers, network, and named volumes
- D) The images that the services were built from

**Correct: C.** `-v` adds the volumes, so the data is gone. B is plain `down`. Images (D) are kept either way.

---

## Session quiz

**Q1.** `depends_on: [db]`, but web fails with "connection refused" at first and works 10 seconds later. Why?
- A) depends_on waits for start, not readiness
- B) web and db were put on separate networks
- C) db needs `-p 5432:5432` to accept web
- D) The volume for db was never declared

**Correct: A.** The db **container** started, but the database **program** inside needed time to get ready. B: Compose puts services on one shared network automatically. C: containers don't need `-p` to talk to each other. D: a missing volume causes data loss, not connection refusal.

---

**Q2.** Which `-v` is a named volume?
- A) `-v /data:/var/lib/mysql`
- B) `-v dbdata:/var/lib/mysql`
- C) `-v ./data:/var/lib/mysql`
- D) `-v ~/data:/var/lib/mysql`

**Correct: B.** Just a name = a volume. A, C, and D start with `/`, `./`, or `~`, so they're **paths**, which makes them bind mounts.

---

**Q3.** The web container connects to `localhost:6379` for Redis in the `cache` service. The fix?
- A) Connect to `cache:6379` instead
- B) Publish Redis with `-p 6379:6379`
- C) Put Redis on the host network only
- D) Restart Colima and try again

**Correct: A.** Inside `web`, `localhost` is **web itself**. Use the service name. B publishes Redis to the **Mac**, but web's `localhost` still points at web.

---

**Q4.** Two services both map `"8080:80"`. What happens?
- A) Both start and share port 8080
- B) Compose moves one to port 8081
- C) Neither service is allowed to start
- D) The second fails: 8080 taken

**Correct: D.** A host port can have only **one** listener (networking session). The first service gets 8080, and the second fails with `port is already allocated`. Compose never renumbers ports on its own (B).

---

**L1.** Volume vs bind mount
**Model answer:** *"A volume is storage created and managed by Docker, used for things like database data, whereas a bind mount maps a specific folder from the host into the container, used for things like live code editing during development."*

**L2.** YAML full form + two rules
**Model answer:** *"YAML Ain't Markup Language (originally Yet Another Markup Language). Rules: indent with spaces, never tabs; items at the same level must be indented by exactly the same amount."* (Also accepted: put a space after every colon; list items start with `- `.)
