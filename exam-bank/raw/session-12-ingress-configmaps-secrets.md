# Labs · Session 12: Ingress, ConfigMaps and Secrets

## Lab 1 · Deploy an app and route traffic to the right Service by host and URL path with an Ingress
**Answer (what to write in the exam):**
```bash
minikube addons enable ingress                       # install the NGINX Ingress controller
kubectl create deployment web --image=nginx:1.27
kubectl expose deployment web --port=80
kubectl create deployment api --image=kicbase/echo-server:1.0
kubectl expose deployment api --port=80 --target-port=8080
kubectl apply -f shop-ingress.yaml
kubectl get ingress shop-ingress                     # wait until ADDRESS is filled
curl -H "Host: shop.example.com" http://$(minikube ip)/      # nginx welcome page (web)
curl -H "Host: shop.example.com" http://$(minikube ip)/api   # "Request served by api-..." (api)
```
`shop-ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```
**What each line does:**
- `minikube addons enable ingress`: an Ingress object is only a set of rules; this installs the **controller** (an NGINX proxy Pod) that reads the rules and actually forwards traffic. No controller = rules do nothing.
- `create deployment` + `expose`: two apps, each behind its own ClusterIP Service (`--target-port=8080` because echo-server listens on 8080).
- `apiVersion: networking.k8s.io/v1` / `kind: Ingress`: the current Ingress API.
- `ingressClassName: nginx`: "this rule is for the NGINX controller".
- `host: shop.example.com`: only requests whose HTTP (HyperText Transfer Protocol) `Host` header is this name match.
- `path: /api` + `pathType: Prefix`: any URL (Uniform Resource Locator) path starting with `/api` goes to Service `api`; everything else (`/`) goes to `web`. The longest matching prefix wins.
- `curl -H "Host: ..."`: fakes the domain name, so you don't need a real DNS (Domain Name System) record.
**Verify:** `kubectl describe ingress shop-ingress` → Rules list `shop.example.com /  web:80` and `/api  api:80`; the two `curl` calls return different apps.
(Mac with Docker driver: run `minikube tunnel` in a second terminal and curl `http://127.0.0.1/` instead of `$(minikube ip)`.)

**🧠 In simple words:** An Ingress is one front door (a single IP address) for many Services. The controller reads the host name and the URL path of every request and sends it to the matching Service. In short: Ingress = HTTP routing rules, the controller = the thing that obeys them.

## Lab 2 · Create a ConfigMap, inject it into a Deployment as environment variables, verify inside the Pod
**Answer (what to write in the exam):**
```bash
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod
kubectl get configmap app-config -o yaml             # see the key/value data
kubectl create deployment web --image=nginx:1.27 --dry-run=client -o yaml > web.yaml
# edit web.yaml: add the envFrom block under the container (shown below)
kubectl apply -f web.yaml
kubectl rollout status deployment/web
kubectl exec deploy/web -- env | grep APP_
```
`web.yaml` (final):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          envFrom:
            - configMapRef:
                name: app-config
```
**What each line does:**
- `create configmap ... --from-literal=KEY=value`: stores plain, non-secret settings as key/value pairs in the cluster.
- `--dry-run=client -o yaml > web.yaml`: generates a Deployment file without creating anything, so you don't type it from scratch.
- `envFrom: - configMapRef: name: app-config`: every key in the ConfigMap becomes an environment variable in the container (`APP_COLOR`, `APP_MODE`).
- `kubectl exec deploy/web -- env`: runs `env` inside a Pod of the Deployment; `grep APP_` filters our variables.
- (One variable only: use `env: - name: APP_COLOR  valueFrom: configMapKeyRef: {name: app-config, key: APP_COLOR}`.)
**Verify:** `kubectl exec deploy/web -- env | grep APP_` → `APP_COLOR=blue` and `APP_MODE=prod`.
Note: env vars are read only when the container starts, so after changing the ConfigMap run `kubectl rollout restart deployment/web`.

**🧠 In simple words:** A ConfigMap keeps settings outside the image, so the same image runs in dev and prod with different values. Kubernetes copies those values into the container as environment variables when it starts. In short: settings live in the ConfigMap, the Pod reads them as env vars.

## Lab 3 · Create a Secret for a credential and consume it in a Pod without hard-coding it in the manifest
**Answer (what to write in the exam):**
```bash
kubectl create secret generic db-secret --from-literal=DB_PASSWORD='ChangeMe123'
kubectl get secret db-secret -o yaml                 # value is base64, not plain text
kubectl apply -f db-client.yaml
kubectl rollout status deployment/db-client
kubectl exec deploy/db-client -- printenv DB_PASSWORD
```
`db-client.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db-client
  template:
    metadata:
      labels:
        app: db-client
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "sleep 3600"]
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
```
**What each line does:**
- `create secret generic db-secret --from-literal=...`: stores the credential in a Secret object (only in the cluster, never in the YAML file you commit). In real use prefer `--from-file=DB_PASSWORD=./pass.txt` so the password is not left in shell history.
- `secretKeyRef: name: db-secret key: DB_PASSWORD`: "fill env var `DB_PASSWORD` from key `DB_PASSWORD` of Secret `db-secret`". The manifest holds only a reference, not the password.
- `printenv DB_PASSWORD`: proves the running container received the value.
- `get secret -o yaml`: shows `DB_PASSWORD: Q2hhbmdlTWUxMjM=`; that is base64 **encoding**, not encryption (anyone can decode it), so limit access with RBAC (Role-Based Access Control).
**Verify:** `kubectl exec deploy/db-client -- printenv DB_PASSWORD` → `ChangeMe123`; `grep -i password db-client.yaml` shows only the reference names, no value.
(Decode check: `kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d`.)

**🧠 In simple words:** The password is stored once in a Secret, and the Deployment only says "take it from that Secret". So the file you share or put in Git never contains the real credential. In short: Secret holds the value, `secretKeyRef` points to it.
