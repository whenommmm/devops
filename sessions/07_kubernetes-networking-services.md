# Session 07 · Kubernetes III: Pod networking, Services, DNS & service discovery (Oct 14, Session 1)

**Time:** ~90–100 min · **Concepts:** 54 → 57 · **Answers:** `MY_ANSWERS.md` or say "Services questions" · **Answer key:** `07_kubernetes-networking-services_ANSWERS.md`

**Before you start:**
```bash
minikube status          # host, kubelet, apiserver: Running (if not: minikube start, then wait ~1 min)
kubectl get nodes        # minikube   Ready
kubectl get deploy,svc   # your Deployments AND Services (the comma = both kinds at once)
mkdir -p ~/devops-practice/k8s-services && cd ~/devops-practice/k8s-services
```
- You'll see `service/kubernetes`: Kubernetes' own Service for the API server. **Never delete it.** If a Deployment `web` is left over from last session, remove it: `kubectl delete deployment web`.
- ⚠️ Never delete the `minikube` or `localstack` containers, and leave the other namespaces (`kube-system`, `ingress-nginx`, `argocd`, `final-app`…) alone.

## 🔁 Warm-up
**W1.** A Deployment's ReplicaSet has `replicas: 3`. You run `kubectl delete pod web-7d9f8-abcde` (one of its Pods). What does `kubectl get pods` show about 10 seconds later?
- A) Two Pods, until you apply the YAML file again
- B) Three Pods, the deleted one back with its name
- C) Three Pods, one with a new name and young age
- D) Two Pods, and the ReplicaSet is marked Failed

**W2.** You run `kubectl set image deployment/web nginx=nginx:1.99`. That tag doesn't exist, so the new Pods can't download their image (status `ImagePullBackOff`), while the old Pods keep running. Which command puts the Deployment back on the previous working version?
- A) `kubectl rollout restart deployment/web`
- B) `kubectl rollout undo deployment/web`
- C) `kubectl rollout status deployment/web`
- D) `kubectl rollout pause deployment/web`

**W3.** What happens when you run this?
```bash
APP=web
kubectl get pods -l 'app=$APP'
```
- A) It lists Pods labelled app=web, as $APP expands
- B) It lists all Pods, as quoted selectors are ignored
- C) It lists nothing, as zsh deletes $APP from it
- D) It fails, as kubectl gets the literal text $APP

## Concept 54 · Pod networking: every Pod gets its own IP

**Words you'll learn:** Pod network, Pod IP, network model, flat network, NAT (Network Address Translation), Pod CIDR, CNI (Container Network Interface), CNI plugin, Flannel, Calico, Cilium, eBPF, kindnet, overlay network, busybox.

### The problem
Riya's `frontend` Pods must call her 3 `backend` Pods. In Docker she'd just call a container by its name, but here the backend Pods may sit on **different nodes** (machines), and they're **replaced all the time** (a ReplicaSet swaps any dead Pod for a fresh one), each time with a new name. So: **how does any Pod reach any other Pod?** (this concept) And **how does the frontend find "the backend" when its Pods keep changing?** (the rest of this session)

### The Kubernetes network model: three rules
1. **Every Pod gets its own IP address**, shared by all its containers (they reach each other on `localhost`).
2. **Every Pod can reach every other Pod directly by its IP**, even on another node, **without NAT**.
3. Programs on a node (such as the kubelet) can reach every Pod on that node.

**NAT** (**Network Address Translation**) means a device **rewrites the addresses** on packets as they pass through it. Your home router does this (many private IPs inside share one public IP outside), and so does Docker's `-p 8080:80`. Rule 2 says that **between Pods nothing is rewritten**, so a Pod always sees the real IP of whoever called it.

A network where every Pod reaches every other Pod directly, as if all were plugged into one giant switch, is called a **flat network**. The network joining all the Pods is the **Pod network**.

**Analogy: an office phone system.** Every desk (Pod) has its own extension (Pod IP). Any desk can dial any other desk directly, even in another building (node), without an operator changing the numbers.
```
 Node 1: [frontend 10.244.0.5] [backend 10.244.0.6]     Node 2: [backend 10.244.1.7] [backend 10.244.1.8]
 └──────────────────────── Pod network (built by the CNI plugin) ─────────────────────────┘
          frontend → 10.244.1.7:8080 works directly: no NAT, real IPs on both sides
```
**Pod IPs are private:** nothing outside the cluster can reach them. Your minikube hands them out from **`10.244.0.0/16`** (CIDR notation: the `/16` means the first 16 bits are fixed, so `10.244.0.0` to `10.244.255.255`). This range is the **Pod CIDR**, and each node gets its own slice of it. Your single node has `10.244.0.0/24`, so all your Pods get `10.244.0.x` addresses.

### Who builds the network: CNI plugins
Kubernetes **defines** these rules, but it **doesn't build the network itself**. It hands that job to a plugin.
- **CNI** (**Container Network Interface**) is a **standard**: an agreed way for Kubernetes to tell a networking program *"here's a new Pod: connect it and give it an IP"* and *"this Pod is gone: clean up"*.
- A **CNI plugin** is a program that follows that standard. When the kubelet starts a Pod, it calls the plugin, which creates the Pod's network interface, gives it an IP from the Pod CIDR, and sets up routes so other Pods can reach it.

| Plugin (know the names) | In one line |
|---|---|
| **Flannel** | Simple and popular. Builds an overlay network (below). Doesn't enforce NetworkPolicies (firewall rules for Pods) on its own |
| **Calico** | Routes Pod traffic like normal IP routing, and **enforces NetworkPolicies** |
| **Cilium** | Fast and advanced, built on **eBPF** (extended Berkeley Packet Filter: small programs that run safely inside the Linux kernel) |
| **kindnet** | Small and simple. **Your minikube uses this one** |

An **overlay network** (like Docker's `overlay` driver) wraps each Pod-to-Pod packet inside a normal node-to-node packet: like a letter for "desk 1.7" inside an envelope addressed to "Building 2". The receiving node opens the envelope and hands the letter to the right Pod.

### Why Pod IPs aren't enough: they're short-lived
**A Pod's IP lives and dies with the Pod.** A new Pod (after a crash, a rolling update, a move to another node, or scaling) gets a **new IP**. Hard-coding `10.244.0.6` into the frontend is like writing down a hotel guest's room number: tomorrow they've checked out, and a stranger has the room. And with 3 backends, which IP would the frontend use? It would need its own list, and its own load balancing.

What's needed is **one stable address** that always leads to "whichever backend Pods are healthy right now". That's a **Service**.

### 🛠 Do (~10 min)
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3 --port=80 --dry-run=client -o yaml > web-deploy.yaml   # only WRITES the YAML
kubectl apply -f web-deploy.yaml             # 3 nginx Pods labelled app=web (Services will use it), containerPort 80
kubectl rollout status deployment/web        # deployment "web" successfully rolled out
kubectl get pods -l app=web -o wide          # -o wide adds IP and NODE: three 10.244.0.x IPs, all on minikube
kubectl run -it --rm pod-test --image=busybox:1.36 --restart=Never -- sh   # a temporary Pod you can type in
```
*(`-it` = type inside it, like `docker run -it`; `--rm` = delete it when you leave; **busybox** = a tiny image full of basic tools such as `wget` and `nslookup`; `--restart=Never` = a plain Pod that isn't restarted when its command ends.)* At the `/ #` prompt you're **inside a Pod**:
```bash
wget -qO- http://10.244.0.102 | head -4      # (YOUR Pod IP) <title>Welcome to nginx!</title>: Pod → Pod, directly. Then: exit
kubectl delete pod <one-of-your-pod-names>
kubectl get pods -l app=web -o wide          # the replacement: new name, young AGE, usually a new IP
kubectl get pods -n kube-system              # kindnet-… (the CNI plugin), kube-proxy-… (writes the Service forwarding rules), coredns-… (the cluster's DNS server)
kubectl get node minikube -o jsonpath='{.spec.podCIDR}{"\n"}'   # prints just one field of the node: 10.244.0.0/24
```

### ✅ Check 54
**54.1** A Pod has two containers: `app` (listening on port 8080) and `helper`. How does `helper` reach `app`?
- A) At `localhost:8080`, as both share the Pod's IP
- B) At `app:8080`, through the container's DNS name
- C) At the Pod IP on port 80, mapped automatically
- D) Only through a Service that selects this Pod

**54.2** Which statement about Pod IPs is **true**?
- A) A Pod keeps its IP when its ReplicaSet replaces it
- B) Pods on different nodes need NAT to reach each other
- C) A replacement Pod usually comes up with a new IP
- D) Pod IPs are public, so a browser can open them

**54.3** (one line) What does a CNI plugin do? Name the one your minikube uses.

## Concept 55 · Services: a stable address in front of changing Pods

**Words you'll learn:** Service, virtual IP, ClusterIP, Service type, NodePort, LoadBalancer, ExternalName, headless Service, StatefulSet, kube-proxy, iptables, IPVS (IP Virtual Server), `kubectl expose`, `minikube service`, `kubectl port-forward`, `minikube tunnel`.

### What a Service is
> A **Service** is a Kubernetes object that gives a **group of Pods** (chosen by their labels) **one stable address**.

It gives you three things:
1. **A stable virtual IP**, called the **ClusterIP**. It stays the same as long as the Service exists, however often the Pods are replaced. **Virtual** means no machine or network card owns it: it exists only as forwarding rules (below).
2. **A stable DNS name**: the Service's name, such as `web`. Kubernetes runs its own DNS inside the cluster, which turns that name into the Service's IP, so other Pods can just call `http://web`.
3. **Load balancing**: connections are **spread across** all the matching, healthy Pods.

**Analogy: a company's main phone number.** Customers call **one number**, and the switchboard forwards each call to whichever agent is free. Agents come and go, but the number **never changes**.
```
                                          ┌──► Pod 10.244.0.102:80
 client Pod ──► web  (10.97.95.169:80) ───┼──► Pod 10.244.0.103:80
                (never changes)           └──► Pod 10.244.0.104:80   (Pods change; the Service keeps up)
```

### How it works: kube-proxy
**kube-proxy** runs on **every node**. It **watches** the API server for Services and the Pod IPs behind each one, and **writes forwarding rules** into the node's Linux kernel: *"anything sent to `10.97.95.169:80`, send to one of these three Pod IPs, picked at random"*. The rules are written with **iptables** (Linux's built-in tables of packet rules; **your minikube uses this**) or, in very big clusters, **IPVS** (**IP Virtual Server**, a faster load balancer built into the kernel).

⚠️ kube-proxy only **writes the rules**; it doesn't carry any traffic. The **kernel** forwards every packet. **Analogy:** kube-proxy is the office manager who updates the switchboard's list when agents come and go; the switchboard (the kernel) connects the calls.

### The Service types
A Service's `spec.type` decides **who can reach it**. Each type **builds on** the one before it (LB = load balancer):
```
 ClusterIP    =                                   ClusterIP ──► Pods
 NodePort     =              node-IP:nodePort ──► ClusterIP ──► Pods
 LoadBalancer = cloud LB ──► node-IP:nodePort ──► ClusterIP ──► Pods
```
- **ClusterIP** (the default, if you write no `type`): a virtual IP that works only **inside** the cluster. Use it for anything only other Pods should call: backend APIs, databases, caches.
- **NodePort**: everything ClusterIP does, **plus** Kubernetes opens **the same port on every node's IP**, picked from **30000–32767**. Anyone who can reach a node can use `<node-IP>:<nodePort>`. Use it for testing, or to put your own load balancer in front. *Analogy:* a side door with the same number on every branch building.
- **LoadBalancer**: everything NodePort does, **plus** Kubernetes asks the **cloud provider** (AWS, Azure, Google Cloud) for a real load balancer with a public IP, shown as **EXTERNAL-IP**. Use it to put a service on the internet. Each one is **billed separately** (Ingress solves this: one shared entry point that routes to many Services). On minikube there's no cloud, so EXTERNAL-IP stays `<pending>` until you run `minikube tunnel`.
- **ExternalName**: no Pods, no ClusterIP. Just a **DNS alias** (a CNAME record): looking up `db` returns `mydb.example.com`. It gives an outside service a name inside the cluster.
- **Headless** (`clusterIP: None`): **no virtual IP and no load balancing**. A DNS lookup of the name returns **every Pod's IP** instead. Used when clients must talk to **specific** Pods, like a database run by a **StatefulSet** (a Deployment-like object whose Pods keep fixed names such as `db-0`, `db-1`).

**Why NodePorts time out from your Mac:** the minikube "node" is a **container inside Colima's VM** (Mac → Colima VM → minikube container `192.168.49.2` → Pods `10.244.0.x`), and the Mac has **no route** to `192.168.49.2`. Three ways in, each keeping its terminal busy:

| Command | What it does |
|---|---|
| `minikube service <name> --url` | Opens an SSH tunnel (a secure forwarding connection) from a random `127.0.0.1` port on your Mac to the Service, and prints that URL |
| `kubectl port-forward svc/<name> 8080:80` | Carries traffic from your Mac's port 8080, through the API server, to port 80 of **one** Pod. ⚠️ A debugging tool, **not** load balancing |
| `minikube tunnel` | Makes LoadBalancer Services (and Ingress, the shared HTTP entry point) reachable on `127.0.0.1`. May ask for your Mac password |

### 🛠 Do (~15 min)
```bash
kubectl expose deployment web --port=80 --target-port=80   # Service "web"; copies the selector app=web; no --type = ClusterIP
kubectl get svc -o wide      # web  ClusterIP  10.97.95.169  <none>  80/TCP … app=web
kubectl run -it --rm client --image=busybox:1.36 --restart=Never -- sh
wget -qO- http://web | head -4               # (inside) nginx's page by NAME; repeat it and a different Pod may answer
wget -qO- http://10.97.95.169 | head -4      # (inside; YOUR ClusterIP) the same page by IP. Then: exit
curl -m 5 http://10.97.95.169                # (on the Mac; -m 5 = give up after 5 s) Connection timed out: inside-only
```
*`--port` = the port the Service listens on; `--target-port` = the port on the Pods, where the app really listens. The CLUSTER-IP comes from a separate range (`10.96.0.0/12`), not the Pod range.*
```bash
kubectl expose deployment web --type=NodePort --port=80 --name=web-np
kubectl get svc web-np                       # 80:31474/TCP = Service port : nodePort (it still has a CLUSTER-IP)
curl -m 5 http://$(minikube ip):31474        # (YOUR nodePort) $(minikube ip) → 192.168.49.2; times out from the Mac
minikube service web-np --url                # http://127.0.0.1:58168 + "the terminal needs to be open"
curl -I http://127.0.0.1:58168               # (in Tab 2, Cmd + T; YOUR port) HTTP/1.1 200 OK. Then Ctrl + C in Tab 1
kubectl port-forward svc/web 8080:80         # Mac 8080 → 80 (the order of -p HOST:CONTAINER)
curl -I http://localhost:8080                # (in Tab 2) HTTP/1.1 200 OK. Then Ctrl + C in Tab 1
kubectl expose deployment web --type=LoadBalancer --port=8080 --target-port=80 --name=web-lb   # optional
kubectl get svc web-lb                       # EXTERNAL-IP <pending>: no cloud to build a load balancer
kubectl delete svc web-lb                    # (optionally run minikube tunnel in Tab 2 first → 127.0.0.1; untested on your Mac)
```

### ✅ Check 55
**55.1** `kubectl get svc` shows:
```
NAME   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
api    NodePort   10.106.29.192   <none>        8080:31474/TCP   2m
```
Which statement is correct?
- A) Pods use `api:31474`; outside clients use `<node-IP>:8080`
- B) Only `<node-IP>:31474` works, as NodePorts lack a ClusterIP
- C) Pods use `api:8080`; outside clients use `<EXTERNAL-IP>:8080`
- D) Pods use `api:8080`; outside clients use `<node-IP>:31474`

**55.2** On your Mac, `curl http://$(minikube ip):31474` times out, although `kubectl get svc` shows that NodePort and the Pods are Running. Why?
- A) NodePorts open only while `minikube tunnel` is running
- B) NodePort Services accept traffic only from other Pods
- C) The Mac has no route to the node inside Colima's VM
- D) `minikube ip` prints the Service's ClusterIP, not a node's

**55.3** (one line) What does kube-proxy do, and does the traffic itself pass through the kube-proxy program?

## Concept 56 · Selectors, Endpoints, and the four ports

**Words you'll learn:** Service selector, Endpoints, EndpointSlice, Ready, `containerPort`, `targetPort`, `port`, `nodePort`, named port, `protocol`, "connection refused" vs "timed out".

### The problem
Riya's Service looks perfect and her Pods are `Running`, yet every request fails with **`Connection refused`**. Nothing is down: the **wiring** is wrong. Only two things connect a Service to its Pods: **labels** (which Pods) and **ports** (which door on them). These are favourite exam debugging questions.

### How a Service finds its Pods: Endpoints
A Service's `spec.selector` is a set of labels. Kubernetes keeps looking for Pods that are in the **same namespace**, have **all** those labels (extra labels are fine), and are **Ready** (passing their readiness check, a repeated test of whether the Pod can take traffic yet).

The IP and port of every matching Pod go into a list called **Endpoints**, which kube-proxy turns into its rules. **Analogy:** the switchboard's list of agents **on shift right now**, updated within a second or two whenever a Pod dies or a new one becomes Ready. An **EndpointSlice** is the newer form of that list, split into slices so huge Services update efficiently; `kubectl get endpoints` still works but prints a deprecation warning.

⚠️ **Selector traps:**
- A **Service** selector is plain `key: value` pairs, while a **Deployment** uses `matchLabels:`:
  ```yaml
  # Service                       # Deployment
  spec:                           spec:
    selector:                       selector:
      app: web                        matchLabels:
                                        app: web
  ```
- It matches the **Pods'** labels (the template's), **not** the Deployment's own `metadata.labels`. Two labels = Pods need **both**. Labels are **case-sensitive** (`app: Web` ≠ `app: web`).

### The four ports
The most-confused part of Services. Follow one request from left to right:
```
  OUTSIDE (NodePort only)        THE SERVICE                     THE POD
  <node-IP>:30080   ──────►   10.105.180.161:8080   ──────►   10.244.0.102:80   (nginx listens on 80)
     nodePort                   port (web-demo:8080)             targetPort
```
| Field | Written in | What it is | Used by |
|---|---|---|---|
| **`nodePort`** | Service `ports:` (NodePort/LoadBalancer) | The port opened on **every node**, 30000–32767. Picked for you if left out | Clients **outside** |
| **`port`** | Service `ports:` | The port the **Service itself** listens on (its ClusterIP and name). Required | Clients **inside**: `web-demo:8080` |
| **`targetPort`** | Service `ports:` | The port **on the Pod** where traffic is delivered. **Must be where the app really listens.** Defaults to `port` | kube-proxy's rules |
| **`containerPort`** | Pod template `containers[].ports:` | The port the container **says** it uses: mainly **documentation**, like `EXPOSE` in a Dockerfile | Humans, named ports |

**Memory line:** nodePort = the building's side door · port = reception's extension · targetPort = the room the guest is really in · containerPort = the name card on the door. ⚠️ Traffic still works with a missing/wrong `containerPort`, **never** with a wrong `targetPort`. **Named port:** name a container port (`- name: http` + `containerPort: 80`) and write `targetPort: http` in the Service, so a port change happens in one place.

### A worked example
```yaml
apiVersion: v1              # Services are in the core API group
kind: Service
metadata:
  name: web-demo            # the Service's name (also its DNS name)
spec:
  type: NodePort            # reachable inside the cluster AND on every node
  selector:
    app: web                # plain key: value; must match the Pods' labels (from web-deploy.yaml)
  ports:
    - name: http            # a name for this port
      protocol: TCP         # the default; UDP is also possible (DNS uses UDP)
      port: 8080            # inside the cluster: http://web-demo:8080
      targetPort: 80        # delivered to port 80 on each Pod (where nginx listens)
      nodePort: 30080       # outside: http://<node-IP>:30080
```
From a Pod, `http://web-demo:8080` ✅ works. `http://web-demo` (that means port 80) ❌ **times out**: the Service has no port 80. `http://web-demo:30080` ❌ times out too: nodePorts live on **nodes**, not on the ClusterIP. From a node, `http://<node-IP>:30080` ✅ works.

### The classic bugs
Two errors, two meanings: **connection refused** = something answered at once, "nothing is listening here"; **timed out** = nothing answered at all, so the client gave up waiting.

| Symptom | `describe svc` / `get endpoints` shows | Cause → fix |
|---|---|---|
| `Connection refused` | **Endpoints `<none>`** | **The selector matches no Ready Pod** (typo, wrong case, label only on the Deployment, other namespace, not Ready) → compare `Selector:` with `kubectl get pods --show-labels` |
| `Connection refused` | Endpoints with the **wrong port** (`…:8080`, nginx on 80) | **`targetPort` is wrong** → set it to the app's real port |
| `Connection refused` | Everything looks right | The app listens on **`127.0.0.1`**, which accepts connections only from inside its own Pod → make it listen on `0.0.0.0` (all interfaces) |
| **Timed out** | Everything looks right | You called the **wrong Service port** → use the port from `kubectl get svc` |

**Debugging order:** `kubectl get svc` → `kubectl describe svc <name>` (Selector, TargetPort, **Endpoints**) → `kubectl get pods --show-labels` → test from a client Pod.

### 🛠 Do (~15 min)
```bash
code web-svc.yaml                      # paste the worked-example YAML, save
kubectl apply -f web-svc.yaml          # ("provided port is already allocated"? use nodePort 30081)
kubectl get svc web-demo               # web-demo   NodePort   10.105.180.161   <none>   8080:30080/TCP
kubectl describe svc web-demo          # Port http 8080/TCP · TargetPort 80/TCP · NodePort 30080 · Endpoints 10.244.0.102:80,…
kubectl get endpoints web-demo         # "Warning: v1 Endpoints is deprecated…" (harmless) + the same Pod-IP:80 list
kubectl get endpointslices -l kubernetes.io/service-name=web-demo   # the same IPs, in the newer format
kubectl run client --image=busybox:1.36 --restart=Never -- sleep 3600   # a client Pod that stays Running for an hour
kubectl exec client -- wget -qO- -T 3 http://web-demo:8080 | head -4    # exec = run one command inside (like docker exec); -T 3 = 3 s limit
kubectl exec client -- wget -qO- -T 3 http://web-demo                   # wget: download timed out (the Service has no port 80)
```
*(After an hour the client Pod finishes; if `exec` complains, run `kubectl delete pod client` and start it again.)*

**Break the selector:** in `web-svc.yaml`, change `app: web` to **`app: wbe`** (a typo), then:
```bash
kubectl apply -f web-svc.yaml
kubectl get endpoints web-demo                                          # ENDPOINTS <none>
kubectl exec client -- wget -qO- -T 3 http://web-demo:8080              # can't connect to remote host … Connection refused
kubectl describe svc web-demo | grep -E 'Selector|Endpoints'            # grep -E 'A|B' = A or B → Selector: app=wbe, empty Endpoints
kubectl get pods --show-labels         # the Pods say app=web: mismatch found. Fix it back, apply, check the endpoints again
```
**Break the targetPort:** change `targetPort: 80` to `8080`, apply, and run `kubectl get endpoints web-demo` and the `wget` again: the endpoints **are** listed this time, but as `…:8080`, and you get `Connection refused` again. Same error, different cause; the Endpoints line tells them apart. Change it back to `80` and apply.

### ✅ Check 56
**56.1** 🐞
```
$ kubectl get pods --show-labels
NAME             READY   STATUS    LABELS
api-6c9f-x2k7q   1/1     Running   app=api,tier=backend,pod-template-hash=6c9f
$ kubectl describe svc api | grep -E 'Selector|Endpoints'
Selector:   app=api,tier=back-end
Endpoints:
```
What causes the empty Endpoints?
- A) `tier=back-end` doesn't match the Pod's `tier=backend`
- B) The Pod isn't Ready yet, so it's left out of the list
- C) A Service selector has to be written under `matchLabels`
- D) The selector must also include the `pod-template-hash`

**56.2** A Service has `port: 80` and `targetPort: 8080`. The Deployment says `containerPort: 8080`, but the app inside actually listens on **3000**. Client Pods get `Connection refused` on `http://svc:80`. What's the fix?
- A) Set `containerPort: 3000` in the Deployment
- B) Set `targetPort: 3000` in the Service YAML
- C) Set `port: 3000` in the Service YAML
- D) Set `nodePort: 3000` in the Service YAML

**56.3** (one line) Explain `port` vs `targetPort` in contrast form.

## Concept 57 · Kubernetes DNS and service discovery

**Words you'll learn:** service discovery, CoreDNS, `kube-dns`, A record, FQDN (Fully Qualified Domain Name), cluster domain, `/etc/resolv.conf`, search domains, `ndots`, cross-namespace call, NXDOMAIN, environment-variable discovery.

### The problem
The frontend could call the backend at its ClusterIP, `10.97.95.169`. But if the Service is re-created, or the app moves to another cluster (say, from testing to production), that IP **changes**, and numbers are easy to mistype anyway. A **name** fixes both: `http://backend:8080` works in any cluster that has a Service called `backend`.

> **Service discovery** = how a program **automatically finds the address** of another service it needs. In Kubernetes, that's mainly done with **DNS**.

### CoreDNS: the cluster's own DNS server
- **CoreDNS** is a DNS server running as Pods in `kube-system`, reached through a Service named **`kube-dns`** (an older name, kept) at a fixed ClusterIP: **`10.96.0.10`** on your cluster.
- It **watches the API server**. As soon as you create a Service, CoreDNS answers lookups of its name with its ClusterIP (an **A record**: name → IPv4 address). Delete the Service, and the name disappears.
- The kubelet writes every Pod's **`/etc/resolv.conf`** (the Linux file that says which DNS server to ask) so that it points at CoreDNS.

**Analogy:** the company's internal phone directory, **updated automatically** whenever a department opens or closes. Staff look up "Accounts", never a desk extension.

### The name format
```
   web  .  default  .   svc    .  cluster.local
 Service   namespace  "this is a   the cluster domain
  name                 Service"    (default value)
```
The complete name, with nothing left out, is the **FQDN** (**Fully Qualified Domain Name**): `web.default.svc.cluster.local`.

**Short names work thanks to search domains.** A Pod in `default` has this `/etc/resolv.conf`:
```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```
- `nameserver` = ask CoreDNS. `search` = the **search domains**: endings tried in turn on a **short** name, so `web` is first tried as `web.default.svc.cluster.local`, which exists.
- `ndots:5` = a name with fewer than 5 dots counts as "maybe short", so those endings are tried first.

| From a Pod in `default`, you type | It becomes | It reaches |
|---|---|---|
| `web` | `web.default.svc.cluster.local` | Service `web` in the **same namespace** |
| `web.shop` or `web.shop.svc` | `web.shop.svc.cluster.local` | Service `web` in the namespace **`shop`** |
| `web.shop.svc.cluster.local` | (already complete) | the same, from anywhere |

⚠️ **A short name only works inside the same namespace.** From the namespace `shop`, plain `web` is tried as `web.shop.svc.cluster.local`, which doesn't exist, so you get `bad address 'web'`. To cross namespaces, add the namespace: `web.default`. (Like Docker Compose's service names, plus a namespace level, and pointing at a load-balancing Service rather than one container.)

### Discovery by environment variables (awareness)
When a container starts, the kubelet also injects **environment variables** for every Service that **already exists** in its namespace: for `web`, `WEB_SERVICE_HOST=10.97.95.169` and `WEB_SERVICE_PORT=80` (upper-cased, `-` becomes `_`: `web-np` → `WEB_NP_…`). The catch: a Service created **after** the Pod started never appears in it. DNS has no such gap, which is why **DNS is preferred**.

### Reading busybox's `nslookup`
busybox's `nslookup` tries each search ending and prints a line for every miss, like `** server can't find web.svc.cluster.local: NXDOMAIN`. **NXDOMAIN** = "**N**on-e**X**istent **DOMAIN**": that attempt doesn't exist. **Ignore those lines** and read the `Name:` / `Address:` pair: that's the real answer. Two busybox quirks (not Kubernetes problems):
- It may exit with an error (`pod … terminated (Error)`) even though the answer it printed is right.
- It **skips the search endings for names that contain a dot**, so `nslookup web.default` fails: use the full name. (`wget http://web.default` is fine.)

**Three-step test when "the Service doesn't work"**, from a client Pod: `nslookup <service>` (is **DNS** fine?) → `wget <ClusterIP>:<port>` (is the **Service** fine?) → `wget <Pod-IP>:<targetPort>` (is the **app** fine?). The first step that fails names the broken layer.

### 🛠 Do (~12 min)
```bash
kubectl run -it --rm dns-test --image=busybox:1.36 --restart=Never -- sh
cat /etc/resolv.conf                          # (inside) the search, nameserver 10.96.0.10 and ndots:5 lines
nslookup web                                  # NXDOMAIN lines, then Name: web.default.svc.cluster.local · Address: <web's CLUSTER-IP>
nslookup web.default.svc.cluster.local        # just the Name/Address answer
wget -qO- http://web.default.svc.cluster.local | head -4   # nginx's page via the full name
nslookup kube-dns.kube-system.svc.cluster.local            # Address: 10.96.0.10, the DNS Service itself
env | grep WEB                                # WEB_SERVICE_HOST=…, WEB_SERVICE_PORT=80, WEB_NP_…, WEB_DEMO_… Then: exit
kubectl create namespace dns-demo
kubectl run -it --rm far-client -n dns-demo --image=busybox:1.36 --restart=Never -- sh
wget -qO- -T 3 http://web                     # (inside) wget: bad address 'web' (no web Service in dns-demo)
wget -qO- -T 3 http://web.default | head -4   # works: it names the namespace. Then: exit
kubectl delete namespace dns-demo             # deletes everything inside it
kubectl get pods -n kube-system -l k8s-app=kube-dns   # one coredns-… Pod Running (the label keeps the old name)
kubectl get svc -n kube-system kube-dns       # 10.96.0.10  53/UDP,53/TCP,9153/TCP: DNS port 53 (mostly UDP); 9153 = metrics
kubectl delete pod client                     # clean up
kubectl delete svc web web-np web-demo        # (and web-lb, if made); the "kubernetes" Service stays
kubectl delete deployment web
```

### ✅ Check 57
**57.1** A Pod in the namespace `shop` runs `curl http://payments:9000`. The `payments` Service exists only in the namespace `billing`. What happens?
- A) It works, since Service names are unique cluster-wide
- B) The name doesn't resolve; it needs `payments.billing`
- C) It works, but only when both Pods are on one node
- D) DNS answers, but traffic between namespaces is blocked

**57.2** Which is the FQDN of the Service `db` in the namespace `prod`?
- A) `prod.db.svc.cluster.local`
- B) `db.svc.prod.cluster.local`
- C) `db.prod.pod.cluster.local`
- D) `db.prod.svc.cluster.local`

**57.3** (one line) Why is DNS preferred over the `<NAME>_SERVICE_HOST` environment variables?

## ➕ Small but important

| Item | What it means |
|---|---|
| `kubectl expose` | Creates a Service for a Deployment: `kubectl expose deployment web --port=80 --target-port=8080 --type=NodePort --name=web-np`. With no `--port` and no `containerPort`, it fails: `couldn't find port via --port flag or introspection` |
| **NodePort range** | Only **30000–32767**: `nodePort: 8080` is rejected (`provided port is not in the valid range`). A `nodePort` on a ClusterIP Service is rejected too |
| **`sessionAffinity: ClientIP`** | "Sticky sessions": requests from one client IP keep going to the **same Pod** (default `None`). For apps that keep each user's data in memory |
| **`externalTrafficPolicy`** | NodePort/LoadBalancer only. **`Cluster`** (default): any node forwards to any Pod, but the app sees a node's IP. **`Local`**: a node only sends to Pods **on itself**, so the **real client IP is kept** (nodes without such a Pod drop the traffic) |
| **Headless Service + StatefulSet** | `clusterIP: None`; DNS returns every Pod's IP, and each StatefulSet Pod gets its own name, like `db-0.db.prod.svc.cluster.local` |
| **Service without a selector** | Kubernetes creates **no** endpoints; you add an EndpointSlice yourself (e.g. for an outside database), so Pods can still use a normal Service name |
| **Multi-port Services** | A Service can list several ports (e.g. 80 and 443); then **every port needs a `name:`** |
| **NetworkPolicy** | By default every Pod can talk to every Pod. A NetworkPolicy is a set of firewall rules: it selects Pods by label and allows only the listed incoming (**ingress**) or outgoing (**egress**) traffic. Needs an enforcing CNI (Calico, Cilium; not Flannel alone) |
| **The `kubernetes` Service** | In `default`, ClusterIP `10.96.0.1`, port 443: how Pods reach the API server. Don't delete it |

**Quick checks:** (1) Users' real IP addresses must appear in the app's logs behind a NodePort Service. Which setting helps? (2) You create a Service with **no** `selector`. What does `kubectl get endpoints` show for it, and why?
*Answers:* (1) `externalTrafficPolicy: Local`: traffic stays on the node that received it, so the client's source IP isn't rewritten. (2) `<none>`: without a selector, Kubernetes doesn't know which Pods to list, so you must add an EndpointSlice yourself.

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory
**Q1.** A Flask app runs `app.run(host="127.0.0.1", port=5000)` in its container. Its Service has `port: 80` and `targetPort: 5000`, and `kubectl get endpoints` lists three `<Pod-IP>:5000` entries. Client Pods calling `http://svc` get `Connection refused`. What's the cause?
- A) The selector matches no Pods, so nothing can answer
- B) The app accepts connections only from its own Pod
- C) targetPort must equal port, so both must be 80
- D) containerPort 5000 is missing from the Pod spec

**Q2.** A Service `shop` has `type: NodePort`, `port: 8080`, `targetPort: 80` and `nodePort: 30080`, and selects nginx Pods. Which pair of requests **both** succeed?
- A) From a Pod: `shop:80` · from a node: `localhost:30080`
- B) From a Pod: `shop:30080` · from a node: `localhost:30080`
- C) From a Pod: `shop:8080` · from a node: `localhost:30080`
- D) From a Pod: `shop:8080` · from a node: `localhost:80`

**Q3.** A Pod in the namespace `web` started at 10:00. At 10:05, a Service `cache` is created in the namespace `web`. At 10:10, the Pod runs `wget http://cache` and `echo $CACHE_SERVICE_HOST`. What happens?
- A) Both work: DNS answers and the variable holds the IP
- B) Both fail until the Pod is restarted or recreated
- C) The variable is set, but DNS waits for a restart
- D) DNS resolves `cache`, but the variable stays empty

**Q4.** Which statement about Services is **NOT** true?
- A) kube-proxy load-balances a headless Service
- B) A NodePort Service also gets its own ClusterIP
- C) A ClusterIP is virtual: no network card owns it
- D) A LoadBalancer Service also opens a nodePort

**L1.** Explain ClusterIP vs NodePort vs LoadBalancer in contrast form, saying who can reach each one.

**L2.** Write the full forms of **CNI** and **FQDN**, and write the FQDN of the Service `api` in the namespace `prod`.

### ✍️ Write the command
**C1.** Forward port 8080 on your Mac to port 80 of the Service `web`.

**C2.** Create a **NodePort** Service named `web-np` for the Deployment `web`. The Service listens on port 80 and sends traffic to port 8080 on the Pods.

**C3.** Show only the `Selector` and `Endpoints` lines from the details of the Service `web-demo`.

**C4.** From the already-running Pod `client`, download `http://web-demo:8080` quietly to the screen with a 3-second time limit.
