# Session 05 · Kubernetes I: why Kubernetes, architecture, kubectl & manifests (Oct 13, Session 1)

**Time:** ~100 min · **Concepts:** 46 → 49 · **Answers:** `MY_ANSWERS.md` or say "Kubernetes I questions" · **Answer key:** `05_kubernetes-fundamentals_ANSWERS.md`

**Before you start:** run three checks, in this order (each one needs the one before it):
```bash
colima status        # Docker's hidden Linux VM (minikube lives inside it)
minikube status      # your Kubernetes cluster
kubectl get nodes    # can kubectl talk to the cluster?
```
**You should see** this for `minikube status`:
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```
and `minikube   Ready   control-plane   …` for `kubectl get nodes`. If minikube says `Stopped`, run `minikube start` and wait 1–2 minutes. (You'll learn what "kubelet", "apiserver" and "control plane" mean in Concept 47.)

> ⚠️ **Never delete the `minikube` or `localstack` containers.** Your cluster also already contains things set up for later sessions (namespaces called `argocd`, `ingress-nginx`, `final-app`, and maybe others). **Leave them alone.** In this session you only create and delete things in the `default` namespace, using the exact names given.

---

## 🔁 Warm-up

**W1.** What does this print?
```bash
NAME="web app"
echo '$NAME is "ready"'
```
- A) `web app is "ready"`
- B) `$NAME is "ready"`
- C) `$NAME is ready`
- D) `web app is ready`

**W2.** A compose file has two services, `api` and `db` (the `db` service uses the image `postgres:16`). The api is configured with `DB_HOST=postgres` and crashes with `could not translate host name "postgres"`. What's the cause?
- A) The DNS name to use is the service name, `db`
- B) `db` needs a `ports:` entry to get a name
- C) depends_on is missing, so DNS isn't ready
- D) Compose puts the services on the default bridge

**W3.** What does this print, and does anything else happen?
```bash
x=5
if [ $x > 10 ]; then echo big; else echo small; fi
```
- A) `small`, because 5 is less than 10
- B) `big`, and a file named `10` appears
- C) An error: `>` can't be used in `[ ]`
- D) `small`, and a file named `10` appears

---

## Concept 46 · Why Kubernetes: from one machine to many

**Words you'll learn:** orchestration, container orchestrator, Kubernetes, K8s, numeronym, open source, Borg, CNCF, cloud native, scheduling, self-healing, scaling, rolling update, rollback, service discovery, load balancing, desired state, declarative / imperative (preview), OCI, Docker Swarm, Nomad.

### Step 1: the problem, as a story
Remember the visit-counter app from Session 04 (web + Redis, started with `docker compose up`)? Imagine it becomes popular: 50,000 visitors a day. Riya, who runs it, now has five headaches:

1. **The machine dies at 3 a.m.** Compose runs everything on **one machine**. If that machine crashes, the whole app is down until Riya wakes up. (Docker's restart policies from Session 04 only help while the machine itself is alive.)
2. **One machine isn't enough.** She needs 10 copies of `web`, spread across 5 servers. Which server has free memory for copy number 7? Someone has to decide by hand, every time.
3. **Updating means downtime.** `docker compose up` with a new image replaces the containers, so for a moment nothing answers. And if the new version is broken, **every** copy is broken.
4. **Containers move, and their IPs change.** If copy 3 is recreated on another server, how do the other parts of the app find it? And who spreads visitors evenly over the 10 copies?
5. **Settings and passwords** must be copied to 5 servers and kept the same everywhere.

Doing all of that by hand, 24 hours a day, is impossible. She needs software that does it **automatically**.

### Step 2: the idea: orchestration
To **orchestrate** means **to arrange and coordinate many parts so they work together**.

> A **container orchestrator** is **software that automatically places, runs, connects, scales and heals containers across a group of machines.**

**Analogy: an airport control tower.** Pilots (containers) know how to fly their own planes. But the **tower** decides which runway (machine) each plane uses, keeps track of every plane at all times, spaces them out so no runway is overloaded, and sends planes elsewhere when a runway closes. Nobody phones the tower to say "plane 7 is missing": the tower notices by itself.
(You'll also hear "an orchestra conductor": the musicians play, and the conductor coordinates them.)

### Step 3: Kubernetes
**Kubernetes** is by far the most widely used container orchestrator.

- **The name:** Greek for **"helmsman"**, the person who steers a ship. It fits the shipping-container idea from Session 02, and the logo is a ship's wheel.
- **K8s:** a short way to write it: **K**, then the **8** letters "ubernete", then **s**. This kind of abbreviation is called a **numeronym** (like "i18n" for "internationalization"). People still say "Kubernetes" (or "kates").
- **Where it came from:** Google ran its own services in containers for years, on an internal system called **Borg**. In **2014** Google released Kubernetes, built on Borg's lessons, as **open source** (its code is public, and anyone may use and improve it for free).
- **Who looks after it now:** in 2015 Google handed it to the **CNCF**: the **Cloud Native Computing Foundation**, a neutral organisation (part of the Linux Foundation) that looks after open-source cloud projects. So no single company controls Kubernetes. **Cloud native** means **apps designed to run as containers on clusters, so they scale and recover easily**. (Prometheus, Helm and Argo CD, which you'll meet later, are CNCF projects too.)
- **Your Mac:** you run Kubernetes **v1.37** through **minikube**, a tool that creates a small Kubernetes on one computer, for learning.

### Step 4: the big idea: desired state
With Docker, you give **instructions**: "start this container", "stop that one". With Kubernetes, you mostly describe the **result you want**:

> "There should always be **3 copies** of `web`, running the image `nginx:1.27`."

That description is called the **desired state**. Kubernetes stores it, then **keeps comparing reality with it, forever**, and fixes any difference. A copy crashed? Start a new one. A machine died? Start its copies somewhere else. You changed 3 to 5? Start 2 more.

**Analogy: a thermostat.** You don't tell the heater "run for 12 minutes". You set **22 °C** (the desired state). The thermostat keeps measuring the room (the actual state) and switches the heater on or off to close the gap, all day, without you.

Describing the result instead of the steps is called **declarative**. Giving step-by-step commands is called **imperative**. (Full details in Concept 49.)

### Step 5: what Kubernetes gives you
| Feature | Plain meaning | Fixes headache | Session |
|---|---|---|---|
| **Scheduling** | Automatically choosing **which machine** runs each container, based on free CPU and memory | #2 | today (Concept 47) |
| **Self-healing** | Replacing crashed containers, and moving work off dead machines, automatically | #1 | 06 |
| **Scaling** | Changing the **number of copies** with one command (or automatically, based on load) | #2 | 06, 09 |
| **Rolling updates & rollbacks** | Replacing copies **a few at a time** so the app never goes down, and going back to the old version if the new one is broken | #3 | 06 |
| **Service discovery & load balancing** | A stable name and address for a group of copies, with visitors spread across them | #4 | 07 |
| **Config & secrets** | Settings and passwords stored once in the cluster, and handed to containers | #5 | 08 |
| **Storage orchestration** | Attaching disks to containers wherever they run | (databases) | 09 |

**Service discovery** = **how one part of an app finds another part** (by a name, not an IP). **Load balancing** = **spreading requests over several copies** (from the networking session).

### Step 6: Compose vs Kubernetes
| | **Docker Compose** | **Kubernetes** |
|---|---|---|
| Runs on | **One** machine | A **cluster** of many machines (Concept 47) |
| A machine dies | The app is down | Its containers are started on healthy machines |
| A container crashes | Restarted only if you set a restart policy | Always replaced (it's part of the desired state) |
| More copies | `--scale web=3`, on that one machine | `replicas: 3`, spread over machines; can autoscale |
| New version | Containers replaced; brief downtime | Rolling update with no downtime, plus rollback |
| Finding each other | Service names on one Compose network | **Services** + cluster DNS (Session 07) |
| Smallest unit | A container (a "service") | A **Pod** (Concept 47) |
| Config files | One `compose.yaml` | Many YAML **manifests** (Concept 49) |
| Best for | Development, small single-server apps | Production apps that must stay up and grow |

> ⚠️ **Exam traps.**
> - **Kubernetes doesn't build images.** You still build them with Docker (Session 03) and push them to a registry. Kubernetes **pulls and runs** them.
> - **Kubernetes doesn't need the Docker engine.** It uses a **container runtime** such as **containerd** (Concept 47). Images built with Docker still work, because all these tools follow the same image standard, **OCI** (**Open Container Initiative**).

### Step 7: other orchestrators (awareness only)
- **Docker Swarm:** Docker's own built-in orchestrator (`docker swarm init`). Simpler, but much less used today.
- **Nomad** (by HashiCorp, the makers of Terraform): orchestrates containers **and** ordinary programs.
- **Red Hat OpenShift:** not a competitor, but a company's packaged version of Kubernetes with extra tools on top.

### 🛠 Do (~3 min)
```bash
kubectl version
```
- `kubectl` = the command-line tool for talking to Kubernetes (Concept 49). `version` = print version information.

**You should see:**
```
Client Version: v1.37.1          ← kubectl itself, on your Mac
Kustomize Version: v5.8.1        ← a helper built into kubectl (ignore it)
Server Version: v1.37.0          ← the Kubernetes "server" inside minikube
```
Two parts, **client** and **server**, exactly like `docker version` showed Client and Server in Session 02. The client asks; the cluster does the work.

```bash
docker ps --format '{{.Names}}'
```
- `--format '{{.Names}}'` = print only the NAMES column. The single quotes stop the shell touching the braces.

*You'll see `minikube` (and `localstack`). Your whole Kubernetes cluster runs **inside one Docker container** called `minikube`, inside Colima's Linux VM. Leave it alone.*

### ✅ Check 46

**46.1** A team runs its app with Docker Compose on a single server, with `restart: unless-stopped` on every service. Which problem would moving to Kubernetes solve that their current setup can't?
- A) Restarting a crashed container on the same server
- B) Moving the app to another server when it dies
- C) Letting one service find another by its name
- D) Keeping db data after a container is removed

**46.2** Which statement describes Kubernetes' **desired state** idea?
- A) You list the steps; K8s runs them once in order
- B) You describe the result; K8s checks it once
- C) You describe the result; K8s keeps it true
- D) You list the steps; K8s reruns them each minute

**46.3** (one line) Give the full form of **CNCF**, and explain where the names "Kubernetes" and "K8s" come from.

---

## Concept 47 · Kubernetes architecture: the control plane and the worker nodes

**Words you'll learn:** node, cluster, control plane, worker node, master (old name), component, kube-apiserver, authentication, authorization, admission, etcd, key-value store, kube-scheduler, kube-controller-manager, controller, cloud-controller-manager, agent, kubelet, heartbeat, container runtime, containerd, CRI, kube-proxy, Pod (preview), add-on, CoreDNS, CNI, high availability.

### Step 1: nodes and clusters
- A **node** is **one machine in Kubernetes**: a real server, or a VM (Session 02).
- A **cluster** is **a group of nodes managed together as one system**. You give work to the cluster, not to a particular machine.

Every node has one of two jobs:
1. **Control plane** = **the brain**. Its programs **make the decisions** (where things run, what needs fixing) and **store the cluster's state**. Older documents call it the **master** node.
2. **Worker nodes** = **the muscles**. They **run your apps' containers**.

Each job is done by several small programs called **components**. Let's meet them inside a **factory**:

```
FACTORY                                   KUBERNETES
Head office                          =    Control plane
  Reception desk (every request)     =      kube-apiserver
  Records room (the master files)    =      etcd
  Planner (assigns jobs to lines)    =      kube-scheduler
  Floor supervisors (checklists)     =      kube-controller-manager
  Landlord liaison                   =      cloud-controller-manager
Production line                      =    Worker node
  Line foreman                       =      kubelet
  The machines on the line           =      container runtime (containerd)
  Mail-routing desk                  =      kube-proxy
```

### Step 2: the control plane, component by component

**1. kube-apiserver: the front door**
The **API server** is the **only entrance** to the cluster. (API = Application Programming Interface, from Session 02. Like Docker's, it's a **REST API**, spoken over **HTTPS**.)
- **Everyone talks to it:** you (through `kubectl`), and every other component too. The scheduler, the controllers and the kubelets **never talk to each other directly**. They all read and write through the API server.
- For every request it checks, in order:
  1. **Authentication** = *who are you?* (kubectl shows a certificate, like an ID card)
  2. **Authorization** = *are you allowed to do this?* (permissions; you'll hear **RBAC**, **Role-Based Access Control**, later)
  3. **Admission** = *does this follow the cluster's rules?* (it also fills in default values you left out)
  4. **Validation** = *is the object written correctly?*

  …and only then saves the result in etcd.
- ⚠️ **It's the only component that reads and writes etcd.**
- **Analogy:** the factory's **reception desk**. Every order, from customers and staff alike, goes through it. It checks your ID and paperwork, then files the order in the records room. Nobody walks into the records room themselves.

**2. etcd: the cluster's memory**
- Pronounced **"et-cee-dee"**: the Linux **`/etc`** folder (where configuration lives, from the Linux session) + **d** for **distributed** (spread over several machines).
- It's a **key-value store**: a database that saves data as **key → value** pairs, like a dictionary, or like YAML's `key: value`. For example, the key `/registry/pods/default/web` holds the full description of the Pod called `web`.
- It holds **the whole state of the cluster**: every object you created (Pods, Deployments, Services, Secrets…), what you **want** (desired state), and what Kubernetes **observed** (actual state).
- **If etcd's data is lost, the cluster forgets everything** it was supposed to be running. That's why teams **back it up** regularly (with a command called `etcdctl snapshot save`).
- **Analogy:** the **records room**, holding the one true copy of every order. Burn it down, and the factory no longer knows what it was making.
- ⚠️ **Trap:** etcd stores Kubernetes' own records, **not your app's data**. Your database's rows go on volumes (Session 09).

**3. kube-scheduler: picks the node**
- It **watches for new Pods that don't have a node yet** (a Pod is the box Kubernetes runs your containers in; Step 4).
- For each one, it chooses a node in two rounds:
  1. **Filtering:** which nodes *can* run it? (enough free CPU and memory, plus any rules the Pod asks for)
  2. **Scoring:** of those, which one is *best*? (for example, the least busy, or one that spreads copies apart)
- It then **writes the chosen node's name into the Pod** (through the API server). That's all it does.
- ⚠️ **It doesn't start containers.** It only decides *where*. If no node fits, the Pod stays **`Pending`**, with an event like `FailedScheduling: 0/3 nodes are available: 3 Insufficient memory`.
- **Analogy:** the **planner** who writes "Line 2" on a job card. The planner never touches a machine.

**4. kube-controller-manager: runs the controllers**
- A **controller** is **a program that runs in an endless loop: watch one kind of object, compare what exists with what's wanted, and fix the difference.** That loop is called a **reconciliation loop** (Concept 48).
- Kubernetes has dozens of controllers, each with one job. Some examples:
  - **ReplicaSet controller:** "is the right number of copies running?" (Session 06)
  - **Deployment controller:** rolls out new versions (Session 06)
  - **Node controller:** "has every node checked in recently?" If one hasn't, it marks it `NotReady`
  - **Job controller:** runs one-off tasks until they complete
  - **EndpointSlice controller:** keeps track of which Pods sit behind each Service (Session 07)
- To keep things simple, all these controllers run inside **one program**: the controller **manager**.
- Controllers don't start containers either. They **create, change or delete objects** through the API server, and other components act on that.
- **Analogy:** **floor supervisors**, each with a checklist ("3 of these must be running"), walking the floor all day and raising new orders whenever something's off.

**5. cloud-controller-manager (short)**
It only exists in clusters running on a cloud (AWS, Google Cloud, Azure). It talks to the **cloud provider's API**: it creates cloud load balancers, attaches cloud disks, and notices when a cloud VM has been deleted. **minikube doesn't have one.**
**Analogy:** the **liaison with the landlord**, who arranges extra space and utilities for the factory.

### Step 3: the worker node, component by component

**1. kubelet: the node's agent**
An **agent** is **a small program that runs on each machine and acts on behalf of a central system**.
- The **kubelet** runs on **every node**, as a **systemd** service (from the Linux session: a daemon, not a container).
- It **watches the API server for Pods assigned to its node**, then makes them happen:
  - tells the container runtime to **pull the image** and **start the containers**
  - **restarts containers** that crash (Session 06)
  - runs **health checks** on them (probes, Session 09)
  - regularly **reports back** the status of its Pods and of the node itself. This regular "I'm alive" message is called a **heartbeat**
- It only manages containers that **Kubernetes** asked for. A container you start by hand with `docker run` is ignored.
- **Analogy:** the **line foreman**, who reads the job cards for their line, gets the machines going, and reports progress to head office.

**2. Container runtime (containerd)**
- The **container runtime** is **the software that actually runs containers**: it pulls images, sets up the namespaces and cgroups (Session 02), and starts the processes.
- Common ones are **containerd** (say "container-dee"; you met it in Session 02 as the program the Docker daemon uses inside) and **CRI-O**. Your minikube uses **containerd**.
- **CRI** = **Container Runtime Interface**: **a standard "plug" between the kubelet and any runtime**. Any runtime with the CRI plug works, the way any appliance with the right plug works in a wall socket.
- ⚠️ Since Kubernetes v1.24, the kubelet **can't use the Docker engine directly** (the adapter that allowed it, called "dockershim", was removed). Docker-built images still run fine (the OCI standard, Concept 46).
- **Analogy:** the **machines on the production line**, which do the physical work.

**3. kube-proxy: the traffic rules**
- It runs on **every node**.
- It writes **network rules into the node's Linux kernel** (using kernel features called **iptables** or **IPVS**), so that traffic sent to a **Service** (a stable address for a group of Pods, Session 07) is forwarded to one of the Pods behind it, on any node.
- It only sets up the rules. The kernel then does the actual forwarding.
- **Analogy:** the **mail-routing desk**, keeping an up-to-date table like "letters for the Packing Team go to these three desks".

### Step 4: Pods (preview)
Kubernetes never runs a bare container. It always wraps containers in a **Pod**:

> A **Pod** is **the smallest thing Kubernetes runs: a wrapper around one container (sometimes a few) that share one IP address.**

**Analogy:** a container is a person, and a Pod is the flat they live in. The flat has **one postal address** (the Pod's IP), and flatmates share it. Full details in Session 06.

### Step 5: the whole picture
```
                        YOU:  kubectl  (on your Mac)
                                 │  HTTPS (REST API)
                                 ▼
┌─────────────────────── CONTROL PLANE (the brain) ───────────────────────┐
│                                                                         │
│   ┌──────────────────┐   reads / writes   ┌──────────────────┐          │
│   │  kube-apiserver  │◄──────────────────►│       etcd       │          │
│   │   (front door)   │   (ONLY the API    │   (the memory)   │          │
│   └────────▲─────────┘    server does)    └──────────────────┘          │
│            │ watch & update                                             │
│   ┌────────┴─────────┐  ┌─────────────────────────┐  ┌───────────────┐  │
│   │  kube-scheduler  │  │ kube-controller-manager │  │ cloud-control-│  │
│   │  (picks a node)  │  │ (runs the controllers)  │  │ ler-manager   │  │
│   └──────────────────┘  └─────────────────────────┘  │ (cloud only)  │  │
│                                                      └───────────────┘  │
└────────────────────────────────▲────────────────────────────────────────┘
                                 │  kubelets watch for work and report status
          ┌──────────────────────┼────────────────────────┐
          ▼                      ▼                        ▼
┌─── WORKER NODE 1 ────┐ ┌─── WORKER NODE 2 ────┐ ┌─── WORKER NODE 3 ────┐
│ kubelet    (foreman) │ │ kubelet              │ │ kubelet              │
│ kube-proxy (rules)   │ │ kube-proxy           │ │ kube-proxy           │
│ containerd (runtime) │ │ containerd           │ │ containerd           │
│ [Pod] [Pod] [Pod]    │ │ [Pod] [Pod]          │ │ [Pod]                │
└──────────────────────┘ └──────────────────────┘ └──────────────────────┘
```

| Component | Runs on | Its one job | Analogy | Exam fact |
|---|---|---|---|---|
| **kube-apiserver** | Control plane | Front door for every request; checks and stores | Reception desk | The only component that talks to etcd |
| **etcd** | Control plane | Key-value store of the whole cluster state | Records room | Back it up; it's not for app data |
| **kube-scheduler** | Control plane | Chooses a node for each new Pod | Planner | Doesn't start containers; no fit → `Pending` |
| **kube-controller-manager** | Control plane | Runs the reconciliation loops (controllers) | Floor supervisors | Many controllers in one program |
| **cloud-controller-manager** | Control plane (cloud only) | Talks to the cloud provider | Landlord liaison | Not on minikube |
| **kubelet** | Every node | Makes assigned Pods run; reports status | Line foreman | A systemd service, not a Pod |
| **kube-proxy** | Every node | Network rules for Services | Mail-routing desk | Uses iptables or IPVS |
| **container runtime** | Every node | Actually runs the containers | The machines | containerd, reached through the CRI |

**Add-ons** are extra programs that run **as Pods** to give the cluster more features. The ones on your minikube:
- **CoreDNS:** the cluster's own DNS server (Session 07).
- **kindnet:** a **CNI** (**Container Network Interface**) plugin. It gives every Pod its IP address and connects Pods together.
- **metrics-server:** collects CPU and memory numbers (Session 09).
- **storage-provisioner:** creates storage when an app asks for it (Session 09).

### Step 6: your minikube is a one-node cluster
In production, a cluster usually has **3 control-plane nodes**, for **high availability** (the cluster keeps working even if one part fails), plus **many worker nodes**. Your minikube has **one node, called `minikube`, which is both the control plane and the worker**. And that "machine" is really **a Docker container** inside Colima, pretending to be a server. Same components, just squeezed together.

### 🛠 Do (~5 min)
```bash
kubectl get nodes -o wide
```
- `get nodes` = list the cluster's nodes. `-o wide` = **o**utput in a **wide** format, with extra columns.

**You should see** (one long line, shortened here):
```
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    …   OS-IMAGE                         …   CONTAINER-RUNTIME
minikube   Ready    control-plane   5d    v1.37.0   192.168.49.2   …   Debian GNU/Linux 12 (bookworm)   …   containerd://2.3.4
```
- **STATUS `Ready`**: the kubelet on this node is healthy and sending heartbeats.
- **ROLES `control-plane`**: this node runs the brain (and, on minikube, your apps too).
- **VERSION**: the kubelet's version.
- **INTERNAL-IP `192.168.49.2`**: a **private** address (the `192.168.x.x` range from the networking session), only reachable inside Colima.
- **CONTAINER-RUNTIME `containerd://2.3.4`**: the runtime the kubelet uses, through the CRI.

```bash
kubectl cluster-info
```
**You should see:**
```
Kubernetes control plane is running at https://127.0.0.1:32771
CoreDNS is running at https://127.0.0.1:32771/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```
*That first URL is the **API server's address**. `127.0.0.1` is your Mac's loopback (networking session): Docker publishes the API server's port (8443, inside the minikube container) on your Mac's port 32771. Every kubectl command you type goes there, over HTTPS. (Your port number may be different.)*

### ✅ Check 47

**47.1** Which component is the **only** one that reads and writes etcd directly?
- A) kube-apiserver
- B) kube-scheduler
- C) kubelet on each node
- D) controller manager

**47.2** A new Pod has been `Pending` for 10 minutes. `kubectl describe pod` shows the event `FailedScheduling: 0/1 nodes are available: 1 Insufficient memory`. Which statement best describes the cause?
- A) The kubelet couldn't pull the image for the container
- B) The controller manager hasn't created the Pod yet
- C) The scheduler found no node with enough free memory
- D) etcd is full, so the Pod's record couldn't be saved

**47.3** (one line) Name the three components that run on every worker node, with one job each.

---

## Concept 48 · What happens when you run `kubectl apply`: desired state and the reconciliation loop

**Words you'll learn:** watch, event, binding, `spec` (desired state), `status` (actual state), reconciliation loop (control loop), controller chain, heartbeat (recap), Lease, `NotReady`, eviction, bare Pod.

### Step 1: the mystery
You type one command, and 2 seconds later nginx is running on a node. No single component did all that. **Each component did one small job, then wrote the result into the API server, where the next component was waiting for it.**

Two words first:
- To **watch** = to **subscribe to changes** in the API server, so you're told the moment something changes. **Analogy:** parcel-tracking notifications on your phone. You don't keep refreshing the page; the update comes to you.
- An **event** = a short **log entry about something that happened to an object** ("Scheduled", "Pulling image"…). You read them with `kubectl describe` and `kubectl get events`.

**Analogy: a noticeboard in the factory.** Nobody phones anybody. Reception pins up the order; the planner sees it and writes a line number on it; the foreman of that line sees it and gets to work; then the foreman writes "running" on it.

### Step 2: the journey, step by step
```
 1  you ── kubectl apply -f pod.yaml ──► kube-apiserver
 2                                       │ checks: who? allowed? rules? valid?
                                         ▼
                                       etcd: Pod "web" saved, node = (none)       → STATUS Pending
 3  kube-scheduler (watching for Pods with no node) ── picks "minikube" ──► API server → etcd
 4  kubelet on minikube (watching for Pods on its node) sees "web"
 5  kubelet ── CRI ──► containerd: pull nginx:1.27, create + start the container
 6  kubelet ── "Running, IP 10.244.0.15" ──► API server → etcd                   → STATUS Running
```

1. **kubectl** reads `pod.yaml`, finds the API server's address and your credentials in its config file (Concept 49), and sends the object to the API server over HTTPS.
2. The **API server** checks it (authentication, authorization, admission, validation) and **saves it in etcd**. The Pod now **exists as a record**, but it has no node and no container yet. kubectl prints `pod/web created`.
   ⚠️ "**created**" means **"recorded"**, not "running"!
3. The **scheduler**, which watches for Pods without a node, sees `web`, filters and scores the nodes, and writes **`nodeName: minikube`** into the Pod. Attaching a Pod to a node like this is called a **binding**. Event: `Scheduled`.
4. The **kubelet on `minikube`**, which watches for Pods assigned to its node, sees the new Pod.
5. The kubelet asks **containerd**, through the **CRI**, to **pull the image** and **start the container**. Events: `Pulling`, `Pulled`, `Created`, `Started`.
6. The kubelet **reports the status** (Running, plus the Pod's IP address) back to the API server, which stores it in etcd. Now `kubectl get pods` shows `Running`.

These are real events from `kubectl describe pod` on your Mac, with **who** wrote each one:
```
Type    Reason     From               Message
Normal  Scheduled  default-scheduler  Successfully assigned default/web to minikube     ← step 3
Normal  Pulling    kubelet            Pulling image "nginx:1.27"                         ← step 5
Normal  Pulled     kubelet            Successfully pulled image "nginx:1.27" in 1.444s
Normal  Created    kubelet            Container created
Normal  Started    kubelet            Container started
```
(Your kubectl also prints `spec.containers{…}:` at the start of the kubelet messages. If the image is already on the node, you see `Container image "nginx:1.27" already present on machine` instead of Pulling/Pulled.)

### Step 3: desired state vs actual state
Every object has two halves:
- **`spec`** (short for **specification**) = the **desired state**: what **you** asked for.
- **`status`** = the **actual state**: what Kubernetes **observed**. **Kubernetes writes it, never you.**

The **reconciliation loop** (also called a **control loop**) is how the gap between them gets closed:
```
        ┌───────────► 1. OBSERVE: what actually exists? (status)
        │                         │
        │                         ▼
  4. REPEAT forever       2. COMPARE it with what's wanted (spec)
        ▲                         │
        │                         ▼
        └─────────────── 3. ACT: create / delete / update to close the gap
```
**Analogy: a supermarket shelf-stacker.** The shelf plan says "10 tins of beans here". Every few minutes, the stacker counts: 7 tins? Bring 3 more. 12? Take 2 away. The stacker doesn't care *why* tins went missing; they just compare and fix. That's exactly how a controller behaves.

### Step 4: controllers form a chain
For a **Deployment** (Session 06), several controllers pass the work along, each one reconciling its own kind of object:
```
you apply a Deployment (replicas: 3)
   └─► Deployment controller: "I need a ReplicaSet for this version" → creates a ReplicaSet
          └─► ReplicaSet controller: "I need 3 Pods, I see 0" → creates 3 Pods (no node yet)
                 └─► scheduler: picks a node for each Pod
                        └─► kubelets: start the containers, report status
```
Nobody "runs" the whole chain. Each link watches the API server and does its own part.

### Step 5: what if a node dies? (preview)
1. Every kubelet sends a **heartbeat** every few seconds (as small objects called **Leases**, kept in the `kube-node-lease` namespace).
2. If a node's heartbeats stop for about **40–50 seconds**, the **node controller** marks the node **`NotReady`**.
3. After about **5 more minutes** (the default), the Pods on that node are **evicted**: Kubernetes removes them from that node.
4. If those Pods belonged to a ReplicaSet or Deployment, its controller sees "I want 3, I have 2" and creates replacements, which the scheduler places on **healthy** nodes.
5. A **bare Pod** (one created on its own, with no controller looking after it) is **not** recreated. It's simply gone.

⚠️ **And if the control plane itself goes down?** Containers that are already running **keep running** (the kubelets and containerd still work). But nothing new can be scheduled, scaled, updated or moved to another node until the control plane is back. (A kubelet can still restart a crashed container on its own node, because that doesn't need the control plane.)

### 🛠 Do (~6 min): watch the journey happen
You need **two Terminal tabs** (Cmd + T opens a new one).

**Tab 1:**
```bash
kubectl get pods -w
```
- `-w` = **watch**: keep running, and print a new line every time a Pod changes. (It says `No resources found in default namespace.` at first. That's fine; leave it running.)

**Tab 2:**
```bash
kubectl run journey --image=nginx:1.27
```
- `kubectl run` = create one Pod straight from the command line (Concept 49). `journey` = its name. `--image=nginx:1.27` = the image to run.
- It prints `pod/journey created`.

**Now look at Tab 1.** You should see the stages appear:
```
NAME      READY   STATUS              RESTARTS   AGE
journey   0/1     Pending             0          0s
journey   0/1     Pending             0          0s     ← the scheduler just bound it to a node
journey   0/1     ContainerCreating   0          0s     ← the kubelet is pulling and starting
journey   1/1     Running             0          2s     ← the kubelet reported "Running"
```
**READY `0/1`** = 0 of its 1 container is ready. Press **Ctrl + C** in Tab 1 to stop watching.

**Back in Tab 2:**
```bash
kubectl describe pod journey
```
*Scroll to **Events** at the bottom and match each line to steps 3–5 of the journey. Look at the **From** column: `default-scheduler` for `Scheduled`, `kubelet` for everything else.*

```bash
kubectl get pod journey -o wide
```
*The **NODE** column says `minikube` (the scheduler's choice), and **IP** is something like `10.244.0.15` (given to the Pod when it started).*

```bash
kubectl delete pod journey
kubectl get pods
```
*The first prints `pod "journey" deleted from default namespace` (older kubectl versions print just `pod "journey" deleted`). The second prints `No resources found in default namespace.` Nobody brought it back, because no controller was looking after it.*

### ✅ Check 48

**48.1** You run `kubectl apply -f pod.yaml` and see `pod/web created`. At that exact moment, which is **guaranteed** to be true?
- A) The container has started on a node and is serving
- B) The scheduler has already picked a node for the Pod
- C) The image has been pulled to the node's local cache
- D) The Pod's record is stored, maybe not yet running

**48.2** Which is the right order for a new Pod's journey?
- A) API server → scheduler → etcd → runtime → kubelet
- B) kubelet → API server → etcd → scheduler → runtime
- C) API server → etcd → scheduler → kubelet → runtime
- D) API server → kubelet → scheduler → etcd → runtime

**48.3** (one line) A worker node loses power. The Pods of a Deployment come back on another node, but a Pod you created with `kubectl run` doesn't. Why?

---

## Concept 49 · kubectl and manifests: talking to the cluster

**Words you'll learn:** kubectl, kubeconfig, cluster / user / context, current context, object, resource, manifest, `apiVersion`, `kind`, `metadata`, `spec`, `status`, API group, imperative, declarative, idempotent, namespace, `default`, `kube-system`, `-n`, `-A`, `get`, `describe`, `apply`, `delete`, `logs`, `exec`, events, `-o wide`, `-o yaml`, port-forward, dry run.

### Step 1: kubectl
**kubectl** (short for **"kube control"**; people say "kube-control", "kube-cuttle" or "kube-C-T-L") is **the command-line client for Kubernetes**. Just like the `docker` client in Session 02, it does no real work itself: it turns your command into **HTTPS requests to the API server** and prints the answer.

Almost every kubectl command has the same shape:
```
kubectl  <verb>     <type>   <name>   <flags>
kubectl  get        pods     web      -o wide
kubectl  describe   pod      web
kubectl  delete     pod      web      -n shop
```

### Step 2: how kubectl finds the cluster: kubeconfig
kubectl reads a YAML file called the **kubeconfig**, at **`~/.kube/config`** (`~` = your home folder, from the Linux session). It contains three lists:

| Section | What it holds | On your Mac |
|---|---|---|
| **clusters** | Each cluster's API server address, plus the certificate used to trust it | `minikube` → `https://127.0.0.1:32771` |
| **users** | Credentials: how you prove who you are | `minikube` → a client certificate file |
| **contexts** | A named combination of **cluster + user + default namespace** | `minikube` = cluster minikube + user minikube + namespace default |

Plus **`current-context`**: the context kubectl is using **right now**.

**Analogy:** saved Wi-Fi networks on your phone. Each saved entry knows a network's name and password, and the phone uses whichever one is selected. Switching context = switching network.

```bash
kubectl config get-contexts          # list the contexts; * marks the current one
kubectl config current-context       # print just the current one: minikube
kubectl config use-context NAME      # switch to another cluster
```
⚠️ **Real-world trap:** engineers with several clusters (dev, staging, production) have deleted things in **production** because the wrong context was selected. Always check the context before you delete anything.

### Step 3: objects and manifests
Everything you create in Kubernetes (a Pod, a Deployment, a Service…) is an **object**: **a record of what you want, stored in etcd**. Each type of object is also called a **resource** (as in `kubectl api-resources`).

A **manifest** is **a YAML file that describes one or more objects**. Every manifest has the same four top-level fields (YAML from Session 04):
```yaml
apiVersion: v1            # 1. which version of the Kubernetes API defines this kind
kind: Pod                 # 2. what type of object this is
metadata:                 # 3. identity: who is this object?
  name: web               #    its name (unique inside its namespace)
  labels:                 #    tags used to find it later (Session 06)
    app: web
spec:                     # 4. the desired state: what you want
  containers:             #    a LIST of containers (note the dash below)
    - name: nginx         #    the container's name inside the Pod
      image: nginx:1.27   #    the image to run
      ports:
        - containerPort: 80   # the port the app listens on (documentation only)
```
| Field | Meaning | Who writes it |
|---|---|---|
| `apiVersion` | Which API group and version the kind belongs to | You |
| `kind` | The type: `Pod`, `Deployment`, `Service`… (**case-sensitive**, starts with a capital) | You |
| `metadata` | Name, namespace, labels, annotations | You (Kubernetes adds more, like a unique ID) |
| `spec` | The **desired** state | You |
| `status` | The **actual** state | ⚠️ **Kubernetes only** (you see it in `kubectl get … -o yaml`) |

**`apiVersion` values you must know.** The format is **`group/version`**. An **API group** is a family of related kinds.
| apiVersion | Kinds | Why |
|---|---|---|
| `v1` | Pod, Service, ConfigMap, Secret, Namespace, Node | The original **core** group, which has no group name |
| `apps/v1` | **Deployment**, ReplicaSet, DaemonSet, StatefulSet | The `apps` group |
| `batch/v1` | Job, CronJob | The `batch` group |
| `networking.k8s.io/v1` | Ingress | The networking group |

Using the wrong one fails. `kind: Deployment` with `apiVersion: v1` gives: `no matches for kind "Deployment" in version "v1"`.

**About `containerPort`:** it **documents** which port the app listens on. It doesn't open or publish anything (very different from Docker's `-p`).

Several objects can go in one file, separated by a line containing just **`---`**.

### Step 4: imperative vs declarative
| | **Imperative** ("do this now") | **Declarative** ("make it look like this") |
|---|---|---|
| You say | **How**: each action, step by step | **What**: the final result |
| Commands | `kubectl run`, `kubectl create deployment`, `kubectl scale`, `kubectl set image`, `kubectl delete` | `kubectl apply -f file.yaml` |
| Repeatable? | You must remember and retype the commands | The file is the record: apply it again any time |
| Reviewable in Git? | No | Yes: files go through pull requests (and GitOps, Oct 17) |
| Good for | Quick tests, learning, emergencies | Real projects and production |

**Analogy: a taxi.** Imperative = "left here, straight on, second right…". Declarative = you give the **address**, and the driver works out the route, even when a road is closed.

**`kubectl apply` is idempotent.** **Idempotent** means **running it once or ten times gives the same result**:
- First time: `pod/web created`
- Again, with no changes: `pod/web unchanged`
- After you edit the file: `pod/web configured` (only what changed is updated)

Compare **`kubectl create -f pod.yaml`**, which is imperative: the second time, it fails with `AlreadyExists`.

### Step 5: namespaces
A **namespace** is **a named section inside one cluster, used to keep groups of objects apart** (by team, app or environment). A name only has to be unique **within** its namespace: you can have a Pod called `web` in `shop` **and** another one in `blog`.
**Analogy:** folders on a computer, or departments in one office building.

| Namespace | What's in it |
|---|---|
| `default` | Where your objects go if you don't name a namespace |
| `kube-system` | Kubernetes' own components and add-ons. ⚠️ Don't change things here |
| `kube-public` | Readable by everyone. Rarely used |
| `kube-node-lease` | The nodes' heartbeat Leases (Concept 48) |

```bash
kubectl get pods                    # the namespace from your context (default)
kubectl get pods -n kube-system     # -n = in this namespace (long form: --namespace)
kubectl get pods -A                 # -A = in ALL namespaces (long form: --all-namespaces)
kubectl create namespace shop       # make a new namespace
```
Some objects belong to the whole cluster, not to a namespace: **nodes**, **namespaces** themselves, and PersistentVolumes (Session 09).

⚠️ **Trap:** `kubectl get pods` says `No resources found in default namespace.`, but the Pod exists… **in another namespace**. Add `-n <namespace>`, or use `-A`.

### Step 6: the core commands
| Command | What it does |
|---|---|
| `kubectl get pods` | List Pods: NAME, READY, STATUS, RESTARTS, AGE |
| `kubectl get pods -o wide` | The same, plus each Pod's **IP** and **NODE** |
| `kubectl get pod web -o yaml` | The **full object** as YAML, including `status` |
| `kubectl describe pod web` | A readable report: settings, state, and the **Events** at the bottom. The first thing to run when something's wrong |
| `kubectl apply -f pod.yaml` | Create or update the objects in a file |
| `kubectl delete pod web` / `kubectl delete -f pod.yaml` | Delete by name / delete everything the file describes |
| `kubectl logs web` | What the container printed (like `docker logs`). `-f` = follow live; `--previous` = the output of the last crashed run; `-c NAME` = choose a container |
| `kubectl exec -it web -- sh` | Run a command inside the container. ⚠️ Note the **`--`**: everything after it is the command for the container |
| `kubectl get events` | Recent events in the namespace |
| `kubectl port-forward pod/web 8082:80` | Temporarily connect your Mac's port 8082 to the Pod's port 80 (Part 4 below) |

### 🛠 Do (~20 min)

**Part 1: look at your cluster's own components.**
```bash
kubectl get nodes
kubectl get pods -A
```
*`-A` adds a **NAMESPACE** column. Find the `kube-system` rows. They look like this (the random endings will differ on your Mac):*
```
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-559f6c778d-mb9q4           1/1     Running   0          5d
kube-system   etcd-minikube                      1/1     Running   0          5d
kube-system   kindnet-vtqnp                      1/1     Running   0          5d
kube-system   kube-apiserver-minikube            1/1     Running   0          5d
kube-system   kube-controller-manager-minikube   1/1     Running   0          5d
kube-system   kube-proxy-b8g6s                   1/1     Running   0          5d
kube-system   kube-scheduler-minikube            1/1     Running   0          5d
kube-system   metrics-server-768f9f6999-d4rzf    1/1     Running   0          5d
kube-system   storage-provisioner                1/1     Running   0          5d
```
*(You'll also see Pods in `argocd`, `ingress-nginx`, `final-app` and other namespaces, set up for later sessions. Leave them alone.)*

**Map them to the architecture:**
| Pod you see | Component | Part of |
|---|---|---|
| `kube-apiserver-minikube` | API server (the front door) | Control plane |
| `etcd-minikube` | etcd (the memory) | Control plane |
| `kube-scheduler-minikube` | Scheduler | Control plane |
| `kube-controller-manager-minikube` | Controller manager | Control plane |
| `kube-proxy-…` | kube-proxy (traffic rules) | Worker |
| `coredns-…` | Cluster DNS | Add-on |
| `kindnet-…` | Pod networking (a CNI plugin) | Add-on |
| `metrics-server-…`, `storage-provisioner` | Metrics, storage | Add-ons |

Two things to notice:
1. **There's no kubelet Pod and no containerd Pod.** They run directly on the node as system services, and they're what start all of these Pods in the first place.
2. Names ending in **`-minikube`** are **static Pods**: the kubelet starts them straight from files on the node (see ➕). Names with a **random ending** (`coredns-559f6c778d-mb9q4`) were created by a **controller** (Session 06).

**Part 2: imperative.**
```bash
kubectl run hello --image=nginx:1.27
kubectl get pods
kubectl get pods -o wide
```
*You get `pod/hello created`, then a row `hello   1/1   Running`. With `-o wide`, the **IP** is like `10.244.0.20`: a **private** address from the `10.0.0.0/8` range (networking session). The **NODE** is `minikube`.*

```bash
kubectl logs hello | tail -3
kubectl exec -it hello -- sh
```
*`logs` shows nginx's start-up lines. `exec -it … -- sh` opens a shell **inside** the container (`-it` = interactive terminal, as in Docker). Try `hostname` (it prints `hello`, the Pod's name), then type `exit`.*

```bash
kubectl delete pod hello
```

**Part 3: declarative.**
```bash
mkdir -p ~/devops-practice/k8s-1 && cd ~/devops-practice/k8s-1
code pod.yaml
```
Paste the manifest from **Step 3** (from `apiVersion: v1` down to `containerPort: 80`) and save.

```bash
kubectl apply -f pod.yaml --dry-run=client
```
- `-f pod.yaml` = from this **f**ile. `--dry-run=client` = **check the file and show what would happen, but don't send it to the cluster**.
- It prints `pod/web created (dry run)`. A good habit for catching typos.

```bash
kubectl apply -f pod.yaml
kubectl apply -f pod.yaml
```
*First: `pod/web created`. Second: `pod/web unchanged`. That's idempotent.*

```bash
kubectl get pod web -o yaml | grep -E "nodeName|phase|podIP:"
```
- `-o yaml` = print the whole object as YAML. `grep -E "a|b|c"` = keep only lines matching any of these (`-E` = extended patterns, where `|` means "or").

*You should see `nodeName: minikube` (written by the scheduler), plus `phase: Running` and `podIP: 10.244.0.x` (written by the kubelet into `status`). **You never wrote those lines**: Kubernetes did.*

**Part 4: reach the Pod from your Mac with port-forward.**
The Pod's IP (`10.244.x.x`) lives inside the minikube container's private network, so your Mac can't reach it directly.
```bash
kubectl port-forward pod/web 8082:80
```
- `port-forward` = open a temporary tunnel. `pod/web` = to this Pod. `8082:80` = **MAC-PORT:POD-PORT**, the same order as Docker's `-p`.

**You should see** this (and the command keeps running):
```
Forwarding from 127.0.0.1:8082 -> 80
Forwarding from [::1]:8082 -> 80
```
*`127.0.0.1` is your Mac's loopback, and `[::1]` is the same thing in IPv6 (the newer, longer address format). So only your Mac can use this tunnel.*

**In Tab 2:**
```bash
curl -s http://localhost:8082 | head -4
```
*You get nginx's HTML (`<title>Welcome to nginx!</title>`), and Tab 1 prints `Handling connection for 8082`. Your request travelled Mac → API server → kubelet → Pod. Press **Ctrl + C** in Tab 1 to close the tunnel.*

**Part 5: delete it, and notice what doesn't happen.**
```bash
kubectl delete -f pod.yaml
kubectl get pods
```
*`pod "web" deleted from default namespace`, then `No resources found in default namespace.` The Pod is **gone for good**. A bare Pod has **no controller** watching it, so nothing recreates it. In Session 06 you'll put Pods under a **ReplicaSet/Deployment**, and deleting one will make a new one appear within seconds.*

### ✅ Check 49

**49.1 🐞** `kubectl apply -f app.yaml` fails with `no matches for kind "Deployment" in version "v1"`. What's the fix?
- A) Set `apiVersion: apps/v1` in the file
- B) Write `kind: deployment` in lowercase
- C) Create the namespace before applying
- D) Use `kubectl create` instead of apply

**49.2** `kubectl get pods` prints `No resources found in default namespace.`, but a teammate says the Pod `api` is running in the namespace `shop`. Which command shows it?
- A) `kubectl get pods --all`
- B) `kubectl get pods -l shop`
- C) `kubectl get pods -n shop`
- D) `kubectl get pods shop/api`

**49.3** (one line) Explain imperative vs declarative in one contrast sentence, with one kubectl command for each.

---

---

## ➕ Small but important (Kubernetes basics): read before the quiz

| Item | What it is |
|---|---|
| `kubectl explain pod.spec.containers` | **Built-in documentation** for any field, straight from the API server. Add `--recursive` to see every field underneath |
| `kubectl api-resources` | Lists every **kind** with its **short name**, **apiVersion**, and whether it's **namespaced** |
| **Short names** | `po` (pods), `svc` (services), `deploy` (deployments), `rs` (replicasets), `ns` (namespaces), `no` (nodes), `cm` (configmaps), `ds` (daemonsets), `sts` (statefulsets), `ing` (ingresses), `pvc` (persistentvolumeclaims). So `kubectl get po` = `kubectl get pods` |
| `--dry-run=client -o yaml` | **Generate YAML** instead of typing it: `kubectl run web --image=nginx:1.27 --dry-run=client -o yaml > pod.yaml`. Works with `kubectl create deployment …` too. `--dry-run=server` also asks the API server to validate it (and still saves nothing) |
| `kubectl config get-contexts` / `use-context NAME` / `set-context --current --namespace=shop` | List contexts / switch cluster / change the default namespace of the current context |
| **Labels (preview)** | `kubectl run web …` automatically adds the label `run=web`. See labels with `kubectl get pods --show-labels`. More in Session 06 |
| **Pods are mostly unchangeable** | On an existing Pod you can change the **image**, but not most other fields (ports, env vars, command). `apply` fails with `pod updates may not change fields other than spec.containers[*].image, …`. Delete and recreate the Pod (or use a Deployment, Session 06) |
| `kubectl create -f` vs `kubectl apply -f` | `create` fails if the object already exists (`AlreadyExists`); `apply` creates **or** updates |
| **Static Pods** | Pods the **kubelet** starts directly from YAML files in a folder on the node (usually `/etc/kubernetes/manifests`), without the scheduler. That's how minikube runs its control plane: `etcd-minikube`, `kube-apiserver-minikube`… (the names end with the node's name). Delete one with kubectl and it comes straight back |
| **Ways to get a cluster** | **kubeadm** (the official tool for building a real cluster on your own machines), **minikube** (a one-node practice cluster: yours), **kind** (**K**ubernetes **in** **D**ocker: each node is a Docker container), **k3s** (a lightweight Kubernetes). **Managed** clusters, where the cloud company runs the control plane for you: **EKS** (Amazon **E**lastic **K**ubernetes **S**ervice), **GKE** (**G**oogle **K**ubernetes **E**ngine), **AKS** (**A**zure **K**ubernetes **S**ervice) |
| **etcd quorum (awareness)** | Production etcd runs **3 or 5** copies (an **odd** number). A change needs a **majority** to agree, called the **quorum**. 3 copies survive 1 failure; 5 survive 2. **4 copies still survive only 1**, so an even number adds cost but no safety |
| **API maturity** | `v1` = stable. `v1beta1` = beta (mostly stable, may still change). `v1alpha1` = alpha (experimental, may disappear) |
| `kubectl get all` | Shows the common objects in a namespace (Pods, Services, Deployments, ReplicaSets), but **not everything** (ConfigMaps and Secrets, for example, aren't included) |
| `kubectl get pods -w` | Watch: print a new line whenever something changes |
| **Node roles** | `kubectl get nodes` shows `ROLES control-plane`. Older clusters showed `master` |

**Quick checks:** (1) How do you get a ready-made Deployment YAML for `nginx:1.27` with 3 replicas, without typing it? (2) Why do etcd clusters use 3 or 5 members rather than 4?
*Answers:* (1) `kubectl create deployment web --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > deploy.yaml`. (2) Every change needs a majority. 4 members survive only 1 failure, the same as 3, so the extra member adds cost but no extra safety.

---

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory

**Q1.** In a 3-node cluster, the control plane becomes unreachable for 10 minutes. During that time, what happens to the Pods **already running** on the worker nodes?
- A) They stop, since the kubelets lose their orders
- B) They keep running; lost ones move to other nodes
- C) They keep running, but nothing new is scheduled
- D) They move to whichever worker node is least busy

**Q2.** Which of these is **NOT** stored in etcd?
- A) The replica count you set for a Deployment
- B) The current status and IP of every Pod
- C) The contents of Secrets and ConfigMaps
- D) The rows in your app's PostgreSQL tables

**Q3.** A new Pod stays `Pending`. `kubectl describe pod` shows `Node: <none>`, and its Events list is **completely empty**: not even a `FailedScheduling` warning. Older Pods are running fine, and kubectl answers normally. Which component is most likely not working?
- A) The kubelet on the node
- B) The kube-scheduler
- C) The etcd database
- D) The kube-apiserver

**Q4.** You applied `pod.yaml` earlier (`pod/web created`). Now you change only `containerPort: 80` to `containerPort: 8080`, and run `kubectl apply -f pod.yaml` again. What happens?
- A) An error: most Pod fields can't change in place
- B) `pod/web configured`; the container restarts on 8080
- C) `pod/web unchanged`, as containerPort is only a note
- D) The old Pod is deleted, and a new one replaces it

**L1.** Contrast the control plane and the worker nodes in one sentence, naming two components of each.

**L2.** Explain `spec` vs `status` in one contrast sentence, and say who writes each.

---

### 📖 Words from this session (also add them to `notes/GLOSSARY.md`)
orchestration · container orchestrator · Kubernetes · K8s · numeronym · Borg · CNCF · cloud native · OCI · desired state · scheduling · self-healing · scaling · rolling update · rollback · service discovery · node · cluster · control plane · worker node · kube-apiserver · authentication · authorization · admission · etcd · key-value store · kube-scheduler · kube-controller-manager · controller · reconciliation loop · cloud-controller-manager · kubelet · heartbeat · container runtime · containerd · CRI · kube-proxy · iptables · Pod · add-on · CoreDNS · CNI · high availability · watch · event · binding · spec · status · Lease · NotReady · eviction · kubectl · kubeconfig · context · object · manifest · apiVersion · API group · imperative · declarative · idempotent · namespace · kube-system · port-forward · dry run · static Pod · kubeadm · kind · EKS · GKE · AKS · quorum
