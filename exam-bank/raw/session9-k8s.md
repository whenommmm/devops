# Labs · Session 9 · Kubernetes basics (cluster, kubectl, architecture)

## Lab 1 · Connect to a cluster and inspect nodes, namespaces and resources with kubectl
**Answer (what to write in the exam):**
```bash
kubectl config get-contexts               # which clusters can I talk to? (* = current)
kubectl config use-context minikube       # pick the cluster
kubectl cluster-info                      # where the API server and CoreDNS run
kubectl version                           # client + server versions
kubectl get nodes -o wide                 # all machines in the cluster
kubectl describe node minikube            # one node in detail
kubectl get ns                            # all namespaces
kubectl get pods -n kube-system           # the cluster's own system Pods
kubectl get all -A                        # common objects in ALL namespaces
kubectl api-resources                     # every object type the API knows
```
**What each line does (record this in the exam):**
- `kubectl`: the command-line tool (CLI, command-line interface) that sends requests to the Kubernetes API (application programming interface) server.
- `config get-contexts` / `use-context`: a *context* = cluster + user + namespace saved in `~/.kube/config`; this lists them and switches to one.
- `cluster-info`: shows the control-plane (API server) URL, e.g. `https://127.0.0.1:32771`, and the CoreDNS (cluster DNS, Domain Name System) URL. Proves you are connected.
- `version`: client version and server version (they should be within one minor version).
- `get nodes -o wide`: NAME, STATUS (`Ready`), ROLES (`control-plane`), VERSION, plus INTERNAL-IP, OS image, kernel and container runtime (`-o wide` = extra columns).
- `describe node`: labels, capacity (CPU (central processing unit) / memory), conditions (MemoryPressure, DiskPressure, Ready), and the Pods running on that node.
- `get ns`: namespaces = logical folders for objects: `default`, `kube-system` (cluster components), `kube-public`, `kube-node-lease`.
- `get pods -n kube-system`: shows etcd, kube-apiserver, kube-scheduler, kube-controller-manager, kube-proxy, coredns running as Pods (`-n` = namespace).
- `get all -A`: Pods, Services, Deployments, ReplicaSets, DaemonSets etc. in every namespace (`-A` = `--all-namespaces`). It does NOT include ConfigMaps/Secrets.
- `api-resources`: NAME, SHORTNAMES (`po`, `svc`, `deploy`, `ns`), APIVERSION, NAMESPACED (true/false) and KIND for every resource type.

**Verify:** `kubectl get nodes` → node `minikube` with STATUS `Ready`; `kubectl cluster-info` → "Kubernetes control plane is running at https://…".

**🧠 In simple words:** kubectl never touches containers directly; it only talks to the API server of the cluster selected by the current context. These commands answer "which cluster am I on, which machines does it have, which namespaces exist, what is running, and what object types can I create". In short: context → cluster-info → nodes → namespaces → all objects → resource types.

## Lab 2 · Deploy a simple web app and inspect it with get, describe and logs until it is healthy
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=2 --dry-run=client -o yaml > web.yaml
kubectl apply -f web.yaml                     # (or run the create command without --dry-run)
kubectl rollout status deployment/web         # waits until all replicas are ready
kubectl get deployment web                    # READY 2/2
kubectl get pods -l app=web -o wide           # Pod names, STATUS Running, IP, NODE
kubectl describe deployment web               # replicas, image, events
kubectl describe pod <web-pod-name>           # State Running, Ready True, Events
kubectl logs deployment/web                   # nginx startup output
kubectl port-forward deployment/web 8080:80   # then in a 2nd terminal: curl localhost:8080
```
**What each line does:**
- `create deployment … --dry-run=client -o yaml > web.yaml`: generates the Deployment manifest (YAML file) without creating anything; `--replicas=2` = two Pods.
- `apply -f web.yaml`: sends the file to the API server; the Deployment creates a ReplicaSet, which creates 2 Pods with label `app=web`.
- `rollout status`: blocks until "deployment "web" successfully rolled out" = healthy.
- `get … -l app=web -o wide`: `-l` filters by label; `-o wide` adds Pod IP (Internet Protocol address) and NODE.
- `describe`: human-readable details; the **Events** section at the bottom shows Scheduled → Pulling → Pulled → Created → Started (or errors like ImagePullBackOff).
- `logs deployment/web`: prints the container's stdout of one Pod of the Deployment (add `-f` to follow).
- `port-forward`: tunnels your laptop's port 8080 to container port 80 so `curl localhost:8080` returns the nginx welcome page.

**Verify:** `kubectl get pods -l app=web` → 2 Pods, READY `1/1`, STATUS `Running`, RESTARTS `0`; `curl localhost:8080` → "Welcome to nginx!".

**🧠 In simple words:** You describe the app (image + number of copies) and Kubernetes starts and watches the Pods for you. `get` gives the quick status, `describe` explains why something is or is not working (read the Events), and `logs` shows what the app itself printed. In short: apply → rollout status → get (Running, 1/1) → describe (Events) → logs.

## Lab 3 · Draw the cluster architecture and give the role of each component
**Answer (what to write in the exam):**
```
  you: kubectl apply / get   (CLI)
                 |  HTTPS (REST API calls)
                 v
+------------------- CONTROL PLANE (master) --------------------+
|                                                               |
|  kube-scheduler --->  +----------------+  <--- controller-    |
|  (picks a node)       | kube-apiserver |       manager        |
|                       |  (front door)  |  (keeps desired =    |
|                       +----------------+   actual state)      |
|                               |                              |
|                               v                              |
|                        +-------------+                       |
|                        |    etcd     |  (key-value store,    |
|                        +-------------+   all cluster state)  |
+---------------------------------------------------------------+
                 |  (only the API server talks to etcd)
      +----------+-----------+
      v                      v
+-- WORKER NODE 1 ---+  +-- WORKER NODE 2 ---+
| kubelet            |  | kubelet            |
| kube-proxy         |  | kube-proxy         |
| container runtime  |  | container runtime  |
| [Pod] [Pod]        |  | [Pod]              |
+--------------------+  +--------------------+
```
**One line per component:**
- **kubectl**: the CLI the user runs; it turns commands into REST (HTTP-based) calls to the API server. It is a client, not part of the cluster.
- **kube-apiserver**: the single front door of the cluster; authenticates and validates every request and is the ONLY component that reads/writes etcd.
- **etcd**: distributed key-value database that stores the whole cluster state (desired and current); back it up.
- **kube-scheduler**: watches for new Pods with no node and chooses the best node (resources, taints, affinity); it does not start the Pod.
- **kube-controller-manager**: runs control loops (Deployment, ReplicaSet, Node, Job controllers) that keep actual state equal to desired state, e.g. recreate a crashed Pod.
- **kubelet**: agent on every node; receives Pods assigned to its node, tells the container runtime to start them, and reports health back to the API server.
- (extra) **kube-proxy**: on every node, programs network rules so Service IPs route to Pods. **Container runtime** (containerd): actually pulls images and runs containers.

**Flow of `kubectl apply -f web.yaml`:** kubectl → API server (validate) → etcd (save) → controller-manager creates Pods → scheduler assigns each Pod a node → kubelet on that node starts the containers → status written back via API server.

**Verify (minikube):** `kubectl get pods -n kube-system` → etcd-minikube, kube-apiserver-minikube, kube-scheduler-minikube, kube-controller-manager-minikube, kube-proxy-…, coredns-….

**🧠 In simple words:** The control plane is the "brain": the API server receives every request, etcd remembers everything, the scheduler decides where Pods go, and the controller manager keeps fixing differences between what you asked for and what exists. Worker nodes are the "muscle": kubelet runs the Pods there. In short: kubectl → API server ↔ etcd; scheduler + controllers decide; kubelet on each node does the work.
