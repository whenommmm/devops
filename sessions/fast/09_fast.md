# Session 09 · Storage, Probes & Autoscaling · FAST (10-min read)
*How Kubernetes keeps files when containers die, checks that apps are truly healthy, and adds or removes Pods by itself.*

## 1 · Volumes: a folder that belongs to the Pod
A container's own files are ephemeral (short-lived): when it crashes, the kubelet (the agent on each node that runs containers) restarts it with a fresh, empty filesystem. A volume is a folder that belongs to the Pod rather than to one container, so every container in the Pod can mount (plug in) it, and its files survive container restarts.
- An `emptyDir` is an empty folder created when the Pod starts and deleted with the Pod; it is the usual way to share files with a sidecar (a helper container in the same Pod).
- A `hostPath` is a folder on the node's own disk, which is risky because a Pod recreated on another node sees that node's different data.
- A `configMap` or `secret` volume turns each key into a file whose content is the value, and these files update by themselves within about a minute, whereas environment variables are read only once, at container start.

```yaml
  containers:
    - name: app
      volumeMounts:
        - name: shared-data
          mountPath: /shared
  volumes:
    - name: shared-data
      emptyDir: {}
```
- `volumes` (Pod level) says **what** storage exists, and `volumeMounts` (per container) says **where** that container sees it; the two names must match exactly.

⚠️ **Exam trap:** An `emptyDir` survives a container restart but dies with the Pod, whereas a `hostPath` survives Pod deletion but only on that one node.

## 2 · PersistentVolume, PersistentVolumeClaim and StorageClass
A database needs storage that outlives the Pod and follows it to any node. A PersistentVolume (PV) is a real piece of storage (a disk or network folder), and a PersistentVolumeClaim (PVC) is a request for storage, such as "100Mi, read-write"; the Pod names the claim, never the PV. Like a hotel: the PVC is your booking, the PV is room 204, and binding is reception matching the two.
- A StorageClass is a storage category plus its provisioner, which creates a matching PV automatically when a claim arrives (dynamic provisioning); a claim naming no class gets the default StorageClass.
- In static provisioning there is no provisioner: an admin creates every PV by hand beforehand.
- Binding is always one PVC to one PV, and the PV must have the same class, the requested access mode and at least the requested size.
- The reclaim policy decides what happens to a PV when its PVC is deleted: `Delete` removes the PV and its storage, whereas `Retain` keeps the PV and its data in status `Released` until an admin cleans up.

| Access mode | Meaning |
|---|---|
| RWO (ReadWriteOnce) | Read-write by one **node** at a time |
| ROX (ReadOnlyMany) | Read-only by many nodes |
| RWX (ReadWriteMany) | Read-write by many nodes |
| RWOP (ReadWriteOncePod) | Read-write by one single Pod |

⚠️ **Exam trap:** RWO means one node, so two Pods on that node can share it, whereas RWOP means exactly one Pod. A `Released` PV is not `Available`, so it is never handed to a new claim automatically.

## 3 · Probes: liveness, readiness and startup
By default Kubernetes only checks that the main process runs, so a deadlocked app (stuck waiting forever) still shows `Running`, and a slow-loading app gets users too early. A probe is a health check that the kubelet repeats on a container, and the probe type decides what happens when it fails. Like a restaurant manager: a frozen waiter is replaced (liveness), but one still setting up just gets no customers yet (readiness).

| | Liveness | Readiness | Startup |
|---|---|---|---|
| Asks | Is it alive, not stuck? | Can it serve traffic now? | Has it finished starting? |
| On failure | Killed and restarted (RESTARTS climbs) | Shows `0/1 Running`, removed from the Service's endpoints, **no restart** | Killed and restarted |

- A Service's endpoints are the Pod IP (Internet Protocol) addresses it sends traffic to, and only Ready Pods are on that list.
- A startup probe runs only until it first succeeds, and until then liveness and readiness are paused, which protects slow-starting apps.
- An `httpGet` probe passes on status codes 200–399, a `tcpSocket` probe passes if a TCP (Transmission Control Protocol) connection to the port opens, and an `exec` probe passes if its command exits with 0.
- A probe acts after `failureThreshold` (default 3) failures in a row, checking every `periodSeconds` (default 10), so a startup probe with 30 × 10 s allows up to 300 s to start.

⚠️ **Exam trap:** A liveness probe that fires before a slow app is up causes a restart loop ending in `CrashLoopBackOff` (ever-longer waits between restarts), which a startup probe fixes and a readiness probe does not. A dependency check like "can I reach the database?" belongs in readiness, whereas in liveness one database blip restarts every Pod at once.

## 4 · Resource requests and limits
Each container can declare how much CPU (Central Processing Unit) and memory it needs. A request is the amount reserved for it, and a limit is the maximum it may use.
```yaml
resources:
  requests:
    cpu: 200m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```
- `200m` means 200 millicores, a fifth of one core, and `Mi`/`Gi` mean mebibytes/gibibytes (powers of 1024).
- The scheduler adds up the requests, not the actual use, to pick a node with enough room.
- A container over its CPU limit is throttled (slowed down, not killed), whereas one over its memory limit is killed as `OOMKilled` (OOM = Out Of Memory) with exit code 137.

⚠️ **Exam trap:** An HPA target of "50% CPU" means 50% of the request, never 50% of the limit.

## 5 · Metrics Server and the HPA (Horizontal Pod Autoscaler)
Scaling by hand with `kubectl scale` needs someone awake during a traffic spike, so the HPA adds or removes Pods by itself. Metrics Server is an add-on (an optional extra component) that collects current CPU and memory use from every kubelet, keeps only the latest numbers (no history), and feeds `kubectl top` and the HPA. Every 15 seconds the HPA compares the average with its target and sets the Deployment's `replicas`.
```
desiredReplicas = ceil( currentReplicas × currentUtilization ÷ targetUtilization )
```
- `ceil` rounds up to a whole number, and utilization is usage ÷ request, so 2 Pods averaging 90% with a 50% target give ceil(3.6) = 4 Pods.
- The result is always kept between `minReplicas` and `maxReplicas`.
- The HPA scales up fast but waits a 5-minute stabilization window before scaling down, to avoid flapping (removing Pods in a short lull, then re-adding them).
- `kubectl autoscale deployment web --cpu=50% --min=1 --max=5` creates an HPA (older spelling: `--cpu-percent=50`).
- `TARGETS cpu: <unknown>/50%` usually means the Pods have no CPU request to take a percentage of.

⚠️ **Exam trap:** The HPA changes how many Pods run (horizontal scaling), whereas the VPA (Vertical Pod Autoscaler) changes how big each Pod is (vertical scaling).

## 🎯 Exam checklist (everything that could be asked)
*MCQ = know the fact · CMD = be able to write it from memory.*

| Asked as | Item | What you must know / the exact command |
|---|---|---|
| MCQ | **Volumes** · what a volume is | A folder that belongs to the Pod, not to one container: all its containers (e.g. the app and a sidecar helper) can mount it, and its files survive container restarts. |
| MCQ | `emptyDir` vs `hostPath` | An `emptyDir` is created empty when the Pod starts and is deleted with the Pod, whereas a `hostPath` (a folder on the node) survives the Pod, but only on that one node. |
| MCQ | `configMap` / `secret` volume | Each key becomes a file whose content is the value; mounted files update by themselves within about a minute, whereas env vars are read only at container start. |
| CMD | Mount a volume (→ = indented under) | Pod level: `volumes:` → `- name: shared-data` → `emptyDir: {}`. In the container: `volumeMounts:` → `- name: shared-data` → `mountPath: /shared`. Different names make `kubectl apply` fail with `Not found`. |
| CMD | Run a command in one container | `kubectl exec shared-pod -c reader -- cat /data/log.txt`: `-c` picks the container (needed when the Pod has two); everything after `--` runs inside it. |
| MCQ | **PV, PVC & StorageClass** · PV vs PVC | A PersistentVolume (PV) is a real piece of storage (cluster-scoped), whereas a PersistentVolumeClaim (PVC) is a request for storage (in a namespace); a Pod names the PVC, never the PV. |
| MCQ | StorageClass · dynamic vs static | A StorageClass is a storage category plus a provisioner that creates a PV automatically when a claim arrives (dynamic), whereas in static provisioning an admin creates every PV by hand. |
| MCQ | Binding rules | Always one PVC to one PV, which needs the same class, the requested access mode and at least the requested size; a claim never splits across PVs or grows one. |
| MCQ | Access modes (full forms) | RWO = ReadWriteOnce (read-write by one node), ROX = ReadOnlyMany (read-only by many nodes), RWX = ReadWriteMany (read-write by many nodes), RWOP = ReadWriteOncePod (one single Pod). |
| MCQ | Trap: RWO | RWO means one node, not one Pod: Pods on that node can share it, whereas a Pod on another node gets `Multi-Attach error` and stays `ContainerCreating`. |
| MCQ | Reclaim policy `Delete` vs `Retain` | When the PVC is deleted, `Delete` removes the PV and its real storage, whereas `Retain` keeps the PV and its data in status `Released` until an admin cleans up. |
| MCQ | PV and PVC statuses | A PV goes Available → Bound → Released; a PVC goes Pending → Bound. A `Released` PV is not `Available`, so a new claim never binds to it automatically. |
| CMD | Write a PVC and use it | `kind: PersistentVolumeClaim` with `spec.accessModes: [ReadWriteOnce]` and `spec.resources.requests.storage: 100Mi` (no `storageClassName` = default class). Pod: `volumes:` → `- name: data` → `persistentVolumeClaim:` → `claimName: notes-pvc`. |
| CMD | List storage objects | `kubectl get pv,pvc` lists PVs and PVCs in one command (comma, no space); `kubectl get storageclass` lists the classes, the default one marked `(default)`. |
| CMD | Why is a PVC `Pending`? | `kubectl describe pvc bad-pvc \| tail -3` shows only the last 3 lines (the Events), e.g. `storageclass "fast-ssd" not found` or `no persistent volumes available`. |
| MCQ | **Probes** · why they exist | Without probes, Kubernetes only checks that the main process runs, so a deadlocked app still shows `Running`. A probe is a repeated kubelet health check set in the Pod spec (Docker's `HEALTHCHECK` is ignored). |
| MCQ | Liveness vs readiness | A failing liveness probe kills and restarts the container (RESTARTS climbs), whereas a failing readiness probe leaves it `0/1 Running`, out of the Service's endpoints (its traffic list), with no restart. |
| MCQ | Startup probe | Runs only until it first succeeds, and until then liveness and readiness are paused; it allows up to `failureThreshold × periodSeconds` to start (30 × 10 s = 300 s). |
| MCQ | Probe check types | `httpGet` passes on status 200–399 (301 passes, 404/500 fail); `tcpSocket` passes if a TCP connection opens; `exec` passes on exit code 0 (a command missing from the image gives 127, so it always fails). |
| MCQ | Probe timing defaults | `initialDelaySeconds` 0, `periodSeconds` 10, `timeoutSeconds` 1, `failureThreshold` 3 (failures in a row before acting), `successThreshold` 1 (must be 1 for liveness and startup). |
| MCQ | Trap: liveness on a slow app | A liveness probe that fires before a slow app is up causes a restart loop ending in `CrashLoopBackOff`; adding a startup probe fixes it, adding a readiness probe does not. |
| MCQ | Trap: database check | Dependency checks ("can I reach the database?") go in readiness, whereas in liveness one database blip restarts every Pod at once. With no readiness probe, users get errors during rollouts. |
| CMD | Probe YAML (in the container) | `livenessProbe:` → `exec:` → `command: ["cat", "/tmp/healthy"]`, with `periodSeconds: 5` and `failureThreshold: 3` beside `exec:`. `readinessProbe:` → `httpGet:` → `path: /ready`, `port: 80`. |
| MCQ | **Requests & limits** · request vs limit | A request is the amount reserved, and the scheduler adds up requests (not real use) to pick a node, whereas a limit is the maximum the container may use. |
| MCQ | Going over a limit | Over its CPU limit a container is throttled (slowed, not killed), whereas over its memory limit it is `OOMKilled` (OOM = Out Of Memory), exit code 137 (128 + 9, where 9 = SIGKILL). |
| MCQ | QoS (Quality of Service) classes | Guaranteed = requests equal limits for CPU and memory; Burstable = some set; BestEffort = none set. A node short of memory evicts BestEffort Pods first, Guaranteed last. |
| CMD | `resources` YAML (in the container) | `resources:` → `requests:` → `cpu: 200m`, `memory: 128Mi`; `limits:` → `cpu: 500m`, `memory: 256Mi`. `200m` = 200 millicores (a fifth of a core); `Mi` = mebibytes (powers of 1024). |
| MCQ | **Metrics Server & HPA** · Metrics Server | An add-on (optional extra) that collects current CPU and memory use from every kubelet, keeps only the latest numbers (no history), and feeds `kubectl top pods` / `kubectl top nodes` and the HPA. |
| MCQ | HPA formula | Every 15 s the HPA sets replicas = ceil(current replicas × current% ÷ target%), kept between `minReplicas` and `maxReplicas`; 2 Pods at 90% with target 50% → ceil(3.6) = 4. |
| MCQ | Trap: % of the request | "50% CPU" means 50% of the request, never the limit: 4 Pods each using 250m of a 200m request = 125%, so the HPA wants ceil(4 × 125 ÷ 50) = 10. |
| MCQ | Scaling down slowly | The HPA scales up fast but waits a 5-minute stabilization window before scaling down, to avoid flapping (removing then re-adding Pods); a manual `kubectl scale` is overridden at its next check. |
| MCQ | `TARGETS cpu: <unknown>/50%` | The HPA can't calculate, usually because the Pods have no CPU request (`missing request for cpu`); otherwise Metrics Server is broken or the HPA is under a minute old. |
| CMD | Create an HPA by command | `kubectl autoscale deployment web --cpu=60% --min=2 --max=8` (older spelling: `--cpu-percent=60`); watch it with `kubectl get hpa web -w`. |
| CMD | HPA YAML keys | `apiVersion: autoscaling/v2`, `kind: HorizontalPodAutoscaler`, `spec.scaleTargetRef` (what to scale), `minReplicas`, `maxReplicas`, and a `metrics` entry for `cpu` with `averageUtilization: 50`. |
| MCQ | HPA vs VPA | The HPA (Horizontal Pod Autoscaler) changes how many Pods run, whereas the VPA (Vertical Pod Autoscaler) changes each Pod's requests and limits; the Cluster Autoscaler adds or removes nodes. |

## 📝 Quiz (5 MCQs; answer in chat as "1 B, 2 D, …")

**1.** A Pod runs two containers, `app` and `shipper`, which share one `emptyDir` volume mounted at `/logs` in both. `app` writes 50 log files, then crashes, and the kubelet restarts it (RESTARTS 1). An hour later the Pod is deleted, and its Deployment creates a replacement Pod on the **same node**. What does `/logs` hold at each stage?
- A) After the crash: none at all; in the new Pod: empty
- B) After the crash: all 50 files; in the new Pod: all 50
- C) After the crash: all 50 files; in the new Pod: empty
- D) After the crash: only shipper sees them; new Pod: empty

**2.** A cluster uses **static** provisioning only. Its single PV, `pv-db` (10Gi, RWO, reclaim policy `Retain`), is Bound to the claim `db-pvc`. Someone deletes `db-pvc` and immediately re-applies the exact same claim YAML. What does `kubectl get pv,pvc` show?
- A) db-pvc Bound to pv-db again, with the old data intact
- B) db-pvc Pending; pv-db Released, old data still kept
- C) db-pvc Bound to a new PV; pv-db Released with old data
- D) db-pvc Pending; pv-db Available again, data wiped

**3.** When a shared database went down for 40 s, all 6 Pods of a Deployment were killed and restarted at the same moment, and because each Pod needs 2 minutes to start, the site was down far longer than the database. The only probe is a livenessProbe whose `exec` script queries the database. What is the best change?
- A) Add a startupProbe that also queries the database
- B) Switch liveness to tcpSocket on the database port
- C) Add a readinessProbe on `/`, keeping liveness as is
- D) Move the database check into a new readinessProbe

**4.** A container has the `resources:` block below. During a load test it tries to use about `900m` of CPU; an hour later a memory leak pushes it towards `300Mi`. What happens at each stage?
```yaml
resources:
  requests:
    cpu: 200m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```
- A) Held to 500m and slowed; then OOMKilled, exit code 137
- B) Held to 200m and slowed; then OOMKilled, exit code 137
- C) Killed at 500m, exit code 137; then OOMKilled the same way
- D) Held to 500m and slowed; then memory held at 256Mi, slowed

**5.** An HPA targets 50% CPU with `minReplicas: 2` and `maxReplicas: 6`. Its Deployment has 4 Pods; each requests `100m` CPU and is limited to `300m`. At 10:00 the average use is `120m` per Pod. At 10:02 the load stops completely (0%). How many replicas are there at 10:01 and at 10:04?
- A) 6 at 10:01, then 2 at 10:04
- B) 10 at 10:01, and still 10 at 10:04
- C) 6 at 10:01, and still 6 at 10:04
- D) 4 at 10:01, then 2 at 10:04
