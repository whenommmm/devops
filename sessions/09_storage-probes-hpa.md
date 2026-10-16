# Session 09 · Kubernetes V: storage, probes & autoscaling (Oct 15, Session 1)

**Time:** ~100–110 min · **Concepts:** 62 → 65 · **Answers:** `MY_ANSWERS.md` or say "K8s V questions" · **Answer key:** `09_storage-probes-hpa_ANSWERS.md`

**Before you start:**
```bash
minikube status        # host, kubelet, apiserver: Running (if not: minikube start)
kubectl get nodes      # minikube   Ready
mkdir -p ~/devops-practice/k8s-v && cd ~/devops-practice/k8s-v   # today's folder; all YAML goes here
```
> ⚠️ **Never delete the `minikube` or `localstack` containers.** Leave namespaces you didn't create (`argocd`, `ingress-nginx`, `final-app`) and their PersistentVolume alone. Today everything runs in `default`.

---
## 🔁 Warm-up (from Kubernetes IV and your weak list)

**W1.** A teammate says a Secret is safe because `kubectl get secret db-pass -o yaml` shows `password: cGFzc3dvcmQ=` instead of the real password. What's accurate?
- A) It's encrypted with the cluster's private key
- B) It's only encoded; `base64 -d` reveals it
- C) It's hashed, so nobody can ever read it back
- D) It's safe, because only etcd can decode it

**W2.** You apply an Ingress for host `shop.local` → Service `shop-svc:80`. `kubectl get ingress` lists it, but its ADDRESS column stays empty and no request is ever answered. `kubectl get pods -A | grep -i ingress` prints nothing. What's wrong?
- A) The host must be a real public domain name
- B) The Service must be changed to NodePort first
- C) The Ingress needs `type: LoadBalancer` added
- D) No Ingress controller exists to act on it

**W3.** What does this print?
```bash
APP=backend
echo 'Deploying $APP to "prod"'
```
- A) `Deploying $APP to "prod"`
- B) `Deploying backend to "prod"`
- C) `Deploying backend to prod`
- D) `Deploying $APP to prod`

---
## Concept 62 · Volumes in Pods: giving containers somewhere to keep files

**Words you'll learn:** ephemeral, volume, mount, `volumes`, `volumeMounts`, `mountPath`, `emptyDir`, sidecar, `hostPath`, `configMap`/`secret` volume, `readOnly`, tmpfs.

### The problem
Files a container writes go into its **writable layer** (the thin read-write layer on top of the image's read-only layers), which is **ephemeral** (short-lived): when a container crashes, the **kubelet** (the agent on each node) restarts it with a fresh, empty layer. **Story:** Asha's Pod has an `app` container that writes logs to `/var/log/app`, and a `shipper` container that should send them to a log server. But each container has its own filesystem, so `shipper` can't even see the files, and they vanish whenever `app` restarts.

### The idea: a volume
> A **volume** is **a folder that belongs to the Pod**, not to any one container. Each container can **mount** it (plug it in) at a folder path of its choice.

**Analogy: a shared drawer in an office room.** The Pod is the room, the containers are colleagues, and the volume is a drawer that belongs to **the room**. A replacement colleague (a restarted container) still finds the papers. What happens when the room itself is cleared out depends on the drawer's **type**:

| Type | What it is | Survives a container restart? | Survives Pod deletion? |
|---|---|---|---|
| **`emptyDir`** | An empty folder, created when the Pod starts | ✅ | ❌ deleted with the Pod |
| **`hostPath`** | A folder on the **node's** own disk | ✅ | ✅ but only on **that node** |
| **`configMap`** / **`secret`** | Each key of the object shown as a **file** | ✅ | (they're separate objects) |
| **`persistentVolumeClaim`** | Real, lasting storage, requested through a claim | ✅ | ✅ until you delete the claim |

- **`emptyDir`** lives exactly as long as the Pod. It's scratch space, and the usual way to share files with a **sidecar**: a helper container running beside the main app in the same Pod (like `shipper`). With `medium: Memory` it lives in RAM (**tmpfs**, as in Docker): fast, but it counts against the memory limit.
- **`hostPath` is risky:** a Pod recreated on another node sees *that* node's folder (different data), and `hostPath: /` would expose the node's own files. It only *seems* fine on minikube, which has a single node.
- **`configMap` / `secret` volumes**: **each key becomes a file, and the file's content is the value**. Keys `color=blue` and `mode=dev` mounted at `/etc/app` give the files `color` (containing `blue`) and `mode`. Secret files live in memory, already decoded from base64.
- **Updates:** change the ConfigMap, and the **mounted files update by themselves** within about a minute. **Environment variables don't**: they're read once, when the container starts.

**How it's written: two halves, joined by a name.** **`volumes`** (at Pod level) says **what** storage the Pod has: a name plus a type. **`volumeMounts`** (inside each container) says **where** that container sees it: the volume's name plus a **`mountPath`**. ⚠️ **Exam trap:** the names must **match exactly**, or `kubectl apply` fails with `Not found: "config"`.

### 🛠 Do (~10 min): two containers sharing an emptyDir
`code shared-pod.yaml`, paste, save (busybox = a tiny image with `sh`, `date` and `cat`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-pod
spec:
  containers:
    - name: writer                # appends the date to /shared/log.txt every 5 s
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /shared/log.txt; sleep 5; done"]
      volumeMounts:               # ② WHERE this container plugs the volume in
        - name: shared-data       #    must match a name under volumes
          mountPath: /shared      #    the folder path inside THIS container
    - name: reader                # just sleeps, so we can look inside
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data        # same volume, different path: same files
          readOnly: true          # this container can't change the files (like Docker's :ro)
  volumes:                        # ① WHAT storage the Pod has: a name + a type
    - name: shared-data
      emptyDir: {}                # {} = an empty map = default settings
```
```bash
kubectl apply -f shared-pod.yaml
kubectl get pod shared-pod                                # shared-pod   2/2   Running  (2 of 2 containers ready)
kubectl exec shared-pod -c reader -- cat /data/log.txt    # dates the writer wrote; -c = which container, -- = the command follows
kubectl delete pod shared-pod                             # delete the Pod…
kubectl apply -f shared-pod.yaml                          # …and recreate it
kubectl exec shared-pod -c reader -- cat /data/log.txt    # only 1–2 fresh dates: the emptyDir died with the old Pod
kubectl delete pod shared-pod                             # clean up
```

### ✅ Check 62
**62.1** A Pod has an `app` container that writes logs to `/var/log/app`, and a `shipper` container that reads them and sends them to a log server. The logs only need to survive **container restarts**, not Pod deletion. Which volume fits best?
- A) `hostPath` with `/var/log` on the node
- B) A `configMap`, mounted in both containers
- C) `emptyDir`, mounted in both containers
- D) A separate PVC for each of the containers

**62.2 🐞** `kubectl apply` fails with `spec.containers[0].volumeMounts[0].name: Not found: "config"`. Find the bug (one line):
```yaml
  containers:
    - name: app
      image: busybox:1.36
      volumeMounts:
        - name: config
          mountPath: /etc/app
  volumes:
    - name: app-config
      configMap:
        name: app-settings
```

**62.3** You mount ConfigMap `app-settings` (keys `color=blue`, `mode=dev`) as a volume at `/etc/app`. What does `ls /etc/app` show inside the container?
- A) One file, `app-settings`, holding both keys
- B) Two files, `color` and `mode`, one per key
- C) Two env vars, `COLOR` and `MODE`, not files
- D) Nothing until the Pod is restarted by hand

---
## Concept 63 · PersistentVolume, PersistentVolumeClaim & StorageClass: storage that outlives Pods

**Words you'll learn:** PersistentVolume (PV), PersistentVolumeClaim (PVC), StorageClass, provisioner, static/dynamic provisioning, binding, `Mi`/`Gi`, access modes (RWO, ROX, RWX, RWOP), reclaim policy (Retain, Delete, Recycle), statuses (Available, Bound, Released, Pending), cluster-scoped, default StorageClass, `volumeBindingMode`.

### The problem
Ravi's team runs a **PostgreSQL** database. Its storage must **outlive the Pod** (so not emptyDir) and **follow the Pod to any node** (so not hostPath). And developers shouldn't have to be storage experts: they just want to say "I need 1 GiB I can read and write."

### The idea: separate "what I need" from "what exists"
- **PersistentVolume (PV):** a **real piece of storage** (a disk, a network folder) with a size and rules. *Persistent* = lasting. PVs are **cluster-scoped**: they belong to the whole cluster, not to a namespace.
- **PersistentVolumeClaim (PVC):** a developer's **request** for storage ("100 MiB, read-write"). It lives in a namespace, and a Pod names the **claim**, never the PV.
- **StorageClass:** a **category** of storage plus its **provisioner**, the component that **creates PVs automatically** when a claim asks for that category.

**Analogy: booking a hotel room.** The PVC is your **booking** ("one double room": you don't pick the number). The PV is the **actual room 204**. The StorageClass is the **room category plus the housekeeping staff** who prepare rooms on demand. **Binding** is reception assigning room 204 to your booking. The Pod is the **guest**, who quotes the booking.
```
  Pod  ──uses──►  PVC  ──bound to──►  PV  ──is really──►  actual storage
 (guest)        (booking)          (room 204)           (disk / network folder)
```
**Provisioning** = getting storage ready. **Static:** an admin creates PVs by hand, in advance. **Dynamic:** the provisioner creates a PV the moment a PVC arrives; this is used almost everywhere, including minikube. A PVC that names no class gets the **default StorageClass**. **Units:** `Mi` = mebibyte (1024 × 1024 bytes), `Gi` = gibibyte (1024 Mi); `M` and `G` are powers of 1000.

### Access modes, binding and reclaim policy

| Access mode | Full form | Meaning |
|---|---|---|
| **RWO** | ReadWriteOnce | Read-write by **one NODE** at a time |
| **ROX** | ReadOnlyMany | Read-only by **many nodes** |
| **RWX** | ReadWriteMany | Read-write by **many nodes** (needs shared storage like NFS, the Network File System) |
| **RWOP** | ReadWriteOncePod | Read-write by **one single Pod** in the whole cluster |

⚠️ **Exam trap: RWO means one *node*, not one *Pod*.** Pods on the same node can share it. Most cloud disks (like Amazon's EBS) support **only RWO**.

**Binding** connects **one PVC to one PV** (always 1 : 1). A PV qualifies only if it has the **same class**, **includes the access mode** asked for, and is **at least the size** asked for (ask for 150Mi, get a 200Mi PV: you get all 200Mi). A PV's status goes **Available** (free) → **Bound** (in use by a claim) → **Released** (its claim was deleted); a PVC's goes **Pending** (no match yet) → **Bound**.

**Reclaim policy** = what happens to the PV when its PVC is deleted:
- **Retain:** the PV **stays, with its data**, as **`Released`**, and isn't given to a new claim until an admin cleans up (the room stays locked with the guest's things inside). Use it for precious data.
- **Delete:** the PV **and the real storage** are deleted: data gone. Usual for dynamic PVs. (**Recycle**, a wipe-and-reuse, is deprecated.)

**Why a PVC is stuck `Pending`:** run `kubectl describe pvc <name>` and read the **Events** at the bottom.

| Cause | Clue in the Events |
|---|---|
| The StorageClass doesn't exist (a typo, or a cloud class like `gp2` on minikube) | `storageclass… "fast-ssd" not found` |
| Static setup: no PV has enough size, the right mode and the right class | `no persistent volumes available for this claim…` |
| The class's **`volumeBindingMode`** (*when* to bind) is `WaitForFirstConsumer` | `waiting for first consumer…`: **normal**, it binds once a Pod uses it |

### 🛠 Do (~12 min): data that survives the Pod
`code notes-pvc.yaml`, paste, save:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: notes-pvc
spec:                     # no storageClassName → the default class (dynamic provisioning)
  accessModes:
    - ReadWriteOnce       # one node at a time
  resources:
    requests:
      storage: 100Mi      # how much space we ask for
```
Ready-made in `~/DevOPS/sessions/labs/09/` (open with `code` to read): `notes-pod.yaml`, a busybox Pod that appends one timestamped line to `/data/notes.txt`, then sleeps (its volume is `persistentVolumeClaim:` → `claimName: notes-pvc`); and `bad-pvc.yaml`, a 1Gi claim for `storageClassName: fast-ssd`, a class minikube doesn't have.
```bash
kubectl get storageclass                         # standard (default)   k8s.io/minikube-hostpath   Delete   Immediate (= bind at once)
kubectl apply -f notes-pvc.yaml
kubectl get pvc notes-pvc                        # STATUS Bound, VOLUME pvc-33b7… (the auto-made PV), 100Mi, RWO
kubectl apply -f ~/DevOPS/sessions/labs/09/notes-pod.yaml
kubectl exec notes-pod -- cat /data/notes.txt    # written at Thu Oct 15 04:20:55 UTC 2026
kubectl delete pod notes-pod
kubectl get pvc notes-pvc                        # still Bound: the claim doesn't care that the Pod is gone
kubectl apply -f ~/DevOPS/sessions/labs/09/notes-pod.yaml   # a new Pod (give it a few seconds)
kubectl exec notes-pod -- cat /data/notes.txt    # TWO lines now: the data outlived the Pod
kubectl get pv,pvc                               # a comma lists two kinds; your PV: Delete, Bound, default/notes-pvc
kubectl apply -f ~/DevOPS/sessions/labs/09/bad-pvc.yaml
kubectl get pvc bad-pvc                          # STATUS Pending
kubectl describe pvc bad-pvc | tail -3           # ProvisioningFailed … storageclass "fast-ssd" not found
kubectl delete pod notes-pod                     # clean up: Pods first, then claims
kubectl delete pvc notes-pvc bad-pvc
kubectl get pv                                   # your pvc-33b7… PV is gone too: reclaim policy Delete
```

### ✅ Check 63
**63.1** A cluster uses **static** provisioning only. Available PVs: `pv-a` (10Gi, RWO) and `pv-b` (3Gi, RWX), both in the same class. A new PVC asks for **5Gi with RWX**. What happens?
- A) It binds to pv-a, since 10Gi covers 5Gi
- B) It binds to pv-b and expands it to 5Gi
- C) It stays Pending; no PV fits both needs
- D) It splits the request across pv-a and pv-b

**63.2** A PV has `persistentVolumeReclaimPolicy: Retain`. Its PVC is deleted. What state is the PV in now?
- A) Deleted, along with all the data on it
- B) Available again for the next new claim
- C) Released, and the data is still there
- D) Bound, because the Pod still remembers it

**63.3** (one line) Explain PV vs PVC in contrast form, using the hotel analogy.

---
## Concept 64 · Probes: liveness, readiness & startup

**Words you'll learn:** health check, probe, liveness/readiness/startup probe, deadlock, `httpGet`, `tcpSocket`, `exec`, `grpc`, `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `failureThreshold`, `successThreshold`, Ready, endpoints, `Unhealthy` event, restart loop, grace period.

### The problem
An app stuck in a **deadlock** (two parts of the program each waiting for the other, forever) still shows `Running`, because by default Kubernetes only checks that the main process (PID 1) is alive. Every request hangs. Another app needs 20 s to load its data, yet gets users the moment its container starts. **A running process is not the same as a healthy, ready app.**

### The idea: probes
> A **probe** is a **health check** (a small "is this app OK?" test) that the **kubelet runs on a container again and again**. When it fails, the kubelet **restarts** the container or **stops sending it traffic**, depending on the probe type.

**Analogy: a restaurant manager.** **Liveness** asks "Are you awake?": a frozen waiter is replaced. **Readiness** asks "Ready to take orders?": no new customers are seated in that section, but the waiter keeps the job. **Startup** is a new chef's day-one grace period: nobody asks "are you awake?" until they're set up.

| | **Liveness** | **Readiness** | **Startup** |
|---|---|---|---|
| Question | Is it **alive** (not stuck)? | Can it **serve traffic** now? | Has it **finished starting**? |
| On failure | **Killed and restarted**; RESTARTS climbs | **Not Ready** (`READY 0/1`, still `Running`), **removed from the Service's endpoints**; **no restart** | Killed and restarted after `failureThreshold` failures |
| Runs | All the time, after startup succeeds | All the time, after startup succeeds | Only at the start, until it succeeds once |
| If not defined | Assumed alive | Ready as soon as the container starts | No slow-start protection |

While a startup probe hasn't succeeded yet, **liveness and readiness are paused**. Remember: a Service's **endpoints** are the Pod IPs it sends traffic to, and **only Ready Pods are on that list**.

### How a probe checks, and how often

| Mechanism | The kubelet… | Success means |
|---|---|---|
| **`httpGet`** | sends an HTTP GET to a path and port | a status code from **200 to 399** (3xx redirects pass; 404 and 500 fail) |
| **`tcpSocket`** | tries to open a TCP connection to a port (like `nc -z`) | the connection opens |
| **`exec`** | runs a command inside the container | the command's **exit code is 0** |

A fourth kind, `grpc`, is awareness only. ⚠️ An `exec` command must **exist in the image**: `curl` in an image without curl exits **127** ("not found") on every check, so the container restarts forever although the app is fine.

- **`initialDelaySeconds`** (default 0) = wait before the **first** check. **`periodSeconds`** (10) = check every N s. **`timeoutSeconds`** (1) = the longest one check may take.
- **`failureThreshold`** (3) = failures **in a row** before acting. **`successThreshold`** (1) = successes needed to count as healthy again; it **must be 1 for liveness and startup**.
- Time to notice a failure ≈ `failureThreshold × periodSeconds` (about 30 s by default). The same product is a startup probe's maximum start time: 30 × 10 s = **300 s**.

**Classic mistakes:**
- An aggressive liveness probe on a slow-starting app kills it before it's up, again and again: a **restart loop** ending in `CrashLoopBackOff`. Fix: add a **startup probe**.
- No readiness probe: traffic arrives too early, so users see errors on every rollout.
- A liveness probe that checks the database: one database blip restarts **every** Pod at once. Put dependency checks in **readiness**; and remember readiness never restarts a stuck app (that's **liveness**).
- Relying on Docker's **`HEALTHCHECK`** (the Dockerfile line that makes Docker test a container's health): **Kubernetes ignores it**. Probes go in the Pod spec.

### 🛠 Do (~15 min): liveness restarts a container; readiness holds back traffic
**Part A: liveness.** `code liveness-demo.yaml`, paste, save. The app deletes its "health file" after 30 s to fake a freeze:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600"]
      livenessProbe:
        exec:
          command: ["cat", "/tmp/healthy"]   # exits 0 while the file exists
        initialDelaySeconds: 5               # first check 5 s after the start
        periodSeconds: 5                     # then every 5 s
        failureThreshold: 3                  # restart after 3 failures in a row
```
```bash
kubectl get pods -w                       # in a SECOND tab (Cmd + T): -w = watch for changes; Ctrl + C stops
kubectl apply -f liveness-demo.yaml       # back in the first tab
kubectl describe pod liveness-demo        # after ~80 s: read the container section and the Events
kubectl delete pod liveness-demo --now    # clean up (--now skips most of the 30 s wait)
```
- The watch tab shows `liveness-demo 1/1 Running 1 (1s ago) 78s`: RESTARTS 0 → 1, again every ~75 s, later `CrashLoopBackOff`. `describe` shows `Exit Code: 137` and the event `Liveness probe failed: cat: can't open '/tmp/healthy'`.
- **Why ~75 s?** After three failed probes (35–45 s) the kubelet sends **SIGTERM**, the polite "please stop" signal. But `sh` running as PID 1 ignores it (Linux gives PID 1 no default SIGTERM handling), so Kubernetes waits out the **grace period** (the time between SIGTERM and the forced SIGKILL, **30 s by default**), and SIGKILL comes at ~75 s. **137 = 128 + 9**, and 9 is SIGKILL's number.

**Part B: readiness.** `~/DevOPS/sessions/labs/09/ready-demo.yaml` holds an nginx Pod (label `app: ready-demo`) with a readinessProbe doing `httpGet` on `/ready`, port 80, every 5 s, plus a Service `ready-demo` selecting it. nginx has no `/ready` page, so it answers 404.
```bash
kubectl apply -f ~/DevOPS/sessions/labs/09/ready-demo.yaml
kubectl get pod ready-demo                          # ready-demo   0/1   Running   0  (not ready, no restarts)
kubectl describe svc ready-demo | grep Endpoints    # Endpoints:   (empty: no Pod gets traffic)
kubectl exec ready-demo -- sh -c 'echo ok > /usr/share/nginx/html/ready'   # create the page the probe asks for
kubectl get pod ready-demo                          # 1/1 within ~5 s
kubectl describe svc ready-demo | grep Endpoints    # 10.244.0.x:80: the Pod joined, without a restart
kubectl delete -f ~/DevOPS/sessions/labs/09/ready-demo.yaml   # deletes everything the file created
```

### ✅ Check 64
**64.1** `kubectl get pods` has shown this for 10 minutes: `api-6f9c-x2x   0/1   Running   0   10m`. What's the most likely explanation?
- A) The readiness probe is failing on every check
- B) The liveness probe is failing on every check
- C) The image is still being pulled from the registry
- D) The Service selector doesn't match the Pod

**64.2** A container has a startupProbe with `periodSeconds: 10` and `failureThreshold: 30`, plus a livenessProbe. What does this give the app?
- A) Up to 30 s to start; liveness runs alongside
- B) Up to 300 s to start; liveness waits for it
- C) Up to 300 s to start; readiness never runs
- D) Exactly 10 s to start, then 30 restarts

**64.3** A readiness probe does `httpGet` on `/`, and the app answers `301 Moved Permanently`. What's the probe result?
- A) Failure, because only 200 counts as success
- B) Failure, because redirects mean the page moved
- C) Success, but only if the redirect is a 302
- D) Success, because any 2xx or 3xx code passes

---
## Concept 65 · Autoscaling: HPA, Metrics Server, and resource requests & limits

**Words you'll learn:** horizontal/vertical scaling, autoscaling, HPA (Horizontal Pod Autoscaler), metric, Metrics Server, `kubectl top`, requests, limits, millicore (`m`), throttling, OOMKilled, utilization, `ceil`, minReplicas/maxReplicas, stabilization window, flapping, VPA (Vertical Pod Autoscaler), Cluster Autoscaler.

### The problem
A midnight sale brings 10× the traffic. With `kubectl scale` (setting the replica count by hand), someone must be awake to add Pods, and remember to remove them later. We want **autoscaling**: scaling that happens by itself. **Horizontal** scaling ("scale out") adds **more Pods**, like opening more counters; that's the **HPA**'s job. **Vertical** scaling ("scale up") makes **each Pod bigger**, like giving one cook a bigger stove; that's the **VPA** (the Pods are usually recreated).

### First: resource requests and limits
Each container can say how much CPU and memory it needs, under `resources:`:
```yaml
      resources:
        requests:          # RESERVED for this container
          cpu: 200m        # 200 millicores = a fifth of a core
          memory: 128Mi
        limits:            # the MAXIMUM it may use
          cpu: 500m        # half a core
          memory: 256Mi
```
CPU is counted in cores, and **`m` = millicores** (thousandths of a core). Memory uses `Mi`/`Gi`; ⚠️ `128m` for memory would mean 0.128 **bytes**!

| | **Requests** | **Limits** |
|---|---|---|
| Meaning | The amount **reserved** (guaranteed) | The **cap** |
| Used by | The **scheduler** (it picks a node with enough room) and the **HPA** ("CPU 50%" = 50% **of the request**) | Neither |
| Going over it | Allowed, if the node has spare room | **CPU:** **throttled** (slowed down, not killed). **Memory:** **`OOMKilled`** |

**Why the difference?** CPU time can be handed out more slowly (the app just runs slower), but memory can't: if there's no RAM left, something must go. **OOM** = **Out Of Memory**; the kernel's **OOM killer** ends the process with SIGKILL, so you see **exit code 137**. **Analogy: a restaurant table booking.** The request is the table held for you; the limit is "max 6 chairs". Extra CPU guests wait their turn; extra memory guests are thrown out.

### Metrics Server and the HPA
> **Metrics Server** is an **add-on** (an optional extra component) that regularly **collects current CPU and memory use** from every kubelet. It keeps **only the latest numbers** (no history: that's Prometheus) and feeds **`kubectl top`** and the HPA. Your minikube already has it enabled.

> An **HPA (Horizontal Pod Autoscaler)** is an object watched by a **controller** (a control loop). **Every 15 s** it compares the Pods' **current metric** with a **target** and changes the Deployment's **`replicas`**; the Deployment does the rest.

```
desiredReplicas = ceil( currentReplicas × currentMetricValue ÷ targetMetricValue )
```
- **`ceil`** = round **up** to a whole number (3.2 → 4). **Utilization** = usage ÷ **request**: 100m used of a 200m request = 50%.
- The result stays between **`minReplicas`** and **`maxReplicas`**. If current ÷ target is within **10%** of 1 (0.9 to 1.1), nothing changes.

| Pods | Average CPU (target 50%) | Calculation | Result |
|---|---|---|---|
| 2 | 90% | ceil(2 × 90 ÷ 50) = ceil(3.6) | **4** |
| 4 | 20% | ceil(4 × 20 ÷ 50) = ceil(1.6) | **2** (after the scale-down wait) |
| 3 | 52% | 52 ÷ 50 = 1.04: inside the tolerance | **3** (no change) |
| 1 | 370% | ceil(7.4) = 8, but `maxReplicas: 5` | **5** |

- **Scale up fast, down slowly.** Scale-down waits a **stabilization window** of **5 minutes**. Traffic is bumpy, and without the wait the HPA would remove Pods in a lull and add them back moments later; that back-and-forth is **flapping**. ⚠️ A manual `kubectl scale` is **overridden** by the HPA at its next check.
- **As YAML:** `kind: HorizontalPodAutoscaler` (`apiVersion: autoscaling/v2`), with `scaleTargetRef` (what to scale), `minReplicas`, `maxReplicas`, and a `metrics` entry for `cpu` with `averageUtilization: 50`. Memory works the same way (`--memory=70%` = 70% of the memory **request**).
- **`TARGETS cpu: <unknown>/50%`** = the HPA can't calculate. Usually the Pods have **no CPU requests**, so "% of the request" means nothing (`missing request for cpu`). Otherwise: Metrics Server is broken, or the HPA is under a minute old.
- **Other autoscalers (awareness):** the **VPA** changes each Pod's requests/limits (an add-on). The **Cluster Autoscaler** adds or removes **nodes** when Pods are stuck **Pending** for lack of room (cloud only).

### 🛠 Do (~20 min): watch the HPA scale up under load, then back down
`~/DevOPS/sessions/labs/09/hpa-web.yaml` holds a Deployment `hpa-web` (1 nginx Pod, label `app: hpa-web`) and a Service `hpa-web` on port 80. Its CPU request is deliberately tiny (`requests: cpu: 20m`, `limits: cpu: 100m`), so a little load is a big percentage.
```bash
minikube addons list | grep metrics-server        # metrics-server … enabled ✅ (if not: minikube addons enable metrics-server)
kubectl top nodes                                 # minikube   288m   4%   3261Mi   32%
kubectl apply -f ~/DevOPS/sessions/labs/09/hpa-web.yaml
kubectl autoscale deployment hpa-web --cpu=50% --min=1 --max=5   # target 50% of the request, 1 to 5 Pods
kubectl get hpa hpa-web -w                        # SECOND tab: cpu: <unknown>/50% for ~30–60 s, then cpu: 20%/50%
kubectl run load-gen --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://hpa-web > /dev/null; done"
kubectl top pods                                  # each hpa-web Pod's CPU, and load-gen working hard
kubectl describe hpa hpa-web                      # Metrics 120% / 50%; ScalingLimited TooManyReplicas; SuccessfulRescale events
kubectl delete pod load-gen --now                 # stop the load
kubectl delete hpa hpa-web                        # clean up
kubectl delete -f ~/DevOPS/sessions/labs/09/hpa-web.yaml
```
- ⚠️ Exams and older tutorials write **`--cpu-percent=50`**; kubectl 1.37 still accepts it but warns it's deprecated. Know both spellings.
- `load-gen` is a plain Pod (`--restart=Never`) that fetches `http://hpa-web` (the Service's DNS name) in an endless loop and throws the page away (`> /dev/null`).
- REPLICAS climbs 1 → 4 → **5** (`cpu: 370%/50%` wants 8, but the max is 5). After you stop the load, CPU drops to `0%/50%`, yet REPLICAS **stays at 5 for ~5 minutes** (the stabilization window), then: `New size: 1; reason: All metrics below target`.

### ✅ Check 65
**65.1** An HPA targets **60%** CPU (min 1, max 10). There are **3** Pods, and their average CPU use is **80%** of their request. How many replicas will the HPA ask for?
- A) 3
- B) 4
- C) 5
- D) 6

**65.2** An HPA has shown `cpu: <unknown>/50%` for 10 minutes. `kubectl top pods` works fine. What's the most likely cause?
- A) The pods have no CPU `limits` set
- B) maxReplicas is set lower than minReplicas
- C) The pods have no CPU `requests` set
- D) The Deployment has only a single replica

**65.3** (one line) Explain requests vs limits in contrast form, including what happens when a container goes over its CPU limit and over its memory limit.

---
## ➕ Small but important (storage, probes, scaling): read before the quiz

| Item | What it is |
|---|---|
| **QoS (Quality of Service) classes** | Kubernetes sorts every Pod by its resources. **Guaranteed** = requests equal limits for CPU and memory; **Burstable** = some set; **BestEffort** = none. A node that runs out of memory **evicts** (pushes off) BestEffort Pods first, Guaranteed last |
| **LimitRange** (awareness) | A per-namespace rule giving **default** requests/limits to containers that set none. It fixes HPA's `<unknown>` |
| **ResourceQuota** (awareness) | A per-namespace **total** cap (CPU, number of Pods, PVCs). Going over → rejected with `exceeded quota` |
| **`volumeClaimTemplates`** | In a **StatefulSet** (Pods with stable names like `db-0`, `db-1`): makes **one PVC per Pod** (`data-db-0`), kept even on scale-down |
| **`terminationGracePeriodSeconds`** | The wait between SIGTERM and SIGKILL; **default 30 s** (`docker stop` waits 10 s). A **`preStop` hook** is a command run just before SIGTERM |
| **PodDisruptionBudget (PDB)** (awareness) | "Keep at least N Pods up during **planned** disruptions", like emptying a node for an upgrade. No help against crashes |
| **`subPath`** | Mount **one file** from a volume, not the whole folder. ⚠️ Such files don't get automatic ConfigMap updates |
| **PVC expansion** | Grow a PVC only if its class has `allowVolumeExpansion: true`; **never shrink**. minikube's `standard`: `false` |

---
## 📝 Session quiz (4 MCQs + 2 one-liners), from memory

**Q1.** A Java app needs about **60 s** to start. Its Deployment has a livenessProbe (`httpGet /healthz`, `periodSeconds: 10`, `failureThreshold: 3`, no initial delay). The Pods restart roughly every 35 s and never become Ready. What's the best fix?
- A) Add a readinessProbe on the same `/healthz` path
- B) Add a startupProbe allowing about 60 s or more
- C) Set the liveness successThreshold to 3 instead
- D) Raise the liveness timeoutSeconds to 60 seconds

**Q2.** On a **3-node** cloud cluster, a Deployment with `replicas: 3` mounts **one** PVC whose access mode is `ReadWriteOnce`. The three Pods land on three different nodes. One runs; the other two are stuck in `ContainerCreating` with a `Multi-Attach error` event. Why?
- A) The PVC needs 3× the storage for 3 replicas
- B) The StorageClass reclaim policy is Delete
- C) RWO allows only one Pod in the whole cluster
- D) RWO lets just one node mount it read-write

**Q3.** An HPA (min 2, max 10) targets **50%** CPU. The Deployment has **4** Pods; each **requests `200m`** and is **limited to `500m`**. Average use is **`250m`** per Pod. What replica count will the HPA set?
- A) 4, since 250m is 50% of the 500m limit
- B) 5, since 250m is 1.25× the 200m request
- C) 10, since 4 × (125% ÷ 50%) equals 10
- D) 2, since usage is under the 500m limit

**Q4.** During every rolling update of a web Deployment, users get errors for about 20 s per new Pod. The app needs ~20 s to load its data after the container starts. The Pods have a livenessProbe only. Which change fixes the errors?
- A) Add a readinessProbe on a /ready check
- B) Set `maxUnavailable: 0` in the strategy
- C) Raise the liveness initialDelaySeconds
- D) Set terminationGracePeriodSeconds to 60

**L1.** Explain liveness vs readiness probes in contrast form, saying what Kubernetes does when each one fails.

**L2.** Write the full forms of **PVC**, **HPA** and **RWX**, and say what happens to a PV with reclaim policy `Retain` when its PVC is deleted.

### ✍️ Write the command
**C1.** With **one** command, list all PersistentVolumes and PersistentVolumeClaims.

**C2.** Print the file `/data/log.txt` from the container `reader` inside the Pod `shared-pod`.

**C3.** Make the Deployment `web` autoscale between **2** and **8** Pods, aiming for **60%** CPU (of the request). Use the new flag spelling.

**C4.** Show only the **last 3 lines** of the details of the PVC `bad-pvc` (where its Events are).
