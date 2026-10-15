# Session 09 · Kubernetes V: storage, probes & autoscaling (Oct 15, Session 1)

**Time:** ~100–110 min · **Concepts:** 62 → 65 · **Answers:** `MY_ANSWERS.md` or say "K8s V questions" · **Answer key:** `09_storage-probes-hpa_ANSWERS.md`

**Before you start:**
```bash
minikube status        # host, kubelet, apiserver: Running (if not: minikube start)
kubectl get nodes      # minikube   Ready
mkdir -p ~/devops-practice/k8s-v && cd ~/devops-practice/k8s-v   # today's folder; all YAML goes here
```
> ⚠️ **Never delete the `minikube` or `localstack` containers.** Leave namespaces you didn't create (`argocd`, `ingress-nginx`, `final-app`) and their PersistentVolume alone. Today everything runs in `default`.

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
## Concept 62 · Volumes in Pods: giving containers somewhere to keep files
**Words you'll learn:** ephemeral, volume, mount, `volumes`, `volumeMounts`, `mountPath`, `emptyDir`, sidecar, `hostPath`, `configMap`/`secret` volume, `readOnly`, tmpfs.
- **Problem:** files in a container's writable layer are **ephemeral** (short-lived, Session 04), and the **kubelet** (the agent on each node) restarts a crashed container with a fresh layer. Also, Asha's `shipper` container can't even see her `app` container's log files.
- **Idea:** a **volume** is **a folder that belongs to the Pod**; each container **mounts** it (plugs it in) at a path of its choice. **Analogy: a shared drawer in an office room** (Pod = room, containers = colleagues): a replacement colleague still finds the papers.
- **Two halves, joined by a name:** `volumes` (Pod level) = **what** storage; `volumeMounts` (in each container) = **where** (`mountPath`). ⚠️ **Exam trap:** the names must **match exactly**, or apply fails with `Not found: "config"`.

| Type | What it is | Survives a container restart? | Survives Pod deletion? |
|---|---|---|---|
| **`emptyDir`** | An empty folder made when the Pod starts | ✅ | ❌ deleted with the Pod |
| **`hostPath`** | A folder on the **node's** own disk | ✅ | ✅ but only on **that node** |
| **`configMap`** / **`secret`** | Each key becomes a **file**; the value is its content | ✅ | (separate objects) |
| **`persistentVolumeClaim`** | Real, lasting storage (Concept 63) | ✅ | ✅ until you delete the claim |

- **emptyDir** is scratch space, and the usual way to share files with a **sidecar** (a helper container beside the app, like `shipper`). With `medium: Memory` it lives in RAM (**tmpfs**), counted against the memory limit.
- **hostPath is risky:** a Pod recreated on another node sees different data, and `hostPath: /` exposes the node's files. It only *seems* fine on one-node minikube.
- **configMap volume:** keys `color=blue`, `mode=dev` mounted at `/etc/app` → files `color` (containing `blue`) and `mode`. Secret files live in memory, already decoded. Mounted files **auto-update** (within a minute); **env vars don't**.
### 🛠 Do (~10 min): two containers sharing an emptyDir, plus a ConfigMap as files
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
          mountPath: /shared
    - name: reader                # just sleeps, so we can look inside
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data        # same volume, different path
        - name: settings
          mountPath: /etc/app
          readOnly: true          # this container can't change the files (like Docker's :ro)
  volumes:                        # ① WHAT storage the Pod has: a name + a type
    - name: shared-data
      emptyDir: {}                # {} = an empty map = default settings
    - name: settings
      configMap:
        name: shared-settings
```
```bash
kubectl create configmap shared-settings --from-literal=color=blue --from-literal=mode=dev   # a ConfigMap with 2 keys
kubectl apply -f shared-pod.yaml
kubectl get pod shared-pod                                # shared-pod   2/2   Running  (2 of 2 containers ready)
kubectl exec shared-pod -c reader -- cat /data/log.txt    # dates the writer wrote; -c = which container, -- = the command follows
kubectl exec shared-pod -c reader -- ls /etc/app          # color  mode  (one file per key)
kubectl exec shared-pod -c reader -- cat /etc/app/color   # blue
kubectl delete pod shared-pod                             # delete the Pod…
kubectl apply -f shared-pod.yaml                          # …and recreate it
kubectl exec shared-pod -c reader -- cat /data/log.txt    # only 1–2 fresh dates: the emptyDir died with the old Pod
kubectl delete pod shared-pod                             # clean up
kubectl delete configmap shared-settings
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
## Concept 63 · PersistentVolume, PersistentVolumeClaim & StorageClass: storage that outlives Pods
**Words you'll learn:** PersistentVolume (PV), PersistentVolumeClaim (PVC), StorageClass, provisioner, static/dynamic provisioning, binding, `Mi`/`Gi`, access modes (RWO, ROX, RWX, RWOP), reclaim policy (Retain, Delete, Recycle), statuses (Available, Bound, Released, Pending), cluster-scoped, default StorageClass, `volumeBindingMode`.
- **Problem:** Ravi's team runs a PostgreSQL database. Its storage must outlive the Pod (so not emptyDir) and follow the Pod to any node (so not hostPath).
- **PersistentVolume (PV):** a **real piece of storage** with a size and rules; **cluster-scoped** (not in a namespace). **PersistentVolumeClaim (PVC):** a developer's **request** ("1 GiB, read-write"); it's namespaced, and the Pod names the **claim**, never the PV.
- **StorageClass:** a storage **category** plus its **provisioner**, which **creates PVs automatically**. **Static** provisioning = an admin makes PVs by hand; **dynamic** = the provisioner makes one per PVC (almost everywhere, including minikube).
- **Analogy: booking a hotel room.** PVC = your booking; PV = the actual room 204; StorageClass = the room category plus housekeeping who prepare rooms on demand; Pod = the guest.
- **Binding** = reception giving room 204 to your booking: always **1 : 1**. A PV must have the **same class**, **include the access mode** asked for, and be **at least** the size asked for (ask 150Mi, get a 200Mi PV: you get all 200Mi).
- **Statuses:** PV Available → Bound → Released (→ Failed); PVC Pending → Bound. **Units:** `Mi` = mebibyte (1024 × 1024 bytes), `Gi` = gibibyte (1024 Mi). `M` and `G` are powers of 1000.

| Access mode | Full form | Meaning |
|---|---|---|
| **RWO** | ReadWriteOnce | Read-write by **one NODE** at a time |
| **ROX** | ReadOnlyMany | Read-only by **many nodes** |
| **RWX** | ReadWriteMany | Read-write by **many nodes** (needs shared storage like NFS, the Network File System) |
| **RWOP** | ReadWriteOncePod | Read-write by **one single Pod** in the whole cluster |

⚠️ **Exam trap: RWO means one *node*, not one *Pod*.** Pods on the same node can share it. Most cloud disks (like Amazon's EBS) support **only RWO**.

**Reclaim policy** (what happens to the PV when its PVC is deleted):
- **Retain:** the PV **stays, with its data**, as **`Released`**, and isn't reused until an admin cleans up. Use it for precious data.
- **Delete:** the PV **and the real storage** are deleted: data gone. Usual for dynamic PVs. **Recycle:** ⚠️ deprecated wipe-and-reuse.

| Why a PVC is stuck `Pending` | Clue in `kubectl describe pvc` Events |
|---|---|
| The StorageClass doesn't exist (a typo, or a cloud class like `gp2` on minikube) | `storageclass… "fast-ssd" not found` |
| Static setup: no PV with enough size, the right mode and class | `no persistent volumes available for this claim…` |
| The class uses `volumeBindingMode: WaitForFirstConsumer` | `waiting for first consumer…`: **normal**, it binds once a Pod uses it |

### 🛠 Do (~12 min): data that survives the Pod
`code notes-pvc.yaml`. It names no `storageClassName`, so the default class is used (dynamic provisioning):
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: notes-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```
- Ready-made in `~/DevOPS/sessions/labs/09/` (open with `code` to read): `notes-pod.yaml`, a busybox Pod that appends one timestamped line to `/data/notes.txt`, then sleeps. Its `volumes:` entry uses `persistentVolumeClaim:` → `claimName: notes-pvc`.
- `bad-pvc.yaml`: a 1Gi claim with `storageClassName: fast-ssd`, a class that doesn't exist on minikube.
```bash
kubectl get storageclass                         # standard (default)   k8s.io/minikube-hostpath   Delete   Immediate
kubectl apply -f notes-pvc.yaml
kubectl get pvc notes-pvc                        # STATUS Bound, VOLUME pvc-33b7… (the auto-made PV), 100Mi, RWO
kubectl apply -f ~/DevOPS/sessions/labs/09/notes-pod.yaml
kubectl exec notes-pod -- cat /data/notes.txt    # written at Thu Oct 15 04:20:55 UTC 2026
kubectl delete pod notes-pod
kubectl get pvc notes-pvc                        # still Bound: the claim doesn't care that the Pod is gone
kubectl apply -f ~/DevOPS/sessions/labs/09/notes-pod.yaml   # a new Pod (give it a few seconds)
kubectl exec notes-pod -- cat /data/notes.txt    # TWO lines now: the data outlived the Pod
kubectl get pv,pvc                               # a comma lists two kinds; your PV: Delete, Bound, default/notes-pvc
kubectl describe pvc notes-pvc                   # Used By: notes-pod; event ProvisioningSucceeded
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
## Concept 64 · Probes: liveness, readiness & startup
**Words you'll learn:** health check, probe, liveness/readiness/startup probe, deadlock, `httpGet`, `tcpSocket`, `exec`, `grpc`, `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `failureThreshold`, `successThreshold`, Ready, endpoints, `Unhealthy` event, restart loop, grace period.
- **Problem:** a **deadlocked** app (two parts waiting for each other forever) still shows `Running`, because Kubernetes only checks that PID 1 is alive. A slow starter gets users too early. **A running process is not a healthy, ready app.**
- **Idea:** a **health check** is a small "is this app OK?" test. A **probe** is a health check the **kubelet runs on a container again and again**. On failure it **restarts** the container or **stops sending it traffic**.
- **Analogy: a restaurant manager.** Liveness = "Are you awake?" (a frozen waiter is replaced). Readiness = "Ready to take orders?" (no new customers are seated in their section, but they keep the job). Startup = a day-one grace period for a new chef.

| | **Liveness** | **Readiness** | **Startup** |
|---|---|---|---|
| Question | Alive (not stuck)? | Can it serve traffic now? | Finished starting? |
| On failure | **Killed and restarted**; RESTARTS climbs | **Not Ready** (`READY 0/1`, still `Running`), **removed from the Service's endpoints**; **no restart** | Killed and restarted after `failureThreshold` failures |
| Runs | All the time, after startup succeeds | All the time, after startup succeeds | Only at the start, until it succeeds once |
| If not defined | Assumed alive | Ready as soon as the container starts | No slow-start protection |

- While a startup probe hasn't succeeded yet, **liveness and readiness are paused**. Recap (Session 07): a Service's **endpoints** are the Pod IPs it sends traffic to, and **only Ready Pods are on that list**.

| Mechanism | The kubelet… | Success means |
|---|---|---|
| **`httpGet`** | sends an HTTP GET to a path and port | a status code from **200 to 399** (3xx redirects pass; 404 and 500 fail) |
| **`tcpSocket`** | opens a TCP connection to a port (like `nc -z`) | the connection opens |
| **`exec`** | runs a command inside the container | **exit code 0** |

(A fourth kind, `grpc`, calls a gRPC health service: awareness.) ⚠️ An `exec` command must **exist in the image**. `curl` in an image without curl exits **127** ("not found") every time, so the container restarts forever although the app is fine.

- **Timing fields (default):** `initialDelaySeconds` (0) = wait before the **first** check; `periodSeconds` (10) = check every N s; `timeoutSeconds` (1) = max time for one check.
- `failureThreshold` (3) = failures **in a row** before acting; `successThreshold` (1) = successes to count healthy again, and it **must be 1 for liveness and startup**.
- Time to notice a failure ≈ `failureThreshold × periodSeconds` (about 30 s). A startup probe with `failureThreshold: 30`, `periodSeconds: 10` allows **300 s** to start.
- **Classic mistakes:** an aggressive liveness probe on a slow app causes a **restart loop** ending in `CrashLoopBackOff` (fix: add a startup probe). No readiness probe means errors on every rollout.
- A liveness probe that checks the database makes one database blip restart every Pod at once: put dependency checks in **readiness**. And readiness never restarts a stuck app: use **liveness** for that.
- **vs Docker's `HEALTHCHECK`** (Session 03): Docker only labels a container `(unhealthy)`. **Kubernetes ignores the Dockerfile HEALTHCHECK**: probes go in the Pod spec.
### 🛠 Do (~15 min): liveness restarts a container; readiness holds back traffic
**Part A: liveness.** `code liveness-demo.yaml`. The app deletes its "health file" after 30 s to fake a freeze:
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
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
```
```bash
kubectl get pods -w                       # in a SECOND tab (Cmd + T): -w = watch for changes; Ctrl + C stops
kubectl apply -f liveness-demo.yaml       # back in the first tab
kubectl describe pod liveness-demo        # after ~80 s: read the container section and the Events
kubectl delete pod liveness-demo --now    # clean up (--now skips most of the 30 s wait)
```
- The watch tab shows `liveness-demo 1/1 Running 1 (1s ago) 78s`: RESTARTS 0 → 1, again every ~75 s, later `CrashLoopBackOff`. `describe` shows `Exit Code: 137` and `Liveness probe failed: cat: can't open '/tmp/healthy'`.
- **Why ~75 s and 137?** Three failed probes (35–45 s) → SIGTERM, which `sh` as PID 1 ignores (Session 03). After the **grace period** (wait between SIGTERM and SIGKILL, **30 s by default**) comes SIGKILL at ~75 s. **137 = 128 + 9** (SIGKILL).

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
## Concept 65 · Autoscaling: HPA, Metrics Server, and resource requests & limits
**Words you'll learn:** horizontal/vertical scaling, autoscaling, HPA (Horizontal Pod Autoscaler), metric, Metrics Server, `kubectl top`, requests, limits, millicore (`m`), throttling, OOMKilled, utilization, `ceil`, minReplicas/maxReplicas, stabilization window, flapping, VPA (Vertical Pod Autoscaler), Cluster Autoscaler.
- **Problem:** a midnight sale brings 10× traffic, and someone must run `kubectl scale` (Session 06), then scale back later. We want **autoscaling**. **Horizontal** ("scale out") = more Pods (**HPA**); **vertical** ("scale up") = bigger Pods, usually recreated (**VPA**).
- **Requests and limits** go under each container's `resources:`:
```yaml
      resources:
        requests:          # RESERVED for this container
          cpu: 200m
          memory: 128Mi
        limits:            # the MAXIMUM it may use
          cpu: 500m
          memory: 256Mi
```
- CPU is in cores: **`m` = millicores** (thousandths), so `500m` = half a core. Memory uses `Mi`/`Gi`; ⚠️ `128m` for memory means 0.128 **bytes**!
- **Analogy: a restaurant table booking.** The request is the table held for you; the limit is "max 6 chairs". Extra CPU guests wait their turn; extra memory guests are thrown out.

| | **Requests** | **Limits** |
|---|---|---|
| Meaning | The amount **reserved** (guaranteed) | The **cap** |
| Used by | The **scheduler** (it picks a node with room) and the **HPA** ("CPU 50%" = 50% **of the request**) | Neither |
| Going over it | Allowed, if the node has spare room | **CPU:** **throttled** (slowed, not killed). **Memory:** **`OOMKilled`** (**OOM** = Out Of Memory: the kernel's OOM killer sends SIGKILL, exit 137) |

- **Metrics Server:** an **add-on** (optional extra component) that **collects current CPU/memory use** from every kubelet, keeps **only the latest numbers**, and feeds **`kubectl top`** and the HPA. No history (that's Prometheus). Your minikube addon is **already enabled**.
- **HPA:** an object watched by a **controller** (a control loop). **Every 15 s** it compares the Pods' **current metric** with a **target** and edits the Deployment's **`replicas`**.
```
desiredReplicas = ceil( currentReplicas × currentMetricValue ÷ targetMetricValue )
```
- **`ceil`** = round **up** (3.2 → 4). **Utilization** = usage ÷ **request** (100m of a 200m request = 50%). The result stays between `minReplicas` and `maxReplicas`; within a **10% tolerance** (ratio 0.9–1.1) nothing changes.

| Pods | Average CPU (target 50%) | Calculation | Result |
|---|---|---|---|
| 2 | 90% | ceil(2 × 90 ÷ 50) = ceil(3.6) | **4** |
| 4 | 20% | ceil(4 × 20 ÷ 50) = ceil(1.6) | **2** (after the scale-down wait) |
| 3 | 52% | 52 ÷ 50 = 1.04: inside the tolerance | **3** (no change) |
| 1 | 370% | ceil(7.4) = 8, but `maxReplicas: 5` | **5** |

- **Scale up fast, down slowly:** scale-down waits a **stabilization window** of **5 min (300 s)** to avoid **flapping** (adding and removing Pods back and forth). ⚠️ A manual `kubectl scale` is **overridden** by the HPA at its next check.
- **Declarative form:** `kind: HorizontalPodAutoscaler`, `apiVersion: autoscaling/v2`, with `scaleTargetRef` (what to scale), `minReplicas`, `maxReplicas`, and a `metrics` entry for `cpu` with `averageUtilization: 50`.
- **`TARGETS cpu: <unknown>/50%`** means the HPA can't calculate. Usually the Pods have **no CPU requests** (`missing request for cpu`). Other causes: Metrics Server broken, or the HPA was created in the last minute.
- **Other autoscalers:** **VPA** changes each Pod's requests/limits (an add-on). The **Cluster Autoscaler** adds or removes **nodes** when Pods are **Pending** for lack of room (cloud only).
### 🛠 Do (~20 min): watch the HPA scale up under load, then back down
`~/DevOPS/sessions/labs/09/hpa-web.yaml` holds a Deployment `hpa-web` (1 nginx Pod, label `app: hpa-web`) and a Service `hpa-web` on port 80. Its CPU request is deliberately tiny (`requests: cpu: 20m`, `limits: cpu: 100m`), so a little load is a big percentage.
```bash
minikube addons list | grep metrics-server        # metrics-server … enabled ✅ (if not: minikube addons enable metrics-server)
kubectl top nodes                                 # minikube   288m   4%   3261Mi   32%
kubectl top pods -n kube-system                   # the same per Pod; "Metrics API not available" = not ready yet
kubectl apply -f ~/DevOPS/sessions/labs/09/hpa-web.yaml
kubectl rollout status deployment hpa-web         # deployment "hpa-web" successfully rolled out
kubectl autoscale deployment hpa-web --cpu-percent=50 --min=1 --max=5   # target 50% of the request, 1 to 5 Pods
kubectl get hpa hpa-web -w                        # SECOND tab: cpu: <unknown>/50% for ~30–60 s, then cpu: 20%/50%
kubectl run load-gen --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://hpa-web > /dev/null; done"
kubectl top pods                                  # each hpa-web Pod's CPU, and load-gen working hard
kubectl describe hpa hpa-web                      # Metrics 120% / 50%; ScalingLimited TooManyReplicas; SuccessfulRescale events
kubectl delete pod load-gen --now                 # stop the load
kubectl create deployment noreq --image=nginx:1.27                # a Deployment with NO resources
kubectl autoscale deployment noreq --cpu=50% --min=1 --max=3      # the new spelling: no warning
kubectl get hpa noreq                                             # stays cpu: <unknown>/50%
kubectl describe hpa noreq | tail -3                              # missing request for cpu in container nginx
kubectl delete hpa hpa-web noreq                                  # clean up
kubectl delete deployment noreq
kubectl delete -f ~/DevOPS/sessions/labs/09/hpa-web.yaml
```
- ⚠️ **kubectl 1.37:** `--cpu-percent` works but warns it's deprecated; the new form is **`--cpu=50%`** (know both). `load-gen` (a plain Pod) loops `wget` against the Service's DNS name.
- REPLICAS climbs 1 → 4 → **5** (`cpu: 370%/50%` wants 8, but max is 5). After you stop the load, CPU drops to `0%/50%`, but REPLICAS **stays at 5 for ~5 minutes** (the stabilization window). Then: `New size: 1; reason: All metrics below target`.
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
## ➕ Small but important (storage, probes, scaling): read before the quiz
| Item | What it is |
|---|---|
| **QoS (Quality of Service) classes** | **Guaranteed** = requests equal limits for CPU and memory; **Burstable** = some set; **BestEffort** = none. A full node **evicts** (pushes off) BestEffort first, Guaranteed last |
| **LimitRange** (awareness) | Per-namespace **default** requests/limits for containers that set none. Fixes HPA `<unknown>` |
| **ResourceQuota** (awareness) | Per-namespace **total** cap (CPU, Pods, PVCs). Going over → `exceeded quota` |
| **`volumeClaimTemplates`** | In a **StatefulSet** (stable Pod names `db-0`, `db-1`): one PVC per Pod (`data-db-0`), kept on scale-down |
| **`terminationGracePeriodSeconds`** | Wait between SIGTERM and SIGKILL; **default 30 s** (`docker stop`: 10 s). A **`preStop` hook** runs just before SIGTERM |
| **PodDisruptionBudget (PDB)** (awareness) | "Keep ≥ N Pods up during **planned** disruptions" (a node drain). No help against crashes |
| **`subPath`** | Mount **one file** from a volume. ⚠️ No automatic ConfigMap updates |
| **PVC expansion** | Grow a PVC only if the class has `allowVolumeExpansion: true`; **never shrink**. minikube's `standard`: `false` |
| **Memory-based HPA** | `--memory=70%`: uses memory **requests** the same way |
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
