# Answer key · Session 05 · Kubernetes I: why Kubernetes, architecture, kubectl & manifests

⚠️ Only open after answering.

---

## Warm-up

**W1.** What does this print?
```bash
NAME="web app"
echo '$NAME is "ready"'
```
- A) `web app is "ready"`
- B) `$NAME is "ready"`
- C) `$NAME is ready`
- D) `web app is ready`

**Correct: B.** The **outer** quotes are single quotes, so everything inside is literal: `$NAME` isn't expanded, and the inner double quotes are just ordinary characters that get printed. A would need outer double quotes (`"$NAME is \"ready\""`). C drops the inner double quotes, but inside single quotes they're ordinary characters, so they're printed. D both expands `$NAME` and drops the quotes, which would need no single quotes at all.

---

**W2.** A compose file has two services, `api` and `db` (the `db` service uses the image `postgres:16`). The api is configured with `DB_HOST=postgres` and crashes with `could not translate host name "postgres"`. What's the cause?
- A) The DNS name to use is the service name, `db`
- B) `db` needs a `ports:` entry to get a name
- C) depends_on is missing, so DNS isn't ready
- D) Compose puts the services on the default bridge

**Correct: A.** On a Compose network, Docker's DNS resolves **service names** (`db`), not image names (`postgres`). Fix: `DB_HOST=db`. B: `ports:` only publishes to the host and has nothing to do with names. C: `depends_on` only controls start order; even with it, the name `postgres` would never exist. D is false: Compose creates its own user-defined network for the project, which is exactly why service-name DNS works.

---

**W3.** What does this print, and does anything else happen?
```bash
x=5
if [ $x > 10 ]; then echo big; else echo small; fi
```
- A) `small`, because 5 is less than 10
- B) `big`, and a file named `10` appears
- C) An error: `>` can't be used in `[ ]`
- D) `small`, and a file named `10` appears

**Correct: B.** Inside `[ ]`, `>` is **redirection**, not "greater than". The shell runs `[ 5 ]` with its output sent to a new file called `10`. `[ 5 ]` tests "is this string non-empty?", which is true, so it prints `big`. A is what people expect, but the comparison never happens. C: there's no error, which is what makes this bug sneaky. D gets the file right but the result wrong. The correct test is `[ "$x" -gt 10 ]`.

---

## Check 46

**46.1** A team runs its app with Docker Compose on a single server, with `restart: unless-stopped` on every service. Which problem would moving to Kubernetes solve that their current setup can't?
- A) Restarting a crashed container on the same server
- B) Moving the app to another server when it dies
- C) Letting one service find another by its name
- D) Keeping db data after a container is removed

**Correct: B.** Compose works on **one machine**; if that machine dies, nothing is left to restart anything. Kubernetes reschedules the work onto other nodes. A is already handled by the restart policy. C is already handled by Compose's service-name DNS. D is handled by Docker volumes.

---

**46.2** Which statement describes Kubernetes' **desired state** idea?
- A) You list the steps; K8s runs them once in order
- B) You describe the result; K8s checks it once
- C) You describe the result; K8s keeps it true
- D) You list the steps; K8s reruns them each minute

**Correct: C.** You declare the end result, and controllers compare reality with it **continuously** and fix any difference (the thermostat idea). A and D describe imperative step lists, not desired state. B is half right: the "describe the result" part is correct, but checking only once would mean no self-healing.

---

**46.3** (one line) Give the full form of **CNCF**, and explain where the names "Kubernetes" and "K8s" come from.

**Model answer:** *"CNCF is the Cloud Native Computing Foundation, the neutral body that looks after Kubernetes; 'Kubernetes' is Greek for 'helmsman' (the person who steers a ship), and 'K8s' is a numeronym: K, then the 8 letters 'ubernete', then s."*

---

## Check 47

**47.1** Which component is the **only** one that reads and writes etcd directly?
- A) kube-apiserver
- B) kube-scheduler
- C) kubelet on each node
- D) controller manager

**Correct: A.** Every other component reads and writes cluster state **through** the API server. B: the scheduler writes its node choice via the API server. C: kubelets report status via the API server. D: controllers create and delete objects via the API server.

---

**47.2** A new Pod has been `Pending` for 10 minutes. `kubectl describe pod` shows the event `FailedScheduling: 0/1 nodes are available: 1 Insufficient memory`. Which statement best describes the cause?
- A) The kubelet couldn't pull the image for the container
- B) The controller manager hasn't created the Pod yet
- C) The scheduler found no node with enough free memory
- D) etcd is full, so the Pod's record couldn't be saved

**Correct: C.** `FailedScheduling` comes from the **scheduler**: during filtering, no node had enough free memory for what the Pod asked for, so it can't be placed and stays Pending. A: an image problem would show `ErrImagePull`/`ImagePullBackOff`, and only **after** a node was chosen. B: the Pod clearly exists, since you can describe it. D: if it couldn't be saved, `kubectl apply` would have failed and there'd be no Pod to describe.

---

**47.3** (one line) Name the three components that run on every worker node, with one job each.

**Model answer:** *"The kubelet makes the Pods assigned to its node run and reports their status to the API server; the container runtime (e.g. containerd, reached through the CRI) actually pulls images and runs the containers; and kube-proxy writes the network rules that send Service traffic to the right Pods."*

---

## Check 48

**48.1** You run `kubectl apply -f pod.yaml` and see `pod/web created`. At that exact moment, which is **guaranteed** to be true?
- A) The container has started on a node and is serving
- B) The scheduler has already picked a node for the Pod
- C) The image has been pulled to the node's local cache
- D) The Pod's record is stored, maybe not yet running

**Correct: D.** `created` means the API server **validated and stored** the object in etcd. Scheduling, pulling and starting all happen **afterwards**, done by other components watching the API server. A, B and C may all become true a second later, but none is guaranteed at the moment the message is printed.

---

**48.2** Which is the right order for a new Pod's journey?
- A) API server → scheduler → etcd → runtime → kubelet
- B) kubelet → API server → etcd → scheduler → runtime
- C) API server → etcd → scheduler → kubelet → runtime
- D) API server → kubelet → scheduler → etcd → runtime

**Correct: C.** The API server stores the Pod in etcd; the scheduler sees an unscheduled Pod and binds it to a node; that node's kubelet sees it and asks the runtime to start the containers. A: the Pod must be stored **before** the scheduler can see it, and the kubelet calls the runtime, not the other way round. B: the kubelet acts last, not first. D: the kubelet can't act before the scheduler has chosen its node.

---

**48.3** (one line) A worker node loses power. The Pods of a Deployment come back on another node, but a Pod you created with `kubectl run` doesn't. Why?

**Model answer:** *"The Deployment's Pods are owned by a ReplicaSet whose controller notices it has fewer Pods than desired and creates replacements on healthy nodes, whereas a bare Pod from `kubectl run` has no controller watching it, so nothing recreates it."*

---

## Check 49

**49.1 🐞** `kubectl apply -f app.yaml` fails with `no matches for kind "Deployment" in version "v1"`. What's the fix?
- A) Set `apiVersion: apps/v1` in the file
- B) Write `kind: deployment` in lowercase
- C) Create the namespace before applying
- D) Use `kubectl create` instead of apply

**Correct: A.** Deployments live in the `apps` API group, so the apiVersion must be `apps/v1`; `v1` is the core group (Pods, Services…). B: `kind` is case-sensitive and must be `Deployment`, so lowercase breaks it further. C: a missing namespace gives `namespaces "x" not found`, a different error. D: `create` uses the same apiVersion lookup and fails the same way.

---

**49.2** `kubectl get pods` prints `No resources found in default namespace.`, but a teammate says the Pod `api` is running in the namespace `shop`. Which command shows it?
- A) `kubectl get pods --all`
- B) `kubectl get pods -l shop`
- C) `kubectl get pods -n shop`
- D) `kubectl get pods shop/api`

**Correct: C.** `-n shop` looks in the `shop` namespace. A: fails with `unknown flag: --all`; the real flag is `-A` / `--all-namespaces`. B: `-l shop` is a **label** selector ("Pods that have a label called shop"), which has nothing to do with namespaces. D: `shop/api` isn't how you name a namespace; kubectl reads it as `type/name` and fails with `there is no need to specify a resource type as a separate argument…`.

---

**49.3** (one line) Explain imperative vs declarative in one contrast sentence, with one kubectl command for each.

**Model answer:** *"Imperative management tells Kubernetes each action to take, e.g. `kubectl run web --image=nginx:1.27` or `kubectl scale deployment web --replicas=5`, whereas declarative management describes the desired end state in a file and lets Kubernetes work out the changes, e.g. `kubectl apply -f web.yaml`."*

---

## Session quiz

**Q1.** In a 3-node cluster, the control plane becomes unreachable for 10 minutes. During that time, what happens to the Pods **already running** on the worker nodes?
- A) They stop, since the kubelets lose their orders
- B) They keep running; lost ones move to other nodes
- C) They keep running, but nothing new is scheduled
- D) They move to whichever worker node is least busy

**Correct: C.** Running containers are managed locally by each kubelet and containerd, so they carry on. But scheduling, scaling, updates and replacing Pods all need the API server, scheduler and controllers. A: kubelets keep running what they already have; they don't stop containers when the API server is unreachable. B is half right: Pods keep running, but moving lost Pods to other nodes needs the scheduler and controllers, which are down. D: nothing moves without the control plane.

---

**Q2.** Which of these is **NOT** stored in etcd?
- A) The replica count you set for a Deployment
- B) The current status and IP of every Pod
- C) The contents of Secrets and ConfigMaps
- D) The rows in your app's PostgreSQL tables

**Correct: D.** etcd stores Kubernetes **objects** (desired state and status), not your application's data. Database rows live on the database's own storage (a volume, Session 09). A is part of a Deployment's spec, B is part of every Pod's status, and C are Kubernetes objects too, so all three are in etcd.

---

**Q3.** A new Pod stays `Pending`. `kubectl describe pod` shows `Node: <none>`, and its Events list is **completely empty**: not even a `FailedScheduling` warning. Older Pods are running fine, and kubectl answers normally. Which component is most likely not working?
- A) The kubelet on the node
- B) The kube-scheduler
- C) The etcd database
- D) The kube-apiserver

**Correct: B.** `Node: <none>` with no `Scheduled` **and** no `FailedScheduling` event means nobody even tried to place the Pod. Deciding a Pod's node is the scheduler's only job. A: a broken kubelet only matters **after** scheduling; you'd see `Node: minikube` and a `Scheduled` event. C and D: if etcd or the API server were down, kubectl couldn't have created or described the Pod at all.

---

**Q4.** You applied `pod.yaml` earlier (`pod/web created`). Now you change only `containerPort: 80` to `containerPort: 8080`, and run `kubectl apply -f pod.yaml` again. What happens?
- A) An error: most Pod fields can't change in place
- B) `pod/web configured`; the container restarts on 8080
- C) `pod/web unchanged`, as containerPort is only a note
- D) The old Pod is deleted, and a new one replaces it

**Correct: A.** A running Pod's spec is mostly **immutable**. The API server rejects it with `pod updates may not change fields other than spec.containers[*].image, …`. B is what happens if you change the **image** instead. C is tempting because containerPort is indeed only documentation, but the field still counts as a change, and it's rejected rather than ignored. D is what a **Deployment** would do (a new Pod from a new template); a bare Pod is never replaced automatically.

---

**L1.** Contrast the control plane and the worker nodes in one sentence, naming two components of each.

**Model answer:** *"The control plane is the cluster's brain that stores state and makes decisions, using components such as the kube-apiserver, etcd, kube-scheduler and kube-controller-manager, whereas worker nodes run the actual application containers, using the kubelet, kube-proxy and a container runtime such as containerd."*

---

**L2.** Explain `spec` vs `status` in one contrast sentence, and say who writes each.

**Model answer:** *"`spec` is the desired state that the user writes in the manifest, whereas `status` is the actual, observed state that Kubernetes itself writes (for example the kubelet reporting the phase and Pod IP); controllers keep working to make status match spec."*
