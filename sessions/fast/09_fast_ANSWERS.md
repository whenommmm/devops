# Session 09 · Storage, Probes & Autoscaling · FAST · Answer key

**1.** A Pod runs two containers, `app` and `shipper`, which share one `emptyDir` volume mounted at `/logs` in both. `app` writes 50 log files, then crashes, and the kubelet restarts it (RESTARTS 1). An hour later the Pod is deleted, and its Deployment creates a replacement Pod on the **same node**. What does `/logs` hold at each stage?
- A) After the crash: none at all; in the new Pod: empty
- B) After the crash: all 50 files; in the new Pod: all 50
- C) After the crash: all 50 files; in the new Pod: empty
- D) After the crash: only shipper sees them; new Pod: empty

**Correct: C.** An `emptyDir` belongs to the Pod, not to a container, so when only the `app` container crashes and is restarted, the folder and all 50 files are still there; when the Pod itself is deleted, its `emptyDir` is deleted with it, and the replacement Pod starts with a brand-new, empty one, even on the same node. A is wrong because a container restart wipes only the container's own filesystem, never a volume. B describes a `hostPath` (a folder on the node's own disk), which would still hold the files on the same node; an `emptyDir` is tied to the Pod, not to the node. D is wrong because the volume belongs to the whole Pod, so the restarted `app` sees exactly the same files as `shipper`.

---

**2.** A cluster uses **static** provisioning only. Its single PV, `pv-db` (10Gi, RWO, reclaim policy `Retain`), is Bound to the claim `db-pvc`. Someone deletes `db-pvc` and immediately re-applies the exact same claim YAML. What does `kubectl get pv,pvc` show?
- A) db-pvc Bound to pv-db again, with the old data intact
- B) db-pvc Pending; pv-db Released, old data still kept
- C) db-pvc Bound to a new PV; pv-db Released with old data
- D) db-pvc Pending; pv-db Available again, data wiped

**Correct: B.** With `Retain`, deleting the claim leaves `pv-db` and its data in place in status `Released`, and a Released PV is never handed to a new claim automatically (an admin must clean it up first). Because the cluster uses static provisioning only, there is no provisioner to create a fresh PV either, so the new `db-pvc` has nothing to bind to and stays `Pending`. A is the common belief that a claim with the same name gets its old PV back; it does not, because the PV is Released, not Available. C would only happen with dynamic provisioning, where a StorageClass's provisioner creates a new PV for the claim. D describes a wipe-and-reuse behaviour; `Retain` keeps the data and does not make the PV Available again.

---

**3.** When a shared database went down for 40 s, all 6 Pods of a Deployment were killed and restarted at the same moment, and because each Pod needs 2 minutes to start, the site was down far longer than the database. The only probe is a livenessProbe whose `exec` script queries the database. What is the best change?
- A) Add a startupProbe that also queries the database
- B) Switch liveness to tcpSocket on the database port
- C) Add a readinessProbe on `/`, keeping liveness as is
- D) Move the database check into a new readinessProbe

**Correct: D.** A failing liveness probe kills and restarts the container, so a liveness check that depends on the database restarts every Pod at once whenever the database blips. As a readiness check, a database outage only marks the Pods not Ready and removes them from the Service's endpoints; nothing restarts, so they take traffic again the moment the database returns, instead of after a 2-minute restart. A is wrong because a startup probe runs only until the app first starts, so it does nothing about a later outage, and the liveness check would still restart the Pods. B still makes liveness depend on the database (it now tests the database's port instead of running a query), so a database outage still restarts every Pod. C is half right: it adds a readiness probe, but it leaves the database query in liveness, so the mass restarts continue.

---

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

**Correct: A.** Limits are the caps. Wanting more CPU than the `500m` limit only throttles the container: it is held at 500m and runs slower, but keeps running. Memory cannot be handed out more slowly, so going past the `256Mi` memory limit gets the container killed as `OOMKilled` (Out Of Memory), with exit code 137. B is wrong because `200m` is the request, the amount reserved for the container, which it may exceed when the node has spare room; the cap is the 500m limit. C is wrong because going over the CPU limit is never a reason to kill a container, only to slow it down. D is wrong because memory is never throttled: over the memory limit means a kill.

---

**5.** An HPA targets 50% CPU with `minReplicas: 2` and `maxReplicas: 6`. Its Deployment has 4 Pods; each requests `100m` CPU and is limited to `300m`. At 10:00 the average use is `120m` per Pod. At 10:02 the load stops completely (0%). How many replicas are there at 10:01 and at 10:04?
- A) 6 at 10:01, then 2 at 10:04
- B) 10 at 10:01, and still 10 at 10:04
- C) 6 at 10:01, and still 6 at 10:04
- D) 4 at 10:01, then 2 at 10:04

**Correct: C.** Utilization is measured against the request: 120m ÷ 100m = 120%. So desiredReplicas = ceil(4 × 120 ÷ 50) = ceil(9.6) = 10, which is capped by `maxReplicas` at 6, and since the HPA checks every 15 s and scales up fast, there are 6 Pods by 10:01. At 10:04 the load has been gone for only 2 minutes, which is inside the 5-minute stabilization window, so the HPA still keeps 6 Pods (it drops to 2, the `minReplicas`, only after the window ends). A gets the scale-up right but ignores the stabilization window. B forgets that the result is capped by `maxReplicas`. D measures against the 300m limit instead of the request (120 ÷ 300 = 40%, so ceil(4 × 40 ÷ 50) = ceil(3.2) = 4) and also ignores the window.

---
