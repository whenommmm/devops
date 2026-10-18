# Answer key · Session 10 · Kubernetes VI: troubleshooting

⚠️ Only open after answering (and after trying the lab on your own).

---

## Warm-up

**W1.** A Pod's readinessProbe fails on every check, but its livenessProbe passes. Which set of observations is right?
- A) READY 0/1, RESTARTS rising, still in endpoints
- B) READY 1/1, RESTARTS 0, still receiving traffic
- C) READY 0/1, RESTARTS 0, not in the endpoints
- D) CrashLoopBackOff, and left out of the endpoints

**Correct: C.** A failing **readiness** probe marks the Pod not Ready (`0/1`) and removes it from the Service's endpoints, but **never restarts** it. A mixes in the liveness behaviour (restarts) and wrongly keeps it in the endpoints. B describes a healthy Pod. D: CrashLoopBackOff needs the container to keep **exiting**, which a readiness failure doesn't cause.

---

**W2.** On minikube, you delete a PVC that was dynamically provisioned by the `standard` StorageClass (no Pod uses it). What happens?
- A) The PV becomes Released and keeps the data
- B) The PV stays Bound until someone deletes it
- C) The PV goes back to Available for reuse
- D) The PV and its data are deleted with it

**Correct: D.** `standard` has `RECLAIMPOLICY Delete`, and dynamically provisioned PVs copy it, so deleting the claim deletes the PV **and** the data. A is what **Retain** does. B: Bound requires a claim, and the claim is gone. C: PVs are never automatically recycled for new claims today (`Recycle` is deprecated).

---

**W3.** What happens when this runs?
```bash
x=3
if [ $x > 5 ]; then echo big; else echo small; fi
```
- A) Prints `big` and creates a file named `5`
- B) Prints `small`, because 3 is less than 5
- C) Prints nothing, because `>` compares text
- D) Gives an error, because `[ ]` can't use `>`

**Correct: A.** Inside `[ ]`, `>` is **redirection**: the shell sends the output of `[ 3 ]` into a new file named `5`. `[ 3 ]` tests a single non-empty string, which is **true**, so it prints `big`. B is what `[ $x -gt 5 ]` would do. C: `\>` (escaped) would compare text, but plain `>` redirects. D: there's no error; that's why the bug is sneaky.

---

## Check 66

**66.1** A Pod has restarted 4 times. You want to see the **exit code and reason** of the container's last crash. Which command shows them?
- A) `kubectl logs <pod>`
- B) `kubectl describe pod <pod>`
- C) `kubectl get pod <pod> -o wide`
- D) `kubectl top pod <pod>`

**Correct: B.** `describe` shows `Last State: Terminated` with **Reason** and **Exit Code**, plus the Events. A shows what the **app printed**, not how the container ended. C adds IP and node, not exit codes. D shows live CPU/memory only. (`kubectl get pod -o yaml` also contains it, under `lastState`, but less readably.)

---

**66.2** `kubectl logs web-7d4b9` prints `Defaulted container "app" out of: app, log-shipper`, then the app's logs. You need the **log-shipper's** logs. What do you run?
- A) `kubectl logs web-7d4b9 --previous`
- B) `kubectl exec -it web-7d4b9 -- log-shipper`
- C) `kubectl logs deploy/log-shipper`
- D) `kubectl logs web-7d4b9 -c log-shipper`

**Correct: D.** `-c` picks the container inside a multi-container Pod. A shows the **previous instance** of the default container (`app`), not another container. B tries to **run** a program called `log-shipper` inside the Pod. C looks for a **Deployment** named log-shipper, but log-shipper is a container inside this Pod.

---

**66.3** (one line) Why does plain `kubectl logs <pod>` sometimes show nothing useful for a Pod in CrashLoopBackOff, and which flag helps?
**Model answer:** *"In a crash loop the current container may have only just restarted (or be waiting to restart), so plain `kubectl logs` shows little or nothing, whereas `kubectl logs <pod> --previous` shows the output of the container that crashed, which usually contains the error message."*

---

## Check 67

**67.1** `kubectl describe pod` shows `Last State: Terminated   Reason: OOMKilled   Exit Code: 137`. What happened?
- A) The app itself chose to exit with the code 137
- B) It went over its memory limit and was killed
- C) It went over its CPU limit, so it was killed
- D) Its liveness probe failed, so it got SIGTERM

**Correct: B.** OOMKilled = the container went over its **memory limit**, and the kernel's OOM killer sent **SIGKILL**: 128 + 9 = 137. A: the Reason would then be `Error`, not `OOMKilled`. C: going over a **CPU** limit only **throttles** (slows) a container; it's never killed for it. D: a liveness failure shows `Liveness probe failed` events, and a clean SIGTERM exit is 143 (or 137 only if it had to be SIGKILLed later), with Reason `Error`, not `OOMKilled`.

---

**67.2** A Deployment's Pod shows `CreateContainerConfigError`, and RESTARTS stays at 0. What's the most likely cause?
- A) The app exits with code 1 right after start
- B) The image tag doesn't exist in the registry
- C) An env var refers to a missing Secret key
- D) The liveness probe fails before it's ready

**Correct: C.** CreateContainerConfigError means the container **can't be created** because a ConfigMap/Secret (or a key in one) it references is missing, so nothing ever runs (RESTARTS 0). A gives `Error`/`CrashLoopBackOff` with rising restarts. B gives `ErrImagePull`/`ImagePullBackOff`. D gives restarts and `Liveness probe failed` events.

---

**67.3** (one line) A container's command is `["sh", "-c", "pyhton app.py"]`. Predict the log line and the exit code, and say what the number means.
**Model answer:** *"The log shows `sh: pyhton: not found` and the container exits with code 127, which is the shell's 'command not found' code; the Pod then cycles through Error and CrashLoopBackOff."*

---

## Check 68

**68.1** Events: `Failed to pull image "acme/billing:2.1": … pull access denied, repository does not exist or may require authorization`. The name is spelled correctly, and the image exists in your company's **private** Docker Hub repository. What's the fix?
- A) Set `imagePullPolicy: Never` on the container
- B) Add `imagePullSecrets` with registry login
- C) Change the tag to `latest` so it's found
- D) Run `docker pull` on your Mac first

**Correct: B.** The kubelet must **log in** to pull from a private repository: create a `docker-registry` Secret and reference it in `imagePullSecrets`. A stops pulling altogether, giving `ErrImageNeverPull` because the image isn't on the node. C: the tag isn't the problem; access is. D pulls into **Colima's** Docker on your Mac, which is separate from the cluster's container runtime (and wouldn't help a real cluster anyway).

---

**68.2** A Pod is Pending. Events: `0/3 nodes are available: 3 Insufficient memory.` Each node has 4 GiB; `kubectl top nodes` shows only 30% memory **in use**. The Pod **requests** `6Gi`. What's the best fix?
- A) Wait, since actual use is only 30% now
- B) Lower the request so it fits a node
- C) Raise the Pod's memory limit above 6Gi
- D) Add a toleration for memory pressure

**Correct: B.** The scheduler compares **requests** with each node's free allocatable room, and **no** node can ever give 6 GiB when it has 4 GiB in total. Lower the request to what the app really needs (or add bigger nodes). A: actual use doesn't matter to the scheduler, and waiting changes nothing. C: the **limit** isn't used for scheduling, and the request still wouldn't fit. D: tolerations relate to **taints**, not to lack of room.

---

**68.3** (one line) Explain taint vs toleration in contrast form.
**Model answer:** *"A taint is set on a node and repels Pods (e.g. `dedicated=gpu:NoSchedule` means 'don't schedule here'), whereas a toleration is set on a Pod and allows it to be scheduled onto nodes with a matching taint; a toleration permits but doesn't force placement on that node."*

---

## Check 69

**69.1** Pods run with the label `app: web` (all `1/1 Running`). The Service has `selector: app: Web`. `kubectl describe svc` shows `Endpoints:` empty. Why?
- A) Selectors are case-sensitive; `Web` ≠ `web`
- B) Endpoints only appear after a DNS lookup
- C) The pods need a readiness probe to be listed
- D) The Service must be a NodePort to get them

**Correct: A.** Labels and selectors are **case-sensitive** strings, so `Web` matches no Pod. B: endpoints are computed from the selector continuously; DNS has nothing to do with them. C: without a readiness probe, a running Pod counts as Ready and **is** listed. D: every Service type uses the same selector → endpoints mechanism.

---

**69.2** Through the Ingress, `shop.local` returns `503 Service Temporarily Unavailable`, while `nothing.local` returns `404`. What does the 503 tell you?
- A) No Ingress rule matched the host shop.local
- B) A rule matched, but no backend is Ready
- C) The Ingress controller itself isn't running
- D) The app returned 503 because of a code bug

**Correct: B.** ingress-nginx returns **503** when a rule **matched** but the backend Service has **no ready endpoints** (selector mismatch, readiness failing, no Pods). A is what **404** means, as `nothing.local` shows. C: if the controller weren't running, nothing would answer at all, not a 404 for one host and a 503 for another. D: possible in theory, but an app answering would mean there **are** ready endpoints; the classic Ingress 503 points to "no ready backends". Check `kubectl describe svc` first.

---

**69.3** (one line) `wget` from one Pod to a Service fails. Contrast what `bad address`, `Connection refused` and a timeout each point to.
**Model answer:** *"`bad address` means DNS couldn't resolve the name (wrong Service name or namespace, or CoreDNS down), whereas `Connection refused` means the IP was reached but nothing is listening on that port (wrong targetPort, app on 127.0.0.1, or no endpoints), and a timeout means packets are being dropped on the way (NetworkPolicy, firewall, or wrong IP)."*

---

## Session quiz

**Q1.** `kubectl describe pod` shows `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`. The container's memory limit is `128Mi`, and `kubectl top` showed it at ~125Mi just before each crash. CPU use is low. What's the best fix?
- A) Raise the CPU limit so it can finish its work
- B) Add a livenessProbe so it restarts in time
- C) Raise the memory limit (or fix the leak)
- D) Let an HPA add pods when memory gets high

**Correct: C.** The container needs more memory than its limit allows, so give it a realistic limit, or fix the leak if usage grows forever. A: CPU isn't the problem and CPU limits never cause OOM kills. B: it's already being restarted; a probe adds nothing. D: more Pods don't give **each** Pod more memory; every copy would still hit 128Mi and be killed.

---

**Q2.** Service `api-svc` shows `Endpoints: 10.244.0.12:3000,10.244.0.13:3000`. From another Pod, `wget http://api-svc` gives `Connection refused`. Inside an api Pod, `wget -qO- http://localhost:3000` works. What's the most likely cause?
- A) The selector doesn't match the pod labels
- B) The app listens on 127.0.0.1, not 0.0.0.0
- C) CoreDNS can't resolve the name api-svc
- D) The readiness probe is failing on the pods

**Correct: B.** Endpoints exist on the right port, the app answers on **localhost** inside its Pod, but refuses connections from outside the Pod: the classic **loopback-only** listener. A would leave the endpoints **empty**. C would give `bad address`, not `Connection refused` (the error even shows the resolved IP). D would also remove the Pods from the endpoints.

---

**Q3.** A Pod is Pending. Events: `0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims`. On minikube, `kubectl get pvc` shows the claim `Pending` with STORAGECLASS `gp2`. What fixes it?
- A) Lower the Pod's CPU request so it can fit
- B) Delete the Pod so the scheduler retries it
- C) Add a toleration for the minikube node
- D) Use `standard` or omit storageClassName

**Correct: D.** `gp2` is an AWS StorageClass that doesn't exist on minikube, so the claim can never bind and the Pod can't be scheduled. Point the claim at `standard` (or leave `storageClassName` out to get the default). A PVC's class can't be changed in place, so **delete and recreate the PVC**, and the Pod then schedules. A: the message is about the PVC, not CPU. B: a new Pod would hit the same unbound claim. C: the minikube node has no taints.

---

**Q4.** You ran `kubectl set image deploy/web web=nginx:1.28-alpne`. `kubectl get pods` now shows one new Pod `0/1 ImagePullBackOff` and the three old Pods still `1/1 Running`. Events: `… nginx:1.28-alpne: not found`. What's the situation, and the quickest safe fix?
- A) Old pods still serve; run rollout undo
- B) All pods are down; recreate the Deployment
- C) New pods serve; delete the old ReplicaSet
- D) Old pods were deleted; scale replicas to 0

**Correct: A.** A rolling update only removes old Pods as new ones become available. The new Pod can't pull its image (`alpne` is a typo for `alpine`), so the rollout is stuck and the **old Pods keep serving**. `kubectl rollout undo deploy/web` returns to the previous ReplicaSet (or `set image` again with the correct tag). B and D: the old Pods are still Running, as `get pods` shows. C: the new Pod isn't serving anything, and deleting the old ReplicaSet would remove the Pods that **are** working.

---

**L1.** Explain CrashLoopBackOff vs ImagePullBackOff in contrast form, naming the first command you'd run for each.
**Model answer:** *"CrashLoopBackOff means the image was pulled and the container started but keeps exiting, and Kubernetes is waiting longer between restarts, so the first command is `kubectl logs <pod> --previous` (then `describe` for the exit code), whereas ImagePullBackOff means the image could never be downloaded, so the container never started and has no logs, and the first command is `kubectl describe pod <pod>` to read the pull error in the Events."*

**L2.** What does exit code **137** mean? Show the arithmetic, and give two different situations in Kubernetes that produce it.
**Model answer:** *"137 = 128 + 9, meaning the process was killed by signal 9, SIGKILL, which can't be caught. In Kubernetes it happens when a container goes over its memory limit (Reason: OOMKilled), or when a container that was asked to stop with SIGTERM (Pod deleted, failed liveness probe, rolling update) didn't exit within the grace period (30 s by default) and was force-killed."*

---

## 🧪 Broken cluster lab: solutions

| # | Symptom | Command that revealed the cause | Cause | Fix |
|---|---|---|---|---|
| **lab-1** (`lab-frontend`) | STATUS flips between `ErrImagePull` and `ImagePullBackOff`; `0/1`; RESTARTS 0 | `kubectl describe pod -l app=lab-frontend` → `Failed to pull image "nginx:1.277": … docker.io/library/nginx:1.277: not found` | The **tag** `1.277` doesn't exist (a typo for `1.27`) | `image: nginx:1.27`, then `kubectl apply -f lab-1.yaml` |
| **lab-2** (`lab-worker`) | Cycles `Running` → `Error` → `CrashLoopBackOff`; RESTARTS rising | `kubectl logs deploy/lab-worker` (or `--previous`) → `ERROR: QUEUE_URL is not set`; `describe` → `Exit Code: 1` | The app needs the **`QUEUE_URL` environment variable**, which isn't set | Add to the container:<br>`env:`<br>`  - name: QUEUE_URL`<br>`    value: "redis://queue:6379"`<br>then apply. Logs now say `connected to redis://queue:6379` |
| **lab-3** (`lab-reports`) | `Pending`; NODE and IP `<none>` | `kubectl describe pod -l app=lab-reports` → `0/1 nodes are available: 1 Insufficient cpu.` | `cpu: "50"` requests **50 whole cores**; your node has about 6 | `cpu: 100m` (a tenth of a core), then apply |
| **lab-4** (`lab-shop`, `lab-shop-svc`) | Pods `1/1 Running`, but `wget http://lab-shop-svc` → `Connection refused` | `kubectl describe svc lab-shop-svc` → `Selector: app=lab-shopp`, `Endpoints:` empty; `kubectl get pods --show-labels` → `app=lab-shop` | **Selector typo** (`lab-shopp`), so it matches no Pod | Service `selector: app: lab-shop`, then apply. `describe svc` shows two endpoints and `wget` returns nginx's HTML |
| **lab-5** (`lab-api`, `lab-api-svc`) | Pod `1/1 Running`, but `wget http://lab-api-svc` → `Connection refused` | `kubectl describe svc lab-api-svc` → `TargetPort: 8080/TCP`, `Endpoints: 10.244.0.x:8080`; nginx listens on **80** | **Wrong `targetPort`**: traffic is sent to 8080, where nothing listens | `targetPort: 80`, then apply. `wget` returns nginx's HTML |

**What the lab teaches:**
- **lab-1 and lab-3** never ran a single line of app code, so `kubectl logs` is useless there; **`describe` → Events** is the tool. (lab-1: the kubelet couldn't pull. lab-3: the scheduler couldn't place it.)
- **lab-2** is the only one where **`kubectl logs`** gives the answer.
- **lab-4 and lab-5** look **perfectly healthy** in `kubectl get pods`, and give the **same** client error. The difference shows only in **`describe svc`**: no endpoints at all (selector) vs endpoints on the wrong port (targetPort).

---

## ✍️ Write the command: answers
**C1.** Pod `web-7d4b9` is in CrashLoopBackOff. Show what its container printed just before it died the last time.
`kubectl logs web-7d4b9 --previous`: `logs` = what the app printed; `web-7d4b9` = the Pod; `--previous` = read the **dead** (previous) container's output, not the freshly restarted one, which may not have printed anything yet. (Alternative: `kubectl logs web-7d4b9 -p`, the short form of `--previous`.)

**C2.** Pod `web-7d4b9` has two containers, `app` and `log-shipper`. Watch the `log-shipper` container's logs live, as new lines arrive.
`kubectl logs web-7d4b9 -c log-shipper -f`: `logs web-7d4b9` = that Pod's output; `-c log-shipper` = pick this container (without it you get the default one, `app`); `-f` = follow, keep printing new lines as they arrive until Ctrl + C. (Alternative: `kubectl logs -f web-7d4b9 -c log-shipper`; the flag order doesn't matter.)

**C3.** List every event in the current namespace, ordered by time so the newest event is at the bottom.
`kubectl get events --sort-by=.lastTimestamp`: `get events` = the namespace's logbook of what happened (Scheduled, Pulled, BackOff…); `--sort-by=` = order the list by a field; `.lastTimestamp` = each event's "last seen" time (the leading `.` means "this field of each event"), oldest first, so the newest ends up at the bottom.

**C4.** Open an interactive shell (`sh`) inside the Pod `api-5f6c8`.
`kubectl exec -it api-5f6c8 -- sh`: `exec` = run a command inside a running container; `-i` = interactive (keep your typing connected); `-t` = give it a terminal, so you get a prompt; `api-5f6c8` = the Pod; `--` = kubectl's own flags end here; `sh` = the program to run inside (type `exit` to leave). (Alternative: `kubectl exec -it api-5f6c8 -- bash`, but only if the image has bash; small images like busybox and alpine have only `sh`.)
