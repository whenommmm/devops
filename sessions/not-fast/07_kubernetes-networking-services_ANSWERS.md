# Answer key · Session 07 · Pod networking, Services, DNS & service discovery

⚠️ Only open after answering.

---

## Warm-up

**W1.** A Deployment's ReplicaSet has `replicas: 3`. You run `kubectl delete pod web-7d9f8-abcde` (one of its Pods). What does `kubectl get pods` show about 10 seconds later?
- A) Two Pods, until you apply the YAML file again
- B) Three Pods, the deleted one back with its name
- C) Three Pods, one with a new name and young age
- D) Two Pods, and the ReplicaSet is marked Failed

**Correct: C.** The ReplicaSet keeps comparing desired (3) with actual (2) and creates a **replacement** straight away: that's self-healing. A deleted Pod is never "brought back": the replacement is a brand-new Pod, so it gets a new random name suffix and a young AGE (B is wrong). Nobody has to re-apply anything (A), and deleting a Pod doesn't make the ReplicaSet "fail" (D).

---

**W2.** You run `kubectl set image deployment/web nginx=nginx:1.99`. That tag doesn't exist, so the new Pods can't download their image (status `ImagePullBackOff`), while the old Pods keep running. Which command puts the Deployment back on the previous working version?
- A) `kubectl rollout restart deployment/web`
- B) `kubectl rollout undo deployment/web`
- C) `kubectl rollout status deployment/web`
- D) `kubectl rollout pause deployment/web`

**Correct: B.** `rollout undo` switches the Deployment back to the previous revision (the old ReplicaSet scales up again and the broken one scales down). `restart` (A) re-creates the Pods from the **current**, broken template, so they'd still fail to pull. `status` (C) only reports progress. `pause` (D) freezes the rollout where it is; it doesn't go back.

---

**W3.** What happens when you run this?
```bash
APP=web
kubectl get pods -l 'app=$APP'
```
- A) It lists Pods labelled app=web, as $APP expands
- B) It lists all Pods, as quoted selectors are ignored
- C) It lists nothing, as zsh deletes $APP from it
- D) It fails, as kubectl gets the literal text $APP

**Correct: D.** The **outer single quotes** stop the shell from expanding `$APP`, so kubectl sends the literal text `app=$APP`. A `$` isn't allowed in a label value, so the API server rejects it: `unable to parse requirement: … Invalid value: "$APP": a valid label must be an empty string or consist of alphanumeric characters…`. A would need double quotes (`"app=$APP"`). Quoted selectors aren't ignored (B), and zsh never touches anything inside single quotes (C).

---

## Check 54

**54.1** A Pod has two containers: `app` (listening on port 8080) and `helper`. How does `helper` reach `app`?
- A) At `localhost:8080`, as both share the Pod's IP
- B) At `app:8080`, through the container's DNS name
- C) At the Pod IP on port 80, mapped automatically
- D) Only through a Service that selects this Pod

**Correct: A.** All the containers in a Pod share one network namespace and one IP, so `localhost:8080` reaches `app`. B is Docker Compose thinking: in Kubernetes, container names aren't DNS names (DNS names belong to Services). C: nothing is remapped; the port stays 8080. D: Services are for reaching **other** Pods, not for talking inside one Pod.

---

**54.2** Which statement about Pod IPs is **true**?
- A) A Pod keeps its IP when its ReplicaSet replaces it
- B) Pods on different nodes need NAT to reach each other
- C) A replacement Pod usually comes up with a new IP
- D) Pod IPs are public, so a browser can open them

**Correct: C.** A replacement is a brand-new Pod, and it gets a new IP from the Pod CIDR: Pod IPs are ephemeral. A is exactly the misconception that makes Services necessary. B contradicts rule 2 of the network model (Pod to Pod, no NAT). D: Pod IPs are private (`10.244.x.x` on your minikube).

---

**54.3** (one line) What does a CNI plugin do? Name the one your minikube uses.

**Model answer:** *"A CNI (Container Network Interface) plugin builds the Pod network: when the kubelet starts a Pod, the plugin gives it a network interface and an IP from the Pod CIDR and sets up routes so every Pod can reach every other Pod. My minikube uses kindnet."* (Calico, Flannel, and Cilium are other examples.)

---

## Check 55

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

**Correct: D.** `8080:31474/TCP` means **Service port 8080 : nodePort 31474**. Inside the cluster you call the name and the Service port (`api:8080`); outside, you call any node's IP and the nodePort. A swaps the two ports. B: every NodePort Service also has a ClusterIP (`10.106.29.192` is right there). C: EXTERNAL-IP is `<none>`; only LoadBalancer Services get one.

---

**55.2** On your Mac, `curl http://$(minikube ip):31474` times out, although `kubectl get svc` shows that NodePort and the Pods are Running. Why?
- A) NodePorts open only while `minikube tunnel` is running
- B) NodePort Services accept traffic only from other Pods
- C) The Mac has no route to the node inside Colima's VM
- D) `minikube ip` prints the Service's ClusterIP, not a node's

**Correct: C.** With the docker driver, the node is a container inside Colima's VM, and `192.168.49.2` lives on a Docker network inside that VM. The Mac can't route there, so the connection times out. A: the NodePort is open all the time (you can `curl` it from inside the node); `minikube tunnel` is for LoadBalancer Services and Ingress. B: NodePorts exist precisely for outside traffic. D: `minikube ip` prints the **node's** IP.

---

**55.3** (one line) What does kube-proxy do, and does the traffic itself pass through the kube-proxy program?

**Model answer:** *"kube-proxy runs on every node, watches Services and their endpoints, and writes iptables (or IPVS) rules so that traffic sent to a ClusterIP is redirected to one of the backing Pods. The traffic doesn't pass through kube-proxy itself: the Linux kernel forwards it using those rules."*

---

## Check 56

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

**Correct: A.** A Pod must have **every** label in the selector. `back-end` and `backend` differ by one hyphen, so no Pod matches and the Endpoints list is empty. B: `READY 1/1` shows the Pod is Ready. C: `matchLabels` is Deployment/ReplicaSet syntax; Service selectors are plain `key: value`. D: the selector only needs to be a **subset** of the Pod's labels; `pod-template-hash` is added by the Deployment and is never needed.

---

**56.2** A Service has `port: 80` and `targetPort: 8080`. The Deployment says `containerPort: 8080`, but the app inside actually listens on **3000**. Client Pods get `Connection refused` on `http://svc:80`. What's the fix?
- A) Set `containerPort: 3000` in the Deployment
- B) Set `targetPort: 3000` in the Service YAML
- C) Set `port: 3000` in the Service YAML
- D) Set `nodePort: 3000` in the Service YAML

**Correct: B.** `targetPort` decides where traffic is delivered on the Pod, so it must be the port the process really listens on: 3000. A is the tempting one, but `containerPort` is only documentation: changing it doesn't change where the Service sends traffic (still 8080). C changes the port clients call, not where traffic lands. D: a nodePort is for outside access, and 3000 isn't even in the 30000–32767 range.

---

**56.3** (one line) Explain `port` vs `targetPort` in contrast form.

**Model answer:** *"`port` is the port the Service itself listens on, which clients call (e.g. `web:8080`), whereas `targetPort` is the port on the Pods that the traffic is delivered to, which must match the port the app really listens on."*

---

## Check 57

**57.1** A Pod in the namespace `shop` runs `curl http://payments:9000`. The `payments` Service exists only in the namespace `billing`. What happens?
- A) It works, since Service names are unique cluster-wide
- B) The name doesn't resolve; it needs `payments.billing`
- C) It works, but only when both Pods are on one node
- D) DNS answers, but traffic between namespaces is blocked

**Correct: B.** The short name is tried as `payments.shop.svc.cluster.local`, which doesn't exist, so the lookup fails. `payments.billing` (or the full FQDN) works. A: Service names are unique only **within** a namespace. C: which node a Pod runs on has nothing to do with DNS. D: by default, traffic between namespaces is allowed (only a NetworkPolicy would block it), and here DNS doesn't even answer.

---

**57.2** Which is the FQDN of the Service `db` in the namespace `prod`?
- A) `prod.db.svc.cluster.local`
- B) `db.svc.prod.cluster.local`
- C) `db.prod.pod.cluster.local`
- D) `db.prod.svc.cluster.local`

**Correct: D.** The format is `<service>.<namespace>.svc.cluster.local`. A swaps the Service and the namespace. B puts `svc` in the wrong place. C uses `pod`, which belongs to the (rarely used) Pod DNS names.

---

**57.3** (one line) Why is DNS preferred over the `<NAME>_SERVICE_HOST` environment variables?

**Model answer:** *"DNS resolves a Service's name at any time, including Services created after the Pod started, whereas the `<NAME>_SERVICE_HOST` variables are set only when the container starts, so they never include Services created later (and they only cover the Pod's own namespace)."*

---

## Session quiz

**Q1.** A Flask app runs `app.run(host="127.0.0.1", port=5000)` in its container. Its Service has `port: 80` and `targetPort: 5000`, and `kubectl get endpoints` lists three `<Pod-IP>:5000` entries. Client Pods calling `http://svc` get `Connection refused`. What's the cause?
- A) The selector matches no Pods, so nothing can answer
- B) The app accepts connections only from its own Pod
- C) targetPort must equal port, so both must be 80
- D) containerPort 5000 is missing from the Pod spec

**Correct: B.** `127.0.0.1` is the loopback address (networking session), so only programs in the same network namespace (the same Pod) can connect. kube-proxy delivers traffic to `<Pod-IP>:5000`, where nothing is listening, so the Pod refuses it. The fix is `host="0.0.0.0"`. A: endpoints are listed, so the selector does match. C: `port` and `targetPort` are allowed to differ; that's their whole point. D: `containerPort` is documentation, and leaving it out never blocks traffic.

---

**Q2.** A Service `shop` has `type: NodePort`, `port: 8080`, `targetPort: 80` and `nodePort: 30080`, and selects nginx Pods. Which pair of requests **both** succeed?
- A) From a Pod: `shop:80` · from a node: `localhost:30080`
- B) From a Pod: `shop:30080` · from a node: `localhost:30080`
- C) From a Pod: `shop:8080` · from a node: `localhost:30080`
- D) From a Pod: `shop:8080` · from a node: `localhost:80`

**Correct: C.** Inside the cluster, clients use the Service port (`shop:8080`). The nodePort (30080) is opened on **every node**, so a request to `localhost:30080` on a node works. (On your minikube, `minikube ssh`, then `curl localhost:30080`, proves it.) A: the Service has no port 80; that's the Pods' targetPort. B: the nodePort lives on **nodes**, not on the ClusterIP, so a Pod calling `shop:30080` times out. D: nothing about this Service opens port 80 on a node.

---

**Q3.** A Pod in the namespace `web` started at 10:00. At 10:05, a Service `cache` is created in the namespace `web`. At 10:10, the Pod runs `wget http://cache` and `echo $CACHE_SERVICE_HOST`. What happens?
- A) Both work: DNS answers and the variable holds the IP
- B) Both fail until the Pod is restarted or recreated
- C) The variable is set, but DNS waits for a restart
- D) DNS resolves `cache`, but the variable stays empty

**Correct: D.** CoreDNS answers for a Service as soon as it exists, so `wget http://cache` works. The `CACHE_SERVICE_HOST` variable would only have been injected if the Service had existed when the container **started** (10:00), so it's empty. A forgets the start-time rule for variables. B: DNS needs no restart. C gets both halves backwards.

---

**Q4.** Which statement about Services is **NOT** true?
- A) kube-proxy load-balances a headless Service
- B) A NodePort Service also gets its own ClusterIP
- C) A ClusterIP is virtual: no network card owns it
- D) A LoadBalancer Service also opens a nodePort

**Correct: A.** A headless Service (`clusterIP: None`) has **no** virtual IP, so kube-proxy writes no rules for it: DNS returns the Pod IPs and the client picks one itself. B, C, and D are all true: the types build on each other (on your minikube, a LoadBalancer Service showed `80:30587/TCP`, a nodePort included), and a ClusterIP exists only as forwarding rules.

---

**L1.** Explain ClusterIP vs NodePort vs LoadBalancer in contrast form, saying who can reach each one.

**Model answer:** *"A ClusterIP Service is reachable only from inside the cluster through its virtual IP, whereas a NodePort Service also opens the same port (30000–32767) on every node so outside clients can use `<node-IP>:<nodePort>`, and a LoadBalancer Service also gets a cloud load balancer with an external IP that forwards to those nodePorts, so it's reachable from the internet."*

---

**L2.** Write the full forms of **CNI** and **FQDN**, and write the FQDN of the Service `api` in the namespace `prod`.

**Model answer:** *"CNI = Container Network Interface; FQDN = Fully Qualified Domain Name. The FQDN is `api.prod.svc.cluster.local`."*

---

## ✍️ Write the command: answers
**C1.** Forward port 8080 on your Mac to port 80 of the Service `web`.
`kubectl port-forward svc/web 8080:80`: `port-forward` = tunnel a local port into the cluster through the API server · `svc/web` = the Service `web` (it picks **one** of its Pods) · `8080:80` = Mac port : Service port (like Docker's `-p HOST:CONTAINER`); it keeps the terminal busy until Ctrl + C. (Alternative: `kubectl port-forward service/web 8080:80`.)

**C2.** Create a **NodePort** Service named `web-np` for the Deployment `web`. The Service listens on port 80 and sends traffic to port 8080 on the Pods.
`kubectl expose deployment web --type=NodePort --port=80 --target-port=8080 --name=web-np`: `expose deployment web` = make a Service that copies `web`'s selector · `--type=NodePort` = also open a port (30000–32767) on every node · `--port=80` = the Service's own port · `--target-port=8080` = the port on the Pods · `--name=web-np` = the Service's name (default would be `web`). (Alternative: write a Service YAML with `type: NodePort`, `port: 80`, `targetPort: 8080` and `kubectl apply -f` it.)

**C3.** Show only the `Selector` and `Endpoints` lines from the details of the Service `web-demo`.
`kubectl describe svc web-demo | grep -E 'Selector|Endpoints'`: `describe svc web-demo` = the Service's full details · `|` = pipe that text into the next command · `grep -E` = filter lines using extended patterns · `'Selector|Endpoints'` = keep lines containing Selector **or** Endpoints (single quotes stop the shell treating `|` as a pipe).

**C4.** From the already-running Pod `client`, download `http://web-demo:8080` quietly to the screen with a 3-second time limit.
`kubectl exec client -- wget -qO- -T 3 http://web-demo:8080`: `exec client` = run one command inside the Pod `client` (like `docker exec`) · `--` = everything after this is the command to run inside · `wget` = download a URL · `-q` = quiet · `O-` = write the page to the screen · `-T 3` = give up after 3 seconds · `http://web-demo:8080` = the Service name and **Service port**.
