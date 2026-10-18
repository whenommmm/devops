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

## ⚡ Must-remember
| Term | Meaning |
|---|---|
| PV statuses | Available → Bound → Released (PVC: Pending → Bound) |
| `Retain` / `Delete` | Keep PV and data as Released / delete PV and storage |
| `0/1 Running`, RESTARTS 0 | Readiness failing: no traffic, no restart |
| `OOMKilled`, exit 137 | Went over the memory limit |
| HPA math | ceil(replicas × current ÷ target), capped by min/max |
| `cpu: <unknown>/50%` | Pods have no CPU request |

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
