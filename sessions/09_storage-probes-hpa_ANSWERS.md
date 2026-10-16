# Answer key · Session 09 · Kubernetes V: storage, probes & autoscaling

⚠️ Only open after answering.

---

## Warm-up

**W1.** A teammate says a Secret is safe because `kubectl get secret db-pass -o yaml` shows `password: cGFzc3dvcmQ=` instead of the real password. What's accurate?
- A) It's encrypted with the cluster's private key
- B) It's only encoded; `base64 -d` reveals it
- C) It's hashed, so nobody can ever read it back
- D) It's safe, because only etcd can decode it

**Correct: B.** Secret values are **base64-encoded**, which is just a different way of writing the same bytes. `echo cGFzc3dvcmQ= | base64 -d` prints `password`. A is wrong: no encryption happens by default (encryption at rest in etcd is an extra setting, and it doesn't change what `kubectl get` shows). C is wrong: a hash can't be turned back, but base64 can. D is wrong: anyone with permission to read the Secret can decode it, no etcd needed.

---

**W2.** You apply an Ingress for host `shop.local` → Service `shop-svc:80`. `kubectl get ingress` lists it, but its ADDRESS column stays empty and no request is ever answered. `kubectl get pods -A | grep -i ingress` prints nothing. What's wrong?
- A) The host must be a real public domain name
- B) The Service must be changed to NodePort first
- C) The Ingress needs `type: LoadBalancer` added
- D) No Ingress controller exists to act on it

**Correct: D.** An Ingress is only a **set of rules**. An **Ingress controller** (e.g. ingress-nginx, a Pod) must be running to read those rules and route traffic; no controller Pods = nobody acts on them, and the ADDRESS stays empty. A: hosts like `shop.local` work fine with `/etc/hosts` or a `Host:` header. B: Ingress backends are normally plain ClusterIP Services. C: `type:` is a **Service** field; an Ingress has no such field.

---

**W3.** What does this print?
```bash
APP=backend
echo 'Deploying $APP to "prod"'
```
- A) `Deploying $APP to "prod"`
- B) `Deploying backend to "prod"`
- C) `Deploying backend to prod`
- D) `Deploying $APP to prod`

**Correct: A.** The **outer** quotes are single quotes, so **everything** inside is literal: `$APP` isn't expanded, and the inner double quotes are printed as ordinary characters. B would need outer double quotes. C and D wrongly drop the inner quotes, which are just text here.

---

## Check 62

**62.1** A Pod has an `app` container that writes logs to `/var/log/app`, and a `shipper` container that reads them and sends them to a log server. The logs only need to survive **container restarts**, not Pod deletion. Which volume fits best?
- A) `hostPath` with `/var/log` on the node
- B) A `configMap`, mounted in both containers
- C) `emptyDir`, mounted in both containers
- D) A separate PVC for each of the containers

**Correct: C.** An emptyDir belongs to the Pod, is shared by every container that mounts it, survives container restarts, and is deleted with the Pod: exactly what's needed. A works on one node but is risky (node files, tied to that node) and more than needed. B: a ConfigMap volume is for **reading** config, and the app can't write logs into it. D: two separate PVCs are two separate storages, so the shipper couldn't see the app's files.

---

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
**Model answer:** *"The volumeMount's `name: config` doesn't match the volume's `name: app-config`; the names must be identical, so change the mount's name to `app-config` (or rename the volume to `config`)."* (Note: `configMap.name: app-settings` is the **ConfigMap object's** name, a different thing from the **volume's** name.)

---

**62.3** You mount ConfigMap `app-settings` (keys `color=blue`, `mode=dev`) as a volume at `/etc/app`. What does `ls /etc/app` show inside the container?
- A) One file, `app-settings`, holding both keys
- B) Two files, `color` and `mode`, one per key
- C) Two env vars, `COLOR` and `MODE`, not files
- D) Nothing until the Pod is restarted by hand

**Correct: B.** In a configMap volume, **each key becomes a file** whose content is the value (`/etc/app/color` contains `blue`). A: the ConfigMap's name is not used as a file name. C describes the **env var** method (`envFrom`), not a volume. D: the files appear as soon as the container starts.

---

## Check 63

**63.1** A cluster uses **static** provisioning only. Available PVs: `pv-a` (10Gi, RWO) and `pv-b` (3Gi, RWX), both in the same class. A new PVC asks for **5Gi with RWX**. What happens?
- A) It binds to pv-a, since 10Gi covers 5Gi
- B) It binds to pv-b and expands it to 5Gi
- C) It stays Pending; no PV fits both needs
- D) It splits the request across pv-a and pv-b

**Correct: C.** A PV must satisfy **all** conditions at once: same class, the requested access mode, **and** capacity ≥ request. pv-a is big enough but lacks RWX (A is wrong); pv-b has RWX but is too small, and binding never resizes a PV (B is wrong). D: binding is always **one PVC to one PV**; claims are never split.

---

**63.2** A PV has `persistentVolumeReclaimPolicy: Retain`. Its PVC is deleted. What state is the PV in now?
- A) Deleted, along with all the data on it
- B) Available again for the next new claim
- C) Released, and the data is still there
- D) Bound, because the Pod still remembers it

**Correct: C.** With **Retain**, the PV stays with its data, in status **Released**. An admin must clean it up by hand. A is the **Delete** policy. B: a Released PV is **not** automatically offered to new claims (it still holds the old claim's data). D: Bound needs an existing claim, and the claim is gone.

---

**63.3** (one line) Explain PV vs PVC in contrast form, using the hotel analogy.
**Model answer:** *"A PV is the actual storage that exists in the cluster, like a real hotel room, whereas a PVC is a request for storage of a certain size and access mode, like a booking; Kubernetes binds one booking (PVC) to one matching room (PV), and the Pod uses the room by quoting the booking's name."*

---

## Check 64

**64.1** `kubectl get pods` has shown this for 10 minutes: `api-6f9c-x2x   0/1   Running   0   10m`. What's the most likely explanation?
- A) The readiness probe is failing on every check
- B) The liveness probe is failing on every check
- C) The image is still being pulled from the registry
- D) The Service selector doesn't match the Pod

**Correct: A.** `Running` + `0/1` + `RESTARTS 0` is the signature of a failing **readiness** probe: the container runs, it's never restarted, but it's not Ready. B: a failing liveness probe would make RESTARTS climb. C: during a pull, STATUS shows `ContainerCreating` (or `ImagePullBackOff`), not `Running`. D: a Service selector has no effect on the Pod's own READY column; it only changes the Service's endpoints.

---

**64.2** A container has a startupProbe with `periodSeconds: 10` and `failureThreshold: 30`, plus a livenessProbe. What does this give the app?
- A) Up to 30 s to start; liveness runs alongside
- B) Up to 300 s to start; liveness waits for it
- C) Up to 300 s to start; readiness never runs
- D) Exactly 10 s to start, then 30 restarts

**Correct: B.** Maximum start time = `failureThreshold × periodSeconds` = 30 × 10 = **300 s**, and liveness (and readiness) **pause until the startup probe succeeds**. A uses the wrong number and wrongly says liveness runs at the same time. C: readiness **does** run, once startup succeeds. D misreads the fields: 10 s is the gap between checks, and 30 is the number of allowed failures, not restarts.

---

**64.3** A readiness probe does `httpGet` on `/`, and the app answers `301 Moved Permanently`. What's the probe result?
- A) Failure, because only 200 counts as success
- B) Failure, because redirects mean the page moved
- C) Success, but only if the redirect is a 302
- D) Success, because any 2xx or 3xx code passes

**Correct: D.** An httpGet probe succeeds for any status code **from 200 to 399**, so 301 and 302 both pass. A is too strict. B confuses "moved" with "broken". C invents a difference: 301 (permanent) vs 302 (temporary) matters to browsers and search engines, not to probes.

---

## Check 65

**65.1** An HPA targets **60%** CPU (min 1, max 10). There are **3** Pods, and their average CPU use is **80%** of their request. How many replicas will the HPA ask for?
- A) 3
- B) 4
- C) 5
- D) 6

**Correct: B.** ceil(3 × 80 ÷ 60) = ceil(4.0) = **4**. Ceil only rounds **up** when there's a fraction, and 4.0 has none, so C (adding one "to be safe") is wrong. A ignores that 80% is well above the 60% target (the ratio 1.33 is outside the 10% tolerance). D would need 120% average use.

---

**65.2** An HPA has shown `cpu: <unknown>/50%` for 10 minutes. `kubectl top pods` works fine. What's the most likely cause?
- A) The pods have no CPU `limits` set
- B) maxReplicas is set lower than minReplicas
- C) The pods have no CPU `requests` set
- D) The Deployment has only a single replica

**Correct: C.** CPU **utilization** is "usage ÷ **request**", so without a request the HPA can't compute a percentage (`missing request for cpu`). `kubectl top` working rules out a Metrics Server problem. A: HPA percentages ignore limits. B: the API rejects an HPA whose max is below its min. D: one replica is a normal starting point for an HPA.

---

**65.3** (one line) Explain requests vs limits in contrast form, including what happens when a container goes over its CPU limit and over its memory limit.
**Model answer:** *"A request is the amount of CPU/memory reserved for a container, which the scheduler uses to choose a node and the HPA uses as the 100% mark, whereas a limit is the maximum the container may use; going over the CPU limit only throttles (slows) the container, but going over the memory limit gets it killed as OOMKilled (exit code 137)."*

---

## Session quiz

**Q1.** A Java app needs about **60 s** to start. Its Deployment has a livenessProbe (`httpGet /healthz`, `periodSeconds: 10`, `failureThreshold: 3`, no initial delay). The Pods restart roughly every 35 s and never become Ready. What's the best fix?
- A) Add a readinessProbe on the same `/healthz` path
- B) Add a startupProbe allowing about 60 s or more
- C) Set the liveness successThreshold to 3 instead
- D) Raise the liveness timeoutSeconds to 60 seconds

**Correct: B.** Liveness fails 3 × 10 s ≈ 30 s in, long before the app is up, so it's killed in a loop. A startupProbe (e.g. `failureThreshold: 12`, `periodSeconds: 10` = 120 s) pauses liveness until the app has started. A: a readiness probe never stops liveness from killing the container. C is rejected by the API (`must be 1` for liveness), and it's about successes anyway. D: `timeoutSeconds` is how long **one** check may wait; while the app isn't listening yet, each check is refused instantly, so it still fails.

---

**Q2.** On a **3-node** cloud cluster, a Deployment with `replicas: 3` mounts **one** PVC whose access mode is `ReadWriteOnce`. The three Pods land on three different nodes. One runs; the other two are stuck in `ContainerCreating` with a `Multi-Attach error` event. Why?
- A) The PVC needs 3× the storage for 3 replicas
- B) The StorageClass reclaim policy is Delete
- C) RWO allows only one Pod in the whole cluster
- D) RWO lets just one node mount it read-write

**Correct: D.** **ReadWriteOnce = one node** at a time. The volume is attached to the first Pod's node, so the Pods on the other two nodes can't attach it. C is the definition of **RWOP** (ReadWriteOncePod); under RWO, several Pods **on the same node** could share it. A: size has nothing to do with attaching. B: reclaim policy only matters when the PVC is deleted. (Fixes: use RWX storage, or give each replica its own PVC with a StatefulSet's `volumeClaimTemplates`.)

---

**Q3.** An HPA (min 2, max 10) targets **50%** CPU. The Deployment has **4** Pods; each **requests `200m`** and is **limited to `500m`**. Average use is **`250m`** per Pod. What replica count will the HPA set?
- A) 4, since 250m is 50% of the 500m limit
- B) 5, since 250m is 1.25× the 200m request
- C) 10, since 4 × (125% ÷ 50%) equals 10
- D) 2, since usage is under the 500m limit

**Correct: C.** Utilization is measured against the **request**: 250m ÷ 200m = **125%**. Then ceil(4 × 125 ÷ 50) = ceil(10) = **10**, which is within max 10. A uses the **limit**, which HPA ignores. B forgets to divide by the 50% target (it computes 4 × 1.25). D confuses the limit with the target.

---

**Q4.** During every rolling update of a web Deployment, users get errors for about 20 s per new Pod. The app needs ~20 s to load its data after the container starts. The Pods have a livenessProbe only. Which change fixes the errors?
- A) Add a readinessProbe on a /ready check
- B) Set `maxUnavailable: 0` in the strategy
- C) Raise the liveness initialDelaySeconds
- D) Set terminationGracePeriodSeconds to 60

**Correct: A.** Without a readiness probe, a Pod counts as Ready the moment its container starts, so the Service sends it traffic 20 s too early. A readiness probe keeps it out of the endpoints until it really answers. B is tempting: it keeps old Pods until new ones are "available", but without a readiness probe new Pods are "available" instantly, so nothing changes. C only delays the first **liveness** check; it doesn't affect traffic. D is about how Pods **stop**, not how they start.

---

**L1.** Explain liveness vs readiness probes in contrast form, saying what Kubernetes does when each one fails.
**Model answer:** *"A liveness probe checks whether the container is still alive (not stuck), and when it fails the kubelet kills and restarts the container, whereas a readiness probe checks whether the container can serve traffic right now, and when it fails the Pod is marked not Ready and removed from the Service's endpoints, without being restarted."*

**L2.** Write the full forms of **PVC**, **HPA** and **RWX**, and say what happens to a PV with reclaim policy `Retain` when its PVC is deleted.
**Model answer:** *"PVC = PersistentVolumeClaim, HPA = Horizontal Pod Autoscaler, RWX = ReadWriteMany. With reclaim policy Retain, deleting the PVC leaves the PV and its data in place in status Released; it isn't reused automatically, and an admin must clean it up or recreate it by hand."*

---

## ✍️ Write the command: answers
**C1.** With **one** command, list all PersistentVolumes and PersistentVolumeClaims.
`kubectl get pv,pvc`: `get` = list objects · `pv` = PersistentVolumes (cluster-wide) · `,` = list several kinds at once (no spaces) · `pvc` = PersistentVolumeClaims (in the current namespace, `default`). (Alternative: `kubectl get persistentvolumes,persistentvolumeclaims`.)

**C2.** Print the file `/data/log.txt` from the container `reader` inside the Pod `shared-pod`.
`kubectl exec shared-pod -c reader -- cat /data/log.txt`: `exec shared-pod` = run a command inside that Pod · `-c reader` = pick the container `reader` (needed: the Pod has two) · `--` = everything after this runs inside the container · `cat /data/log.txt` = print the file (the emptyDir is mounted at `/data` in `reader`).

**C3.** Make the Deployment `web` autoscale between **2** and **8** Pods, aiming for **60%** CPU (of the request). Use the new flag spelling.
`kubectl autoscale deployment web --cpu=60% --min=2 --max=8`: `autoscale deployment web` = create an HPA that targets `web` · `--cpu=60%` = target average CPU use of 60% of the Pods' CPU **requests** · `--min=2` = never fewer than 2 Pods · `--max=8` = never more than 8 Pods. (Alternative: `kubectl autoscale deployment web --cpu-percent=60 --min=2 --max=8`, the older spelling that kubectl 1.37 warns is deprecated.)

**C4.** Show only the **last 3 lines** of the details of the PVC `bad-pvc` (where its Events are).
`kubectl describe pvc bad-pvc | tail -3`: `describe pvc bad-pvc` = the claim's full details, Events at the bottom · `|` = pipe that text into the next command · `tail -3` = keep only the last 3 lines (e.g. `ProvisioningFailed … storageclass "fast-ssd" not found`). (Alternative: `kubectl describe pvc bad-pvc | tail -n 3`.)
