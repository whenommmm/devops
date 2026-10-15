# Answer key · Session 03 · Dockerfile

⚠️ Only open after answering.

---

## Warm-up

**W1.** `check() { [ -d /nope ]; }`, then `check`, then `echo $?`
- A) `0`
- B) `1`
- C) Nothing at all
- D) `/nope`

**Correct: B.** A function with no `return` gives back the exit code of its **last command**. `[ -d /nope ]` is false (that folder doesn't exist), so it exits with **1**, and `$?` prints 1.

---

**W2.** Which command shows the exact unstaged lines you changed?
- A) `git status`
- B) `git log -p`
- C) `git diff`
- D) `git show`

**Correct: C.** A only lists **file names**. B shows changes inside **past commits**. D shows the **last commit**. Only `git diff` shows your current, unstaged edits.

---

**W3.** `docker stop api` an hour ago; now `docker run --name api nginx` fails. Why?
- A) A stopped container named api still exists
- B) The nginx image was deleted when api stopped
- C) Docker blocks reusing a name for one hour
- D) The api container is still holding port 80

**Correct: A.** Stopped containers still exist and keep their names. Remove it (`docker rm api`) or start it (`docker start api`). D: stopped containers release their ports, and this command doesn't even use `-p`.

---

## Check 38

**38.1** `EXPOSE 5000`, run without `-p`. Can the Mac open `localhost:5000`?
- A) Yes, EXPOSE publishes it on the Mac
- B) No, EXPOSE only documents the port
- C) Yes, once the container is restarted
- D) No, port 5000 only works with --rm

**Correct: B.** EXPOSE is a note for humans and tools. Only `-p` publishes a port to the Mac.

---

**38.2** You edit `app.py` often. Which order rebuilds fastest?
- A) `COPY . .` → `RUN pip install -r requirements.txt`
- B) `COPY requirements.txt .` → `RUN pip install …` → `COPY . .`
- C) `RUN pip install …` → `COPY requirements.txt .` → `COPY . .`
- D) Any order, since Docker caches every layer anyway

**Correct: B.** The slow install sits above your code, so code edits don't break its cache. A: every code change re-runs the install. C fails, because pip can't read `requirements.txt` before it's copied in. D: once a layer changes, every later layer rebuilds.

---

## Check 39

**39.1** `ENTRYPOINT ["ping"]` + `CMD ["localhost"]`, run as `docker run myping google.com`. What runs?
- A) `ping localhost`
- B) `ping google.com`
- C) `google.com` on its own
- D) `ping localhost google.com`

**Correct: B.** What you type after the image name **replaces CMD** and is **added to ENTRYPOINT**. A would happen with no argument. C would need `--entrypoint`. D: CMD gets replaced, not added to.

---

**39.2** Which is correct?
- A) RUN executes each time a container starts
- B) CMD executes once, while the image is built
- C) RUN runs at build; CMD runs at start
- D) RUN and CMD both execute only during the build

**Correct: C.** A and B swap the two.

---

## Check 40

**40.1** Main benefit of multi-stage builds?
- A) Builds run in parallel on several machines
- B) The final image leaves out the build tools
- C) Containers start up in two separate stages
- D) Base images update themselves automatically

**Correct: B.** Only the last stage becomes the image, so it's smaller and safer. C mixes up build stages with how containers start.

---

**40.2** Two things for `.dockerignore`, and why
**Model answer:** *"`.git` (large, and not needed in the image, so builds are faster and images smaller) and `.env` (holds secrets that must never be copied into an image)."* Also accepted: `*.log`, `node_modules`, `__pycache__`.

---

## Check 41

**41.1** Which does NOT make the image smaller?
- A) Using `python:3.12-slim` instead of `python:3.12`
- B) Adding `--no-cache-dir` to the `pip install` line
- C) Splitting one long RUN into five RUN lines
- D) Moving the build steps into a separate stage

**Correct: C.** More RUN lines = more layers. Files saved in an earlier layer stay in the image even if a later layer deletes them. A, B, and D all genuinely shrink it.

---

**41.2** Why `USER appuser`?
- A) It makes the container start noticeably faster
- B) A break-in gets far fewer powers than root
- C) Docker Hub rejects images that run as root
- D) It shrinks the image by removing root's files

**Correct: B.** Least privilege. It has no effect on speed (A) or size (D), and Docker Hub accepts root images (C).

---

## Session quiz

**Q1.** `COPY . .` comes before `pip install`; you change a comment in `app.py`. What happens?
- A) Everything is cached, so the build is instant
- B) `COPY . .` and every step after it run again
- C) Only the `CMD` step runs again
- D) Only the `FROM` step downloads again

**Correct: B.** `COPY . .` includes `app.py`, so its layer changed. Every layer after it, including the slow `pip install`, rebuilds. That's exactly why requirements should be copied first. A is the trap: **any** change to a copied file counts, even a comment.

---

**Q2.** `CMD python app.py`, and every `docker stop` takes 10 seconds. Why?
- A) The image is too large to stop quickly
- B) The Dockerfile has no EXPOSE instruction
- C) PID 1 is a shell that drops SIGTERM
- D) The project folder has no .dockerignore

**Correct: C.** Shell form runs `/bin/sh -c "python app.py"`, so the shell is PID 1 and may not pass `docker stop`'s SIGTERM on to Python. Docker waits 10 seconds, then sends SIGKILL. The fix is exec form: `CMD ["python", "app.py"]`.

---

**Q3.** Which line leaves the password readable via `docker inspect` on the final image?
- A) `ENV DB_PASS=s3cr3t`
- B) `ARG DB_PASS` (set with `--build-arg`)
- C) `LABEL owner=platform-team`
- D) `EXPOSE 5432/tcp`

**Correct: A.** ENV values are **baked into the image** and visible to anyone through `docker inspect` (and in every running container's environment). B is the tempting one: an `ARG` exists **only during the build** and isn't kept in the final image's settings. *(Its value can still leak into `docker history` if a RUN line uses it, which is why real secrets use build secrets or runtime injection instead.)* C stores a harmless label. D only documents a port.

---

**Q4.** Old `myapp:latest`, new `myapp:2.0`; you run `docker run myapp`. What runs?
- A) `myapp:2.0`, as it's the newest image
- B) An error, as `latest` is unclear
- C) Both images, one after the other
- D) Whatever `myapp:latest` points to

**Correct: D.** No tag means Docker uses the tag **`latest`**, which is still the **old** image. `latest` is just a label, not "the newest". That's why teams use explicit version tags.

---

**L1.** RUN vs CMD
**Model answer:** *"RUN executes a command once, while the image is being built, and saves the result into the image, whereas CMD sets the default command that runs every time a container starts."*

**L2.** Why `0.0.0.0` inside a container?
**Model answer:** *"Inside a container, 127.0.0.1 means only the container itself, so traffic forwarded by Docker's `-p` from outside would be refused. Listening on 0.0.0.0 accepts connections arriving on any of the container's addresses."*
