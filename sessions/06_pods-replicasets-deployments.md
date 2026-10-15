# Session 06 · Kubernetes II: Pods, labels & selectors, ReplicaSets, Deployments (Oct 13, Session 2)

**Time:** ~100 min · **Concepts:** 50 → 53 · **Answers:** `MY_ANSWERS.md` or say "Kubernetes II questions" · **Answer key:** `06_pods-replicasets-deployments_ANSWERS.md`

**Before you start:**
```bash
minikube status                   # host/kubelet/apiserver Running (else: minikube start)
kubectl config current-context    # context: minikube
kubectl get pods                  # expect "No resources found" (delete leftover web/hello/journey: kubectl delete pod <name>)
mkdir -p ~/devops-practice/k8s-2 && cd ~/devops-practice/k8s-2
```
> ⚠️ Never delete the `minikube` or `localstack` containers, and don't touch the other namespaces (`argocd`, `final-app`, `ingress-nginx`…). Everything today happens in `default`.

## 🔁 Warm-up
**W1.** An nginx Pod is `Running` with IP `10.244.0.12`. On your Mac, `curl http://10.244.0.12` just hangs. Which statement explains it and gives a working approach?
- A) Pod IPs are cluster-internal; use `kubectl port-forward`
- B) Pods serve nginx on 8080, so curl `10.244.0.12:8080` instead
- C) The Pod lacks `containerPort: 80`; add it and the IP answers
- D) 10.244.x.x is a public range that macOS blocks by default

**W2.** For a new Pod, which component **decides the node**, and which component **starts the containers** on it?
- A) The controller manager decides; kubelet starts them
- B) The scheduler decides; kube-proxy starts them
- C) The scheduler decides; the kubelet starts them
- D) The API server decides; containerd starts them alone

**W3.** A user's umask is `027`. They create a new **file** (not a folder). What permissions does it get?
- A) `750` (rwxr-x---)
- B) `640` (rw-r-----)
- C) `650` (rw-r-x---)
- D) `740` (rwxr-----)

## Concept 50 · Pods: the smallest thing Kubernetes runs
**Words you'll learn:** Pod, network namespace, `emptyDir`, Pod phase, container state, reason, `restartPolicy`, back-off, crash loop, `CrashLoopBackOff`, grace period, init container, sidecar, ephemeral, cattle vs pets.
> **Problem:** a web server and its helper (which keeps downloading fresh files for it) must share a machine and start/stop together, but separately scheduled containers could land on different nodes.
>
> **Idea:** a **Pod** is **the smallest unit Kubernetes creates and schedules: one or more containers that always run together on one node, share one IP and set of ports, and can share storage.** Most Pods hold one container.

**Analogy: a flat.** The Pod is the flat with one postal address (the Pod IP); containers are flatmates with private rooms (own filesystem and processes) who share the kitchen (a shared volume) and talk across the hall on `localhost`.
- They share one **network namespace** (Session 02): B reaches A's port 80 at **`localhost:80`**, so ⚠️ two containers can't both listen on one port (`Address already in use`). ⚠️ **Compose vs Pod:** two Compose services = two IPs, talking by name (`cache:6379`); one Pod = one IP, talking on `localhost`.
- Each Pod gets a **private** IP from the Pod range (`10.244.0.0/16` on minikube); a replacement Pod gets a **new IP** (that's why Services exist, Session 07).

| **Phase** (exactly five) | Meaning | ⚠️ The STATUS column often shows |
|---|---|---|
| `Pending` | Accepted; not all containers running yet (scheduling, image download) | `ContainerCreating` |
| `Running` | On a node; ≥ 1 container running, starting or **restarting** | **`CrashLoopBackOff`** |
| `Succeeded` | All containers exited with **0**; no restarts | `Completed` |
| `Failed` | All stopped, ≥ 1 with a non-zero code; no restarts | `Error` (with `Never`) |
| `Unknown` | The state can't be read (node not answering) | |

- STATUS shows a **reason** (why a container is in its **state**: `Waiting`, `Running` or `Terminated`), not the phase; `Terminating` (being deleted) isn't a phase either.
- **`restartPolicy`** (once per Pod, under `spec`): **`Always`** (default; restart after any exit, even 0; the only value Deployments allow) · **`OnFailure`** (only after a non-zero exit) · **`Never`**. Restarts happen **in the same Pod, on the same node** (RESTARTS goes up).
- **Back-off** = growing waits between restarts (10 s, 20 s, 40 s… max **5 min**); a container stuck in that cycle is in a **crash loop**, shown as **`CrashLoopBackOff`** *(minikube v1.37 may keep showing `Error` plus a `BackOff` event; same meaning)*.
- **Deleting:** STATUS `Terminating`; SIGTERM to each container's PID 1, then SIGKILL after the **grace period** (`terminationGracePeriodSeconds`, default **30 s**; `docker stop` waited 10).
- **Init container** (`initContainers:`) = runs **before** the app, one at a time; each must exit **0** or the app never starts (STATUS `Init:0/1`).
- **Sidecar** (another item under `containers:`) = a helper running **alongside** the app the whole time (e.g. a log shipper).
- **Ephemeral (short-lived): cattle, not pets.** Pets (a server named `zeus`) are nursed back to health; cattle (Pods like `web-6cfbd9d4d-gvvmj`) are replaced (new name, new IP). Bare Pods have nobody to replace them: ReplicaSets do (Concept 52).
```yaml
apiVersion: v1                      # Pods are in the core API group
kind: Pod
metadata:
  name: shop
spec:
  volumes:                          # storage the containers can share
    - name: shared
      emptyDir: {}                  # emptyDir = an empty folder that lives as long as the Pod
  initContainers:                   # run first, until finished
    - name: prepare
      image: busybox:1.36           # busybox = a tiny image of basic Linux tools
      command: ["sh", "-c", "echo '<h1>Prepared by the init container</h1>' > /work/index.html"]   # replaces ENTRYPOINT
      volumeMounts:
        - name: shared
          mountPath: /work          # the shared folder appears here in "prepare"…
  containers:                       # a LIST: each container starts with "- " (a missing dash is a classic bug)
    - name: web                     # container name (used with -c)
      image: nginx:1.27
      volumeMounts:
        - name: shared
          mountPath: /usr/share/nginx/html   # …and here in "web", so nginx serves the file
    - name: helper                  # the sidecar: fetches the page every 5 s via localhost
      image: busybox:1.36
      command: ["sh", "-c", "while true; do wget -qO- http://localhost:80; sleep 5; done"]
```
### 🛠 Do (~15 min)
```bash
kubectl run once --image=busybox:1.36 --restart=Never -- echo hi         # -- = the rest runs inside the container
kubectl run fail --image=busybox:1.36 --restart=Never -- sh -c 'exit 3'  # a pretend crash (exit code 3)
kubectl run loop --image=busybox:1.36 -- sh -c 'echo bye; exit 1'        # no --restart = Always
kubectl get pods     # after ~30 s: fail Error (Failed) · loop Error/CrashLoopBackOff, RESTARTS climbing · once Completed
for p in once loop; do kubectl get pod $p -o jsonpath='{.status.phase}'; echo; done   # Succeeded · Running (!)
kubectl logs once                                # hi
kubectl describe pod loop                        # State Terminated, Exit Code 1, Restart Count, BackOff event
kubectl delete pod once fail loop                # clean up
code shop.yaml                                   # paste the shop manifest, then save
kubectl apply -f shop.yaml && kubectl get pod shop -w   # && = only if apply worked; Init:0/1 → 2/2 Running; Ctrl + C
kubectl logs shop -c helper | head -3            # -c picks a container → <h1>Prepared by the init container</h1>
kubectl port-forward pod/shop 8083:80            # Tab 2: curl -s localhost:8083 → the same <h1>; then Ctrl + C
kubectl delete pod shop     # ~30 s: helper's shell is PID 1 and ignores SIGTERM → SIGKILL after the grace period
```
### ✅ Check 50
**50.1** One Pod has two containers: `app`, which listens on port 8080, and a sidecar `metrics`, which needs to call it. Which address should the sidecar use?
- A) `app:8080`
- B) `localhost:8080`
- C) `app.default:8080`
- D) `metrics:8080`

**50.2** `kubectl get pods` shows `loop   0/1   CrashLoopBackOff   5 (40s ago)   4m`. What is this Pod's **phase**?
- A) `Failed`
- B) `Pending`
- C) `Unknown`
- D) `Running`

**50.3** A Pod has `restartPolicy: OnFailure`. Its only container finishes with exit code **0**. What happens next?
- A) It restarts, since the container stopped
- B) No restart; the Pod shows `Completed`
- C) It restarts after a 10-second back-off
- D) No restart; the Pod shows `Error`

## Concept 51 · Labels and selectors: how objects find each other
**Words you'll learn:** label, prefix, label selector, equality-based, set-based, `matchLabels`, `matchExpressions`, annotation, `-l`, `--show-labels`, `-L`, `--overwrite`.
> **Problem:** Pod names (`web-6cfbd9d4d-gvvmj`) are random and change on every replacement, yet a ReplicaSet must count "its" Pods and a Service must find Pods to send traffic to. **Idea:** a **label** is **a key–value tag in `metadata.labels`** (`app: shop`, `env: prod`); a **label selector** is **a filter that picks objects by their labels**.

**Analogy: airport luggage tags.** The baggage system ignores a suitcase's brand; it reads the tag ("AI202, Priority") and sorts by that.
- Keys/values: up to **63 characters**, letters, digits, `-` `_` `.`, no spaces; a key may have a **prefix** ending in `/` (`app.kubernetes.io/name`). ⚠️ **Values are text:** `version: 2` fails (`cannot unmarshal number…`); write `version: "2"`.
- **Annotations** are also key–value pairs in `metadata`, but they **describe** objects (notes for people/tools, can be long: `kubernetes.io/change-cause: "upgrade to 1.28"`, a Git commit ID) and **can't be used in selectors**. Set them with `kubectl annotate` (labels: `kubectl label`).

| Selector | Kind | Picks objects whose… |
|---|---|---|
| `app=shop` (or `==`) · `app=shop,env=prod` | equality-based | `app` is `shop` · the comma means **AND** |
| `env!=prod` · `env notin (dev)` | equality · set-based | `env` isn't that value. ⚠️ **Includes objects with no `env` label at all** |
| `env in (prod,staging)` | set-based | `env` is prod **or** staging |
| `tier` · `!tier` | set-based | **have** a `tier` label (any value) · have **no** `tier` label |

Wrap set-based selectors and `!` in **single quotes** (`-l 'env in (prod,staging)'`), or the shell interprets them (the outer quotes decide). A Service's YAML selector is just labels (`selector:` with `app: shop` under it); ReplicaSets and Deployments write:
```yaml
selector:
  matchLabels:              # equality: every pair listed must match (extra Pod labels are fine)
    app: shop
  matchExpressions:         # set-based (optional)
    - key: env
      operator: In          # In, NotIn, Exists, DoesNotExist
      values:
        - prod
        - staging
```

**Selectors are the glue:** objects never point at each other by name.
- Deployment `web` ─ selector `app=web` ─► its ReplicaSet ─ `app=web` ─► Pods labelled `app=web` ◄─ `app=web` ─ Service `web` (Session 07).
- So a Deployment has the label in 3 places (①②③ in Concept 53's manifest):
  - ① `metadata.labels` (a tag on the Deployment itself, optional)
  - ② `spec.selector.matchLabels` ("the Pods I manage")
  - ③ `spec.template.metadata.labels` (stamped on every Pod it creates)
> ⚠️ ③ **must include everything in** ②, or the controller couldn't find its own Pods; Kubernetes refuses: `` `selector` does not match template `labels` ``. The selector **can't be changed** later (`field is immutable`), and `--dry-run=client` misses a mismatch (`--dry-run=server` catches it).
### 🛠 Do (~10 min)
```bash
kubectl run red   --image=nginx:1.27 --labels="app=shop,env=prod"   # --labels = set labels at creation
kubectl run green --image=nginx:1.27 --labels="app=shop,env=dev"
kubectl run blue  --image=nginx:1.27 --labels="app=blog,env=prod"
kubectl get pods --show-labels                  # adds a LABELS column, e.g. app=shop,env=prod
kubectl get pods -l app=shop                    # predict first! green, red   (-l = label selector)
kubectl get pods -l app=shop,env=prod           # red only (AND)
kubectl get pods -l 'env in (prod,staging)'     # blue, red
kubectl get pods -l env!=prod                   # green
kubectl get pods -L app,env                     # all three, plus APP and ENV columns
kubectl label pod green env=prod                # error: 'env' already has a value (dev), and --overwrite is false
kubectl label pod green env=prod --overwrite    # pod/green labeled (labels never change by accident)
kubectl label pod blue tier=frontend
kubectl get pods -l tier                        # blue
kubectl get pods -l '!tier'                     # green, red
kubectl label pod blue tier-                    # key + minus = remove the label
kubectl annotate pod red owner=vansh
kubectl get pods -l owner=vansh                 # No resources found: annotations aren't selectable
kubectl delete pods -l 'app in (shop,blog)'     # deletes all three (⚠️ preview with the same get -l first)
```
### ✅ Check 51
**51.1** Three Pods: `a` has `app=web,env=prod`; `b` has only `app=web`; `c` has `app=api,env=dev`. Which Pods does `kubectl get pods -l env!=prod` return?
- A) Only `c`
- B) Only `b`
- C) `b` and `c`
- D) `a`, `b` and `c`

**51.2** A Deployment has `selector.matchLabels: {app: web}`, but its template's labels are `{app: webapp}`. What happens on `kubectl apply`?
- A) Rejected: selector doesn't match template
- B) Created; each Pod gets both labels merged
- C) Created, but it never manages any Pods
- D) Created; the selector updates to `app: webapp`

**51.3** (one line) Contrast labels and annotations, with one example of each.

## Concept 52 · ReplicaSets: keeping N copies alive
**Words you'll learn:** replica, ReplicaSet (RS), `replicas`, Pod template, DESIRED / CURRENT / READY, self-healing, adoption, orphan, `ownerReferences`, cascading delete, ReplicationController (RC).
> **Problem:** in Session 05 a deleted bare Pod stayed gone; production needs "always 3 copies", enforced automatically.
>
> **Idea:** a **ReplicaSet** (`rs`) is **a controller that keeps a set number of identical Pods (replicas) running at all times**, using `replicas` (how many), `selector` (which Pods are "mine") and `template` (the mould for new Pods).

**Analogy:** a restaurant manager who must keep 3 waiters on the floor: one goes home ill → hire a replacement from the job description (the template); four turn up → send one home. **How many** matters, never **which**.
```yaml
apiVersion: apps/v1               # ReplicaSets live in the "apps" API group
kind: ReplicaSet
metadata:
  name: web-rs
spec:
  replicas: 3                     # the desired number of Pods
  selector:
    matchLabels:
      app: web-rs                 # Pods with this label count as "mine"
  template:                       # the Pod mould: metadata + spec, without a name
    metadata:
      labels:
        app: web-rs               # must match the selector
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```
- **Reconciliation loop** (Session 05): count matching Pods; too few → create from the template (`web-rs-` + 5 random characters); too many → delete extras.
  - Each Pod gets an **`ownerReferences`** entry (its owner), shown by `describe` as `Controlled By:  ReplicaSet/web-rs`.
  - `kubectl get rs`: **DESIRED** (= `replicas`), **CURRENT** (exist), **READY**.
- **Self-healing:** delete a Pod or lose a node → a new Pod (new name, new IP) appears within seconds.
- ⚠️ **Adoption trap:** an RS counts **every** matching Pod in its namespace with no other owner.
  - Hand-create a Pod labelled `app: web-rs` → it's **adopted** (4), then one is deleted (usually yours).
  - Reverse trick: relabel one of its Pods (`kubectl label pod web-rs-xxxxx app=debug --overwrite`) → it becomes an **orphan** (no owner) you can debug, and the RS replaces it.
- ⚠️ **A template change doesn't touch running Pods:** change `nginx:1.27` → `1.28` and apply, and the 3 Pods **stay on 1.27**; only later Pods get 1.28 → **mixed versions**. Proper updates are the Deployment's job.
- `kubectl delete rs` also deletes its Pods: a **cascading delete** (`--cascade=orphan` keeps them). **ReplicationController** (`rc`, `v1`) = the older version (equality selectors only); recognise it, don't use it.
### 🛠 Do (~12 min)
```bash
code rs.yaml                                     # paste the ReplicaSet manifest above, then save
kubectl apply -f rs.yaml
kubectl get rs,pods                              # web-rs 3 3 3 + three web-rs-xxxxx Pods. Tab 2: kubectl get pods -w
kubectl delete pod web-rs-7wrh8                  # use a real name; Tab 2: a NEW Pod appears
kubectl describe pod <any-web-rs-pod> | grep "Controlled By"   # Controlled By:  ReplicaSet/web-rs
kubectl run stray --image=nginx:1.27 --labels=app=web-rs       # the adoption trap: pod/stray created…
kubectl get pods                                               # …only 3 web-rs Pods remain
kubectl describe rs web-rs | tail -2                           # Deleted pod: stray
kubectl set image rs/web-rs nginx=nginx:1.28                   # set the container NAMED nginx to nginx:1.28
kubectl get pods -o 'custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image'   # only these columns: all still 1.27
kubectl scale rs web-rs --replicas=5                           # (delete a Pod + re-run get first: 1.28 + 1.27 mixed)
kubectl get rs                                                 # 5 5 5
kubectl delete rs web-rs
kubectl get pods                                               # cascading delete: Pods Terminating, then gone
```
*⚠️ Keep the single quotes around `custom-columns=…`: in zsh an unquoted `[0]` gives `zsh: no matches found`.*
### ✅ Check 52
**52.1** A ReplicaSet with `replicas: 3` and selector `app: web` has 3 running Pods. You run `kubectl run test --image=busybox:1.36 --labels=app=web -- sleep 3600`. What happens?
- A) 4 Pods run; an RS ignores Pods it didn't make
- B) The API server rejects the clashing label
- C) The RS scales itself up to `replicas: 4`
- D) The RS adopts `test`, then deletes one Pod

**52.2** You change a ReplicaSet's template image from `nginx:1.27` to `nginx:1.28` and apply it. What happens to the 3 running Pods?
- A) All 3 restart on 1.28 at the same time
- B) They're replaced one by one with 1.28
- C) They keep 1.27; only new Pods get 1.28
- D) The apply fails: templates are immutable

## Concept 53 · Deployments: scaling, rolling updates and rollbacks
**Words you'll learn:** Deployment, rollout, revision, `pod-template-hash`, horizontal / vertical scaling, rolling update, `Recreate`, downtime, `maxSurge`, `maxUnavailable`, available, change-cause, `ErrImagePull`, `ImagePullBackOff`, `progressDeadlineSeconds`, rollback.
> **Problem:** upgrading with bare ReplicaSets means scaling a new RS up and the old one down one Pod at a time, keeping the old one for going back, and stopping if new Pods fail. By hand, every release.
>
> **Idea:** a **Deployment** (`deploy`) is **a controller that manages ReplicaSets for you: change the Pod template and it rolls the change out gradually, keeping a history so you can roll back.**

**Analogy:** a restaurant changing its menu: new-menu cooks come in one at a time while old-menu cooks keep serving, and if the new dish makes people ill, the old cooks' details are still on file.
- Deployment `web` → owns ReplicaSet `web-557577df75` (current template) → owns Pods `web-557577df75-d5p5c`…; older ReplicaSets stay at 0 for rollback.
  - `557577df75` = the **`pod-template-hash`** (a fingerprint of the template: new template → new hash → new RS).
  - A **rollout** = moving the Pods to a new template; a **revision** = one numbered version in the history.
```yaml
apiVersion: apps/v1             # "Step 3": the Deployment manifest
kind: Deployment
metadata:
  name: web
  labels:
    app: web                    # ① a tag on the Deployment itself (optional)
spec:
  replicas: 3
  strategy:
    type: RollingUpdate         # the default; the other is Recreate
    rollingUpdate:
      maxSurge: 1               # at most 1 Pod ABOVE replicas during an update
      maxUnavailable: 0         # never fewer than 3 available Pods
  selector:
    matchLabels:
      app: web                  # ② "the Pods I manage"
  template:
    metadata:
      labels:
        app: web                # ③ stamped on every Pod; must include ②
    spec:
      containers:
        - name: nginx           # ← the CONTAINER name: kubectl set image uses it
          image: nginx:1.27
```
- **Scaling** = changing capacity (more Pods = **horizontal**; more CPU/memory per Pod = **vertical**): `kubectl scale`, edit `replicas:` + `kubectl apply`, or `kubectl edit`.
  - It resizes the current RS and **creates no revision** (only `spec.template` changes do).
  - ⚠️ **The file wins:** a later `kubectl apply -f` of a file saying `replicas: 3` goes back to 3, even if only the image changed.
- **Rolling update** (started by any template change: image, env var, template label, port): with replicas 3, maxSurge 1, maxUnavailable 0, it adds 1 new Pod (4 total), removes 1 old one once the new one is ready, and repeats until the old RS is at 0.
  - So ≥ 3 Pods always serve: **zero downtime** (**downtime** = time the app can't serve users).
  - **`maxSurge`** = Pods allowed **above** `replicas` (% rounded **up**, ceil); **`maxUnavailable`** = Pods allowed unavailable **below** `replicas` (% rounded **down**, floor).
  - Default **25%** each; they **can't both be 0**.

| replicas | Settings | Max total Pods | Min available Pods |
|---|---|---|---|
| 3 | 25% / 25% (default) | 3 + ceil(0.75) = **4** | 3 − floor(0.75) = **3** |
| 4 | 25% / 25% | 4 + 1 = **5** | 4 − 1 = **3** |
| 4 | maxSurge 0 / maxUnavailable 1 | **4** | **3** (remove one old, then add one new) |

- **Available** = ready (for `minReadySeconds`).
- **`Recreate`** = delete **all** old Pods, then create new ones → downtime; only when two versions must never run together (e.g. a shared database).
- ⚠️ In `kubectl set image deployment/web nginx=nginx:1.28`, the part before `=` is the **container name** (wrong → `unable to find container named "web"`).
- ⚠️ **Undo renumbers:** history `1, 2, 3` (on 3) → `rollout undo` reuses revision 2's template as **4** → `1, 3, 4`. kubectl also warns that **your YAML file didn't change**: the next `apply -f` brings the file's version back, so fix the file.
- **Broken image** (`nginx:1.99-typo`): the new Pod can't pull → **`ErrImagePull`**, then **`ImagePullBackOff`** (retrying with growing waits).
  - It never gets ready, so (with maxUnavailable 0) no old Pod is removed and **the old Pods keep serving**; after `progressDeadlineSeconds` (default **600 s**) the rollout is marked failed.
  - ⚠️ **No automatic rollback:** run `kubectl rollout undo`.
  - Causes: a typo, a tag never pushed, a private registry without login, no internet.
### 🛠 Do (~25 min): 1.27 → 1.28 → a broken tag → undo
```bash
code deployment.yaml                                   # paste the manifest above, then save
kubectl apply -f deployment.yaml && kubectl rollout status deployment/web   # … successfully rolled out
kubectl get deploy,rs,pods --show-labels               # several types at once; Pods carry pod-template-hash=<hash>
kubectl scale deployment web --replicas=5
kubectl get rs                                                     # the SAME RS shows 5 5 5 (no new revision)
kubectl apply -f deployment.yaml
kubectl get deploy web                                             # the file wins: READY back to 3/3
kubectl port-forward deployment/web 8084:80    # Tab 2: curl -sI localhost:8084 | grep -i server → nginx/1.27.5; Ctrl + C before each rollout
kubectl set image deployment/web nginx=nginx:1.28                  # new revision → rollout (Tab 2: kubectl get pods -w)
kubectl annotate deployment/web kubernetes.io/change-cause="upgrade nginx to 1.28"   # fills CHANGE-CAUSE (else <none>)
kubectl rollout status deployment/web          # exit code 0 = success, non-zero on failure/--timeout (stops set -e CI scripts)
kubectl get rs -o wide                                             # old RS kept at 0
kubectl rollout history deployment/web                             # 1 <none>, 2 upgrade nginx to 1.28
kubectl set image deployment/web nginx=nginx:1.99-typo             # a broken tag
kubectl annotate deployment/web kubernetes.io/change-cause="try 1.99-typo" --overwrite   # exists already → --overwrite
kubectl rollout status deployment/web --timeout=30s; echo $?       # timed out waiting for the condition · 1 = failure
kubectl get pods                                                   # 1 new ImagePullBackOff + 3 old Running
kubectl get deploy web                                             # READY 3/3, UP-TO-DATE 1, AVAILABLE 3
kubectl describe pod <the-broken-pod-name> | tail -8               # Failed to pull image "nginx:1.99-typo" … not found
kubectl rollout undo deployment/web && kubectl rollout history deployment/web   # 1, 3 try 1.99-typo, 4 upgrade… (2 became 4)
kubectl rollout undo deployment/web --to-revision=1 && kubectl get deploy web -o wide   # IMAGES: nginx:1.27
kubectl delete -f deployment.yaml                                  # cascading delete
kubectl get deploy,rs,pods                                         # → No resources found
```
### ✅ Check 53
**53.1** A Deployment has `replicas: 4` and the default strategy (`maxSurge: 25%`, `maxUnavailable: 25%`). During a rolling update, what's the largest number of Pods that can exist, and the smallest number that must stay available?
- A) At most 4 Pods; at least 3 available
- B) At most 5 Pods; at least 4 available
- C) At most 5 Pods; at least 3 available
- D) At most 6 Pods; at least 2 available

**53.2 🐞** Using the Deployment from Step 3, `kubectl set image deployment/web web=nginx:1.28` fails with `error: unable to find container named "web"`. What's wrong?
- A) The Deployment must be scaled to 0 first
- B) `set image` only works on bare Pods
- C) The tag `1.28` doesn't exist on Docker Hub
- D) It needs the container name, `nginx`

**53.3** A Deployment's history shows revisions `1` (nginx:1.27), `2` (nginx:1.28) and `3` (a broken tag, the current one). You run `kubectl rollout undo deployment/web`. What does `kubectl rollout history` show now?
- A) `1, 2, 3, 4`: revision 4 is a copy of 2
- B) `1, 3, 4`: revision 2 became revision 4
- C) `1, 2`: the broken revision 3 is deleted
- D) `1, 2, 3`: the current one is now 2

## ➕ Small but important (workloads & rollouts)
| Item | What it is |
|---|---|
| **DaemonSet** (`ds`) | **One Pod on every node** (new nodes get one automatically): log collectors, monitoring agents (`kube-proxy` on minikube) |
| **Job** / **CronJob** (`cj`) | Job = runs Pods **until a task succeeds** (retries up to `backoffLimit`, default 6; `restartPolicy` `OnFailure`/`Never`). CronJob = a Job **on a cron schedule** (`"0 2 * * *"` = daily 02:00) |
| **StatefulSet** (`sts`) | Like a Deployment, but Pods get **stable names** (`db-0`, `db-1`), **their own storage**, and start/stop **in order**: databases |
| Pod vs RS vs Deployment | Pod (`v1`): not replaced if deleted or its node dies. RS (`apps/v1`): keeps N, no rolling update. Deployment (`apps/v1`): rolling update + rollback; **the normal way** |
| `kubectl rollout pause` / `resume` | Pause, make several template changes, then resume to roll them out as **one** revision |
| `kubectl rollout restart deployment/web` | Replace every Pod gradually **without changing the image** (e.g. to pick up new config) |
| `revisionHistoryLimit` | Old ReplicaSets kept for rollbacks (default **10**); `0` = no rollback possible |
| `progressDeadlineSeconds` | Default **600**; then the rollout is marked failed (`ProgressDeadlineExceeded`); **no automatic rollback** |
| `kubectl create deployment web --image=nginx:1.27 --replicas=3` | Imperative create; add `--dry-run=client -o yaml > deployment.yaml` to get YAML instead |

**Quick checks:** (1) You delete the current ReplicaSet of a running Deployment with `kubectl delete rs web-<hash>`. What happens? (2) Which object would you use to run exactly one log-collector Pod on every node?
*Answers:* (1) The RS and its Pods are deleted, but the Deployment immediately recreates the RS (same hash) and new Pods, because its desired state hasn't changed; to remove the app, delete the **Deployment**. (2) A **DaemonSet**.

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory
**Q1.** A Pod has two containers, both from the `nginx` image (each listens on port 80). One of them keeps restarting, and its logs say `bind() to 0.0.0.0:80 failed (98: Address already in use)`. Why?
- A) Containers in a Pod share one network namespace
- B) A node can run only one port-80 container at once
- C) `containerPort: 80` is missing from one container
- D) A Pod can't run two containers from the same image

**Q2.** A ReplicaSet's selector is `matchLabels: {app: web, tier: front}`. You hand-create two Pods: `p1` with labels `app=web, tier=front, env=dev`, and `p2` with just `app=web`. Which Pods does the ReplicaSet count as its own?
- A) Both, since each one has `app=web`
- B) Neither: the extra `env` label breaks it
- C) Only `p1`: it has all selector labels
- D) Only `p2`: it matches exactly one label

**Q3.** A Deployment has `replicas: 4` and the default strategy (25% / 25%). Someone sets its image to a tag that doesn't exist. Ten minutes later, what does `kubectl get pods` show?
- A) 4 old Running, 1 new in ImagePullBackOff
- B) 3 old Running, 2 new in ImagePullBackOff
- C) 4 new in ImagePullBackOff, no old Pods left
- D) 4 old Running; K8s already rolled back

**Q4.** During a sale, you run `kubectl scale deployment web --replicas=6`. The next morning, a teammate changes **only the image** in `web.yaml` (which still says `replicas: 3`) and runs `kubectl apply -f web.yaml`. What happens?
- A) New image rolls out; 6 replicas are kept
- B) apply fails: the replica counts disagree
- C) Replicas return to 3; image is unchanged
- D) New image rolls out; back to 3 replicas

**L1.** Contrast a ReplicaSet and a Deployment in one sentence.

**L2.** Explain `maxSurge` vs `maxUnavailable` in contrast form, with the numbers for `replicas: 4` and the 25% defaults.
