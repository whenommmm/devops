# Answer key · Session 06 · Kubernetes II: Pods, labels & selectors, ReplicaSets, Deployments

⚠️ Only open after answering.

---

## Warm-up

**W1.** An nginx Pod is `Running` with IP `10.244.0.12`. On your Mac, `curl http://10.244.0.12` just hangs. Which statement explains it and gives a working approach?
- A) Pod IPs are cluster-internal; use `kubectl port-forward`
- B) Pods serve nginx on 8080, so curl `10.244.0.12:8080` instead
- C) The Pod lacks `containerPort: 80`; add it and the IP answers
- D) 10.244.x.x is a public range that macOS blocks by default

**Correct: A.** Pod IPs belong to the cluster's private Pod network, which lives inside the minikube container, so your Mac has no route to them. `kubectl port-forward pod/<name> 8082:80` tunnels through the API server. B: nginx listens on 80 inside the Pod, and the port isn't the problem anyway. C: `containerPort` is documentation only; adding it changes nothing about reachability. D: `10.x.x.x` is a **private** range (networking session), not public.

---

**W2.** For a new Pod, which component **decides the node**, and which component **starts the containers** on it?
- A) The controller manager decides; kubelet starts them
- B) The scheduler decides; kube-proxy starts them
- C) The scheduler decides; the kubelet starts them
- D) The API server decides; containerd starts them alone

**Correct: C.** The scheduler binds the Pod to a node; that node's kubelet then asks the container runtime (containerd, through the CRI) to pull and start the containers. A: controllers create Pods (e.g. a ReplicaSet), but never choose nodes. B: kube-proxy only manages Service network rules. D: the API server only validates and stores; containerd does nothing until the kubelet asks it to.

---

**W3.** A user's umask is `027`. They create a new **file** (not a folder). What permissions does it get?
- A) `750` (rwxr-x---)
- B) `640` (rw-r-----)
- C) `650` (rw-r-x---)
- D) `740` (rwxr-----)

**Correct: B.** New files start from **666** (no execute bits), and the umask removes bits: 666 − 027 → owner 6, group 6 − 2 = 4, others 6 − 7 → 0. So `640` = rw-r-----. A is the answer for a **folder** (folders start from 777). C and D mix up which bits the umask removes.

---

## Check 50

**50.1** One Pod has two containers: `app`, which listens on port 8080, and a sidecar `metrics`, which needs to call it. Which address should the sidecar use?
- A) `app:8080`
- B) `localhost:8080`
- C) `app.default:8080`
- D) `metrics:8080`

**Correct: B.** Containers in one Pod share a single network namespace, so `app`'s port 8080 is on the sidecar's own `localhost`. A is the Docker Compose habit: container names aren't DNS names inside a Pod. C looks like a Service DNS name (Session 07), but no Service called `app` exists here. D is the sidecar's own name, which isn't a hostname either.

---

**50.2** `kubectl get pods` shows `loop   0/1   CrashLoopBackOff   5 (40s ago)   4m`. What is this Pod's **phase**?
- A) `Failed`
- B) `Pending`
- C) `Unknown`
- D) `Running`

**Correct: D.** `CrashLoopBackOff` is a container **reason** shown in the STATUS column, not a phase. The Pod is still bound to its node and its container keeps being restarted, so the phase is `Running`. A: `Failed` needs every container stopped **with no more restarts**, but `restartPolicy: Always` keeps restarting it. B: Pending is before the containers have started at all. C: Unknown means the node can't be contacted.

---

**50.3** A Pod has `restartPolicy: OnFailure`. Its only container finishes with exit code **0**. What happens next?
- A) It restarts, since the container stopped
- B) No restart; the Pod shows `Completed`
- C) It restarts after a 10-second back-off
- D) No restart; the Pod shows `Error`

**Correct: B.** `OnFailure` restarts only on a **non-zero** exit code. Exit 0 = success, so the Pod's phase becomes `Succeeded`, shown as `Completed`. A describes `Always`. C mixes in the back-off idea, which only applies to repeated failures. D: `Error` would mean a non-zero exit code.

---

## Check 51

**51.1** Three Pods: `a` has `app=web,env=prod`; `b` has only `app=web`; `c` has `app=api,env=dev`. Which Pods does `kubectl get pods -l env!=prod` return?
- A) Only `c`
- B) Only `b`
- C) `b` and `c`
- D) `a`, `b` and `c`

**Correct: C.** `!=` matches every object whose `env` isn't `prod`, **including objects with no `env` label at all**, so `b` (no env) and `c` (env=dev) both match. A is the common mistake of thinking `!=` requires the label to exist. B forgets `c`. D wrongly includes `a`, whose env **is** prod.

---

**51.2** A Deployment has `selector.matchLabels: {app: web}`, but its template's labels are `{app: webapp}`. What happens on `kubectl apply`?
- A) Rejected: selector doesn't match template
- B) Created; each Pod gets both labels merged
- C) Created, but it never manages any Pods
- D) Created; the selector updates to `app: webapp`

**Correct: A.** The API server validates this and refuses: `` `selector` does not match template `labels` ``. B: Kubernetes never merges or invents labels. C describes the problem the rule exists to **prevent**: Pods that the controller couldn't find. D: the selector is never changed automatically (and it's immutable after creation anyway). Note: `--dry-run=client` wouldn't catch this; `--dry-run=server` would.

---

**51.3** (one line) Contrast labels and annotations, with one example of each.

**Model answer:** *"Labels are short identifying key–value tags that selectors use to find and group objects, e.g. `app: web`, whereas annotations are non-identifying notes for people and tools that can't be used in selectors, e.g. `kubernetes.io/change-cause: "upgrade to 1.28"`."*

---

## Check 52

**52.1** A ReplicaSet with `replicas: 3` and selector `app: web` has 3 running Pods. You run `kubectl run test --image=busybox:1.36 --labels=app=web -- sleep 3600`. What happens?
- A) 4 Pods run; an RS ignores Pods it didn't make
- B) The API server rejects the clashing label
- C) The RS scales itself up to `replicas: 4`
- D) The RS adopts `test`, then deletes one Pod

**Correct: D.** A ReplicaSet counts **every** ownerless Pod matching its selector. It adopts `test`, counts 4, wants 3, and deletes one (usually the newest, i.e. `test`). The events show `Deleted pod: test`. A is the natural assumption, and exactly the trap. B: labels are never "reserved"; any Pod may carry any label. C: a controller never changes its own desired `replicas`.

---

**52.2** You change a ReplicaSet's template image from `nginx:1.27` to `nginx:1.28` and apply it. What happens to the 3 running Pods?
- A) All 3 restart on 1.28 at the same time
- B) They're replaced one by one with 1.28
- C) They keep 1.27; only new Pods get 1.28
- D) The apply fails: templates are immutable

**Correct: C.** A ReplicaSet only checks **how many** matching Pods exist, never what version they run. The template is only used when it creates a **new** Pod, so you end up with mixed versions. B is what a **Deployment** does (a rolling update). A doesn't happen with any controller. D: a ReplicaSet's template can be changed (it's the selector that's immutable).

---

## Check 53

**53.1** A Deployment has `replicas: 4` and the default strategy (`maxSurge: 25%`, `maxUnavailable: 25%`). During a rolling update, what's the largest number of Pods that can exist, and the smallest number that must stay available?
- A) At most 4 Pods; at least 3 available
- B) At most 5 Pods; at least 4 available
- C) At most 5 Pods; at least 3 available
- D) At most 6 Pods; at least 2 available

**Correct: C.** 25% of 4 = 1. maxSurge → 4 + 1 = **5** Pods at most; maxUnavailable → 4 − 1 = **3** available at least. A forgets the surge. B forgets that 1 Pod may be unavailable. D doubles both numbers (that would be 50%).

---

**53.2 🐞** Using the Deployment from Step 3, `kubectl set image deployment/web web=nginx:1.28` fails with `error: unable to find container named "web"`. What's wrong?
- A) The Deployment must be scaled to 0 first
- B) `set image` only works on bare Pods
- C) The tag `1.28` doesn't exist on Docker Hub
- D) It needs the container name, `nginx`

**Correct: D.** In `set image`, the part before `=` is the **container name** from the template (`- name: nginx`), not the Deployment's name. A: no scaling is needed; set image triggers a normal rolling update. B: it works on Deployments, ReplicaSets, DaemonSets, Pods and more. C: a bad tag would be accepted by kubectl and only fail later, on the node, as `ImagePullBackOff`; the error here is about the container name.

---

**53.3** A Deployment's history shows revisions `1` (nginx:1.27), `2` (nginx:1.28) and `3` (a broken tag, the current one). You run `kubectl rollout undo deployment/web`. What does `kubectl rollout history` show now?
- A) `1, 2, 3, 4`: revision 4 is a copy of 2
- B) `1, 3, 4`: revision 2 became revision 4
- C) `1, 2`: the broken revision 3 is deleted
- D) `1, 2, 3`: the current one is now 2

**Correct: B.** Undo reuses revision 2's ReplicaSet and gives it the **next** number, 4, so 2 disappears from the list. A: a template appears only once in the history, never twice. C: revision 3 stays (its ReplicaSet is kept at 0 replicas, so you could even go back to it). D: the numbering always moves forward; the current revision is the highest number.

---

## Session quiz

**Q1.** A Pod has two containers, both from the `nginx` image (each listens on port 80). One of them keeps restarting, and its logs say `bind() to 0.0.0.0:80 failed (98: Address already in use)`. Why?
- A) Containers in a Pod share one network namespace
- B) A node can run only one port-80 container at once
- C) `containerPort: 80` is missing from one container
- D) A Pod can't run two containers from the same image

**Correct: A.** Both containers live in the same network namespace (one IP, one set of ports), so the second nginx can't bind port 80, which the first already holds. B: different **Pods** have different IPs, so a node can run many port-80 Pods. C: `containerPort` is documentation and has no effect on binding. D: the same image twice is allowed; it's the port that clashes.

---

**Q2.** A ReplicaSet's selector is `matchLabels: {app: web, tier: front}`. You hand-create two Pods: `p1` with labels `app=web, tier=front, env=dev`, and `p2` with just `app=web`. Which Pods does the ReplicaSet count as its own?
- A) Both, since each one has `app=web`
- B) Neither: the extra `env` label breaks it
- C) Only `p1`: it has all selector labels
- D) Only `p2`: it matches exactly one label

**Correct: C.** `matchLabels` is an AND: a Pod must have **every** listed pair. Extra labels don't matter. `p1` has both `app=web` and `tier=front`; `p2` lacks `tier=front`. A treats the selector as OR. B is wrong because extra labels are always allowed. D gets the rule backwards.

---

**Q3.** A Deployment has `replicas: 4` and the default strategy (25% / 25%). Someone sets its image to a tag that doesn't exist. Ten minutes later, what does `kubectl get pods` show?
- A) 4 old Running, 1 new in ImagePullBackOff
- B) 3 old Running, 2 new in ImagePullBackOff
- C) 4 new in ImagePullBackOff, no old Pods left
- D) 4 old Running; K8s already rolled back

**Correct: B.** With 4 replicas, maxSurge = 1 (total ≤ 5) and maxUnavailable = 1 (≥ 3 available). The controller removes **one** old Pod (4 → 3 still keeps 3 available) and creates new Pods up to the total of 5: **3 old + 2 new**. The new ones never become ready, so it stops there. (Verified on minikube: `READY 3/4, UP-TO-DATE 2, AVAILABLE 3`.) A is the answer for `replicas: 3`, where maxUnavailable rounds down to 0. C would need the `Recreate` strategy. D: Kubernetes never rolls back automatically; after 600 s it only marks the rollout as failed.

---

**Q4.** During a sale, you run `kubectl scale deployment web --replicas=6`. The next morning, a teammate changes **only the image** in `web.yaml` (which still says `replicas: 3`) and runs `kubectl apply -f web.yaml`. What happens?
- A) New image rolls out; 6 replicas are kept
- B) apply fails: the replica counts disagree
- C) Replicas return to 3; image is unchanged
- D) New image rolls out; back to 3 replicas

**Correct: D.** `apply` makes the live object match the file, and the file says both the new image **and** `replicas: 3`, so it rolls out the new image and scales back down to 3. A is the common assumption ("they only changed the image"), but the file's `replicas` field still applies. B: there's no conflict check; the file simply wins. C gets the replicas right but ignores the image change, which does start a rollout.

---

**L1.** Contrast a ReplicaSet and a Deployment in one sentence.

**Model answer:** *"A ReplicaSet only keeps a fixed number of identical Pods running (it self-heals and scales, but a template change affects only new Pods), whereas a Deployment manages ReplicaSets for you, so changing its template triggers a controlled rolling update with revision history and `kubectl rollout undo`."*

---

**L2.** Explain `maxSurge` vs `maxUnavailable` in contrast form, with the numbers for `replicas: 4` and the 25% defaults.

**Model answer:** *"`maxSurge` is how many extra Pods above the desired count may exist during a rolling update (rounded up: 25% of 4 = 1, so at most 5 Pods), whereas `maxUnavailable` is how many Pods below the desired count may be unavailable (rounded down: 25% of 4 = 1, so at least 3 must stay available)."*
