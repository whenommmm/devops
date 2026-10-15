# Session 07 · Kubernetes III: Pod networking, Services, DNS & service discovery (Oct 14, Session 1)

**Time:** ~90–100 min · **Concepts:** 54 → 57 · **Answers:** `MY_ANSWERS.md` or say "Services questions" · **Answer key:** `07_kubernetes-networking-services_ANSWERS.md`

**Before you start:**
```bash
minikube status          # host, kubelet, apiserver: Running (else: minikube start, wait ~1 min)
kubectl get nodes        # minikube   Ready
kubectl get deploy,svc   # both kinds at once; service/kubernetes = the API server's own Service: never delete it. Leftover web? kubectl delete deployment web
mkdir -p ~/devops-practice/k8s-services && cd ~/devops-practice/k8s-services
```
> ⚠️ Never delete the `minikube` or `localstack` containers, and leave the other namespaces (`kube-system`, `ingress-nginx`, `argocd`, `final-app`…) alone.

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
> **Problem:** Riya's `frontend` Pods must call her 3 `backend` Pods, which may sit on different nodes and are replaced all the time (new names, new IPs). How does any Pod reach any other Pod (this concept), and how does the frontend find "the backend" (Concepts 55–57)?

**The Kubernetes network model (3 rules):**
- ① every Pod gets its own IP (shared by its containers, which reach each other on `localhost`);
- ② every Pod can reach every other Pod by IP, even on another node, **without NAT**;
  - **NAT** (**Network Address Translation**) = a device rewriting the addresses on passing packets, like your home router (many private IPs behind one public IP) or Docker's `-p`; rule ② means a Pod sees its caller's real IP.
- ③ agents on a node (e.g. the kubelet) can reach every Pod on it.

A network where every Pod reaches every Pod directly is a **flat network**; the network joining all Pods is the **Pod network**.

**Analogy: an office phone system.** Every desk (Pod) has its own extension (Pod IP), and any desk can dial any other directly, even in another building (node), without an operator changing numbers.
```
 Node 1: [frontend 10.244.0.5] [backend 10.244.0.6]        Node 2: [backend 10.244.1.7] [backend 10.244.1.8]
        └────────── Pod network (built by the CNI plugin): frontend → 10.244.1.7:8080 directly, no NAT ──────────┘
```
- **Pod IPs are private** (`10.244.x.x` is inside `10.0.0.0/8`). minikube hands them out from **`10.244.0.0/16`** (**CIDR**, **Classless Inter-Domain Routing**: first 16 bits fixed → `10.244.0.0`–`10.244.255.255`). That range is the **Pod CIDR**; each node gets a slice (yours: `10.244.0.0/24`; one node, so all your Pods are `10.244.0.x`).
- **CNI** (**Container Network Interface**) = a **standard** way for Kubernetes to tell a networking program "connect this new Pod and give it an IP" / "this Pod is gone, clean up". A **CNI plugin** follows it: the kubelet calls it when a Pod starts, and it creates the Pod's network interface, assigns an IP from the Pod CIDR and sets up routes.
- **CNI plugins** (awareness):
  - **Flannel** (simple; builds an overlay network; doesn't enforce NetworkPolicies on its own)
  - **Calico** (normal IP routing; **enforces NetworkPolicies**, i.e. firewall rules for Pods)
  - **Cilium** (fast, built on **eBPF**, **extended Berkeley Packet Filter**: small programs running safely inside the Linux kernel)
  - **kindnet** (small and simple: **your minikube uses it**).
- **Overlay network** (like Docker's `overlay` driver) = each Pod-to-Pod packet is wrapped inside a normal node-to-node packet, like a letter to "desk 1.7" inside an envelope addressed to "Building 2".
- ⚠️ **Pod IPs are ephemeral:** a new Pod (crash, rolling update, rescheduling, scaling) gets a **new IP**, so hard-coding one is like noting a hotel guest's room number. With 3 backends you'd also need your own list and load balancing. The fix is **one stable address**: a **Service** (Concept 55).
### 🛠 Do (~10 min)
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3 --port=80 --dry-run=client -o yaml > web-deploy.yaml   # writes the YAML (open it: code web-deploy.yaml)
kubectl apply -f web-deploy.yaml && kubectl rollout status deployment/web   # 3 nginx Pods labelled app=web (Services will use it), containerPort 80
kubectl get pods -l app=web -o wide          # -l = label selector; -o wide adds IP + NODE: three 10.244.0.x IPs, all on minikube
kubectl run -it --rm pod-test --image=busybox:1.36 --restart=Never -- sh   # -it = type inside, --rm = delete on exit; busybox = tiny tool image
wget -qO- http://10.244.0.102 | head -4      # (inside, at / #; YOUR Pod IP) <title>Welcome to nginx!</title>: Pod → Pod directly. Then: exit
kubectl delete pod <one-of-your-pod-names>
kubectl get pods -l app=web -o wide          # replacement: new name, young AGE, usually a new IP
kubectl get pods -n kube-system              # kindnet-… (the CNI plugin), kube-proxy-… (Concept 55), coredns-… (Concept 57)
kubectl get node minikube -o jsonpath='{.spec.podCIDR}{"\n"}'   # print one field → 10.244.0.0/24 (single quotes protect { } and ")
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
> **Problem:** the frontend needs **one** address for "the backend", but backend Pods come and go.
>
> **Idea:** a **Service** is **a Kubernetes object that gives a group of Pods (chosen by labels, Concept 56) one stable address**: a **virtual IP** called the **ClusterIP** (no network card owns it; it exists only as forwarding rules), a **DNS name** (Concept 57), and **load balancing** across the matching healthy Pods.

**Analogy: a company's main phone number.** Customers call one number; the switchboard forwards each call to whichever agent is free; agents come and go, but the number never changes.
```
 client Pod ──► web (10.97.95.169:80, never changes) ──┬──► Pod 10.244.0.102:80
                                                       ├──► Pod 10.244.0.103:80   (Pods change; the Service keeps up)
                                                       └──► Pod 10.244.0.104:80
```
**kube-proxy** (on every node) **watches** the API server for Services and their Pod lists and **writes forwarding rules** into each node's kernel ("traffic to `10.97.95.169:80` → one of these Pod IPs, at random").
- It uses **iptables** (Linux's packet-rule tables; **your minikube's mode**) or **IPVS** (**IP Virtual Server**, a kernel load balancer, faster with thousands of Services; newer: nftables).
- ⚠️ kube-proxy only **writes the rules**; the **kernel** carries the traffic (the manager updating the switchboard's list, not connecting calls).

**Service types** (`spec.type`; each builds on the one before; LB = load balancer):
```
 ClusterIP    =                                   ClusterIP ──► Pods
 NodePort     =              node-IP:nodePort ──► ClusterIP ──► Pods
 LoadBalancer = cloud LB ──► node-IP:nodePort ──► ClusterIP ──► Pods
```
| Type | Who can reach it, and how | Typical use | On your minikube |
|---|---|---|---|
| **ClusterIP** (default) | Only things inside the cluster: `web:80` or the ClusterIP | Internal APIs, databases, caches | Works from Pods |
| **NodePort** | Anyone who can reach a node: `<node-IP>:<nodePort>`, the **same port on every node**, from **30000–32767** | Testing; your own LB in front | Not directly from the Mac (below) |
| **LoadBalancer** | The internet: Kubernetes asks the **cloud** (AWS = Amazon Web Services, Azure, Google Cloud) for a real LB with a public **EXTERNAL-IP** that forwards to the nodePorts; each one is billed separately (Ingress fixes that, Session 08) | Public services in a cloud | `<pending>` until `minikube tunnel` |
| **ExternalName** | Just a **DNS alias** (a CNAME record): `db` → `mydb.example.com`; no Pods, no ClusterIP | An inside name for an outside service | Works |
| **Headless** (`clusterIP: None`) | No virtual IP, no load balancing: DNS returns **every Pod's IP** | Talking to specific Pods: **StatefulSets** (Pods with fixed names like `db-0`), databases | Works |

**Why NodePorts time out from your Mac:** the minikube "node" is a **container inside Colima's VM** (Mac → Colima VM → minikube container `192.168.49.2` → Pods `10.244.0.x`), and the Mac has **no route** to `192.168.49.2`. Three ways in, each keeping its terminal busy:
- `minikube service <name> --url` = an SSH tunnel (a secure forwarding connection) from a random `127.0.0.1` port to the Service; prints the URL.
- `kubectl port-forward svc/<name> 8080:80` = Mac port 8080 → through the API server → port 80 of **one** Pod. ⚠️ A debugging tool, **not** load balancing.
- `minikube tunnel` = makes LoadBalancer Services (and Ingress) reachable on `127.0.0.1`; may ask for your Mac password.
### 🛠 Do (~15 min)
```bash
kubectl expose deployment web --port=80 --target-port=80   # Service "web", copies the selector app=web; Service port → Pod port; no --type = ClusterIP
kubectl get svc -o wide      # web  ClusterIP  10.97.95.169  <none>  80/TCP … app=web (ClusterIPs come from 10.96.0.0/12, not the Pod range)
kubectl run -it --rm client --image=busybox:1.36 --restart=Never -- sh
wget -qO- http://web | head -4               # (inside) nginx's page by NAME; repeat it and a different Pod may answer
wget -qO- http://10.97.95.169 | head -4      # (inside; YOUR ClusterIP) the same page by IP; then: exit
curl -m 5 http://10.97.95.169                # on the Mac (-m 5 = give up after 5 s): Connection timed out. A ClusterIP is inside-only
kubectl expose deployment web --type=NodePort --port=80 --name=web-np
kubectl get svc web-np                       # 80:31474/TCP = port:nodePort (it still has a CLUSTER-IP)
curl -m 5 http://$(minikube ip):31474        # $( ) = command substitution → 192.168.49.2; times out from the Mac (no route)
minikube service web-np --url                # http://127.0.0.1:58168 + "terminal needs to be open". Tab 2: curl -I <that URL> → 200 OK; Ctrl + C
kubectl port-forward svc/web 8080:80         # Mac 8080 → 80 (like -p HOST:CONTAINER). Tab 2: curl -I http://localhost:8080 → 200 OK; Ctrl + C
kubectl expose deployment web --type=LoadBalancer --port=8080 --target-port=80 --name=web-lb   # optional
kubectl get svc web-lb                       # EXTERNAL-IP <pending>
kubectl delete svc web-lb                    # (optionally run minikube tunnel in Tab 2 first → EXTERNAL-IP 127.0.0.1; untested on your Mac)
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
> **Problem:** Riya's Service looks perfect and the Pods are `Running`, yet every request gets **`Connection refused`**. Only two things wire a Service to Pods: **labels** (which Pods) and **ports** (which door on them). A favourite exam debugging topic.

**Endpoints:** the Service's `spec.selector` picks Pods that are in the **same namespace**, have **all** its labels (extras are fine) and are **Ready** (passing readiness checks, Session 09).
- Their IP:port pairs form the **Endpoints** list that kube-proxy turns into rules.
- **Analogy:** the switchboard's list of agents **on shift right now**, updated within a second or two.
- **EndpointSlice** = the newer form that splits big lists into slices (≤ 100 addresses each); `kubectl get endpoints` still works but prints a deprecation warning.

⚠️ **Selector traps:** a **Service** selector is plain `key: value` pairs, while a Deployment's uses `matchLabels:` (Session 06):
```yaml
# Service                       # Deployment
spec:                           spec:
  selector:                       selector:
    app: web                        matchLabels:
                                      app: web
```
It matches the **Pods'** labels (the template's), not the Deployment's own `metadata.labels`. Two labels = Pods need **both**. Labels are **case-sensitive** (`app: Web` ≠ `app: web`).
```
  OUTSIDE (NodePort only)         THE SERVICE                      THE POD
  <node-IP>:30080  ──────────►  10.105.180.161:8080  ──────────►  10.244.0.102:80  (nginx listens on 80)
      nodePort                    port (web-demo:8080)               targetPort  (containerPort documents it)
```
| Field | Written in | What it is | Used by |
|---|---|---|---|
| **`nodePort`** | Service `ports:` (NodePort/LoadBalancer) | The port opened on **every node**, 30000–32767 (picked for you if left out) | Clients **outside** |
| **`port`** | Service `ports:` | The port the **Service** listens on (ClusterIP and name). Required | Clients **inside**: `web-demo:8080` |
| **`targetPort`** | Service `ports:` | The port **on the Pod** that receives traffic: **must be where the app really listens**. Defaults to `port` | kube-proxy's rules |
| **`containerPort`** | Pod template `containers[].ports:` | The port the container **says** it uses: mainly **documentation**, like `EXPOSE` (Session 03) | Humans, **named ports** |

**Memory line:** nodePort = the building's side door · port = reception's extension · targetPort = the room the guest is really in · containerPort = the name card on the door. ⚠️ Traffic still works with a missing/wrong `containerPort`, **never** with a wrong `targetPort`. **Named port:** name a container port (`- name: http` + `containerPort: 80`) and write `targetPort: http` in the Service, so a port change happens in one place.
```yaml
apiVersion: v1            # the worked example: save as web-svc.yaml
kind: Service
metadata:
  name: web-demo
spec:
  type: NodePort
  selector:
    app: web              # plain key: value; must match the Pods' labels (from web-deploy.yaml)
  ports:
    - name: http
      protocol: TCP       # the default; UDP is also possible (DNS uses UDP)
      port: 8080          # inside the cluster: http://web-demo:8080
      targetPort: 80      # delivered to port 80 on each Pod (where nginx listens)
      nodePort: 30080     # outside: http://<node-IP>:30080
```
From a Pod: `http://web-demo:8080` ✅ · `http://web-demo` (= port 80) ❌ **times out** (no such Service port) · `http://web-demo:30080` ❌ times out (nodePorts live on **nodes**, not the ClusterIP). From a node: `http://<node-IP>:30080` ✅.

| Symptom | `describe svc` / `get endpoints` shows | Cause → fix |
|---|---|---|
| `Connection refused` | **Endpoints `<none>`** | The selector matches no Ready Pod (typo, case, label only on the Deployment, other namespace, not Ready) → compare `Selector:` with `kubectl get pods --show-labels` |
| `Connection refused` | Endpoints with the **wrong port** (`…:8080`, nginx on 80) | **`targetPort` is wrong** → set it to the app's real port |
| `Connection refused` | Everything looks right | The app listens on **`127.0.0.1`** inside the container → make it listen on `0.0.0.0` |
| **Timed out** | Everything looks right | You called the **wrong Service port** → use the port from `kubectl get svc` |

**Debugging order:** `kubectl get svc` → `kubectl describe svc <name>` (Selector, TargetPort, **Endpoints**) → `kubectl get pods --show-labels` → test from a client Pod.
### 🛠 Do (~15 min)
```bash
code web-svc.yaml                                         # paste the web-demo YAML above, then save
kubectl apply -f web-svc.yaml                             # ("port is already allocated"? use 30081)
kubectl get svc web-demo                                  # NodePort … 8080:30080/TCP
kubectl describe svc web-demo          # Port http 8080/TCP · TargetPort 80/TCP · NodePort 30080 · Endpoints 10.244.0.102:80,…
kubectl get endpoints web-demo         # "Warning: v1 Endpoints is deprecated…" (harmless) + the same Pod-IP:80 list
kubectl get endpointslices -l kubernetes.io/service-name=web-demo   # the same IPs, newer format
kubectl run client --image=busybox:1.36 --restart=Never -- sleep 3600   # a client Pod that stays alive for an hour (then recreate it)
kubectl exec client -- wget -qO- -T 3 http://web-demo:8080 | head -4    # exec = run one command inside (like docker exec); -T 3 = 3 s limit
kubectl exec client -- wget -qO- -T 3 http://web-demo                   # wget: download timed out (the Service has no port 80)
```
**Break the selector:** in `web-svc.yaml` change `app: web` to `app: wbe` (a typo), then:
```bash
kubectl apply -f web-svc.yaml
kubectl get endpoints web-demo                                          # ENDPOINTS <none>
kubectl exec client -- wget -qO- -T 3 http://web-demo:8080              # can't connect to remote host … Connection refused
kubectl describe svc web-demo | grep -E 'Selector|Endpoints'            # grep -E 'A|B' = A or B → Selector: app=wbe, empty Endpoints
kubectl get pods --show-labels         # the Pods say app=web: mismatch found. Fix it back, apply, and check the endpoints again
```
**Break the targetPort:** change `targetPort: 80` to `8080`, apply, and run `kubectl get endpoints web-demo; kubectl exec client -- wget -qO- -T 3 http://web-demo:8080`: endpoints `…:8080`, `Connection refused` again. Same error, different cause; the Endpoints line tells them apart. Change it back to `80` and apply.
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
> **Problem:** a ClusterIP changes when a Service is re-created or the app moves to another cluster, and numbers are easy to mistype; a name like `http://backend:8080` works anywhere a Service `backend` exists. **Service discovery** = **how a program automatically finds the address of a service it needs**; in Kubernetes, mainly via **DNS**.

**Analogy:** the company's internal phone directory, updated automatically whenever a department opens or closes: staff look up "Accounts", never a desk extension.
- **CoreDNS** = the cluster's DNS server, running as Pods in `kube-system`, reached via the Service **`kube-dns`** (an old name, kept) at **`10.96.0.10`**. It **watches the API server**: create a Service and its name answers with its ClusterIP (an **A record**: name → IPv4 address); delete it and the name vanishes.
- The kubelet writes each Pod's **`/etc/resolv.conf`** (the Linux file naming the DNS server): `search default.svc.cluster.local svc.cluster.local cluster.local` · `nameserver 10.96.0.10` · `options ndots:5`.
  - **Search domains** = endings tried in turn on a short name (`web` → `web.default.svc.cluster.local`).
  - **`ndots:5`** = names with fewer than 5 dots try those endings first.
```
   web  .  default  .  svc  .  cluster.local     ← the FQDN (Fully Qualified Domain Name): the complete name, nothing left out
 Service   namespace  "a Service"  cluster domain (the default value)
```
| From a Pod in `default`, you type | It becomes | It reaches |
|---|---|---|
| `web` | `web.default.svc.cluster.local` | Service `web` in the **same namespace** |
| `web.shop` or `web.shop.svc` | `web.shop.svc.cluster.local` | Service `web` in the namespace **`shop`** |
| `web.shop.svc.cluster.local` | (already complete) | the same, from anywhere |

- ⚠️ **A short name works only in the same namespace:** from `shop`, plain `web` → `web.shop.svc.cluster.local` → `bad address 'web'`; use `web.default`. (Like Compose's service names, plus a namespace level, and pointing at a load-balancing Service rather than one container.)
- **Environment-variable discovery** (awareness): the kubelet injects `WEB_SERVICE_HOST=10.97.95.169` and `WEB_SERVICE_PORT=80` (upper-cased, `-` → `_`: `web-np` → `WEB_NP_…`) only for Services that **already existed** when the container started; later ones never appear. DNS has no such gap, so **DNS is preferred**.
- **busybox `nslookup` quirks:** it prints `** server can't find web.svc.cluster.local: NXDOMAIN` for every search ending that misses (**NXDOMAIN** = **N**on-e**X**istent **DOMAIN**): ignore those and read the `Name:` / `Address:` pair.
  - It may exit non-zero (`pod … terminated (Error)`) though the answer is right.
  - It **skips search endings for names with a dot** (`nslookup web.default` fails: use the full name; `wget http://web.default` is fine).
- **Three-step test** from a client Pod: `nslookup <service>` (DNS ok?) → `wget <ClusterIP>:<port>` (Service ok?) → `wget <Pod-IP>:<targetPort>` (app ok?). The first one that fails names the broken layer.
### 🛠 Do (~12 min)
```bash
kubectl run -it --rm dns-test --image=busybox:1.36 --restart=Never -- sh
cat /etc/resolv.conf                          # (inside) the search, nameserver 10.96.0.10 and ndots:5 lines
nslookup web                                  # NXDOMAIN lines, then Name: web.default.svc.cluster.local · Address: <web's CLUSTER-IP>
nslookup web.default.svc.cluster.local        # just the Name/Address answer
wget -qO- http://web.default.svc.cluster.local | head -4   # nginx's page via the full name
nslookup kube-dns.kube-system.svc.cluster.local            # Address: 10.96.0.10, the DNS Service itself
env | grep WEB                                # WEB_SERVICE_HOST=…, WEB_SERVICE_PORT=80, WEB_NP_…, WEB_DEMO_… (made earlier). Then: exit
kubectl create namespace dns-demo
kubectl run -it --rm far-client -n dns-demo --image=busybox:1.36 --restart=Never -- sh
wget -qO- -T 3 http://web                     # (inside) wget: bad address 'web' (no web Service in dns-demo)
wget -qO- -T 3 http://web.default | head -4   # works: it names the namespace. Then: exit
kubectl delete namespace dns-demo             # deletes everything inside it
kubectl get pods -n kube-system -l k8s-app=kube-dns   # one coredns-… Pod Running (the label keeps the old name)
kubectl get svc -n kube-system kube-dns       # 10.96.0.10  53/UDP,53/TCP,9153/TCP: DNS port 53 (mostly UDP, TCP for big answers); 9153 = metrics
kubectl delete pod client                     # clean up
kubectl delete svc web web-np web-demo        # (web-lb too, if made); keep "kubernetes"
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

## ➕ Small but important (Services, DNS, Pod networking)
| Item | What it is |
|---|---|
| `kubectl expose` | Creates a Service for a Deployment/Pod: `kubectl expose deployment web --port=80 --target-port=8080 --type=NodePort --name=web-np`. With no `--port` and no `containerPort`: `couldn't find port via --port flag or introspection` |
| `kubectl get svc -o wide` / `describe svc` | `-o wide` adds the **SELECTOR** column; `describe` shows Selector, Port, TargetPort, NodePort, **Endpoints** (debugging step one) |
| **NodePort range** | **30000–32767**. `nodePort: 8080` → `provided port is not in the valid range`. A `nodePort` on a ClusterIP Service is rejected too |
| **`sessionAffinity: ClientIP`** | "Sticky sessions": one client IP keeps going to the **same Pod** (default `None`). For apps keeping user data in memory |
| **`externalTrafficPolicy`** | NodePort/LoadBalancer. **`Cluster`** (default): any node → any Pod, but the app sees a node's IP. **`Local`**: only Pods on that node, so the **real client IP is kept** (nodes without such a Pod drop traffic) |
| **Headless Service + StatefulSet** | `clusterIP: None`; DNS returns every Pod IP, and each StatefulSet Pod gets a name like `db-0.db.prod.svc.cluster.local` |
| **Service without a selector** | Kubernetes creates **no** endpoints; you add an EndpointSlice yourself (e.g. to an outside database) to keep a normal Service name |
| **ExternalName** / `protocol` | ExternalName = just a CNAME (`type: ExternalName` + `externalName: mydb.example.com`). `protocol`: `TCP` (default), `UDP`, `SCTP` (**Stream Control Transmission Protocol**, rare) |
| **Multi-port Services** | Several ports (e.g. 80 and 443): then **every port needs a `name:`** |
| **NetworkPolicy** | By default every Pod can talk to every Pod. A NetworkPolicy = firewall rules selecting Pods by label, allowing only listed incoming (**ingress**) / outgoing (**egress**) traffic. Needs an enforcing CNI (Calico, Cilium; not Flannel alone) |
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
