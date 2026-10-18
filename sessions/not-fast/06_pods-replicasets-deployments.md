# Session 06 · Kubernetes II: Pods, labels & selectors, ReplicaSets, Deployments (Oct 13, Session 2)

**Time:** ~90 min · **Concepts:** 50 → 53 · **Answers:** say "Kubernetes II questions" · **Answer key:** `06_pods-replicasets-deployments_ANSWERS.md`

**Before you start:** `minikube status` should show `Running` (if not, run `minikube start`). Work only in the `default` namespace. ⚠️ Never delete the `minikube`/`localstack` containers, and leave the `argocd`, `ingress-nginx` and `final-app` namespaces alone.

---

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

---

## Concept 50 · Pods: the smallest thing Kubernetes runs

**Words you'll learn:** Pod, shared network, sidecar, init container, phase, restart policy, CrashLoopBackOff.

### What a Pod is
In Docker you ran **containers** directly. Kubernetes doesn't. It runs **Pods**.

> A **Pod** is a small wrapper around **one or more containers** that should always run together on the same machine. It's the smallest thing Kubernetes can create, schedule onto a node, and manage.

Most Pods contain **just one container** (your app). The wrapper exists because Kubernetes needs one "unit" to give an IP address, place on a node, and attach storage to.

**Analogy: a hotel room.** The Pod is the room, and the containers are the guests staying in it. Everyone in the room shares:
- **the same address** → all containers in a Pod share **one IP address**
- **the same phone line** → they can call each other on **`localhost`**
- **the same cupboard** → they can share **volumes** (folders)

Because they share one IP, two containers in the same Pod **can't both use port 80**, just as two guests can't both answer the same extension.

### A Pod's YAML, line by line
```yaml
apiVersion: v1              # Pods are in the core API group → "v1"
kind: Pod                   # what we're creating
metadata:
  name: web                 # the Pod's name
  labels:
    app: web                # a key=value tag that other objects use to find this Pod
spec:                       # the desired state
  containers:               # a LIST of containers in this Pod
    - name: nginx           # the container's name inside the Pod
      image: nginx:1.27     # which image to run
      ports:
        - containerPort: 80 # documents the port the app listens on (like EXPOSE)
```

### Helper containers: sidecars and init containers
Sometimes a Pod needs more than one container:
- **Sidecar:** a helper container that runs **alongside** the main app the whole time. Example: a container that ships the app's log files to a log server. Because they share `localhost`, the helper can talk to the app directly.
- **Init container:** a container that runs **before** the app starts, does one setup job, and exits. Example: wait until the database is reachable. The app only starts once every init container has finished **successfully** (exit code 0).

### A Pod's life: phases
Every Pod has a **phase**, its big-picture stage:

| Phase | Meaning |
|---|---|
| **Pending** | Accepted by Kubernetes, but not running yet: still waiting for a node, or still downloading the image |
| **Running** | Placed on a node, and at least one container is running (or restarting) |
| **Succeeded** | All containers finished with exit code 0, and won't be restarted |
| **Failed** | All containers finished, and at least one failed (non-zero exit code) |
| **Unknown** | Kubernetes can't reach the node to find out |

⚠️ **Phase vs the STATUS column.** `kubectl get pods` shows a STATUS column that's **more detailed** than the phase. It can say things like `ContainerCreating` or `CrashLoopBackOff`. Those are **reasons**, not phases. A Pod showing `CrashLoopBackOff` is still in the **Running** phase, because it's placed on a node and Kubernetes keeps restarting its container.

### What happens when a container stops: restartPolicy
`restartPolicy` (one setting for the whole Pod, under `spec`) tells the **kubelet** what to do when a container exits:

| restartPolicy | Restarts the container when… | Used for |
|---|---|---|
| **Always** (the default) | it exits for **any** reason, even exit code 0 | Apps that should run forever (web servers). Deployments only allow this one |
| **OnFailure** | it exits with a **non-zero** code (failed) | One-off jobs that should retry until they succeed |
| **Never** | never | Jobs you only want to run once |

Restarts happen **inside the same Pod, on the same node**, and the **RESTARTS** counter goes up. *(This is the kubelet, the agent on each node that starts and restarts containers, working on its own.)*

**CrashLoopBackOff:** if a container keeps crashing, the kubelet doesn't restart it instantly forever. It **waits longer each time** (10 s, then 20 s, then 40 s… up to 5 minutes). That growing wait is called **back-off**, and a Pod stuck in this cycle shows **`CrashLoopBackOff`** = "keeps crashing, waiting before the next try". The fix is always to find out **why** it crashes: `kubectl logs` shows what the container printed before it died.

### Pods are disposable
Pods are **not repaired, they're replaced**. If a Pod's node dies, that Pod is gone for good, and a **bare Pod** (one you created directly) is **never** recreated. Something has to watch Pods and replace them: that's the **ReplicaSet**, which keeps a set number of Pod copies running.

### 🛠 Do (~8 min)
```bash
mkdir -p ~/devops-practice/k8s2 && cd ~/devops-practice/k8s2
code pod.yaml          # paste the Pod YAML above, save
kubectl apply -f pod.yaml
kubectl get pods       # web   1/1   Running
kubectl describe pod web | tail -8     # the Events at the bottom: Scheduled → Pulled → Created → Started
kubectl delete -f pod.yaml
```
**Watch a crash loop:** this container just exits with code 1, again and again.
```bash
kubectl run crasher --image=busybox:1.36 -- sh -c "exit 1"
kubectl get pod crasher -w        # watch STATUS: Error → CrashLoopBackOff, RESTARTS climbing (Ctrl+C to stop)
kubectl delete pod crasher
```
*(`kubectl run` creates a bare Pod. `--` separates kubectl's options from the command to run inside the container. `-w` = watch: keep printing updates.)*

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

---

## Concept 51 · Labels & selectors: how Kubernetes objects find each other

**Words you'll learn:** label, selector, matchLabels, annotation.

### The problem
A ReplicaSet needs to know **which Pods are "its" Pods**. A Service (one stable address that spreads traffic across Pods) needs to know **which Pods to send traffic to**. Pods are created and destroyed all the time, with random names, so finding them **by name** doesn't work.

### Labels
> A **label** is a **key=value tag** you stick on any Kubernetes object, e.g. `app=web`, `env=prod`, `tier=frontend`.

**Analogy:** colour stickers on boxes in a warehouse. The boxes' names don't matter; you find "all red boxes" by their sticker.

### Selectors
> A **selector** is a **filter that finds objects by their labels**: "give me everything with `app=web`".

From the command line:
```bash
kubectl get pods --show-labels        # show each Pod's labels
kubectl get pods -l app=web           # only Pods labelled app=web  (-l = label selector)
kubectl get pods -l app=web,env=prod  # BOTH labels (comma = AND)
kubectl get pods -l 'env!=prod'       # env is NOT prod (also matches Pods with no env label at all)
kubectl label pod web env=prod        # add a label to an existing Pod
```
*(Quote selectors containing `!` or spaces. The outer-quotes rule from shell scripting!)*

### Selectors in YAML: matchLabels
ReplicaSets and Deployments say **which Pods they own** with `selector.matchLabels`, and they create Pods from a `template` that carries labels:
```yaml
spec:
  selector:
    matchLabels:
      app: web          # "my Pods are the ones labelled app=web"
  template:
    metadata:
      labels:
        app: web        # every Pod I create gets this label
```
⚠️ **The template's labels must match the selector.** If they don't, the Deployment would create Pods it can't recognise as its own, so Kubernetes **rejects** the YAML when you apply it. A Pod may have **extra** labels, as long as it has all the ones the selector asks for.

### Annotations: labels' cousin
An **annotation** is also key=value information on an object, but it's **for humans and tools, never for selecting**. For example, `description: "frontend for the shop"` or a link to documentation.
- **Labels** = for finding and grouping (selectors use them)
- **Annotations** = extra notes (nothing selects by them)

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

---

## Concept 52 · ReplicaSets: keeping N copies alive

**Words you'll learn:** ReplicaSet, replicas, desired count, self-healing, adopt.

### The problem
A bare Pod that dies stays dead. Real apps need **several copies** running at all times, both to share the load and so one crash doesn't take the app down.

### What a ReplicaSet does
> A **ReplicaSet** keeps a set number of **identical Pods** (its **replicas**) running at all times.

It's a **controller** (a background program that keeps making the cluster match what you asked for) running a loop forever:
1. **Count** the running Pods that match its selector.
2. **Compare** with the desired number (`replicas`).
3. **Fix the difference:** too few → create Pods from the template; too many → delete some.

This is **self-healing**: delete a Pod, and a new one appears within seconds.

**Analogy:** a shop manager who must always have 3 cashiers on the floor. If one goes home sick, the manager calls in a replacement immediately. If a 4th shows up, one is sent home.

### A ReplicaSet's YAML
```yaml
apiVersion: apps/v1         # ReplicaSets and Deployments live in the "apps" group
kind: ReplicaSet
metadata:
  name: web-rs
spec:
  replicas: 3               # the desired number of Pods
  selector:
    matchLabels:
      app: web-rs           # Pods with this label count as "mine"
  template:                 # the "mould" used to create each Pod
    metadata:
      labels:
        app: web-rs         # must match the selector
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

### Two traps
1. **It counts ANY Pod with matching labels.** If you manually create a Pod with the label `app=web-rs`, the ReplicaSet **adopts** it (counts it as its own), sees 4 instead of 3, and **deletes one**.
2. **It doesn't update existing Pods.** If you change the template (say, a new image), the Pods already running **keep the old version**. Only Pods created **later** use the new template. To roll out a new version properly, you need a **Deployment** (next concept).

### 🛠 Do (~6 min)
```bash
code rs.yaml                    # paste the ReplicaSet YAML above, save
kubectl apply -f rs.yaml
kubectl get pods -l app=web-rs  # 3 Pods with random suffixes, e.g. web-rs-x7k2p
kubectl delete pod <one-of-the-pod-names>
kubectl get pods -l app=web-rs  # still 3: a brand-new one replaced it (self-healing)
kubectl delete -f rs.yaml       # deletes the ReplicaSet AND its Pods
```

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

---

## Concept 53 · Deployments: updating apps safely

**Words you'll learn:** Deployment, scaling, rollout, rolling update, maxSurge, maxUnavailable, Recreate, revision, rollback.

### What a Deployment is
> A **Deployment** manages ReplicaSets for you, so you can **update** your app to a new version **without downtime**, and **roll back** if something goes wrong.

You almost never create Pods or ReplicaSets yourself. You create a **Deployment**, and it creates everything below it:
```
Deployment "web"
   └── ReplicaSet "web-6cfbd9d4d"        (one per version of the app)
          ├── Pod "web-6cfbd9d4d-gvvmj"
          ├── Pod "web-6cfbd9d4d-x8k2p"
          └── Pod "web-6cfbd9d4d-q9m4t"
```
The names show who owns what: **Pod name = ReplicaSet name + random suffix**.

### Step 3: the Deployment YAML
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web                    # ← labels on the DEPLOYMENT ITSELF (lets `-l app=web` find it too)
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx           # ← the CONTAINER's name (needed by `kubectl set image`)
          image: nginx:1.27
```
It looks just like a ReplicaSet. The difference is what happens when you **change** it.

### Scaling
**Scaling** = changing how many copies run.
```bash
kubectl scale deployment web --replicas=5
```
⚠️ This changes the cluster, **not your YAML file**. If someone later runs `kubectl apply -f` with a file that still says `replicas: 3`, it goes **back to 3**. The file is the desired state.

### Rolling updates: how a new version goes out
When you change the image, the Deployment does **not** stop everything at once. It does a **rolling update**:
1. Creates a **new ReplicaSet** for the new version.
2. Starts a few new Pods. Once they're up, removes a few old Pods.
3. Repeats until all Pods run the new version.

Users keep being served throughout, because some Pods are always up.

**Analogy:** replacing the tyres on a bus one at a time while it keeps driving, instead of parking it for the day.

Two settings control the speed and safety:

| Setting | Meaning | Default |
|---|---|---|
| **maxSurge** | How many **extra** Pods (above `replicas`) may exist during the update | 25% |
| **maxUnavailable** | How many Pods may be **missing** (below `replicas`) during the update | 25% |

**Worked example**, with `replicas: 4` and the defaults:
- maxSurge 25% of 4 = **1** → at most **5** Pods at any moment
- maxUnavailable 25% of 4 = **1** → at least **3** Pods always available

**The other strategy, `Recreate`:** delete **all** old Pods first, then start the new ones. There's **downtime** in between. It's only used when two versions must never run at the same time.

### Watching, history and rollback
Each change to the Pod template creates a new **revision** (a numbered version), so you can go back:
```bash
kubectl set image deployment/web nginx=nginx:1.28   # nginx= is the CONTAINER name, then the new image
kubectl rollout status deployment/web               # follow the update until it's done
kubectl rollout history deployment/web              # list the revisions
kubectl rollout undo deployment/web                 # go back to the previous revision
kubectl rollout undo deployment/web --to-revision=1 # go back to a specific revision
```
**If the new version is broken** (say, an image tag that doesn't exist): the new Pods fail (`ImagePullBackOff`), and the rollout **stops**. Because of maxUnavailable, most **old Pods keep running**, so users are still served. Then you run `rollout undo`.

**How undo numbers things:** rolling back doesn't delete history. It **re-uses** the old revision and gives it the **next number**. So with revisions 1, 2, 3, undoing to 2 makes it revision **4**, and the history shows **1, 3, 4**.

### Other workload types (know what each is for)

| Kind | Runs… | Example |
|---|---|---|
| **Deployment** | N identical copies, forever | Web apps, APIs |
| **DaemonSet** | exactly **one Pod on every node** | Log collectors, monitoring agents |
| **StatefulSet** | Pods with **stable names and their own storage** | Databases |
| **Job** | Pods that do a task and **finish** | A database migration |
| **CronJob** | a Job **on a schedule** (cron syntax!) | A nightly backup |

### 🛠 Do (~12 min)
```bash
code deploy.yaml                         # paste the Step 3 YAML, save
kubectl apply -f deploy.yaml
kubectl get deploy,rs,pods -l app=web    # see all three layers
kubectl scale deployment web --replicas=4
kubectl set image deployment/web nginx=nginx:1.28
kubectl rollout status deployment/web    # "successfully rolled out"
kubectl get rs -l app=web                # TWO ReplicaSets: the new one with 4 Pods, the old one with 0
kubectl set image deployment/web nginx=nginx:does-not-exist
kubectl get pods -l app=web              # old Pods still Running; new ones in ImagePullBackOff
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl delete -f deploy.yaml
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

---

## ➕ Small but important

| Item | What it means |
|---|---|
| `kubectl get rs` | List ReplicaSets (rs = short name; also `po`, `deploy`) |
| `kubectl create deployment web --image=nginx:1.27 --replicas=3` | Create a Deployment without writing YAML (imperative) |
| `… --dry-run=client -o yaml > web.yaml` | Same, but only **print** the YAML into a file; nothing is created |
| `kubectl edit deployment web` | Open the live object in an editor; saving applies the change |
| `kubectl rollout pause / resume deployment/web` | Pause a rollout halfway (e.g. to check the new Pods), then continue |
| `revisionHistoryLimit` | How many old ReplicaSets are kept for rollback (default 10) |
| `kubernetes.io/change-cause` annotation | Text shown in `rollout history`, explaining why each revision was made |
| ReplicationController | The old, pre-ReplicaSet version. You may see it in exam options; don't use it |

**Quick checks:** (1) Which kind runs exactly one Pod per node? (2) After `kubectl scale … --replicas=6`, what does a later `kubectl apply -f` with `replicas: 3` in the file do?
*Answers:* (1) DaemonSet. (2) It sets the Deployment back to 3. The file is the desired state.

---

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

### ✍️ Write the command
**C1.** List only the Pods that have **both** labels `app=shop` and `env=prod`.

**C2.** In the Deployment `web`, change the container named `nginx` to the image `nginx:1.28`.

**C3.** Roll the Deployment `web` back to **revision 1** (not just the previous revision).

**C4.** Without creating anything in the cluster, write the YAML for a Deployment `api` (image `nginx:1.27`, 3 replicas) into a file called `api.yaml`.
