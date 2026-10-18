# Session 10 · Kubernetes VI: troubleshooting (Oct 15, Session 2)

**Time:** ~110–120 min (includes a 25-min lab) · **Concepts:** 66 → 69 · **Answers:** `MY_ANSWERS.md` or say "K8s troubleshooting questions" · **Answer key:** `10_kubernetes-troubleshooting_ANSWERS.md`

**What this covers:** the official "Kubernetes 6–8" lecture: `get`, `describe`, `logs`, `exec`, Events, CrashLoopBackOff, ImagePullBackOff, Pending Pods, Service & DNS. Expect **"you see X, what's the cause?"** questions.

**Before you start:**
```bash
minikube status        # Running?
kubectl get nodes      # minikube   Ready
mkdir -p ~/devops-practice/k8s-vi/lab && cd ~/devops-practice/k8s-vi   # session folder + a lab folder
```
**What each line does:**
- `minikube status`: checks that your local cluster is up: `host`, `kubelet` and `apiserver` should say `Running` (if not: `minikube start`).
- `kubectl get nodes`: lists the cluster's nodes; you should see `minikube` with STATUS `Ready`.
- `mkdir -p … && cd …`: makes the session folder **and** a `lab` folder inside it in one go (`-p` = also create missing parent folders, no error if they exist), then moves into the session folder (`&&` = only if `mkdir` worked).

> ⚠️ **Never delete the `minikube` or `localstack` containers**, and leave alone namespaces/resources you didn't create (`argocd`, `ingress-nginx`, `final-app`). Today everything runs in `default`, with names starting `tour`, `crash-`, `pull-`, `pending-`, `bind-`, `dns-` or `lab-`.

---
## 🔁 Warm-up (from Kubernetes V and your weak list)

**W1.** A Pod's readinessProbe fails on every check, but its livenessProbe passes. Which set of observations is right?
- A) READY 0/1, RESTARTS rising, still in endpoints
- B) READY 1/1, RESTARTS 0, still receiving traffic
- C) READY 0/1, RESTARTS 0, not in the endpoints
- D) CrashLoopBackOff, and left out of the endpoints

**W2.** On minikube, you delete a PVC that was dynamically provisioned by the `standard` StorageClass (no Pod uses it). What happens?
- A) The PV becomes Released and keeps the data
- B) The PV stays Bound until someone deletes it
- C) The PV goes back to Available for reuse
- D) The PV and its data are deleted with it

**W3.** What happens when this runs?
```bash
x=3
if [ $x > 5 ]; then echo big; else echo small; fi
```
- A) Prints `big` and creates a file named `5`
- B) Prints `small`, because 3 is less than 5
- C) Prints nothing, because `>` compares text
- D) Gives an error, because `[ ]` can't use `>`

---
## Concept 66 · The troubleshooting toolkit, and a method that always works

**Words you'll learn:** troubleshooting, symptom, root cause, `kubectl get` flags, STATUS, `kubectl describe`, Events, `kubectl logs` flags, `kubectl exec`, `kubectl get events`, `kubectl port-forward`, layer.

### The problem
Friday, 6 p.m.: "The shop is down!" A junior engineer deletes random Pods for 40 minutes; it's still broken. A senior types five commands and finds a one-letter typo in an image tag within three minutes. The difference is a **method**.
- **Troubleshooting** = finding and fixing the cause of a problem, step by step.
- A **symptom** is **what you see** (the Pod shows `CrashLoopBackOff`). The **root cause** is **why** it happens (the `DB_HOST` variable is missing). You fix the root cause, not the symptom.

**Analogy: a doctor's visit.** Five kubectl tools match what a doctor does:

| Doctor | kubectl | What it tells you |
|---|---|---|
| Glance at the vital signs | **`kubectl get`** | The **state** of things, at a glance |
| Read the patient's chart | **`kubectl describe`** | Full details + **Events** (what happened, in order) |
| Ask "what do you feel?" | **`kubectl logs`** | What the **app itself** printed |
| Examine the patient | **`kubectl exec`** | Step **inside** the container and test things |
| Read the ward's logbook | **`kubectl get events`** | Everything that happened in the namespace, in time order |

### Tool 1: `kubectl get` and the STATUS column
Handy flags: **`-o wide`** adds each Pod's IP and NODE (`-o` = output format); **`-o yaml`** prints the whole object, including its live `status:`; **`-w`** watches for changes; **`--show-labels`** adds a LABELS column; **`-l app=web`** shows only Pods with that label; **`-A`** covers all namespaces. ⚠️ Despite its name, `kubectl get all` does **not** show ConfigMaps, Secrets, PVCs or Ingresses.

READY `0/1` means 0 of the Pod's 1 containers are ready. **STATUS** is the most useful single word in Kubernetes:

| STATUS | Meaning | Look next |
|---|---|---|
| `Pending` | Accepted, but **not yet placed on a node** | `describe` → scheduler Events (68) |
| `ContainerCreating` | On a node; pulling the image, mounting volumes | `describe` if stuck (68) |
| `Running` | The main process runs (**not necessarily healthy or ready!**) | probes, `describe svc` (69) |
| `Completed` | The main process exited with **0** | normal for a Job; see `restartPolicy` (67) |
| `Error` | The main process exited with a **non-zero** code | `logs --previous` (67) |
| `CrashLoopBackOff` | It keeps exiting; Kubernetes **waits** before the next restart | `logs --previous` + `describe` (67) |
| `OOMKilled` | Killed for using more memory than its limit | `describe` (67) |
| `ErrImagePull` / `ImagePullBackOff` | The image can't be downloaded | `describe` → kubelet Events (68) |
| `CreateContainerConfigError` | A ConfigMap/Secret the container needs is missing | `describe` (67) |

### Tool 2: `kubectl describe` (the full chart)
`kubectl describe pod <p>` shows the Pod's node, then each container's **State** (what it's doing now) and **Last State** (how it ended last time, with a **Reason**, an **Exit Code** and the Restart Count). At the very bottom come the **Events**: a time-ordered list of what Kubernetes did and what went wrong. **The Events are the gold.**
- Each event has a Type, Reason, Age, From and Message, e.g. `Warning  Failed  10s (x5 over 3m9s)  kubelet  Failed to pull image "nginx:1.277": … not found`.
- Read the **`Warning`** lines first (`Normal` = fine). **From** = who reported it: `default-scheduler` places Pods on nodes, and the `kubelet` runs containers. `(x5 over 3m9s)` = it happened 5 times in 3 min 9 s.
- ⚠️ Events are kept for only about **1 hour**. `describe` works on any object (`describe svc`, `describe node`, `describe pvc`…).

### Tools 3 to 5: logs, exec, events (plus port-forward)
- **`kubectl logs <pod>`** = what the app printed. `-f` follows it live. **`--previous`** shows the output of the **previous, dead** container: essential for crash loops, because the new container may not have printed anything yet. `-c <container>` picks one container in a multi-container Pod; `deploy/web` uses one of the Deployment's Pods; `--tail=20` shows only the last 20 lines.
- ⚠️ A container that **never started** (image not pulled, config missing) has **no logs**: you get `… is waiting to start`. Use `describe` instead.
- **`kubectl exec -it <pod> -- sh`** opens a shell inside the container: `-i` = interactive, `-t` = a terminal, as in `docker exec -it`. **`--`** means "kubectl's own flags end here; the rest runs inside". One-off commands need no `-it`: `kubectl exec <pod> -- env`.
- **`kubectl get events --sort-by=.lastTimestamp`** lists all recent events in the namespace, oldest first, so the newest are at the **bottom**. Add `--field-selector type=Warning` to see only problems.
- **`kubectl port-forward svc/<s> 8090:80`** connects your Mac's port 8090 straight to the Service (like Docker's `-p`). It skips NodePort and Ingress, so it shows whether the app and Service work on their own.

### The method, and "which layer is broken?"
**The method: get → describe → logs → exec → events.** Start with `get`: its STATUS tells you which tool to use next (the table above). Then ask **which layer is broken**: a request to your app passes through layers, and each one can break (the same idea as checking a network problem one layer at a time: cable, IP, DNS, port, app):
```
 Node Ready? → Pod scheduled? → Image pulled? → Container running? → App ready (probes)?
   → Service has endpoints? → DNS name resolves? → Ingress rule matches?
```
**Work from the inside out:** test the app inside its own Pod, then from another Pod, then by Service name, then through the Ingress. The first layer that fails holds the problem.

### 🛠 Do (~12 min): a toolkit tour on a healthy app
```bash
kubectl create deployment tour --image=nginx:1.27 --replicas=2   # 2 Pods, labelled app=tour
kubectl expose deployment tour --port=80                         # a ClusterIP Service "tour"
kubectl get pods -o wide                     # 1/1 Running, IP, NODE
kubectl get pods --show-labels               # app=tour,pod-template-hash=…
kubectl describe pod <one-tour-pod-name>     # Events all Normal
kubectl logs deploy/tour --tail=3            # nginx's last start-up lines
kubectl exec -it deploy/tour -- sh           # now inside a Pod (prompt #)
curl -s localhost | head -4                  #   (inside) the app answers
cat /etc/resolv.conf                         #   (inside) the DNS settings
exit                                         #   leave the Pod
kubectl port-forward svc/tour 8090:80        # Ctrl + C stops
curl -s localhost:8090 | head -4             # in a 2nd tab: Welcome to nginx!
kubectl get events --sort-by=.lastTimestamp | tail -5   # the newest events
kubectl delete deployment tour               # clean up
kubectl delete service tour
```
**What each line does:**
- `kubectl create deployment tour --image=nginx:1.27 --replicas=2`: a Deployment with 2 nginx Pods (`--image` = the image, `--replicas` = how many Pods); they get the label `app=tour` automatically.
- `kubectl expose deployment tour --port=80`: a ClusterIP Service `tour` (a stable in-cluster IP + name in front of the Pods); `--port` = the port it listens on (and, with no `--target-port`, the Pods' port too).
- `kubectl get pods -o wide`: `-o wide` adds the IP and NODE columns: `1/1 Running`, IP `10.244.0.x`, NODE `minikube`.
- `kubectl get pods --show-labels`: adds a LABELS column: `app=tour,pod-template-hash=…` (the hash tells the Deployment's ReplicaSets apart).
- `kubectl describe pod <one-tour-pod-name>`: the full details. Healthy = Events all `Normal`: `Scheduled`, `Pulled`, `Created`, `Started`.
- `kubectl logs deploy/tour --tail=3`: `deploy/tour` = use one of the Deployment's Pods (it says `Found 2 pods, using pod/tour-…`); `--tail=3` = only the last 3 lines of nginx's start-up output.
- `kubectl exec -it deploy/tour -- sh`: opens a shell inside one of the Pods (`-i` = interactive, `-t` = a terminal; `--` = the command to run inside follows). The prompt changes to `#`.
- `curl -s localhost | head -4`: (inside) `-s` = silent (no progress bar); `| head -4` = the first 4 lines. The app answers inside its own Pod.
- `cat /etc/resolv.conf`: (inside) the DNS settings Kubernetes gave the Pod; its `nameserver` is the cluster DNS.
- `exit`: leaves the Pod.
- `kubectl port-forward svc/tour 8090:80`: connects your Mac's port 8090 to the Service's port 80 and prints `Forwarding from 127.0.0.1:8090 -> 80`. It keeps the terminal busy; Ctrl + C stops it.
- `curl -s localhost:8090 | head -4`: (in a 2nd tab) `<title>Welcome to nginx!</title>`: the app and Service work on their own.
- `kubectl get events --sort-by=.lastTimestamp | tail -5`: all recent events in the namespace, sorted by time (`--sort-by` = the field to sort on), so the newest are at the bottom; `tail -5` keeps the last 5: `ScalingReplicaSet`, `SuccessfulCreate`, `Scheduled`, `Started`.
- `kubectl delete deployment tour` + `kubectl delete service tour`: clean up.

**🧠 In simple words:** You started a small, healthy app called `tour`: a Deployment (the object that keeps a chosen number of Pods running) with two nginx web-server Pods, plus a ClusterIP Service, which gives those Pods one stable name and address inside the cluster. Then you tried each troubleshooting tool in the usual order of get, describe, logs, exec and events, starting with `get`, which showed both Pods as `1/1 Running` along with their IP (Internet Protocol) addresses, their node and their labels. The `describe` command showed one Pod's full details, and its events were all `Normal` (scheduled, image pulled, created, started), which is exactly what a healthy Pod looks like, while `logs` showed the last three lines nginx printed when it started. With `exec` you opened a shell inside a Pod, where the app answered your request and the `/etc/resolv.conf` file showed that the Pod uses the cluster's DNS (Domain Name System) server to turn names into addresses. After leaving the Pod, `port-forward` connected your Mac's port 8090 straight to the Service's port 80, and the "Welcome to nginx!" page came back, which proves the app and the Service work on their own, with no NodePort or Ingress (the usual ways in from outside the cluster) involved. Last, the newest events in the namespace (the folder your objects live in), sorted by time, told the story of what Kubernetes had just done, from scaling up to starting the Pods, and then you deleted the Deployment and the Service. You practised on an app with nothing wrong so that you know what "normal" looks like before you go hunting for what is broken. In short: `get`, `describe`, `logs`, `exec` and `events` each show a different view of the same app, and you use them in that order.

### ✅ Check 66
**66.1** A Pod has restarted 4 times. You want to see the **exit code and reason** of the container's last crash. Which command shows them?
- A) `kubectl logs <pod>`
- B) `kubectl describe pod <pod>`
- C) `kubectl get pod <pod> -o wide`
- D) `kubectl top pod <pod>`

**66.2** `kubectl logs web-7d4b9` prints `Defaulted container "app" out of: app, log-shipper`, then the app's logs. You need the **log-shipper's** logs. What do you run?
- A) `kubectl logs web-7d4b9 --previous`
- B) `kubectl exec -it web-7d4b9 -- log-shipper`
- C) `kubectl logs deploy/log-shipper`
- D) `kubectl logs web-7d4b9 -c log-shipper`

**66.3** (one line) Why does plain `kubectl logs <pod>` sometimes show nothing useful for a Pod in CrashLoopBackOff, and which flag helps?

---
## Concept 67 · CrashLoopBackOff and other container start failures

**Words you'll learn:** CrashLoopBackOff, back-off, `restartPolicy` (Always / OnFailure / Never), State / Last State, Reason, Exit Code, `Error` vs `Completed`, OOMKilled, the 128 + N rule, memory leak, CreateContainerConfigError, RunContainerError, StartError.

### What CrashLoopBackOff really means
Meera's worker shows `CrashLoopBackOff   5 (40s ago)`, and she thinks Kubernetes is broken. It isn't: **her app keeps exiting**, and Kubernetes keeps restarting it. Read the name in three parts:
- **Crash:** the container's main process **exited** (a container lives only as long as its main process: when that process ends, the container stops).
- **Loop:** Kubernetes restarted it, and it exited again. And again.
- **Back-off:** so as not to hammer the node, the kubelet **waits longer before each new restart**. The wait doubles each time, up to 5 minutes.

> **CrashLoopBackOff is not the cause.** It's Kubernetes saying "this container keeps dying, so I'm waiting before I try again". The cause is inside the container, or in its settings.

**Analogy: redialling a friend whose phone is off.** You wait 1, then 2, then 4 minutes between calls. You're not broken; the phone is off, and you need to find out **why**. With `kubectl get pods -w` you see STATUS cycle: `Running` → `Error` (or `Completed`, if the exit code was 0) → `CrashLoopBackOff`.

**Why it restarts at all: `restartPolicy`.** **`Always`** (the default, and the only value a Deployment allows) restarts the container after **any** exit, even exit code 0. **`OnFailure`** restarts only after a non-zero exit; it's used by **Jobs** (one-off tasks). **`Never`** doesn't restart (`kubectl run --restart=Never`). ⚠️ So a Deployment whose command just runs `echo hello` and exits 0 still ends in **CrashLoopBackOff**: its container must **keep running**.

### The common causes

| Cause | What you see | Where you find it |
|---|---|---|
| App error, missing env var/config | `Error`, **Exit Code 1**, e.g. `DB_HOST is not set` | `kubectl logs --previous` |
| Typo in a shell command | **Exit Code 127**, `sh: pyhton: not found` | logs + describe |
| Command's file not in the image | **`RunContainerError`**, Last State `StartError`, Exit Code 128 | describe (no logs: it never started) |
| Failing liveness probe (the kubelet's "are you alive?" check; fails → restart) | Restarts, Exit Code **137** or **143** | describe → `Liveness probe failed…` |
| Out of memory | `OOMKilled`, Exit Code **137** | describe |
| Main process simply finishes | `Completed`, then CrashLoopBackOff | describe: `Exit Code: 0` |

### Decoding exit codes: the 128 + N rule
When a process is ended by a **signal** (a numbered message the OS sends to a process, like SIGTERM or SIGKILL), its exit code is **128 + the signal's number**:

| Exit code | Arithmetic (128 + signal number) | Meaning |
|---|---|---|
| **137** | 128 + **9** (SIGKILL) | **Killed by force**: the OOM killer, or SIGKILL after the grace period |
| **143** | 128 + **15** (SIGTERM) | **Asked to stop politely**, and it did: normal on delete or rollout |
| **139** | 128 + **11** (SIGSEGV) | **Segmentation fault**: the program crashed on bad memory access |

**OOMKilled:** the container went over its memory **limit** (the most memory it may use, set under `resources.limits`), which cgroups enforce. The kernel's **OOM (Out Of Memory) killer** sent SIGKILL, so the exit code is 137. Fix it by raising the limit, or by fixing a **memory leak** (a bug that keeps grabbing memory and never gives it back).

**The recipe:** `get pods` (STATUS + RESTARTS) → `logs <pod> --previous` (what the dead container printed) → `describe pod` (Last State, Exit Code, Events) → fix the YAML → `apply`.

### Two errors that are NOT crashes
- **`CreateContainerConfigError`:** something the container needs is missing: a ConfigMap or Secret (or a **key** inside one) used by `env`/`envFrom` (the fields that turn ConfigMap/Secret keys into environment variables). The container is **never created**, so RESTARTS stays **0** and there are **no logs**. Its Events say `configmap "app-config" not found` or `couldn't find key db_host…`. Create the missing object, and the kubelet **retries by itself**.
- **`RunContainerError`:** the runtime tried to start the container and couldn't, for example because the command's file doesn't exist in the image or isn't executable. `describe` shows Last State `StartError` with Exit Code 128. Fix `command:` (or the image).

### 🛠 Do (~12 min): three ways to fail, and how each one looks
The manifest is ready-made: `~/DevOPS/sessions/labs/10/practice/crash-demos.yaml` (open it with `code` to read it). It makes three Pods:
- `crash-oom` runs `tail /dev/zero` (endless zero bytes with no line breaks, so memory grows) with a **32Mi** memory limit.
- `crash-typo` runs `pyhton app.py`, a typo (and busybox has no Python anyway).
- `crash-config` reads `DB_HOST` from a ConfigMap `crash-settings` that **doesn't exist yet**.
```bash
kubectl apply -f ~/DevOPS/sessions/labs/10/practice/crash-demos.yaml
kubectl get pods                                          # three different failures
kubectl describe pod crash-oom | grep -A 5 "Last State"   # OOMKilled, 137
kubectl logs crash-typo                                   # sh: pyhton: not found
kubectl describe pod crash-typo | grep -A 3 "Last State"  # Exit Code: 127
kubectl describe pod crash-config | tail -3               # configmap … not found
kubectl logs crash-config                                 # an error: no container yet
kubectl create configmap crash-settings --from-literal=db_host=postgres   # create the missing ConfigMap
kubectl get pod crash-config                              # 1/1 Running
kubectl logs crash-config                                 # DB_HOST is postgres
kubectl delete -f ~/DevOPS/sessions/labs/10/practice/crash-demos.yaml --now   # clean up
kubectl delete configmap crash-settings
```
**What each line does:**
- `kubectl apply -f …/crash-demos.yaml`: creates the three Pods from the one file (`-f` = file).
- `kubectl get pods`: after about 10 s: `crash-config` `CreateContainerConfigError` with RESTARTS 0; `crash-oom` `OOMKilled` with 1; `crash-typo` `Error` with 1 (later `CrashLoopBackOff`).
- `kubectl describe pod crash-oom | grep -A 5 "Last State"`: finds the `Last State` line, and `-A 5` also prints the 5 lines After it: `Terminated`, `Reason: OOMKilled`, `Exit Code: 137`.
- `kubectl logs crash-typo`: what the container printed: `sh: pyhton: not found`.
- `kubectl describe pod crash-typo | grep -A 3 "Last State"`: `Exit Code: 127` = command not found.
- `kubectl describe pod crash-config | tail -3`: `tail -3` = the last 3 lines (the newest Events): `Error: configmap "crash-settings" not found`.
- `kubectl logs crash-config`: an error instead of logs: the container was never created.
- `kubectl create configmap crash-settings --from-literal=db_host=postgres`: creates the missing ConfigMap (`--from-literal` = a key=value typed on the command line).
- `kubectl get pod crash-config`: within about 10 s, `1/1 Running`: the kubelet retried by itself.
- `kubectl logs crash-config`: `DB_HOST is postgres`.
- `kubectl delete -f … --now` + `kubectl delete configmap crash-settings`: clean up everything the file created (`--now` = skip most of the grace period), then the ConfigMap.

**🧠 In simple words:** You applied one ready-made file that creates three Pods, each broken in a different way on purpose. The first, `crash-oom`, runs a command whose memory keeps growing, but it is only allowed 32Mi (about 32 megabytes), so the OOM (Out Of Memory) killer in the kernel, the core of the operating system, stopped it by force, and `describe` showed the reason `OOMKilled` with exit code 137, which is 128 plus 9, the number of the SIGKILL "stop right now" signal. The second, `crash-typo`, tries to run a misspelt command, `pyhton`, so its log said `sh: pyhton: not found` and its exit code was 127, which means "command not found"; Kubernetes keeps restarting it and it keeps failing, so its STATUS moves from `Error` to `CrashLoopBackOff`. The third, `crash-config`, reads a setting from a ConfigMap (a Kubernetes object that stores settings as key-value pairs) that did not exist yet, so its container was never even created: it showed `CreateContainerConfigError` with zero restarts, its `describe` events said the ConfigMap was not found, and asking for its logs only gave an error. As soon as you created the missing ConfigMap, the kubelet (the agent on the node that starts containers) retried by itself, the Pod became `1/1 Running`, and its log printed `DB_HOST is postgres`. At the end you deleted the three Pods and the ConfigMap. In short: the STATUS word tells you which kind of failure you have, and the exit code, the logs and the `describe` events tell you exactly why it happened.

### ✅ Check 67
**67.1** `kubectl describe pod` shows `Last State: Terminated   Reason: OOMKilled   Exit Code: 137`. What happened?
- A) The app itself chose to exit with the code 137
- B) It went over its memory limit and was killed
- C) It went over its CPU limit, so it was killed
- D) Its liveness probe failed, so it got SIGTERM

**67.2** A Deployment's Pod shows `CreateContainerConfigError`, and RESTARTS stays at 0. What's the most likely cause?
- A) The app exits with code 1 right after start
- B) The image tag doesn't exist in the registry
- C) An env var refers to a missing Secret key
- D) The liveness probe fails before it's ready

**67.3** (one line) A container's command is `["sh", "-c", "pyhton app.py"]`. Predict the log line and the exit code, and say what the number means.

---
## Concept 68 · ImagePullBackOff / ErrImagePull, and Pending Pods

**Words you'll learn:** image pull, ErrImagePull, ImagePullBackOff, `imagePullSecrets`, `docker-registry` Secret, `imagePullPolicy`, CPU architecture (amd64 / arm64), `exec format error`, `ErrImageNeverPull`, `minikube image load`, Pending, scheduler, FailedScheduling, allocatable, Insufficient cpu/memory, preemption, node label, `nodeSelector`, taint, toleration, `NoSchedule`.

### Part 1: the image can't be downloaded
Before a container can start, the kubelet must **pull** (download) its image from a **registry** (Docker Hub, GHCR…). If that fails, the container can't even begin.
- **`ErrImagePull`** = the pull just failed. **`ImagePullBackOff`** = the kubelet is waiting before it tries again (the same growing back-off as crash loops). STATUS flips between the two: one problem, two moments of it.
- The container never started, so there are **no logs** (`… waiting to start: image can't be pulled`). Use **describe**.

| Message in `describe` Events (shortened) | Cause | Fix |
|---|---|---|
| `failed to resolve reference "…nginx:1.277": … not found` | The tag doesn't exist | Use a real tag |
| `pull access denied, repository does not exist or may require authorization` (or `401 Unauthorized`) | ⚠️ **A typo in the name** (`ngnix`), or a private repo without login | **Check the spelling first**; else `imagePullSecrets` |
| `lookup registry-1.docker.io … no such host` / `i/o timeout` | No network/DNS from the node | Fix the node's network |
| `no match for platform in manifest`, or a crash with `exec format error` | Wrong CPU architecture (awareness) | A multi-arch or **arm64** image |

⚠️ **Exam trap:** Docker Hub answers "pull access denied … may require authorization" for **any** repository it can't show you, including ones that **don't exist**. So a misspelt name looks like a login problem.
- **Private registries:** save the login in a Secret of type `docker-registry`: `kubectl create secret docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=<user> --docker-password=<token>`. Then list it under the Pod's `spec:` as `imagePullSecrets:` → `- name: regcred`; the kubelet logs in with it before pulling.
- **`imagePullPolicy`** = when to pull. `IfNotPresent`: only if the image isn't on the node yet (the default for a specific tag like `nginx:1.27`). `Always`: check the registry every time (the default for `latest` or no tag). `Never`: never pull, so a missing image gives `ErrImageNeverPull`.
- **CPU architecture:** your Mac's chip is **arm64**. **Multi-arch** images also contain an **amd64** version (for the usual Intel/AMD chips), and the right one is picked automatically. An amd64-only image fails on arm64 nodes.
- **Tip for your Mac:** an image you build with `docker build` lives in Colima's Docker, not inside minikube. Copy it in with `minikube image load myapp:1.0`.

### Part 2: Pending: the Pod can't be placed
A **Pending** Pod was accepted by the API server, but the **scheduler** (the part that chooses a node for each Pod) hasn't found a node for it. `kubectl get pods -o wide` shows NODE and IP as `<none>`, and the Events say `FailedScheduling … 0/1 nodes are available: <reason>`: **the text after the colon is the reason**. (`ContainerCreating` is different: a node was chosen and is preparing. Stuck there usually means a volume problem, `MountVolume.SetUp failed`.)

| Scheduler message contains | Cause | Fix |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | Requests bigger than any node's **unreserved** room | Lower the requests, or add nodes |
| `unbound immediate PersistentVolumeClaims` | The PVC (the Pod's request for storage) exists but is Pending: no volume matches it yet | `kubectl describe pvc` |
| `persistentvolumeclaim "x" not found` | The PVC doesn't exist | Create it, or fix `claimName` |
| `didn't match Pod's node affinity/selector` | `nodeSelector` matches no node's labels | Label a node, or fix the selector |
| `had untolerated taint {dedicated: gpu}` | Tainted node, no toleration | Add a toleration |
| *(no events at all)* | The scheduler isn't running | Check `kube-scheduler-minikube` in `kube-system` |

- ⚠️ **The scheduler counts requests, not real use** (a **request** is the CPU/memory a Pod reserves up front): a node at 5% CPU can still be "full" if Pods have reserved it all. `kubectl describe node` shows **Allocatable** (what Pods may use in total) and **Allocated resources** (what's already requested). A remark like "Preemption is not helpful" (**preemption** = evicting lower-priority Pods to make room) can be ignored.
- **`nodeSelector`:** a **node label** is a label on a node, just like Pod labels (e.g. `disktype=ssd`). A Pod's `spec.nodeSelector` (`disktype: ssd`) means "only put me on nodes with this label". If no node has it, the Pod waits forever.
- **Taints and tolerations (awareness):** a **taint** is a "keep out" mark on a **node** (`key=value:NoSchedule` = don't schedule Pods here unless they tolerate it). A **toleration** on a **Pod** says "I'm allowed past that mark"; it goes under `spec.tolerations` (`key: dedicated`, `operator: Equal`, `value: gpu`, `effect: NoSchedule`). **Analogy:** the taint is a "Staff only" sign; the toleration is a staff badge, which **allows** you in but doesn't **force** you to go in. Real clusters taint their control-plane nodes; your minikube node has `Taints: <none>`.

### 🛠 Do (~12 min): one pull failure and two Pending Pods
The manifest is ready-made: `~/DevOPS/sessions/labs/10/practice/pull-pending.yaml`. It makes three Pods:
- `pull-typo` uses the image `ngnix:1.27` (misspelt).
- `pending-selector` has `nodeSelector:` → `disktype: ssd`, a label no node has.
- `pending-pvc` mounts a PVC named `does-not-exist`.
```bash
kubectl apply -f ~/DevOPS/sessions/labs/10/practice/pull-pending.yaml
kubectl get pods -o wide                          # 2 × Pending + 1 ErrImagePull
kubectl describe pod pull-typo | tail -4          # pull access denied …
kubectl logs pull-typo                            # waiting to start
kubectl describe pod pending-selector | tail -2   # didn't match … selector
kubectl describe pod pending-pvc | tail -2        # persistentvolumeclaim … not found
kubectl label node minikube disktype=ssd          # add the label
kubectl get pod pending-selector -o wide          # Running on minikube
kubectl label node minikube disktype-             # remove the label
kubectl describe node minikube | grep -A 6 "Allocated resources"   # what's reserved
kubectl describe node minikube | grep Taints      # Taints: <none>
kubectl delete -f ~/DevOPS/sessions/labs/10/practice/pull-pending.yaml   # clean up
```
**What each line does:**
- `kubectl apply -f …/pull-pending.yaml`: creates the three Pods.
- `kubectl get pods -o wide`: the two `pending-…` Pods are `Pending` with NODE and IP `<none>` (never placed). `pull-typo` shows `ErrImagePull` but **has** a node and an IP: it was placed, and only the pull failed.
- `kubectl describe pod pull-typo | tail -4`: the newest Events: `pull access denied, repository does not exist…`: here, the name is misspelt.
- `kubectl logs pull-typo`: no logs, just `… is waiting to start: image can't be pulled`.
- `kubectl describe pod pending-selector | tail -2`: `0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector`.
- `kubectl describe pod pending-pvc | tail -2`: `0/1 nodes are available: persistentvolumeclaim "does-not-exist" not found`.
- `kubectl label node minikube disktype=ssd`: adds the label `disktype=ssd` to the node; the scheduler retries and places `pending-selector`.
- `kubectl get pod pending-selector -o wide`: `Running` on `minikube`.
- `kubectl label node minikube disktype-`: the key followed by a minus removes the label (the Pod already running stays).
- `kubectl describe node minikube | grep -A 6 "Allocated resources"`: that heading plus the 6 lines after it: what Pods have already **requested**, e.g. `cpu 1370m (22%)` requests, `2700m (45%)` limits.
- `kubectl describe node minikube | grep Taints`: `Taints: <none>`: no "keep out" marks on your node.
- `kubectl delete -f …/pull-pending.yaml`: clean up all three Pods.

**🧠 In simple words:** You applied one ready-made file that creates three broken Pods. The first, `pull-typo`, asks for an image with a misspelt name, `ngnix`, so it was placed on the node but its image could never be downloaded: STATUS showed `ErrImagePull`, `describe` said "pull access denied, repository does not exist", which is what Docker Hub says for any image name it cannot show you, even one that simply does not exist, and there were no logs because the container never started. The other two stayed `Pending` with no node and no IP (Internet Protocol) address, because the scheduler (the part of Kubernetes that picks a node for each Pod) could not find a place for them, and the newest event in each Pod's `describe` gave the exact reason after the words "0/1 nodes are available". For `pending-selector`, the reason was that it only accepts a node labelled `disktype=ssd` and no node had that label; for `pending-pvc`, it was that the storage request it needs, a PVC (PersistentVolumeClaim) called `does-not-exist`, really does not exist. When you added the `disktype=ssd` label to the minikube node, the scheduler tried again and placed `pending-selector` there, where it started running, and removing the label afterwards did not stop the Pod that was already running. You also looked at the node itself: it showed how much the Pods have already requested (reserved up front), for example about 22% of its CPU (Central Processing Unit, the processor), and that it has no taints, which are "keep out" marks that turn Pods away unless they carry a matching toleration, a permission to pass that mark. Finally you deleted all three Pods. In short: an image error means the Pod was placed but its image could not be downloaded, while Pending means the Pod was never placed at all, and in both cases `describe` tells you why.

### ✅ Check 68
**68.1** Events: `Failed to pull image "acme/billing:2.1": … pull access denied, repository does not exist or may require authorization`. The name is spelled correctly, and the image exists in your company's **private** Docker Hub repository. What's the fix?
- A) Set `imagePullPolicy: Never` on the container
- B) Add `imagePullSecrets` with registry login
- C) Change the tag to `latest` so it's found
- D) Run `docker pull` on your Mac first

**68.2** A Pod is Pending. Events: `0/3 nodes are available: 3 Insufficient memory.` Each node has 4 GiB; `kubectl top nodes` shows only 30% memory **in use**. The Pod **requests** `6Gi`. What's the best fix?
- A) Wait, since actual use is only 30% now
- B) Lower the request so it fits a node
- C) Raise the Pod's memory limit above 6Gi
- D) Add a toleration for memory pressure

**68.3** (one line) Explain taint vs toleration in contrast form.

---
## Concept 69 · Service & DNS troubleshooting

**Words you'll learn:** endpoints, EndpointSlice, selector mismatch, `targetPort` / `containerPort`, bind address, `Connection refused` vs timeout vs `bad address`, CoreDNS, `kube-dns`, FQDN, search domains, `/etc/resolv.conf`, NXDOMAIN, Ingress 404 / 502 / 503, default backend, NetworkPolicy.

### The problem
All Pods are `1/1 Running` with no restarts, yet `wget http://lab-shop-svc` from another Pod fails. The containers are fine; **the path to them** is broken. A request travels this path, and every arrow can break: name → **DNS** (CoreDNS) → ClusterIP → **endpoints** (the Ready Pods matching the selector) → `PodIP:targetPort` → the app, listening on `0.0.0.0`.

**Analogy: posting a letter to a flat.** The street must be in the directory (DNS), the building must list its residents (endpoints), and the flat number must be right (targetPort). Someone must also answer the door to visitors, not just to housemates (`0.0.0.0` vs `127.0.0.1`).

### The checklist, in order
**Check 1: does the Service have endpoints?** Run `kubectl describe svc <s> | grep -E "Selector|Endpoints"`. The **endpoints** are the IPs of the Pods the Service sends traffic to (an **EndpointSlice** is the newer object that holds the same list). Empty endpoints have three causes:
1. The **selector doesn't match** the Pods' labels: a typo (`lab-shopp`) or a case difference (`Web` vs `web`), because **labels are case-sensitive**. `kubectl get pods -l app=lab-shopp` printing `No resources found` proves it.
2. The Pods **aren't Ready** (a failing readiness probe: the "can you take traffic?" check, which keeps a Pod out of the Service until it passes).
3. The Pods are in **another namespace**: a Service only selects Pods in its own namespace.

**Check 2: is `targetPort` right?** The endpoints show `PodIP:targetPort` (e.g. `10.244.0.16:8080`). If the app really listens on 80, nothing answers on 8080 → **`Connection refused`**. ⚠️ `containerPort` in the Pod spec is mostly **documentation**: it doesn't open any port.

**Check 3: does the app listen on `0.0.0.0`?** An app bound to `127.0.0.1` (**loopback**) accepts connections only from **its own Pod**. It must listen on `0.0.0.0` (all interfaces), as our Flask app does with `app.run(host="0.0.0.0")`. The tell-tale: endpoints are fine, the test **inside** the Pod works, and the test **from another Pod** is refused.

**Check 4: does the name resolve (DNS)?**
- **CoreDNS** is the cluster's DNS server: Pods labelled `k8s-app=kube-dns` in `kube-system`, behind a Service named **`kube-dns`** (`10.96.0.10`).
- Each Pod's `/etc/resolv.conf` has `nameserver 10.96.0.10` and a `search` line. **Search domains** are endings tried automatically on short names, which is why a plain `api` works only from the **same namespace**.
- The **FQDN** (Fully Qualified Domain Name) is `<service>.<namespace>.svc.cluster.local`. From `default`, the Service `api` in namespace `shop` needs at least `api.shop`.
- Use the **Service's** name, not the Deployment's. If nothing resolves, not even `kubernetes.default`, check the CoreDNS Pods and their logs.
- **NXDOMAIN** ("Non-Existent Domain") is DNS's answer for "no such name". ⚠️ Don't test a Service with `ping`: a ClusterIP is virtual and usually doesn't answer ping. Use `wget` and `nslookup`.

**Reading the error words** tells you which check failed:

| `wget`/`curl` says | Meaning | Usual causes |
|---|---|---|
| **`bad address 'x'`** / `could not resolve host` / NXDOMAIN | **DNS** couldn't find the name | Wrong Service name or namespace, CoreDNS down |
| **`Connection refused`** | The IP was reached, but **nothing listens** on that port | Wrong targetPort, app on 127.0.0.1, **no endpoints** |
| **Timeout** (hangs, then gives up) | Packets are **dropped** somewhere | A NetworkPolicy, a firewall, a wrong IP |
| Ingress **404** | No rule matched | Wrong host/path/`ingressClassName`; the **default backend** (the catch-all) answered |
| Ingress **503** | A rule matched, but **no ready endpoints** | Check 1 |
| Ingress **502** | Endpoints exist, but connecting **failed** | Checks 2–3, or a crashing app |

**NetworkPolicy** (awareness): a **firewall rule for Pods** (e.g. "only `app=web` may talk to the database"). A "default deny" policy silently **drops** traffic, so you get timeouts, not refusals. It only works if the cluster's network plugin enforces it (e.g. Calico). Check with `kubectl get networkpolicy`.

### 🛠 Do (~15 min): the 127.0.0.1 trap, and DNS checks
`code bind-demo.yaml`, paste, save. busybox's tiny web server `httpd` serves a page: `-f` = stay in the foreground, `-v` = log requests, **`-p 127.0.0.1:8000` = listen on loopback port 8000 only**, `-h /www` = serve that folder.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bind-demo
  labels:
    app: bind-demo                # kubectl expose copies this label into the Service's selector
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "mkdir -p /www && echo 'hello from the app' > /www/index.html && httpd -f -v -p 127.0.0.1:8000 -h /www"]
```
```bash
kubectl apply -f bind-demo.yaml
kubectl expose pod bind-demo --port=80 --target-port=8000         # Service: 80 → 8000
kubectl run dns-test --image=busybox:1.28 --restart=Never -- sleep 3600   # a client Pod
kubectl describe svc bind-demo | grep -E "TargetPort|Endpoints"   # endpoints are FINE
kubectl exec dns-test -- wget -qO- -T 3 http://bind-demo          # Connection refused
kubectl exec bind-demo -- wget -qO- http://127.0.0.1:8000         # works inside
kubectl exec bind-demo -- netstat -tln                            # 127.0.0.1:8000 LISTEN
```
**What each line does:**
- `kubectl apply -f bind-demo.yaml`: creates the Pod.
- `kubectl expose pod bind-demo --port=80 --target-port=8000`: a Service `bind-demo` for this one Pod. `--port` = the Service's port (80); `--target-port` = the Pod's port, where the app listens (8000).
- `kubectl run dns-test --image=busybox:1.28 --restart=Never -- sleep 3600`: a client Pod that sleeps for an hour so you can run tests from it (`--restart=Never` = a plain Pod; `--` = the command follows). Version 1.28 because its `nslookup` prints clean answers.
- `kubectl describe svc bind-demo | grep -E "TargetPort|Endpoints"`: keeps only those two lines (`grep -E "A|B"` = A or B): `8000/TCP` and `10.244.0.x:8000`. The endpoints are **fine**.
- `kubectl exec dns-test -- wget -qO- -T 3 http://bind-demo`: fetches the Service from the client Pod (`-q` = quiet, `-O-` = print to the screen, `-T 3` = give up after 3 s). Result: `Connection refused`.
- `kubectl exec bind-demo -- wget -qO- http://127.0.0.1:8000`: the same request **inside** the app's own Pod: `hello from the app`. It works there.
- `kubectl exec bind-demo -- netstat -tln`: lists the listening sockets (`-t` = TCP, `-l` = listening only, `-n` = numbers, not names). `127.0.0.1:8000 LISTEN` is the cause: loopback only.

**Fix:** in `bind-demo.yaml`, change `-p 127.0.0.1:8000` to **`-p 8000`** (all interfaces). A Pod's command can't be edited in place, so delete and re-apply:
```bash
kubectl delete pod bind-demo --now
kubectl apply -f bind-demo.yaml
kubectl exec bind-demo -- netstat -tln                    # :::8000
kubectl exec dns-test -- wget -qO- -T 3 http://bind-demo  # hello from the app
kubectl exec dns-test -- nslookup bind-demo               # the FQDN + ClusterIP
kubectl exec dns-test -- nslookup kubernetes.default      # 10.96.0.1
kubectl exec dns-test -- nslookup bind-demo-typo          # can't resolve
kubectl get pods -n kube-system -l k8s-app=kube-dns       # coredns-… Running
kubectl get svc -n kube-system kube-dns                   # 10.96.0.10
kubectl delete pod bind-demo dns-test --now               # clean up
kubectl delete service bind-demo
```
**What each line does:**
- `kubectl delete pod bind-demo --now` + `kubectl apply -f bind-demo.yaml`: replaces the Pod with the fixed one (`--now` = skip most of the grace period).
- `kubectl exec bind-demo -- netstat -tln`: now `:::8000` = all interfaces (the IPv6 form, which covers IPv4 too).
- `kubectl exec dns-test -- wget -qO- -T 3 http://bind-demo`: `hello from the app`: the Service works now.
- `kubectl exec dns-test -- nslookup bind-demo`: asks DNS for the short name: `10.105.x.x bind-demo.default.svc.cluster.local`. The search domains turned it into the FQDN.
- `kubectl exec dns-test -- nslookup kubernetes.default`: `10.96.0.1`, the API server's own Service: a quick "does DNS work at all?" test.
- `kubectl exec dns-test -- nslookup bind-demo-typo`: `can't resolve 'bind-demo-typo'`: what a DNS failure looks like.
- `kubectl get pods -n kube-system -l k8s-app=kube-dns`: the CoreDNS Pods (`-n` = namespace, `-l` = label selector): `coredns-… Running`.
- `kubectl get svc -n kube-system kube-dns`: the DNS Service: `10.96.0.10`, ports `53/UDP,53/TCP,9153/TCP` (DNS mostly uses UDP; 9153 = metrics).
- `kubectl delete pod bind-demo dns-test --now` + `kubectl delete service bind-demo`: clean up.

**🧠 In simple words:** You started a tiny web server Pod that serves the page "hello from the app", but on purpose you told it to listen only on 127.0.0.1, the loopback address, which accepts connections only from inside that same Pod. You put a Service in front of it, sending the Service's port 80 to the Pod's port 8000, and started a second Pod called `dns-test` that just sleeps, so you could send test requests from it. The Service looked perfect, with the right target port and one endpoint (the Pod address and port the Service sends traffic to), yet a request from the client Pod got `Connection refused`, while the same request made inside the app's own Pod worked, and `netstat` (a tool that lists which ports are listening) showed the server listening on `127.0.0.1:8000` only. That was the cause: the Service sends requests to the Pod's own network address, not to loopback, so nothing was listening for them there. A Pod's command cannot be changed while the Pod exists, so you deleted it and created it again listening on all addresses, and this time the client got "hello from the app" through the Service. Then you checked DNS (Domain Name System, which turns names into IP, or Internet Protocol, addresses): the short name `bind-demo` was automatically completed to its full name `bind-demo.default.svc.cluster.local` with the Service's address, `kubernetes.default` also resolved, which proves DNS works at all, and a misspelt name gave "can't resolve", which is what a DNS failure looks like. You also saw that CoreDNS, the cluster's own DNS server, was running in the `kube-system` namespace (where the cluster's own parts live) behind a Service named `kube-dns` at 10.96.0.10, and then you cleaned everything up. In short: a Service can have perfect endpoints and still fail, because an app listening only on 127.0.0.1 refuses every visitor from outside its own Pod.

### ✅ Check 69
**69.1** Pods run with the label `app: web` (all `1/1 Running`). The Service has `selector: app: Web`. `kubectl describe svc` shows `Endpoints:` empty. Why?
- A) Selectors are case-sensitive; `Web` ≠ `web`
- B) Endpoints only appear after a DNS lookup
- C) The pods need a readiness probe to be listed
- D) The Service must be a NodePort to get them

**69.2** Through the Ingress, `shop.local` returns `503 Service Temporarily Unavailable`, while `nothing.local` returns `404`. What does the 503 tell you?
- A) No Ingress rule matched the host shop.local
- B) A rule matched, but no backend is Ready
- C) The Ingress controller itself isn't running
- D) The app returned 503 because of a code bug

**69.3** (one line) `wget` from one Pod to a Service fails. Contrast what `bad address`, `Connection refused` and a timeout each point to.

---
## 🧪 Broken cluster lab (~25 min): diagnose five faults on your own

Five ready-made manifests in `~/DevOPS/sessions/labs/10/`, each with **one** fault. The names don't give it away, and two look perfectly healthy in `get pods`. Use the method: **get → describe → logs → exec → events**.
For each, note **symptom → command that revealed it → cause → fix** in `MY_ANSWERS.md`. To fix one, copy it into `~/devops-practice/k8s-vi/lab/`, edit the copy, `kubectl apply -f` it, and prove it works. (The originals stay broken for next time.)

- **lab-1:** `kubectl apply -f ~/DevOPS/sessions/labs/10/lab-1-frontend.yaml`. A Deployment `lab-frontend` (nginx). Find out why its Pod never runs.
- **lab-2:** `kubectl apply -f ~/DevOPS/sessions/labs/10/lab-2-worker.yaml`. A Deployment `lab-worker` (a busybox script). Find out why it keeps restarting.
- **lab-3:** `kubectl apply -f ~/DevOPS/sessions/labs/10/lab-3-reports.yaml`. A Deployment `lab-reports` (nginx). Find out why it never starts.
- **lab-4:** `kubectl apply -f ~/DevOPS/sessions/labs/10/lab-4-shop.yaml`. Deployment `lab-shop` + Service `lab-shop-svc`. The Pods look fine; test the Service.
- **lab-5:** `kubectl apply -f ~/DevOPS/sessions/labs/10/lab-5-api.yaml`. Deployment `lab-api` + Service `lab-api-svc`. The Pod looks fine; test the Service.
```bash
kubectl run lab-client --image=busybox:1.28 --restart=Never -- sleep 3600   # a client Pod for testing the Services
kubectl get pods                                                            # start the method here
kubectl exec lab-client -- wget -qO- -T 3 http://lab-shop-svc               # test lab-4's Service
kubectl exec lab-client -- wget -qO- -T 3 http://lab-api-svc                # test lab-5's Service
kubectl delete -f ~/DevOPS/sessions/labs/10/                                # clean up all five labs (subfolders aren't touched)
kubectl delete pod lab-client --now
kubectl get all                                                             # only service/kubernetes should remain
```
**What each line does:**
- `kubectl run lab-client --image=busybox:1.28 --restart=Never -- sleep 3600`: a client Pod that sleeps for an hour, so you can test the Services from inside the cluster.
- `kubectl get pods`: step 1 of the method: read each lab Pod's STATUS and RESTARTS to decide which tool to use next.
- `kubectl exec lab-client -- wget -qO- -T 3 http://lab-shop-svc` / `…/lab-api-svc`: request lab-4's and lab-5's Services by name from the client Pod (3-second limit). The error words (`bad address`, `Connection refused`, timeout) tell you which layer fails.
- `kubectl delete -f ~/DevOPS/sessions/labs/10/`: when you're done: `-f` with a **folder** deletes everything defined in the YAML files directly inside it (the `practice` subfolder isn't touched).
- `kubectl delete pod lab-client --now`: removes the client Pod.
- `kubectl get all`: checks the cleanup: only `service/kubernetes` should remain.

**Hints (only if stuck 5+ min):**
- Status flips between two "pull" words → read the `Failed` event's last few words.
- `Error` / `CrashLoopBackOff` → what did the dead container print? What can a container have besides `command`?
- NODE is `<none>` → how many CPU cores does your node have? (`kubectl describe node minikube | grep -A 2 Allocatable`)
- `Connection refused` for both Services → compare `describe svc`: one has **no** endpoints, one has endpoints on a **suspicious port**.

---
## ➕ Small but important (troubleshooting): read before the quiz

| Item | What it is |
|---|---|
| **Exit codes** | **0** success · **1** app error · **126** found but **not executable** · **127** **command not found** · **137** SIGKILL · **139** SIGSEGV · **143** SIGTERM |
| **Stuck `Terminating`** (awareness) | Often a **finalizer**: a note saying "don't delete me until some cleanup is done" (e.g. `kubernetes.io/pvc-protection` keeps a PVC while a Pod uses it). Last resort: `kubectl delete pod <p> --grace-period=0 --force` |
| **`kubectl debug`** (awareness) | `kubectl debug -it <pod> --image=busybox:1.28 --target=<container>` adds an **ephemeral container** (a temporary troubleshooting container) to a Pod whose image has **no shell** |
| **`kubectl rollout status deploy/<d>`** | Waits for a rollout to finish; with `--timeout=60s` it fails with `timed out waiting for the condition`. Old Pods keep serving; undo with `kubectl rollout undo deploy/<d>` |
| **`kubectl auth can-i`** (awareness) | "Am I allowed?": `kubectl auth can-i create pods` → yes/no. Explains `Forbidden` errors (**RBAC**, Role-Based Access Control) |
| **Node `NotReady`** | The node's **kubelet** stopped reporting. `kubectl describe node` → Conditions (`MemoryPressure`, `DiskPressure`, `Ready`) |
| **`Evicted`** | A node short of memory/disk pushed the Pod off; **BestEffort** Pods (no requests or limits set) go first |
| **`Init:0/1`, `Init:CrashLoopBackOff`** | An **init container** (one that must finish **before** the main ones start) is still running or failing: `kubectl logs <pod> -c <init-name>` |
| **Catching YAML mistakes** | `kubectl apply --dry-run=server -f f.yaml` checks a file without creating anything: `error converting YAML to JSON` (indentation), `unknown field "contaners"` (typo) |
| **ResourceQuota rejections** | The Pods **never appear** (not even Pending): `kubectl describe rs <rs>` → `FailedCreate … exceeded quota` |

---
## 📝 Session quiz (4 MCQs + 2 one-liners), from memory

**Q1.** `kubectl describe pod` shows `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`. The container's memory limit is `128Mi`, and `kubectl top` showed it at ~125Mi just before each crash. CPU use is low. What's the best fix?
- A) Raise the CPU limit so it can finish its work
- B) Add a livenessProbe so it restarts in time
- C) Raise the memory limit (or fix the leak)
- D) Let an HPA add pods when memory gets high

**Q2.** Service `api-svc` shows `Endpoints: 10.244.0.12:3000,10.244.0.13:3000`. From another Pod, `wget http://api-svc` gives `Connection refused`. Inside an api Pod, `wget -qO- http://localhost:3000` works. What's the most likely cause?
- A) The selector doesn't match the pod labels
- B) The app listens on 127.0.0.1, not 0.0.0.0
- C) CoreDNS can't resolve the name api-svc
- D) The readiness probe is failing on the pods

**Q3.** A Pod is Pending. Events: `0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims`. On minikube, `kubectl get pvc` shows the claim `Pending` with STORAGECLASS `gp2`. What fixes it?
- A) Lower the Pod's CPU request so it can fit
- B) Delete the Pod so the scheduler retries it
- C) Add a toleration for the minikube node
- D) Use `standard` or omit storageClassName

**Q4.** You ran `kubectl set image deploy/web web=nginx:1.28-alpne`. `kubectl get pods` now shows one new Pod `0/1 ImagePullBackOff` and the three old Pods still `1/1 Running`. Events: `… nginx:1.28-alpne: not found`. What's the situation, and the quickest safe fix?
- A) Old pods still serve; run rollout undo
- B) All pods are down; recreate the Deployment
- C) New pods serve; delete the old ReplicaSet
- D) Old pods were deleted; scale replicas to 0

**L1.** Explain CrashLoopBackOff vs ImagePullBackOff in contrast form, naming the first command you'd run for each.

**L2.** What does exit code **137** mean? Show the arithmetic, and give two different situations in Kubernetes that produce it.

### ✍️ Write the command
**C1.** Pod `web-7d4b9` is in CrashLoopBackOff. Show what its container printed just before it died the last time.

**C2.** Pod `web-7d4b9` has two containers, `app` and `log-shipper`. Watch the `log-shipper` container's logs live, as new lines arrive.

**C3.** List every event in the current namespace, ordered by time so the newest event is at the bottom.

**C4.** Open an interactive shell (`sh`) inside the Pod `api-5f6c8`.
