# Session 10 · Kubernetes VI: troubleshooting (Oct 15, Session 2)

**Time:** ~110–120 min (includes a 25-min lab) · **Concepts:** 66 → 69 · **Answers:** `MY_ANSWERS.md` or say "K8s troubleshooting questions" · **Answer key:** `10_kubernetes-troubleshooting_ANSWERS.md`

**What this covers:** the official "Kubernetes 6–8" lecture: `get`, `describe`, `logs`, `exec`, Events, CrashLoopBackOff, ImagePullBackOff, Pending Pods, Service & DNS. Expect **"you see X, what's the cause?"** questions.

**Before you start:**
```bash
minikube status        # Running (if not: minikube start)
kubectl get nodes      # minikube   Ready
mkdir -p ~/devops-practice/k8s-vi/lab && cd ~/devops-practice/k8s-vi   # session folder + a lab folder
```
> ⚠️ **Never delete the `minikube` or `localstack` containers**, and leave alone namespaces/resources you didn't create (`argocd`, `ingress-nginx`, `final-app`). Today everything runs in `default`, with names starting `tour`, `crash-`, `pull-`, `pending-`, `bind-`, `dns-` or `lab-`.

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

## Concept 66 · The troubleshooting toolkit, and a method that always works
**Words you'll learn:** troubleshooting, symptom, root cause, `kubectl get` flags, STATUS, `kubectl describe`, Events, `kubectl logs` flags, `kubectl exec`, `kubectl get events`, `kubectl port-forward`, layer.
- **Problem:** Friday 6 p.m., "the shop is down!" A junior deletes random Pods for 40 minutes. A senior types five commands and finds a one-letter image-tag typo in three. The difference is a **method**.
- **Troubleshooting** = finding and fixing a problem's cause, step by step. A **symptom** is what you see (`CrashLoopBackOff`); the **root cause** is why (`DB_HOST` is missing). Fix the root cause.
- **Analogy: a doctor's visit.** `kubectl get` = vital signs (state at a glance). `describe` = the patient's chart (details + **Events**: what happened, in order). `logs` = "what do you feel?" (what the app printed). `exec` = examining the patient. `get events` = the ward's logbook.
- **`kubectl get` flags:** `-o wide` (adds IP and NODE; `-o` = output format), `-o yaml` (the whole object, including the live `status:`), `-w` (watch), `--show-labels`, `-l app=web` (only Pods with that label), `-A` (all namespaces).
- READY `0/1` = 0 of the Pod's 1 containers are ready. ⚠️ `kubectl get all` does **not** show ConfigMaps, Secrets, PVCs or Ingresses.

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

- **`kubectl describe pod <p>`** shows the Node, each container's **State / Last State / Reason / Exit Code / Restart Count**, Conditions, and the **Events** at the bottom (the gold).
- In Events, read **`Warning`** lines first. **From** = who reported it (`default-scheduler` places Pods; `kubelet` runs containers). `(x5 over 3m9s)` = it happened 5 times. ⚠️ Events last only ~**1 hour**.
- **`kubectl logs <pod>`** = what the app printed. `-f` follows live; **`--previous`** shows the **dead** container's output (essential for crash loops); `-c <container>` picks one container; `deploy/web` uses one of its Pods; `--tail=20`.
- ⚠️ A container that **never started** has **no logs** (`… is waiting to start`). Use `describe` instead.
- **`kubectl exec -it <pod> -- sh`:** `-i` interactive + `-t` terminal (like `docker exec -it`). **`--`** = kubectl's flags end here; the rest runs inside. One-off commands need no `-it`: `kubectl exec <pod> -- env`.
- **`kubectl get events --sort-by=.lastTimestamp`** = the whole namespace's events, newest at the **bottom**. `--field-selector type=Warning` shows only problems.
- **`kubectl port-forward svc/<s> 8090:80`** connects your Mac's port 8090 to the Service (like Docker's `-p`), skipping NodePort and Ingress.
- **The method: get → describe → logs → exec → events.**
- **Which layer is broken?** Node Ready → Pod scheduled → image pulled → container running → app ready → Service has endpoints → DNS resolves → Ingress matches.
- Work **from the inside out**: test inside the Pod, then from another Pod, then by Service name, then through the Ingress. The first layer that fails holds the problem.

### 🛠 Do (~12 min): a toolkit tour on a healthy app
```bash
kubectl create deployment tour --image=nginx:1.27 --replicas=2   # 2 Pods, labelled app=tour automatically
kubectl expose deployment tour --port=80                         # a ClusterIP Service "tour" (Session 07)
kubectl get pods -o wide                     # 1/1 Running, IP 10.244.0.x, NODE minikube
kubectl get pods --show-labels               # app=tour,pod-template-hash=… (the hash tells ReplicaSets apart)
kubectl describe pod <one-tour-pod-name>     # Events all Normal: Scheduled, Pulled, Created, Started = healthy
kubectl logs deploy/tour --tail=3            # Found 2 pods, using pod/tour-…, then nginx's start-up lines
kubectl exec deploy/tour -- nginx -v         # nginx version: nginx/1.27.5
kubectl exec -it deploy/tour -- sh           # now inside a Pod (prompt #); try the next three lines there
curl -s localhost | head -4                  #   the app answers inside its own Pod
cat /etc/resolv.conf                         #   the DNS settings Kubernetes gave the Pod (Concept 69)
exit                                         #   leave the Pod
kubectl port-forward svc/tour 8090:80        # Forwarding from 127.0.0.1:8090 -> 80 (Ctrl + C stops)
curl -s localhost:8090 | head -4             # in a 2nd tab: <title>Welcome to nginx!</title>
kubectl get events --sort-by=.lastTimestamp | tail -5   # newest: ScalingReplicaSet, SuccessfulCreate, Scheduled, Started
kubectl delete deployment tour               # clean up
kubectl delete service tour
```

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

## Concept 67 · CrashLoopBackOff and other container start failures
**Words you'll learn:** CrashLoopBackOff, back-off, `restartPolicy` (Always / OnFailure / Never), State / Last State, Reason, Exit Code, `Error` vs `Completed`, OOMKilled, the 128 + N rule, memory leak, CreateContainerConfigError, RunContainerError, StartError.
- **Problem:** Meera's worker shows `CrashLoopBackOff   5 (40s ago)`, and she thinks Kubernetes is broken. It isn't: **her app keeps exiting**, and Kubernetes keeps restarting it.
- **CrashLoopBackOff** = **Crash** (the main process exited) + **Loop** (restarted, exited again) + **Back-off** (the kubelet waits longer before each restart).
- The waits double: **10, 20, 40, 80, 160, then 300 s at most**. They reset after 10 minutes of running fine.
- **It's not the cause**, just "this keeps dying, so I'm waiting". The cause is inside the container. With `-w`, STATUS cycles `Running` → `Error` (or `Completed` for exit 0) → `CrashLoopBackOff`.
- **Analogy: redialling a friend whose phone is off.** You wait 1, 2, then 4 minutes between calls. You're not broken, the phone is off, and you must find out **why**.
- **`restartPolicy`:** **`Always`** (the default; restarts on any exit, even 0; the only value allowed in Deployments), **`OnFailure`** (non-zero exits only; Jobs = one-off tasks), **`Never`** (`kubectl run --restart=Never`).
- ⚠️ A Deployment whose command just runs `echo hello` and exits 0 still ends in **CrashLoopBackOff**: its container must **keep running**.

| Cause | What you see | Where you find it |
|---|---|---|
| App error, missing env var/config | `Error`, **Exit Code 1**, e.g. `DB_HOST is not set` | `kubectl logs --previous` |
| Typo in a shell command | **Exit Code 127**, `sh: pyhton: not found` | logs + describe |
| Command's file not in the image | **`RunContainerError`**, Last State `StartError`, Exit Code 128 | describe (no logs: it never started) |
| Failing liveness probe (Session 09) | Restarts, Exit Code **137** or **143** | describe → `Liveness probe failed…` |
| Out of memory | `OOMKilled`, Exit Code **137** | describe |
| Main process simply finishes | `Completed`, then CrashLoopBackOff | describe: `Exit Code: 0` |

| Exit code | Arithmetic (128 + signal number) | Meaning |
|---|---|---|
| **137** | 128 + **9** (SIGKILL) | **Killed by force**: the OOM killer, or SIGKILL after the grace period |
| **143** | 128 + **15** (SIGTERM) | **Asked to stop politely**, and it did: normal on delete or rollout |
| **139** | 128 + **11** (SIGSEGV) | **Segmentation fault**: the program crashed on bad memory access |

- **OOMKilled:** the container went over its memory **limit** (enforced by cgroups). The kernel's **OOM** (Out Of Memory) killer sent SIGKILL, so the code is 137.
- Fix OOMKilled by raising the limit, or by fixing the **memory leak** (a bug that keeps grabbing memory and never gives it back).
- **Recipe:** `get pods` (STATUS + RESTARTS) → `logs <pod> --previous` (the dead container's error) → `describe pod` (Last State, Exit Code, Events) → fix the YAML → `apply`.
- **`CreateContainerConfigError` is not a crash:** a ConfigMap/Secret (or a key in one) used by `env`/`envFrom` is missing, so the container is **never created**. RESTARTS stays **0** and there are **no logs**.
- Its Events say `configmap "app-config" not found` or `couldn't find key db_host…`. Create the missing object and the kubelet **retries by itself** (`optional: true` lets it start without).
- **`RunContainerError`:** the runtime couldn't start the container (command file missing or not executable, bad mount). Fix `command:`.

### 🛠 Do (~12 min): three ways to fail, and how each one looks
The manifest is ready-made: `~/DevOPS/sessions/labs/10/practice/crash-demos.yaml` (open it with `code` to read it). It makes three Pods:
- `crash-oom` runs `tail /dev/zero` (endless zero bytes with no line breaks, so memory grows) with a **32Mi** memory limit.
- `crash-typo` runs `pyhton app.py`, a typo (and busybox has no Python anyway).
- `crash-config` reads `DB_HOST` from a ConfigMap `crash-settings` that **doesn't exist yet**.
```bash
kubectl apply -f ~/DevOPS/sessions/labs/10/practice/crash-demos.yaml
kubectl get pods                                          # ~10 s: crash-config CreateContainerConfigError 0; crash-oom OOMKilled 1; crash-typo Error 1
kubectl describe pod crash-oom | grep -A 5 "Last State"   # -A 5 = 5 lines After: Terminated, Reason: OOMKilled, Exit Code: 137
kubectl logs crash-typo                                   # sh: pyhton: not found
kubectl describe pod crash-typo | grep -A 3 "Last State"  # Exit Code: 127 = command not found
kubectl describe pod crash-config | tail -3               # Error: configmap "crash-settings" not found
kubectl logs crash-config                                 # an error: the container was never created
kubectl create configmap crash-settings --from-literal=db_host=postgres   # create the missing ConfigMap
kubectl get pod crash-config                              # within ~10 s: 1/1 Running (the kubelet retried)
kubectl logs crash-config                                 # DB_HOST is postgres
kubectl delete -f ~/DevOPS/sessions/labs/10/practice/crash-demos.yaml --now   # clean up
kubectl delete configmap crash-settings
```
Run `kubectl get pods` again a little later and `crash-oom` / `crash-typo` show `CrashLoopBackOff`.

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

## Concept 68 · ImagePullBackOff / ErrImagePull, and Pending Pods
**Words you'll learn:** image pull, ErrImagePull, ImagePullBackOff, `imagePullSecrets`, `docker-registry` Secret, `imagePullPolicy`, CPU architecture (amd64 / arm64), `exec format error`, `ErrImageNeverPull`, `minikube image load`, Pending, scheduler, FailedScheduling, allocatable, Insufficient cpu/memory, preemption, node label, `nodeSelector`, taint, toleration, `NoSchedule`.
- **Pull problems:** before a container starts, the kubelet must **pull** (download) its image from a **registry** (Docker Hub, GHCR…).
- **`ErrImagePull`** = the pull just failed. **`ImagePullBackOff`** = waiting before retrying (the same doubling back-off). STATUS flips between them.
- The container never started, so there are no logs (`… waiting to start: image can't be pulled`). Use **describe**.
- **Pending** = accepted by the API server, but the **scheduler** (the part that chooses a node for each Pod) hasn't found a node. NODE and IP show `<none>`.
- Its Events say `FailedScheduling … 0/1 nodes are available: <reason>`. The text after the colon is the reason.
- `ContainerCreating` is different: a node was chosen and it's preparing. Stuck there usually means a volume problem (`MountVolume.SetUp failed`).
- **Analogy: a "Staff only" sign and a staff badge.** A **taint** is a "keep out" mark on a **node** (`key=value:NoSchedule`). A **toleration** on a **Pod** is the badge: it **allows** the Pod in, but doesn't **force** it there.
- Real clusters taint their control-plane nodes. Your minikube node has `Taints: <none>`.

| Message in `describe` Events (shortened) | Cause | Fix |
|---|---|---|
| `failed to resolve reference "…nginx:1.277": … not found` | The tag doesn't exist | Use a real tag |
| `pull access denied, repository does not exist or may require authorization` (or `401 Unauthorized`) | ⚠️ **A typo in the name** (`ngnix`), or a private repo without login | **Check the spelling first**; else `imagePullSecrets` |
| `lookup registry-1.docker.io … no such host` / `i/o timeout` | No network/DNS from the node | Fix the node's network |
| `no match for platform in manifest`, or a crash with `exec format error` | Wrong CPU architecture (awareness) | A multi-arch or **arm64** image |
| `Insufficient cpu` / `Insufficient memory` | Requests bigger than any node's **unreserved** room | Lower the requests, or add nodes |
| `unbound immediate PersistentVolumeClaims` | The PVC exists but is Pending (Session 09) | `kubectl describe pvc` |
| `persistentvolumeclaim "x" not found` | The PVC doesn't exist | Create it, or fix `claimName` |
| `didn't match Pod's node affinity/selector` | `nodeSelector` matches no node's labels | Label a node, or fix the selector |
| `had untolerated taint {dedicated: gpu}` | Tainted node, no toleration | Add a toleration |
| *(no events at all)* | The scheduler isn't running | Check `kube-scheduler-minikube` in `kube-system` |

- ⚠️ **Exam trap:** Docker Hub answers "pull access denied … may require authorization" for **any** repository it can't show you, including ones that don't exist.
- **Private registries:** first `kubectl create secret docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=<user> --docker-password=<token>`.
- Then list it under the Pod's `spec:` as `imagePullSecrets:` → `- name: regcred`. The kubelet logs in with it before pulling.
- **`imagePullPolicy`:** `IfNotPresent` (pull only if missing; the default for a specific tag), `Always` (the default for `latest` or no tag), `Never` (missing image → `ErrImageNeverPull`).
- **Architecture:** your Mac is **arm64**. Multi-arch images also contain **amd64** (Intel/AMD chips); amd64-only images fail on arm64 nodes.
- **Tip:** images from `docker build` live in Colima's Docker, not inside minikube. Copy them in with `minikube image load myapp:1.0`.
- **The scheduler counts requests, not real use:** a node at 5% CPU can still be "full". `kubectl describe node` shows **Allocatable** (what Pods may use) and **Allocated resources** (already requested).
- **Preemption** (evicting lower-priority Pods to make room): "Preemption is not helpful" in a message can be ignored.
- A **node label** (like `disktype=ssd`) plus a Pod's **`nodeSelector`** (`disktype: ssd` under `spec.nodeSelector`) means "only put me on nodes with this label".
- A toleration is written under the Pod's `spec.tolerations`, with `key: dedicated`, `operator: Equal`, `value: gpu`, `effect: NoSchedule`.

### 🛠 Do (~12 min): one pull failure and two Pending Pods
The manifest is ready-made: `~/DevOPS/sessions/labs/10/practice/pull-pending.yaml`. It makes three Pods:
- `pull-typo` uses the image `ngnix:1.27` (misspelt).
- `pending-selector` has `nodeSelector:` → `disktype: ssd`, a label no node has.
- `pending-pvc` mounts a PVC named `does-not-exist`.
```bash
kubectl apply -f ~/DevOPS/sessions/labs/10/practice/pull-pending.yaml
kubectl get pods -o wide                          # 2 × Pending with NODE/IP <none>; pull-typo ErrImagePull HAS a node and IP
kubectl describe pod pull-typo | tail -4          # pull access denied, repository does not exist…: the name is misspelled
kubectl logs pull-typo                            # … is waiting to start: image can't be pulled
kubectl describe pod pending-selector | tail -2   # 0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector
kubectl describe pod pending-pvc | tail -2        # 0/1 nodes are available: persistentvolumeclaim "does-not-exist" not found
kubectl label node minikube disktype=ssd          # add the label: the scheduler retries and places the Pod
kubectl get pod pending-selector -o wide          # Running on minikube
kubectl label node minikube disktype-             # key + minus = remove the label (a running Pod stays)
kubectl describe node minikube | grep -A 6 "Allocated resources"   # cpu 1370m (22%) requests, 2700m (45%) limits…
kubectl describe node minikube | grep Taints      # Taints: <none>
kubectl delete -f ~/DevOPS/sessions/labs/10/practice/pull-pending.yaml   # clean up
```

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

## Concept 69 · Service & DNS troubleshooting
**Words you'll learn:** endpoints, EndpointSlice, selector mismatch, `targetPort` / `containerPort`, bind address, `Connection refused` vs timeout vs `bad address`, CoreDNS, `kube-dns`, FQDN, search domains, `/etc/resolv.conf`, NXDOMAIN, Ingress 404 / 502 / 503, default backend, NetworkPolicy.
- **Problem:** all Pods are `1/1 Running`, yet `wget http://lab-shop-svc` from another Pod fails. The containers are fine; **the path to them** is broken.
- **The path:** name → **DNS** (CoreDNS) → ClusterIP → **endpoints** (IPs of Ready Pods matching the selector) → `PodIP:targetPort` → the app, listening on `0.0.0.0`.
- **Analogy: posting a letter to a flat.** The street must be in the directory (DNS), the building must list residents (endpoints), and the flat number must be right (targetPort). Someone must also answer the door to visitors, not just to housemates (`0.0.0.0` vs `127.0.0.1`).
- **Check 1, endpoints:** run `kubectl describe svc <s> | grep -E "Selector|Endpoints"`. Empty endpoints = no Pods to send traffic to.
- Three causes: the **selector doesn't match** (typo or case: **labels are case-sensitive**), the Pods **aren't Ready**, or they're in **another namespace**.
- `kubectl get pods -l app=lab-shopp` printing `No resources found` proves a selector typo.
- An **EndpointSlice** is the newer form of the same list, so `kubectl get endpoints` warns that it's deprecated. Harmless.
- **Check 2, targetPort:** endpoints show `PodIP:targetPort` (e.g. `:8080`). If the app listens on 80, you get **`Connection refused`**. ⚠️ `containerPort` is mostly **documentation**: it opens no port.
- **Check 3, bind address:** an app on `127.0.0.1` (loopback) accepts connections only from **its own Pod**. It must listen on `0.0.0.0` (all interfaces), as the Flask app did in Session 04.
- The tell-tale: endpoints fine, the test inside the Pod works, the test from another Pod is refused.
- **Check 4, DNS:** **CoreDNS** is the cluster's DNS server: Pods labelled `k8s-app=kube-dns` in `kube-system`, behind the Service **`kube-dns`** (`10.96.0.10`).
- Each Pod's `/etc/resolv.conf` has `nameserver 10.96.0.10` and a `search` line. **Search domains** are endings tried on short names, so a plain `api` works only from the **same namespace**.
- The **FQDN** (Fully Qualified Domain Name) is `<service>.<namespace>.svc.cluster.local`. From `default`, Service `api` in namespace `shop` needs at least `api.shop`.
- Use the **Service** name, not the Deployment's. If nothing resolves, not even `kubernetes.default`, check the CoreDNS Pods and their logs.
- **NXDOMAIN** = "Non-Existent Domain", the DNS answer for "no such name". ⚠️ Don't `ping` a ClusterIP: it's virtual and usually doesn't answer ping. Use `wget` and `nslookup`.

| `wget`/`curl` says | Meaning | Usual causes |
|---|---|---|
| **`bad address 'x'`** / `could not resolve host` / NXDOMAIN | **DNS** couldn't find the name | Wrong Service name or namespace, CoreDNS down |
| **`Connection refused`** | The IP was reached, but **nothing listens** on that port | Wrong targetPort, app on 127.0.0.1, **no endpoints** |
| **Timeout** (hangs, then gives up) | Packets are **dropped** somewhere | A NetworkPolicy, a firewall, a wrong IP |
| Ingress **404** | No rule matched | Wrong host/path/`ingressClassName`; the **default backend** (catch-all) answered |
| Ingress **503** | A rule matched, but **no ready endpoints** | Check 1 |
| Ingress **502** | Endpoints exist, but connecting **failed** | Checks 2–3, or a crashing app |

- **NetworkPolicy** (awareness): a **firewall rule for Pods**. A "default deny" policy silently drops traffic, giving timeouts. It needs a network plugin that enforces it (e.g. Calico). Check with `kubectl get networkpolicy`.

### 🛠 Do (~15 min): the 127.0.0.1 trap, and DNS checks
`code bind-demo.yaml`. busybox's tiny web server `httpd`: `-f` = foreground, `-v` = log requests, **`-p 127.0.0.1:8000` = loopback only**, `-h /www` = serve that folder:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bind-demo
  labels:
    app: bind-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "mkdir -p /www && echo 'hello from the app' > /www/index.html && httpd -f -v -p 127.0.0.1:8000 -h /www"]
---
apiVersion: v1
kind: Service
metadata:
  name: bind-demo
spec:
  selector:
    app: bind-demo
  ports:
    - port: 80
      targetPort: 8000
```
```bash
kubectl apply -f bind-demo.yaml
kubectl run dns-test --image=busybox:1.28 --restart=Never -- sleep 3600   # client Pod; 1.28's nslookup prints clean answers
kubectl describe svc bind-demo | grep -E "TargetPort|Endpoints"   # 8000/TCP and 10.244.0.x:8000: endpoints are FINE
kubectl exec dns-test -- wget -qO- -T 3 http://bind-demo          # -T 3 = 3 s timeout → Connection refused
kubectl exec bind-demo -- wget -qO- http://127.0.0.1:8000         # hello from the app: it works inside
kubectl exec bind-demo -- netstat -tln                            # TCP listening sockets: 127.0.0.1:8000 LISTEN = the cause
```
**Fix:** in `bind-demo.yaml`, change `-p 127.0.0.1:8000` to **`-p 8000`** (all interfaces). A Pod's command can't be edited in place, so delete and re-apply:
```bash
kubectl delete pod bind-demo --now
kubectl apply -f bind-demo.yaml
kubectl exec bind-demo -- netstat -tln                    # :::8000 = all interfaces (IPv6 form, covers IPv4 too)
kubectl exec dns-test -- wget -qO- -T 3 http://bind-demo  # hello from the app
kubectl exec dns-test -- cat /etc/resolv.conf             # search default.svc.cluster.local … / nameserver 10.96.0.10
kubectl exec dns-test -- nslookup bind-demo               # Address 1: 10.105.x.x bind-demo.default.svc.cluster.local
kubectl exec dns-test -- nslookup kubernetes.default      # 10.96.0.1: the API server's own Service
kubectl exec dns-test -- nslookup bind-demo-typo          # can't resolve 'bind-demo-typo': a DNS failure
kubectl get pods -n kube-system -l k8s-app=kube-dns       # coredns-… Running
kubectl get svc -n kube-system kube-dns                   # 10.96.0.10, ports 53/UDP,53/TCP,9153/TCP (DNS mostly uses UDP)
kubectl delete -f bind-demo.yaml --now                    # clean up
kubectl delete pod dns-test --now
```

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
**Hints (only if stuck 5+ min):**
- Status flips between two "pull" words → read the `Failed` event's last few words.
- `Error` / `CrashLoopBackOff` → what did the dead container print? What can a container have besides `command`?
- NODE is `<none>` → how many CPU cores does your node have? (`kubectl describe node minikube | grep -A 2 Allocatable`)
- `Connection refused` for both Services → compare `describe svc`: one has **no** endpoints, one has endpoints on a **suspicious port**.

## ➕ Small but important (troubleshooting): read before the quiz
| Item | What it is |
|---|---|
| **Exit codes** | **0** success · **1** app error · **126** found but **not executable** · **127** **command not found** · **137** SIGKILL · **139** SIGSEGV · **143** SIGTERM |
| **Stuck `Terminating`** (awareness) | Often a **finalizer** (a "don't delete me until cleanup is done" note, e.g. `kubernetes.io/pvc-protection`). Last resort: `kubectl delete pod <p> --grace-period=0 --force` |
| **`kubectl debug`** (awareness) | `kubectl debug -it <pod> --image=busybox:1.28 --target=<container>` adds an **ephemeral container** (a temporary troubleshooting container) for images with **no shell** |
| **`kubectl rollout status deploy/<d>`** | Waits for a rollout; `--timeout=60s` fails with `timed out waiting for the condition`. Old Pods keep serving; undo with `kubectl rollout undo deploy/<d>` |
| **`kubectl auth can-i`** (awareness) | "Am I allowed?": `kubectl auth can-i create pods` → yes/no. Explains `Forbidden` errors (**RBAC**, Role-Based Access Control) |
| **Node `NotReady`** | The node's **kubelet** stopped reporting. `kubectl describe node` → Conditions (`MemoryPressure`, `DiskPressure`, `Ready`) |
| **`Evicted`** | A node short of memory/disk pushed the Pod off; **BestEffort** Pods go first (QoS, Session 09) |
| **`Init:0/1`, `Init:CrashLoopBackOff`** | An **init container** (runs to completion **before** the main ones) is running or failing: `kubectl logs <pod> -c <init-name>` |
| **Catching YAML mistakes** | `kubectl apply --dry-run=server -f f.yaml` checks without creating: `error converting YAML to JSON` (indentation), `unknown field "contaners"` |
| **ResourceQuota rejections** | Pods **never appear** (not even Pending): `kubectl describe rs <rs>` → `FailedCreate … exceeded quota` |

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
