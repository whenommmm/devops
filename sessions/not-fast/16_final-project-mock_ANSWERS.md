# Answer key · Session 16 · Final project walkthrough + mini mock exam

⚠️ Only open after answering. For the mock: mark yourself honestly (a hedge, like "B or C", scores zero), add up your marks, then re-read the session named under each question you missed.

---

## Warm-up

**W1.** A teammate runs `terraform state rm aws_s3_bucket.logs` and then `terraform plan`. The `aws_s3_bucket.logs` block is still in the code. What happens?
- A) The real bucket is deleted at once in the cloud
- B) Plan wants to create it, as Terraform forgot it
- C) Plan shows no changes; the bucket is still in code
- D) Plan wants to destroy the bucket on the next apply

**Correct: B.** `state rm` makes Terraform **forget** the resource; the real bucket is **untouched** (so A is wrong). The code still declares it, the state no longer has it, so the plan shows `+ create`. Applying would fail, because a bucket with that name already exists. The proper fix is `terraform import` to adopt it again. C: the plan compares code with **state**, and the state no longer contains it. D: Terraform can only destroy what's in its state.

---

**W2.** A provider is constrained with `version = "~> 6.4.0"`. Which versions are allowed?
- A) Any 6.x from 6.4.0 up, but not 7.0
- B) Exactly 6.4.0, and no other version
- C) 6.4.0 and newer 6.4.x, not 6.5.0
- D) Any version at all from 6.4.0 upwards

**Correct: C.** `~>` lets only the **last written number** go up. With three numbers (`6.4.0`), only the patch number can rise: ≥ 6.4.0 and < 6.5.0 (6.4.0, 6.4.1, 6.4.2 …). A describes `~> 6.4` (two numbers). B would be `= 6.4.0`. D would be `>= 6.4.0`.

---

**W3.** What does this print?
```bash
deploy() { echo "deploying"; return 1; }
deploy && echo "success" || echo "failed"
echo "code: $?"
```
- A) `deploying` / `failed` / `code: 0`
- B) `deploying` / `failed` / `code: 1`
- C) `deploying` / `success` / `code: 1`
- D) `deploying` / `success` / `code: 0`

**Correct: A.** The function prints `deploying` and returns **1** (its exit code = its `return N`). `&&` is skipped (the left side failed), so `||` runs `echo "failed"`, which **succeeds**. So `$?` is the exit code of that last `echo`: **0**. B is the classic trap: `$?` reports the **last** command, not the function. C and D treat `return 1` as "true", as in C# or C, where 1 means true; in the shell, **0 = success** and anything else = failure.

---

## Check 90

**90.1** In the **pull-based (GitOps)** version of the pipeline, which component actually changes the cluster?
- A) The GitHub Actions runner, using kubectl apply
- B) Argo CD, running inside the cluster itself
- C) Terraform, after it reads the Helm chart
- D) The container registry, when the tag is pushed

**Correct: B.** In GitOps, an agent **inside** the cluster (Argo CD) watches Git and **pulls** changes in; the pipeline only updates Git. A is the **push** model. C: Terraform provisions infrastructure; it doesn't deploy apps from charts in this pipeline. D: a registry only stores images; it never changes a cluster.

---

**90.2** Which pairing of pipeline step → security category is **WRONG**?
- A) Trivy on the built image → image scanning
- B) pip-audit on requirements.txt → SCA scanning
- C) Bandit on app.py → DAST of the running app
- D) gitleaks on the commits → secret scanning

**Correct: C.** Bandit reads **source code without running it**, which makes it **SAST** (Static Application Security Testing). **DAST** attacks a **running** app from outside (e.g. OWASP ZAP). A, B, and D are all correct pairings.

---

## Check 91

**91.1** You built `counter:1.0` with Colima's Docker and deployed it to minikube with `imagePullPolicy: IfNotPresent`. The Pods show `ErrImagePull: pull access denied, repository does not exist`. Which command was missed?
- A) `minikube image load counter:1.0`
- B) `docker image push counter:1.0`
- C) `kubectl rollout restart deploy/web`
- D) `minikube addons enable registry`

**Correct: A.** minikube has **its own** container runtime and can't see Colima's images, so Kubernetes tried to download `docker.io/library/counter:1.0` from Docker Hub, where it doesn't exist. `minikube image load` copies the image in, and `IfNotPresent` then uses it. B would push to Docker Hub's `library/` namespace, which you can't write to. C restarts the Pods, which still can't find the image. D enables a registry add-on but doesn't put your image in it.

---

**91.2** Bandit fails the pipeline with `B104: Possible binding to all interfaces` on `app.run(host="0.0.0.0")` in the containerised app. What's the best fix?
- A) Change it to 127.0.0.1 so Bandit is satisfied
- B) Delete the SAST step so the pipeline goes green
- C) Run Bandit with `|| true` so failures are ignored
- D) Keep 0.0.0.0; add `# nosec B104` with a reason

**Correct: D.** Inside a container the app **must** listen on `0.0.0.0`, so the finding is an **accepted risk**: suppress that **one line**, with the reason visible in code review. A "fixes" the scan but breaks the app (127.0.0.1 accepts connections only from inside the container, so `-p` and Services can't reach it). B and C both remove the **gate** for every future finding, not just this one.

---

## Check 92

**92.1** `kubectl get pods` shows three Pods, each `1/1 Running`. But `kubectl get endpointslices -l kubernetes.io/service-name=web` shows **no endpoints** for the `web` Service. What's the most likely cause?
- A) The Service's port differs from the targetPort
- B) CoreDNS has no record yet for the Service name
- C) The Service's selector doesn't match Pod labels
- D) The Pods' readiness probes are failing right now

**Correct: C.** Endpoints are built from the Pods that the Service's **selector** matches **and** that are Ready. The Pods are Ready (`1/1`), so the selector must match none of them. Check with `kubectl get pods --show-labels`. A: `port` and `targetPort` are *supposed* to differ (that's normal), and a wrong targetPort still produces endpoints. B: DNS has nothing to do with endpoints. D: `1/1` means the readiness probe is passing.

---

**92.2** `terraform apply` fails with `Error acquiring the state lock`, and the lock info says `Who: runner@fv-az123` (a CI runner). What should you do **first**?
- A) Run `terraform force-unlock` right away and apply
- B) Check whether that CI run is still in progress
- C) Delete terraform.tfstate so the lock disappears
- D) Re-run with `-lock=false` to skip the lock

**Correct: B.** The lock usually means **someone is applying right now**; if so, you wait. Only if that run has **crashed** do you `force-unlock` with the lock ID. A could let two applies write the state at once and corrupt it. C destroys Terraform's memory of everything it manages. D skips the very protection that prevents corruption.

---

## Mini mock exam: Part A (MCQs)

**M1.** What does this print?
```bash
MSG="build ok"
for w in $MSG; do echo "[$w]"; done
for w in "$MSG"; do echo "[$w]"; done
```
- A) `[build]` / `[ok]` / `[build]` / `[ok]`
- B) `[build ok]` / `[build]` / `[ok]`
- C) `[build]` / `[ok]` / `[build ok]`
- D) `[build ok]` / `[build ok]`

**Correct: C.** Unquoted `$MSG` is **split at the space** into two words, so the first loop runs twice: `[build]`, `[ok]`. Quoted `"$MSG"` stays **one** word, so the second loop runs once: `[build ok]`. (The `"[$w]"` inside the loop doesn't change how the **list** was split.) A treats the quotes as having no effect; B and D get the first loop wrong. *Re-read: shell scripting quotes (weak list).*

---

**M2.** An engineer runs `kubectl get pods` and sees `api-6f7c9-x2k   0/1   CrashLoopBackOff   5 (40s ago)`. `kubectl logs api-6f7c9-x2k` prints nothing, because the container has just been restarted. Which command most directly shows why it crashed?
- A) `kubectl logs api-6f7c9-x2k -p`
- B) `kubectl get ep api -o wide`
- C) `kubectl top pod api-6f7c9-x2k`
- D) `kubectl rollout undo deploy/api`

**Correct: A.** `-p` (`--previous`) shows the logs of the **previous, crashed** container, where the error was printed. B shows Service endpoints (a networking question, not a crash). C shows resource use, not the reason. D may hide the problem by rolling back, but it doesn't **show why** it crashed (and the crash could be config, not the image). *Re-read: Session 10.*

---

**M3.** A Flask app inside a container calls `app.run(host="127.0.0.1", port=8000)`. It's started with `docker run -d -p 8000:8000 app`. `docker ps` shows `0.0.0.0:8000->8000/tcp`, but `curl localhost:8000` on the Mac fails with `curl: (52) Empty reply from server`. Which statement best describes the cause?
- A) EXPOSE 8000 is missing, so the port was never opened
- B) The app listens only on the container's own loopback
- C) Published ports can't be reached via localhost on a Mac
- D) -p must be written as 127.0.0.1:8000:8000 on a Mac

**Correct: B.** Docker forwards the traffic **into** the container's network, but the app listens only on the container's **own** 127.0.0.1, so nothing answers the forwarded connection and curl gets an empty reply. Fix: `host="0.0.0.0"`. (This exact output was reproduced on a machine like yours.) A: `EXPOSE` only documents; `-p` alone publishes. C: published ports **are** reachable via localhost (you did it with nginx in Session 02). D: that form limits who can connect **to the Mac**; it doesn't change what the app inside listens on. *Re-read: Sessions 02 and 03 + the networking 127.0.0.1 vs 0.0.0.0 rule.*

---

**M4.** A Service has `port: 80` and `targetPort: 8080`. Its Pods run the counter image, which listens on **5000**, and they declare `containerPort: 5000`. From another Pod, `curl http://counter` fails with `Connection refused`. Which single change fixes it?
- A) Change the Service's `port` to 5000
- B) Set `containerPort: 8080` on the Pods
- C) Add a `nodePort: 30080` to the Service
- D) Set `targetPort: 5000` on the Service

**Correct: D.** The Service forwards to each Pod's **`targetPort`**, which must be the port the app **really** listens on (5000). A changes the port **clients** use, but traffic would still be sent to 8080 on the Pods. B: `containerPort` is mostly informational; changing it doesn't change what the app listens on. C adds outside access but keeps forwarding to the wrong port. *Re-read: Session 07.*

---

**M5.** A commit that broke production has already been pushed to the shared `main` branch, and teammates have pulled it. What's the safest way to undo it?
- A) `git reset --hard HEAD~1`, then `git push --force`
- B) `git revert <sha>`, then push the new commit
- C) `git checkout HEAD~1`, then push from there
- D) `git commit --amend`, then push it again

**Correct: B.** `git revert` makes a **new** commit that undoes the bad one, so history is only **added to**, and teammates' copies stay consistent. A **rewrites shared history**: teammates who pulled now have commits that "don't exist" on the remote. C puts you in **detached HEAD** and doesn't undo anything on `main`. D rewrites the last commit (also a force-push problem) and doesn't remove the bad change. *Re-read: Session 01.*

---

**M6.** A workflow's `test` job runs `pytest --junitxml=report.xml`. A second job, `publish`, has `needs: test` and runs `cat report.xml`, which fails with `No such file or directory`. Why?
- A) The publish job must run actions/checkout to get it
- B) needs: was ignored, so both jobs ran at the same time
- C) Jobs don't share files; pass it as an artifact
- D) The file is in .gitignore, so the runner deleted it

**Correct: C.** Every job runs on a **fresh runner**, so files created in one job don't exist in another. Use `actions/upload-artifact` in `test` and `actions/download-artifact` in `publish`. A: checkout brings the **repo's** files, but `report.xml` was generated at runtime and was never in the repo. B: `needs:` was honoured (it ran after `test`). D: `.gitignore` only affects what Git commits; runners don't delete files because of it. *Re-read: Session 12.*

---

**M7.** A Deployment has 4 replicas with `maxSurge: 1` and `maxUnavailable: 0`. CI deploys a new image tag that **doesn't exist** in the registry. A few minutes later, what do you see?
- A) One new Pod in ImagePullBackOff; 4 old Pods serving
- B) All 4 old Pods replaced, and the app is fully down
- C) Kubernetes rolls back to the old version on its own
- D) The Deployment is deleted after its progress deadline

**Correct: A.** `maxSurge: 1` allows **one** extra Pod; `maxUnavailable: 0` forbids removing an old Pod until a new one is **Ready**. The new Pod never becomes Ready, so the rollout **stalls** and the 4 old Pods keep serving. (This exact behaviour was seen on minikube.) B would need old Pods to be removed first. C: Kubernetes **doesn't** roll back automatically; you run `kubectl rollout undo` or `helm rollback`. D: after `progressDeadlineSeconds` the Deployment is only **marked** as failing (`ProgressDeadlineExceeded`); nothing is deleted. *Re-read: Session 06.*

---

**M8.** `terraform plan` shows `-/+ resource "aws_db_instance" "main"` with `# forces replacement`. The database holds customer data. Which `lifecycle` setting would make Terraform **refuse** to run a plan like this?
- A) `create_before_destroy = true`
- B) `ignore_changes = all`
- C) `prevent_destroy = true`
- D) `replace_triggered_by = []`

**Correct: C.** With `prevent_destroy`, any plan that would delete the resource stops with `Error: Instance cannot be destroyed` (seen on Terraform 1.16). A still destroys the old database, just **after** creating the new one. B makes Terraform ignore future changes to the resource, so no plan is made at all; it doesn't **refuse** anything, and it can quietly hide drift. D adds replacement **triggers**; an empty list does nothing. *Re-read: Session 15 ➕.*

---

**M9.** A Pod's **readiness** probe starts failing because its database is down, while its **liveness** probe keeps passing. What does Kubernetes do?
- A) Restarts the container until the probe passes
- B) Deletes the Pod and schedules it on a new node
- C) Marks the Deployment as failed and rolls back
- D) Removes the Pod from the Service's endpoints

**Correct: D.** A failing **readiness** probe means "don't send me traffic yet": the Pod is taken out of the Service's endpoints and **isn't restarted**. When readiness passes again, it's added back. A is what a failing **liveness** probe does. B: probes never move Pods to other nodes. C: Kubernetes doesn't roll back on its own. *Re-read: Session 09.*

---

**M10.** A user's umask is `027`. They create a new **file** with `touch report.txt`. What does `ls -l` show for its permissions?
- A) `-rwxr-x---`
- B) `-rw-r-----`
- C) `-rw-rw-r--`
- D) `-rw-r--r--`

**Correct: B.** Files start from **666**: 666 − 027 → **640** = `rw-` `r--` `---`. (Checked on a Mac: `-rw-r-----`.) A is what a **folder** gets (777 − 027 = 750). C is the result of umask 002; D is the result of umask 022. *Re-read: the Linux permissions lesson (weak list).*

---

**M11.** A pipeline scan reads `requirements.txt` and reports `requests 2.19.0 — CVE-2018-18074 (HIGH), fixed in 2.20.0`. What kind of scan produced this?
- A) SCA: it checked third-party dependencies
- B) SAST: it analysed the app's own source code
- C) DAST: it attacked the running application
- D) Secret scanning: it searched for leaked keys

**Correct: A.** **SCA** (Software Composition Analysis) compares your **dependencies'** versions with lists of known **CVEs** (Common Vulnerabilities and Exposures). The giveaway is "a library, a version, a CVE, a fixed-in version". B (SAST) flags patterns in **your** code. C (DAST) probes a running app. D finds leaked secrets, not vulnerable libraries. *Re-read: Session 13.*

---

**M12.** A Deployment reads `LOG_LEVEL` from a ConfigMap through `envFrom`. You change the ConfigMap to `LOG_LEVEL=debug` with `kubectl apply`. An hour later the app still logs at `info`. Why, and what fixes it?
- A) Env vars are read at start; run kubectl rollout restart
- B) ConfigMaps update only after a kubelet reboot on the node
- C) The ConfigMap must be base64-encoded to be picked up
- D) envFrom works only with Secrets; switch it to a Secret

**Correct: A.** Environment variables are copied into the container **when it starts** and never change afterwards. `kubectl rollout restart deployment/<name>` creates fresh Pods that read the new value. (ConfigMaps mounted as **files** do update by themselves after a short delay.) B: no reboot is involved. C: base64 is for **Secrets**, not ConfigMaps. D: `envFrom` works with both ConfigMaps and Secrets. *Re-read: Session 08.*

---

**M13.** A Dockerfile ends with:
```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "5000"]
```
You run `docker run img --port 8000`. What runs inside the container?
- A) `python app.py --port 5000`; your args are ignored
- B) `--port 8000` alone; ENTRYPOINT is replaced
- C) `python app.py --port 8000`; CMD is replaced
- D) `python app.py`; both CMD and args are dropped

**Correct: C.** Arguments after the image name **replace CMD**, and are **appended to ENTRYPOINT**. A: args are never ignored. B: replacing ENTRYPOINT needs `--entrypoint`. D: nothing is dropped. *Re-read: Session 03.*

---

**M14.** From an app server, `nc -zv db.internal 5432` returns `Connection refused` **instantly**. From a second server, the same command **hangs, then times out**. Which reading is most accurate?
- A) Refused = DNS failed; timeout = wrong port number
- B) Refused = firewall drop; timeout = app not running
- C) Both mean the database rejected the password
- D) Refused = no listener; timeout = firewall drop

**Correct: D.** **Refused** = the host was reached and **actively answered "no"**, usually because nothing is listening on 5432 (a firewall *reject* rule can also do this). A **timeout** = **no answer at all**: packets are being **dropped**, typically by a firewall or security group, or the host is unreachable. B is exactly reversed (the tempting mix-up). A: a DNS failure says "could not resolve" / "unknown host" instead. C: `nc -z` only opens a TCP connection; no password is ever sent. *Re-read: networking commands + Session 15 (security groups).*

---

**M15.** An HPA targets **50%** average CPU utilisation, with `maxReplicas: 10`. The Deployment has **2** Pods, averaging **90%** of their CPU requests. How many replicas will the HPA ask for?
- A) 2 replicas
- B) 3 replicas
- C) 4 replicas
- D) 5 replicas

**Correct: C.** desired = **ceil(current × actual ÷ target)** = ceil(2 × 90 ÷ 50) = ceil(3.6) = **4** (and 4 is within `maxReplicas`). B rounds **down** (the HPA always rounds **up**). D multiplies wrongly. A assumes nothing changes. *Re-read: Session 09.*

---

**M16.** What happens when this runs in an empty folder?
```bash
COUNT=2
if [ $COUNT > 5 ]; then echo "big"; else echo "small"; fi
ls
```
- A) `small`, and `ls` shows nothing new
- B) `big`, and `ls` shows a new file named `5`
- C) An error: `>` isn't allowed inside `[ ]`
- D) `small`, and `ls` shows a new file named `5`

**Correct: B.** Inside `[ ]`, `>` is **redirection**: the shell runs `[ 2 ]` with its output sent to a new file called `5`. `[ 2 ]` tests "is this text non-empty?", which is **true**, so it prints `big`. (Checked: it prints `big`, and `ls` shows `5`.) D is the near-miss: it spots the file but assumes a real comparison happened. A assumes a numeric comparison with no side effect. C: there's no error; that's what makes the bug sneaky. Use `-gt`. *Re-read: shell scripting tests (weak list).*

---

**M17.** Your laptop's `terraform init` installed AWS provider **6.12.0** last month; a teammate's fresh `init` today installed **6.68.0**. Both used `version = "~> 6.0"`. What makes everyone get the same version?
- A) Commit `.terraform.lock.hcl` to Git
- B) Commit the `.terraform/` folder to Git
- C) Commit `terraform.tfstate` to Git
- D) Run `terraform fmt` before each init

**Correct: A.** The **dependency lock file** records the exact provider versions (and their hashes). With it committed, every `init` installs the same versions until someone deliberately runs `terraform init -upgrade`. B: `.terraform/` holds huge downloaded binaries (about 770 MB for AWS) and must be git-ignored. C: the state must **never** go into Git, and it doesn't control provider versions anyway. D: `fmt` only formats code. *Re-read: Session 15.*

---

**M18.** A database server in a **private** subnet must download security patches from the internet, but nothing on the internet may **start** a connection to it. What should you add?
- A) A route 0.0.0.0/0 → internet gateway for the subnet
- B) An inbound security group rule for 443 from 0.0.0.0/0
- C) A NAT gateway in a public subnet as its default route
- D) An Elastic IP attached directly to the database server

**Correct: C.** A **NAT gateway** (sitting in a public subnet, with the private subnet's route table sending `0.0.0.0/0` to it) lets private servers **start** outbound connections, while nothing on the internet can reach them. A makes the subnet **public**. B **opens** the server to inbound traffic, the opposite of what's wanted (and replies to outbound connections don't need inbound rules). D gives it a public address, again making it reachable. *Re-read: Session 15, Concept 89.*

---

**M19.** Prometheus's **Targets** page shows `counter:5000/metrics` as **DOWN** with `connection refused`, and Grafana's panel for the app shows no data. Which is most likely?
- A) Grafana's data source URL points to the wrong server
- B) Prometheus can't scrape the app's metrics endpoint
- C) The app must push its metrics to Prometheus first
- D) Alertmanager has silenced every alert for the app

**Correct: B.** "DOWN" on the Targets page means Prometheus's **scrape** (pull) of that endpoint failed: a wrong port or path, the app not exposing `/metrics`, or the Pod not running. Grafana shows no data **because** Prometheus has none. A would leave the Prometheus target **UP**. C: Prometheus **pulls**; apps don't push to it (except through a special Pushgateway). D: silencing affects notifications, not data collection. *Re-read: Session 14.*

---

**M20.** You run `helm install web ./chart` (revision 1), then `helm upgrade web ./chart --set image.tag=v2` (revision 2), which breaks the app, then `helm rollback web 1`. What does `helm history web` show as the **latest** revision?
- A) Revision 1, marked deployed again
- B) Revision 2, marked as rolled back
- C) Revision 0, a fresh install
- D) Revision 3, "Rollback to 1"

**Correct: D.** A rollback is a **new** release revision that reuses revision 1's settings, so history becomes 1 (superseded), 2 (superseded), **3 (deployed, "Rollback to 1")**. This was checked on Helm 4. A: old revisions are never reactivated in place. B: revision 2 is just `superseded`. C: revisions start at 1 and only go up. *Re-read: Session 11.*

---

**M21.** A Pod in namespace `shop` calls `http://payments:8080`. The `payments` Service exists in namespace **`billing`**. The call fails with `Could not resolve host: payments`. What fixes it?
- A) Call `billing.payments:8080` instead
- B) Call `payments.billing:8080` instead
- C) Expose `payments` as a NodePort Service
- D) Add `hostNetwork: true` to the shop Pod

**Correct: B.** A short name resolves only **inside the Pod's own namespace**. Across namespaces, use **`<service>.<namespace>`** (full form `payments.billing.svc.cluster.local`). A has the order reversed. C changes how the Service is exposed outside the cluster, not DNS. D makes the Pod use the node's network, which doesn't help it resolve cluster names and weakens isolation. *Re-read: Session 07.*

---

**M22.** Which statement about this Dockerfile's **final** image is true?
```dockerfile
FROM python:3.12 AS build
WORKDIR /src
COPY requirements.txt .
RUN pip wheel -r requirements.txt -w /wheels

FROM python:3.12-slim
COPY --from=build /wheels /wheels
RUN pip install /wheels/*
```
- A) It has the slim base plus only the copied wheels
- B) It still contains the full python:3.12 image as a layer
- C) It's larger, because both FROM images are stacked in it
- D) It can't install the wheels, as build tools were dropped

**Correct: A.** In a **multi-stage build**, only the **last stage** becomes the image; earlier stages are thrown away, except for what you copy with `COPY --from`. B and C: the big `python:3.12` build stage isn't in the final image at all; that's the whole point. D: **wheels** are already-built packages, so installing them needs no compiler. *Re-read: Session 03.*

---

**M23.** A teammate says: "Our DB password is safe because it's in a Kubernetes Secret." The Secret's YAML shows `password: cGFzc3dvcmQxMjM=`. Which statement is accurate?
- A) It's encrypted; only the API server holds the key
- B) It's a hash, so the password can't be recovered
- C) It's base64: anyone who can read it can decode it
- D) It's safe to commit, since Git can't decode base64

**Correct: C.** base64 is an **encoding**, not encryption: `echo cGFzc3dvcmQxMjM= | base64 -d` prints `password123`. Protect Secrets with RBAC, encryption at rest in etcd, and by keeping them out of Git (e.g. Sealed Secrets or an external secrets manager). A: not encrypted by default. B: base64 is fully reversible, unlike a hash. D: anyone can decode it, and Git history keeps it forever. *Re-read: Session 08.*

---

**M24.** You delete one Pod of a Deployment that has `replicas: 3`. Which sequence best describes what happens next?
- A) Scheduler creates a Pod → etcd picks a node → kube-proxy runs it
- B) kubelet restarts the same Pod on the same node → scheduler records it
- C) API server restores the old Pod from etcd → kubelet runs it again
- D) ReplicaSet makes a new Pod → scheduler picks node → kubelet runs it

**Correct: D.** The **ReplicaSet controller** (in the controller manager) sees 2 instead of 3 and **creates a new Pod object**; the **scheduler** assigns it a node; that node's **kubelet** starts its containers. A mixes up every role: the scheduler only places Pods, etcd only stores data, and kube-proxy handles Service networking. B: a deleted Pod is gone; a **new** Pod (with a new name) replaces it. C: Kubernetes doesn't restore deleted objects; controllers create replacements. *Re-read: Sessions 05 and 06.*

---

**M25.** The pipeline runs `trivy image --severity CRITICAL --exit-code 1 app:1.4`. The image has **3 HIGH** and **0 CRITICAL** vulnerabilities. What happens?
- A) The step fails, because HIGH counts as a finding
- B) The step fails, since exit-code 1 always fails it
- C) The step passes, but the image push is blocked
- D) The step passes; HIGH ones aren't checked at all

**Correct: D.** `--severity CRITICAL` limits the scan to CRITICAL findings, and `--exit-code 1` applies **only when something matching is found**. Nothing matches, so Trivy exits 0 and the pipeline continues (the HIGH ones aren't even listed). To gate on both, use `--severity HIGH,CRITICAL`. A ignores the severity filter. B misreads `--exit-code`: it's the code to use **if** findings exist. C: a passing step blocks nothing. *Re-read: Session 13.*

---

## Mini mock exam: Part B (debugging)

**Marking:** 1 mark per bug, only if you named the problem **and** gave the right fix. Extra improvements are welcome but earn no marks.

### D1. Shell script (`release.sh`, run as `bash release.sh push`)
```bash
#!/bin/bash
# release.sh: build the counter image, and push it if asked
set -e

TAG = $(git rev-parse --short HEAD)
IMAGE="ghcr.io/vansh/counter:$TAG"

build() {
  docker build -t "$IMAGE" .
}

build
if [ $1 = "push" ]; then
  docker push "$IMAGE"
  echo 'Pushed $IMAGE'
fi
```
**The bugs** (all three reproduced with stand-in `git`/`docker` commands):
1. **`TAG = $(…)`**: spaces around `=`. Bash reads `TAG` as a **command name** with arguments `=` and the SHA: `release.sh: line 5: TAG: command not found`, and `set -e` stops the script immediately (exit code 127). **Fix:** `TAG=$(git rev-parse --short HEAD)` (no spaces).
2. **`[ $1 = "push" ]`**: unquoted. Run with no argument, it becomes `[ = "push" ]` → `[: =: unary operator expected`, and the push section is skipped (an error inside an `if` test doesn't trigger `set -e`). **Fix:** `[ "$1" = "push" ]`.
3. **`echo 'Pushed $IMAGE'`**: single quotes are literal, so it prints `Pushed $IMAGE`. **Fix:** double quotes: `echo "Pushed $IMAGE"`.

**Fixed file:**
```bash
#!/bin/bash
# release.sh: build the counter image, and push it if asked
set -e

TAG=$(git rev-parse --short HEAD)
IMAGE="ghcr.io/vansh/counter:$TAG"

build() {
  docker build -t "$IMAGE" .
}

build
if [ "$1" = "push" ]; then
  docker push "$IMAGE"
  echo "Pushed $IMAGE"
fi
```
*Verified: `bash release.sh push` now builds, pushes, and prints `Pushed ghcr.io/vansh/counter:a1b2c3d`; with no argument it just builds and exits 0.* *Re-read: shell scripting (variables, quotes, tests).*

---

### D2. Dockerfile
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
USER appuser
EXPOSE 5000
CMD ["python" "app.py"]
```
**The bugs** (built and run on Colima):
1. **`COPY . .` before `RUN pip install`**: any change to any file (even a comment in `app.py`) invalidates that layer and **every layer after it**, so `pip install` re-runs on every build. **Fix:** `COPY requirements.txt .` → `RUN pip install …` → then `COPY` the code.
2. **`USER appuser`, but the user is never created**: the build succeeds, but `docker run` fails with `unable to find user appuser: no matching entries in passwd file`. **Fix:** `RUN useradd --create-home appuser` before `USER appuser`.
3. **`CMD ["python" "app.py"]`**: a **missing comma**, so it isn't valid JSON. Docker silently falls back to **shell form** and runs the text as a shell command: `/bin/sh: 1: [python: not found` (the build only warns `JSONArgsRecommended`). **Fix:** `CMD ["python", "app.py"]`.

**Fixed file:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
RUN useradd --create-home appuser
COPY . .
USER appuser
EXPOSE 5000
CMD ["python", "app.py"]
```
*Verified: builds, runs as `appuser` (`docker exec … whoami` → `appuser`), and serves on port 5000.* *Re-read: Session 03.*

---

### D3. Compose file (`compose.yaml`)
```yaml
services:
  web:
    build: .
    ports:
      - "8080:5000"
    environment:
      - REDIS_HOST=localhost
    depends_on:
      - redis
  cache:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
```
**The bugs** (checked with `docker compose config` and `docker compose up`):
1. **The named volume isn't declared**: `redis-data` is used, but there's no top-level `volumes:` section. `service "cache" refers to undefined volume redis-data: invalid compose project`. **Fix:** add `volumes:` / `  redis-data:` at the bottom (top level, not indented under a service).
2. **`depends_on: - redis`**: there's no service called `redis` (it's called `cache`). `service "web" depends on undefined service "redis": invalid compose project`. **Fix:** `- cache`.
3. **`REDIS_HOST=localhost`**: inside the `web` container, `localhost` is **web itself**, so every request fails to reach Redis (`ConnectionError … connecting to localhost:6379`, HTTP 500). **Fix:** `REDIS_HOST=cache` (the service name works as a DNS name on Compose's network).

**Fixed file:**
```yaml
services:
  web:
    build: .
    ports:
      - "8080:5000"
    environment:
      - REDIS_HOST=cache
    depends_on:
      - cache
  cache:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
volumes:
  redis-data:
```
*Verified: `docker compose up` starts both services, and `curl localhost:<port>` counts 1, 2, …* *Re-read: Session 04.*

---

### D4. Kubernetes YAML (`counter.yaml`)
```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: counter
spec:
  replicas: 2
  selector:
    matchLabels:
      app: counter
  template:
    metadata:
      labels:
        app: counter-web
    spec:
      containers:
        - name: counter
          image: ghcr.io/vansh/counter:1.0
          ports:
            - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: counter
spec:
  selector:
    app: counter
  ports:
    - port: 80
      targetPort: 8080
```
**The bugs** (checked with `kubectl apply --dry-run`):
1. **`apiVersion: v1` for a Deployment**: Deployments live in the **`apps/v1`** API group. `kubectl apply` fails with `no matches for kind "Deployment" in version "v1"`. **Fix:** `apiVersion: apps/v1`.
2. **Selector and template labels don't match**: the selector wants `app: counter`, but the Pod template's label is `app: counter-web`. The API server rejects it: ``The Deployment "counter" is invalid: spec.template.metadata.labels: Invalid value: {"app":"counter-web"}: `selector` does not match template `labels` ``. (Note: `--dry-run=client` does **not** catch this; only the API server does, e.g. `--dry-run=server`.) **Fix:** set the template label to `app: counter`, which also makes it match the **Service's** selector. (Changing the selector to `counter-web` instead would fix the Deployment but leave the Service matching nothing.)
3. **Service `targetPort: 8080`**, but the container listens on **5000**: endpoints exist, but every connection is refused. **Fix:** `targetPort: 5000`.

**Fixed file:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: counter
spec:
  replicas: 2
  selector:
    matchLabels:
      app: counter
  template:
    metadata:
      labels:
        app: counter
    spec:
      containers:
        - name: counter
          image: ghcr.io/vansh/counter:1.0
          ports:
            - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: counter
spec:
  selector:
    app: counter
  ports:
    - port: 80
      targetPort: 5000
```
*Verified with `kubectl apply --dry-run=server`: both objects are accepted.* *(Not a bug, but worth adding if an HPA will scale it: `resources.requests.cpu`.)* *Re-read: Sessions 05–07.*

---

### D5. GitHub Actions workflow (`.github/workflows/ci.yaml`)
```yaml
name: CI
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt pytest
      - run: pytest -q

  build:
    needs: tests
    runs-on: ubuntu-latest
    steps:
      - name: Build image
        run: docker build -t ghcr.io/vansh/counter:${{ github.sha }} .
      - name: Log in to GHCR
        run: echo "${{ secret.GHCR_TOKEN }}" | docker login ghcr.io -u vansh --password-stdin
      - name: Push image
        run: docker push ghcr.io/vansh/counter:${{ github.sha }}
```
**The bugs** (1 and 3 were flagged by `actionlint`; 2 shows up when the job runs):
1. **`needs: tests`**: the job is called **`test`**. GitHub rejects the workflow file as invalid before anything runs (actionlint: `job "build" needs job "tests" which does not exist in this workflow`). **Fix:** `needs: test`.
2. **No `actions/checkout` in the `build` job**: each job starts on a **fresh, empty runner**, so there's no Dockerfile: `ERROR: failed to build: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory`. **Fix:** add `- uses: actions/checkout@v4` as the first step of `build`.
3. **`${{ secret.GHCR_TOKEN }}`**: the context is called **`secrets`** (plural). The workflow is invalid (`Unrecognized named-value: 'secret'`; actionlint: `undefined variable "secret"`). **Fix:** `${{ secrets.GHCR_TOKEN }}`.

**Fixed file:**
```yaml
name: CI
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt pytest
      - run: pytest -q

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t ghcr.io/vansh/counter:${{ github.sha }} .
      - name: Log in to GHCR
        run: echo "${{ secrets.GHCR_TOKEN }}" | docker login ghcr.io -u vansh --password-stdin
      - name: Push image
        run: docker push ghcr.io/vansh/counter:${{ github.sha }}
```
*Verified: actionlint reports no errors.* *(Not a bug, but better: use the built-in `GITHUB_TOKEN` with `permissions: packages: write` instead of a personal token, as in Concept 91.)* *Re-read: Sessions 12–13.*

---

## Score yourself

| Marks (out of 40) | Meaning | What to do tonight |
|---|---|---|
| 36–40 | Full-marks level | Read the last-look sheet once, then sleep |
| 30–35 | Strong | Re-read the sessions named under your misses; redo those questions tomorrow morning |
| 22–29 | Gaps in one or two areas | Group your misses by topic; re-read those sessions' ➕ tables and quizzes |
| < 22 | Needs a focused review | Kubernetes troubleshooting (Session 10) + Docker (03–04) + the weak list first: they carry the most marks |

**Topic map of the mock:** Kubernetes: M2, M4, M7, M9, M12, M15, M21, M23, M24, D4 · Docker: M3, M13, M22, D2, D3 · Foundations: M1, M5, M10, M14, M16, D1 · CI/CD + DevSecOps: M6, M11, M25, D5 · Terraform/Cloud: M8, M17, M18 · Monitoring/GitOps/Helm: M19, M20.

---

## ✍️ Write the command: answers
**C1.** Your `git push` to `main` was rejected with `! [rejected] main -> main (fetch first)`. Bring in your teammate's new commits by **replaying your commits on top of** the remote's `main` (no merge commit); then, only if that worked, push again.
`git pull --rebase origin main && git push`: `git pull` = fetch the remote's new commits and combine them with yours; `--rebase` = replay your local commits on top of them instead of making a merge commit; `origin main` = from the remote `origin`, branch `main`; `&& git push` = push only if the pull worked (on a conflict: edit → `git add` → `git rebase --continue`, then push; never `--force` a shared branch). (Alternative: `git fetch origin && git rebase origin/main && git push`.)

**C2.** Save the current commit's **short** SHA in a shell variable `TAG`; then, only if that worked, build an image named `counter`, tagged with that SHA, from the current folder.
`TAG=$(git rev-parse --short HEAD) && docker build -t counter:$TAG .`: `$( … )` runs the command inside and pastes its output; `git rev-parse --short HEAD` prints the current commit's short SHA (e.g. `a1b2c3d`); `TAG=…` (no spaces around `=`) stores it in a shell variable; `&&` = only if that worked; `docker build` builds an image from the `Dockerfile`; `-t counter:$TAG` = name `counter`, tag = the SHA; the final `.` = the build context (the current folder; easy to forget). (Alternative: two lines, `TAG=$(git rev-parse --short HEAD)` then `docker build -t "counter:$TAG" .`.)

**C3.** You changed `app.py` in the counter project. Rebuild the images and start the Compose stack in the background; then, only if that worked, list the stack's containers and their status.
`docker compose up -d --build && docker compose ps`: `up` creates and starts every service in `compose.yaml`; `-d` = in the background (detached); `--build` = rebuild the images first (without it, Compose reuses the old image and your change is missing); `&&` = only if that worked; `ps` lists this stack's containers with their status and ports. (Alternative: `docker compose up --build -d && docker compose ps`.)

**C4.** The `counter-web` Service has no endpoints. List all Pods in the `counter` namespace **with their labels shown**, so you can compare them with the Service's selector.
`kubectl get pods -n counter --show-labels`: `get pods` lists the Pods; `-n counter` = in the `counter` namespace (without it you'd see `default`); `--show-labels` adds a LABELS column (e.g. `app=counter-web,pod-template-hash=…`) to compare with the `Selector:` from `kubectl describe svc counter-web -n counter`. (Alternative: `kubectl get pods --namespace counter --show-labels`.)

**C5.** Deploy the local chart `./charts/counter` as the release `counter` in the namespace `counter`: install it if it doesn't exist yet, otherwise upgrade it; create the namespace if it's missing; set the value `image.tag` to the contents of `$TAG`; and report success only once the Pods are Ready.
`helm upgrade --install counter ./charts/counter -n counter --create-namespace --set image.tag=$TAG --wait`: `helm upgrade` upgrades a release, and `--install` installs it instead if it doesn't exist yet (safe to re-run); `counter` = the release name; `./charts/counter` = the chart folder; `-n counter` = the namespace; `--create-namespace` = create it if missing; `--set image.tag=$TAG` = override one value from `values.yaml` (the shell fills in `$TAG`); `--wait` = report success only after the Pods are Ready (otherwise `deployed` appears as soon as Kubernetes accepts the YAML). (Alternative: `helm upgrade --install counter ./charts/counter --namespace counter --create-namespace --set image.tag=$TAG --wait`.)

**C6.** Make a Terraform plan using the variable values in `prod.tfvars` and save it to a file named `tfplan`; then, only if planning worked, apply **exactly that saved plan**.
`terraform plan -var-file=prod.tfvars -out=tfplan && terraform apply tfplan`: `plan` = the dry run; `-var-file=prod.tfvars` loads variable values from that file (it beats `terraform.tfvars` and `TF_VAR_`); `-out=tfplan` saves the plan to the file `tfplan`; `&&` = only if planning worked; `apply tfplan` carries out exactly the saved plan, without making a new plan and without asking `yes` (what was reviewed is what runs).
