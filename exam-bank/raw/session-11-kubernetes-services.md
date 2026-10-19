# Labs · Session 11 · Kubernetes Services (ClusterIP, NodePort, port vs targetPort)

## Lab 1 · Deploy an app with several replicas, add a ClusterIP Service that selects its Pods, verify endpoints
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3
kubectl rollout status deployment/web
kubectl get pods -l app=web -o wide --show-labels       # note the 3 Pod IPs
kubectl expose deployment web --port=80 --target-port=80   # ClusterIP is the default type
kubectl get svc web                                     # TYPE ClusterIP, CLUSTER-IP 10.x, 80/TCP
kubectl describe svc web                                # Selector app=web, Endpoints
kubectl get endpoints web                               # the same 3 Pod IPs, each :80
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://web
```
Same Service as a manifest (`kubectl apply -f web-svc.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```
**What each line does:**
- `create deployment … --replicas=3`: 3 nginx Pods, each labelled `app=web`.
- `expose deployment web`: creates Service `web` that copies the Deployment's selector `app=web`; `--port` = Service port, `--target-port` = container port.
- `selector: app: web`: the Service sends traffic to every *ready* Pod with this label. A typo here = no endpoints.
- `get svc`: shows the stable virtual CLUSTER-IP (only reachable inside the cluster).
- `get endpoints web`: list of `PodIP:port` behind the Service (on Kubernetes 1.33+ it prints a deprecation warning; `kubectl get endpointslices -l kubernetes.io/service-name=web` is the new form).
- `kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://web`: temporary Pod inside the cluster; `web` is resolved by cluster DNS (Domain Name System) to the ClusterIP; `--rm` deletes the Pod afterwards.

**Verify:** `kubectl get endpoints web` → `10.244.0.11:80,10.244.0.12:80,10.244.0.13:80` (matches `get pods -o wide`); the busybox `wget` prints the "Welcome to nginx!" HTML.

**🧠 In simple words:** Pod IPs change every time a Pod is recreated, so clients should not use them. A ClusterIP Service gives one fixed IP and DNS name (`web`, or `web.default.svc.cluster.local` from another namespace) and load-balances to whichever Pods currently match the label. In short: label the Pods → Service selects that label → endpoints list the Pod IPs.

## Lab 2 · Create a NodePort Service and reach the app through the node's port
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3     # skip if it exists from Lab 1
kubectl expose deployment web --name=web-np --type=NodePort --port=80 --target-port=80
kubectl get svc web-np                    # TYPE NodePort, PORT(S) 80:3xxxx/TCP
kubectl get endpoints web-np              # Pod IPs :80
kubectl get nodes -o wide                 # INTERNAL-IP of a node, e.g. 192.168.49.2
NODE_PORT=$(kubectl get svc web-np -o jsonpath='{.spec.ports[0].nodePort}')
curl http://<NODE_IP>:$NODE_PORT          # minikube on Linux: curl http://$(minikube ip):$NODE_PORT
```
Manifest with a fixed node port (`kubectl apply -f web-np.yaml`, instead of the `expose` line):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-np
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```
**What each line does:**
- `--type=NodePort`: opens the same port (range 30000–32767) on EVERY node; traffic to `NodeIP:nodePort` → Service → a Pod. It also still gets a ClusterIP.
- `PORT(S) 80:30080/TCP`: left = Service port inside the cluster, right = node port from outside (TCP, Transmission Control Protocol).
- `nodePort: 30080`: choose the port yourself; leave it out and Kubernetes picks a random free one.
- `jsonpath='{.spec.ports[0].nodePort}'`: reads the assigned node port into a shell variable.
- `curl http://<NODE_IP>:<nodePort>`: proves access from outside the cluster.
- (Mac, minikube Docker driver: the node IP is not reachable from macOS, so run `minikube service web-np --url` and keep that terminal open; it prints `http://127.0.0.1:<port>` to curl. `kubectl port-forward svc/web-np 8080:80` also works, but it bypasses the node port.)

**Verify:** `kubectl get svc web-np` → `NodePort … 80:30080/TCP`; `curl http://192.168.49.2:30080` (or the `minikube service … --url` address) → "Welcome to nginx!".

**🧠 In simple words:** ClusterIP only works inside the cluster; NodePort additionally opens a high port on every node so someone outside can reach the app with `node-IP:node-port`. It is the simple way to test external access without a cloud load balancer. In short: type NodePort → read the 3xxxx port → curl NodeIP:port.

## Lab 3 · Service port and targetPort differ; prove traffic reaches the container port
**Answer (what to write in the exam):** the app listens on **8080**, clients use Service port **80**.
```bash
kubectl apply -f hello.yaml
kubectl rollout status deployment/hello
kubectl get svc hello                      # PORT(S) 80/TCP
kubectl get endpoints hello                # PodIP:8080  <- the targetPort
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://hello        # :80 works
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 http://hello:8080   # fails
```
`hello.yaml` (imperative alternative for the Service: `kubectl expose deployment hello --port=80 --target-port=8080`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: echo
          image: kicbase/echo-server:1.0
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 8080
```
**What each line does:**
- `image: kicbase/echo-server:1.0` + `containerPort: 8080`: a small HTTP (HyperText Transfer Protocol) server that listens on 8080 and replies with the request it received.
- `port: 80`: the port clients use on the Service (ClusterIP / DNS name `hello`).
- `targetPort: 8080`: the port on the Pod the Service forwards to; it must equal the port the app really listens on.
- `get endpoints hello`: shows `PodIP:8080`, the proof that the Service targets the container port.
- `wget http://hello` (port 80) → reply "Request served by hello-…"; `wget http://hello:8080` fails, because the Service itself only listens on 80.

**Verify:** `kubectl get svc hello` → `80/TCP`; `kubectl get endpoints hello` → `10.244.0.x:8080,10.244.0.y:8080`; busybox `wget -qO- http://hello` → "Request served by hello-xxxxx". (Other common pair: nginx with `port: 8080`, `targetPort: 80`.)

**🧠 In simple words:** `port` is the door on the Service, `targetPort` is the door on the container, and kube-proxy forwards from one to the other, so they do not have to be the same number. If `targetPort` is wrong, the Service still exists but requests fail (connection refused). In short: clients hit Service:80 → kube-proxy → Pod:8080, checked with `get endpoints` and a busybox `wget`.
