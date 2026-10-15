# Session 08 · Kubernetes IV: Ingress, ConfigMaps & Secrets (Oct 14, Session 2)

**Time:** ~90–100 min · **Concepts:** 58 → 61 · **Answers:** `MY_ANSWERS.md` or say "Ingress questions" · **Answer key:** `08_ingress-configmaps-secrets_ANSWERS.md`

**Before you start:**
```bash
minikube status                       # everything Running (else: minikube start)
kubectl get deploy,svc                # Session 07 leftovers? kubectl delete deployment web; kubectl delete svc web web-np web-demo
minikube addons list | grep ingress   # is the Ingress add-on switched on? (Concept 58)
mkdir -p ~/devops-practice/k8s-ingress-config && cd ~/devops-practice/k8s-ingress-config
```
> ⚠️ Never delete the `minikube` or `localstack` containers, and don't delete anything in the `ingress-nginx`, `kube-system`, `argocd` or `final-app` namespaces. If `kubectl get ingress -A` shows Ingresses elsewhere, leave them alone. Work only in `default`.

## 🔁 Warm-up
**W1.** A Service `api` in the namespace `shop` has `type: ClusterIP`, `port: 80`, and `targetPort: 8080`. A Pod in the namespace `web` wants to call it. Which URL works?
- A) `http://api:8080`
- B) `http://api.shop:80`
- C) `http://api.shop:8080`
- D) `http://api:80`

**W2.** `kubectl get endpoints web` shows `<none>`, yet `kubectl get pods -l app=web` lists 3 Pods that are Running and Ready, in the same namespace. What's the most likely cause?
- A) The selector has a label those Pods don't have
- B) The targetPort doesn't match the containerPort
- C) kube-proxy hasn't written its iptables rules yet
- D) CoreDNS hasn't made a DNS record for it yet

**W3.** A shop moves its website to a new domain for good. Which status code should the old URLs return, so browsers and search engines update their saved links?
- A) 302, as the visitor is sent elsewhere
- B) 304, as the old page hasn't changed
- C) 301, as the move is permanent
- D) 307, as the method must be kept

## Concept 58 · Ingress and the Ingress controller
**Words you'll learn:** Ingress resource, Ingress controller, `ingress-nginx`, reverse proxy, OSI model (Open Systems Interconnection), Layer 4 (L4) vs Layer 7 (L7), IngressClass, `ingressClassName`, minikube addon, Host header, virtual hosting, `X-Forwarded-For`, echo server.
> **Problem:** Riya's `web`, `api` and `admin` must be reachable from the internet. Three LoadBalancer Services = 3 public IPs and 3 monthly bills, 3 places to renew HTTPS certificates, and no way to send `shop.example.com/api/…` to the API (a LoadBalancer sees only IPs and ports, never URLs).
>
> **Idea:** **Ingress** ("the way in"; the opposite is *egress*) is Kubernetes' **reverse proxy** (networking session): **one entry point that receives all HTTP/HTTPS traffic and routes each request to the right Service by host name and URL path.**

**Analogy: a shopping mall.** Instead of each shop having its own street entrance and guard, there's one main entrance with an information desk: the **directory board** is the Ingress resource, and the **desk staff** who read it are the controller.
- **Ingress resource** (`kind: Ingress`) = a YAML object holding **routing rules only** ("host `shop.example.com`, path `/api` → Service `api`, port 80"). On its own it **does nothing**.
- **Ingress controller** = the **actual reverse-proxy program**, running as Pods, that **watches** every Ingress and reconfigures itself to follow them: the **NGINX Ingress Controller** (`ingress-nginx`, minikube's), Traefik, HAProxy, the AWS Load Balancer Controller.
  - ⚠️ **A plain cluster has no controller:** Ingresses are stored but get no ADDRESS and route nothing. "My Ingress does nothing" → check the controller first.
  - The controller itself sits behind **one** Service: one cloud LoadBalancer (one bill, not three); on minikube, the NodePort Service `ingress-nginx-controller`.
```
 Browser ─► ONE entry point ─► Ingress controller Pod (NGINX: reads every Ingress; looks at the Host header + path)
                                  ├─ shop.example.com/       ─► Service web   ─► web Pods
                                  └─ shop.example.com/api/…  ─► Service api   ─► api Pods   (admin.example.com ─► admin …)
```
- **OSI model** (**Open Systems Interconnection**: networking as 7 stacked layers). Services/kube-proxy work at **Layer 4 (L4)**, transport: TCP/UDP, IPs and **ports** only, HTTPS passed through untouched, any TCP/UDP app.
  - Ingress works at **Layer 7 (L7)**, application: it reads the HTTP request (host, path, headers, cookies), routes by **host and path**, and can decrypt HTTPS ("terminate" TLS, Concept 59).
  - ⚠️ **Ingress doesn't replace Services:** every rule points at a Service, whose endpoints lead to the Pods.
- **IngressClass** = a small object naming one controller (yours: class **`nginx`** → `k8s.io/ingress-nginx`); `spec.ingressClassName: nginx` picks it.
  - A **default** class takes Ingresses with no class; with no default and no `ingressClassName`, **no controller picks it up**. (The old annotation `kubernetes.io/ingress.class` is deprecated.) **Always write `ingressClassName`.**
- **Host header** = the site name in every HTTP request (`Host: shop.example.com`), filled in by browsers from the URL; it lets **one IP serve many websites** (**virtual hosting**), and it's what the controller reads.
  - Without real DNS, curl sets it: `curl -H "Host: shop.example.com" http://localhost:8080/` (`-H` = add/replace a header).
- **On your minikube:** `minikube addons enable ingress` (an **addon** = an optional extra minikube installs) puts the NGINX controller in namespace `ingress-nginx`.
  - As with NodePorts, the node is inside Colima's VM, so this session **port-forwards to the controller** and uses `curl -H "Host: …"`. *(Untested alternative: `minikube tunnel`, which asks for your Mac password since ports 80/443 are privileged, then `curl -H "Host: shop.example.com" http://127.0.0.1/`.)*
### 🛠 Do (~15 min)
```bash
minikube addons list | grep ingress   # │ ingress │ minikube │ enabled ✅ │ (if disabled: minikube addons enable ingress, 1–2 min)
kubectl get pods,svc,ingressclass -n ingress-nginx   # controller Pod 1/1 Running ("admission" Pods Completed = finished setup jobs) · NodePort 80:31119,443:32766 · nginx (default)
for app in web api; do kubectl create deployment $app --image=kicbase/echo-server:1.0 --port=8080; kubectl expose deployment $app --port=80 --target-port=8080; done   # 2 apps + Services (80 → 8080)
kubectl get deploy,svc                # web, api 1/1 · both ClusterIP 80/TCP. echo-server = tiny app replying with WHICH Pod answered + the request it got
kubectl create ingress shop --class=nginx --rule="shop.example.com/*=web:80" --dry-run=client -o yaml > ingress.yaml   # * = Prefix; open it: rules → host → http.paths → backend.service
kubectl apply -f ingress.yaml
kubectl get ingress                   # shop  nginx  shop.example.com  192.168.49.2  80 (ADDRESS can take ~1 min)
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80   # in Tab 2 (Cmd + T): Mac 8080 → the controller; leave it running for Concept 59
curl -H "Host: shop.example.com" http://localhost:8080/   # Request served by web-… · X-Forwarded-For: 127.0.0.1 (proxy headers keep the original client/host)
curl -i http://localhost:8080/       # -i = show headers. Host "localhost" matches no rule → HTTP/1.1 404 Not Found (<center>nginx</center>)
```
### ✅ Check 58
**58.1** On a fresh cluster, an engineer applies a valid Ingress. `kubectl get ingress` shows no ADDRESS, and requests to the nodes time out. The Pods and Services work fine. What's the most likely cause?
- A) The backend Services must be type LoadBalancer
- B) Every backend Service needs its own NodePort
- C) No Ingress controller is running to act on it
- D) Ingress rules need IP addresses, not host names

**58.2** Which statement correctly contrasts a Service with an Ingress?
- A) Services route by URL path; an Ingress by port only
- B) Ingress routes by host/path; Services by IP/port
- C) Ingress replaces Services; backends need no Service
- D) Services decrypt TLS; Ingress passes HTTPS untouched

## Concept 59 · Host- and path-based routing, and TLS
**Words you'll learn:** host-based routing, path-based routing, `pathType` (`Prefix`, `Exact`, `ImplementationSpecific`), path segment, backend, default backend, TLS termination, CA (Certificate Authority), self-signed certificate, CN (Common Name), SAN (Subject Alternative Name), `kubernetes.io/tls` Secret, SNI (Server Name Indication), 308 redirect, cert-manager, Let's Encrypt.
> **Problem:** through **one** controller and **one** IP, Riya wants `shop.example.com` → `web`, `shop.example.com/api/…` → `api`, `api.example.com` → `api`, and everything on **HTTPS**.

**Analogy:** the mall's security desk opens sealed parcels at the entrance (decrypts), reads the labels (host and path), and sends the contents to the right shop on the internal trolley (plain HTTP).
- **Host-based routing:** a rule's `host` is compared with the Host header (both names can point at one IP; the header alone tells them apart); a rule **without** `host` matches every host; wildcards like `*.example.com` are allowed.
- **Path-based routing:** under a host, `paths` send URL paths to different Services (each a **backend**). When several match, the **longest** wins (`/api/orders` → `/api`, not `/`).
  - ⚠️ The path reaches the backend **unchanged** (`/api/orders`, not `/orders`; see `rewrite-target` in ➕).

| `pathType` | How it matches | `path: /api` matches | Doesn't match |
|---|---|---|---|
| **`Prefix`** | The path and everything **below** it, by **segment** (a part between `/`s) | `/api`, `/api/`, `/api/orders` | `/apix`, `/apis` |
| **`Exact`** | Only that exact path | `/api` | `/api/`, `/api/orders` |
| **`ImplementationSpecific`** | Up to the controller (NGINX uses it for regex paths) | depends | depends |

**The controller's own answers:** no rule matches → **404 Not Found** (unless `spec.defaultBackend` names a Service for everything else) · a rule matches but its Service is missing or has no Ready Pods → **503 Service Temporarily Unavailable** · TLS is on and the request used `http://` → **308 Permanent Redirect** to `https://`.
```yaml
apiVersion: networking.k8s.io/v1        # the worked example: two hosts, two Services, TLS
kind: Ingress
metadata:
  name: shop
spec:
  ingressClassName: nginx
  tls:                                  # HTTPS for these hosts…
    - hosts:
        - shop.example.com
        - api.example.com
      secretName: shop-tls              # …using the certificate in this Secret
  rules:
    - host: shop.example.com            # rule 1: the shop's own name
      http:
        paths:
          - path: /api                  # /api and everything below it → api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /                     # everything else → web
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
    - host: api.example.com             # rule 2: a separate name for the API
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```
- **TLS termination** = **where the encrypted connection ends**. The **controller** holds the certificate (signed by a **CA**, a **Certificate Authority** browsers trust) and private key, decrypts each request, reads host and path (only possible after decrypting) and forwards plain HTTP to the Pods:
  - `Browser ══ HTTPS ══► controller (cert + key) ── HTTP ──► Service ──► Pods`. Benefits: apps need no certificates; renewals happen in one place.
  - The certificate lives in a **Secret of type `kubernetes.io/tls`** with exactly two keys, **`tls.crt`** and **`tls.key`** (`kubectl create secret tls`), in the **same namespace as the Ingress**, valid for the `tls.hosts` names: its **CN** (**Common Name**, the main name) and **SAN** list (**Subject Alternative Name**: every extra name).
- **SNI** (**Server Name Indication**) = the host name the client sends **in plain text** during the TLS handshake, so the controller can pick a certificate before decrypting (the Host header is inside the encryption).
  - No match → NGINX's **"Kubernetes Ingress Controller Fake Certificate"**. So HTTPS tests use `curl --resolve` (real name in SNI and Host, while connecting to `127.0.0.1`).
- **308 redirect:** with `tls:`, plain HTTP gets **`308 Permanent Redirect`** to `https://…` (308 = permanent like 301, but the method stays the same; 307 = the temporary one, like 302). Switch it off: annotation `nginx.ingress.kubernetes.io/ssl-redirect: "false"`.
- **Self-signed certificate** = signed by its own key, not a trusted CA: curl and browsers warn; curl **`-k`** (`--insecure`) skips the check (testing only).
- **cert-manager** (awareness) = a controller that gets real certificates from a CA like **Let's Encrypt** (free, automated), stores them in the TLS Secret and renews them; add an annotation such as `cert-manager.io/cluster-issuer: letsencrypt-prod`.
### 🛠 Do (~15 min)
*openssl:*
- *`-x509` = a finished self-signed certificate (not a request for a CA) · `-nodes` = "no DES" (**Data Encryption Standard**): no password on the key · `-newkey rsa:2048` = a new 2048-bit **RSA** (Rivest–Shamir–Adleman) key*
- *`-days 365` = one year · `-keyout`/`-out` = where to save · `-subj` = the CN · `-addext` = the SAN list · `\` = continues on the next line.*
```bash
openssl req -x509 -nodes -newkey rsa:2048 -days 365 -keyout tls.key -out tls.crt \
  -subj "/CN=shop.example.com" -addext "subjectAltName=DNS:shop.example.com,DNS:api.example.com"   # progress dots; ls → tls.crt tls.key
kubectl create secret tls shop-tls --cert=tls.crt --key=tls.key   # secret/shop-tls created
kubectl get secret shop-tls                                       # kubernetes.io/tls, DATA 2
kubectl describe secret shop-tls                                  # tls.crt: 1168 bytes, tls.key: 1704 bytes (sizes only)
code ingress.yaml                     # replace everything with the worked example above, then save
kubectl apply -f ingress.yaml         # configured (= updated)
kubectl get ingress shop              # HOSTS shop.example.com,api.example.com · PORTS 80, 443
kubectl describe ingress shop         # TLS: shop-tls terminates …; each rule with its real Pod, e.g. /api  api:80 (10.244.0.154:8080)
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80 8443:443   # Tab 2: Ctrl + C the old one first; Mac 8443 → HTTPS
curl -k --resolve shop.example.com:8443:127.0.0.1 https://shop.example.com:8443/api/orders   # --resolve = "this name:port → 127.0.0.1" → api-…, GET /api/orders
curl -k --resolve shop.example.com:8443:127.0.0.1 https://shop.example.com:8443/apix         # web-… (Prefix compares whole segments)
curl -k --resolve api.example.com:8443:127.0.0.1 https://api.example.com:8443/               # api-… (host-based routing)
curl -i -H "Host: shop.example.com" http://localhost:8080/   # HTTP/1.1 308 Permanent Redirect · Location: https://shop.example.com
curl --resolve shop.example.com:8443:127.0.0.1 https://shop.example.com:8443/   # no -k → curl: (60) SSL certificate problem: self signed certificate
curl -kv --resolve shop.example.com:8443:127.0.0.1 https://shop.example.com:8443/ 2>&1 | grep "subject:"   # -v details on stderr, 2>&1 pipes them → subject: CN=shop.example.com
```
### ✅ Check 59
**59.1** An Ingress has two rules for host `shop.example.com`: `/api` (Prefix) → `api`, and `/` (Prefix) → `web`. Where do these three requests go: `/api/v1/items`, `/apis`, `/`?
- A) api, api, web
- B) web, web, web
- C) api, web, 404
- D) api, web, web

**59.2** An Ingress in the namespace `shop` has `tls:` with `secretName: shop-tls`. HTTPS clients get the "Kubernetes Ingress Controller Fake Certificate", and the controller's logs say the Secret can't be found. Yet `kubectl get secret shop-tls -n default` shows it. Why?
- A) The Secret has to be in the Ingress's namespace
- B) TLS Secrets must live in the ingress-nginx namespace
- C) The Secret must be of type Opaque, not TLS
- D) The Secret's name must match the host name exactly

**59.3** (one line) What does "TLS termination at the Ingress" mean? Give one benefit.

## Concept 60 · ConfigMaps
**Words you'll learn:** configuration, hard-coding, Twelve-Factor App, ConfigMap, `data`, `--from-literal`, `--from-file`, `--from-env-file`, `valueFrom`/`configMapKeyRef`, `envFrom`/`configMapRef`, ConfigMap volume, `volumeMounts`, `mountPath`, `kubectl patch`, `CreateContainerConfigError`, MiB (mebibyte).
> **Problem:** Riya **hard-codes** her settings (writes them straight into the image with `ENV`), so production's `DB_HOST=db-prod` needs a rebuild: production no longer runs the image she tested, and every change means rebuild → push → redeploy.
>
> **Idea:** the **Twelve-Factor App** (12 rules for cloud apps), factor III "Config": **store configuration in the environment, not in the code**. One image everywhere; only the attached configuration changes.

**Analogy: a restaurant chain.** Every branch uses the same recipe book (the image) but has its own notice board (the ConfigMap) for local opening hours and specials; changing the board doesn't mean reprinting the book.
- A **ConfigMap** is **a Kubernetes object that stores non-secret configuration as key–value pairs**. Values are always **text** (a word, or a **whole file** with the file name as key).
  - It has **`data:`**, not `spec:`; Pods can use only ConfigMaps in **their own** namespace; the limit is **1 MiB** (**mebibyte** = 1,048,576 bytes).
  - Create one from literals (`--from-literal=APP_COLOR=blue`), from a file (`--from-file=settings.properties`: file name = key; `--from-file=mykey=…` renames it), or from an env file (`--from-env-file=app.env`: each `KEY=VALUE` line → its own key).
  - Or from YAML: `kind: ConfigMap` + `data:` with `KEY: value` lines, where `settings.properties: |` holds a whole file (`|` keeps line breaks).
  - ⚠️ **Quote numbers and booleans:** `MAX_ITEMS: 50` is a YAML **number**, rejected with `json: cannot unmarshal number into Go struct field ConfigMap.data of type string`; write `"50"` (same for `true`/`false`, and for `value:` under a container's `env:`).

| How a Pod uses it (see the Pod below) | The app sees | When the ConfigMap changes |
|---|---|---|
| (a) `env` + `valueFrom.configMapKeyRef` = **one** key → a variable you name · (b) `envFrom` + `configMapRef` = **every** key → a same-named variable | `$COLOR` · `$APP_COLOR`, `$APP_MODE` | **Nothing changes** until the container restarts (env vars are set once, at start) |
| (c) a `configMap` **volume** + `volumeMounts` = every key → a **file** in `mountPath` | `/etc/config/APP_COLOR` | Files **update automatically** after a delay (~1 min on minikube): good for whole config files |

- ⚠️ A missing ConfigMap or key → **`CreateContainerConfigError`** (`describe pod`: `configmap "app-confg" not found`); `optional: true` allows it to be missing.
- ⚠️ You **can't add env vars to a running Pod** (`pod updates may not change fields other than …`): re-create it. In a Deployment, editing the template triggers a rolling update, and **`kubectl rollout restart deployment/<name>`** reloads a changed ConfigMap into env vars.
```yaml
apiVersion: v1                                 # save as cfg-demo.yaml
kind: Pod
metadata:
  name: cfg-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]      # just stay alive for an hour
      env:                                     # (a) ONE key → one variable, with a name you choose
        - name: COLOR
          valueFrom:
            configMapKeyRef:
              name: app-config                 # which ConfigMap
              key: APP_COLOR                   # which key in it
      envFrom:                                 # (b) EVERY key → a variable with the same name
        - configMapRef:
            name: app-config
      volumeMounts:                            # (c) every key → a FILE in this folder
        - name: config-vol
          mountPath: /etc/config               # app-config's keys appear here
        - name: files-vol
          mountPath: /etc/app                  # app-files' key appears here
  volumes:
    - name: config-vol
      configMap:
        name: app-config
    - name: files-vol
      configMap:
        name: app-files
```
### 🛠 Do (~15 min)
```bash
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod   # configmap/app-config created
kubectl get configmap app-config                               # DATA 2 (= keys)
kubectl get configmap app-config -o yaml                       # data: APP_COLOR: blue … (plain, readable text)
printf 'log_level=info\ncache=true\n' > settings.properties    # \n = new line → a two-line file
kubectl create configmap app-files --from-file=settings.properties
kubectl describe configmap app-files                           # one key, settings.properties
kubectl create configmap demo --from-literal=APP_COLOR=blue --dry-run=client -o yaml   # only pretend; prints ready-made YAML (add > demo.yaml)
code cfg-demo.yaml                                             # paste the Pod above, then save
kubectl apply -f cfg-demo.yaml
kubectl get pod cfg-demo                                       # wait until Running
kubectl exec cfg-demo -- env | grep -E 'COLOR|APP_'            # COLOR=blue, APP_COLOR=blue, APP_MODE=prod (grep -E 'A|B' = either)
kubectl exec cfg-demo -- ls /etc/config                        # APP_COLOR APP_MODE (a file per key)
kubectl exec cfg-demo -- cat /etc/config/APP_COLOR             # blue (no newline after it)
kubectl exec cfg-demo -- cat /etc/app/settings.properties      # the two lines
kubectl patch configmap app-config -p '{"data":{"APP_COLOR":"green"}}'   # change only listed fields; outer single quotes keep the JSON's "
kubectl exec cfg-demo -- cat /etc/config/APP_COLOR             # blue; after ~1 min: file green
kubectl exec cfg-demo -- env | grep APP_COLOR                  # blue; after ~1 min: env STILL blue
```
### ✅ Check 60
**60.1** A Deployment's Pods read `APP_MODE` through `envFrom` from the ConfigMap `app-config`. You change `APP_MODE` to `debug` and `kubectl apply` the ConfigMap. Ten minutes later, the app still sees `prod`. What's the reason, and the fix?
- A) The kubelet refreshes env vars hourly; wait for it
- B) ConfigMaps need `immutable: true` before changes apply
- C) envFrom works only with Secrets; use configMapKeyRef
- D) Env vars are set at start; restart the Pods

**60.2** 🐞 `kubectl apply -f cm.yaml` fails with `json: cannot unmarshal number into Go struct field ConfigMap.data of type string`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: prod
  MAX_ITEMS: 50
```
What's the fix?
- A) Add a `spec:` line above `data:`
- B) Write it as `MAX_ITEMS: "50"`
- C) Rename `data:` to `stringData:`
- D) Write it as `MAX_ITEMS=50`

**60.3** (one line) Name the three ways a Pod can use a ConfigMap, and say which one picks up changes without a restart.

## Concept 61 · Secrets and environment variables
**Words you'll learn:** Secret, Secret types (`Opaque`, `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson`), `imagePullSecrets`, base64, encoding vs encryption, `data` vs `stringData`, `secretKeyRef`, `secretRef`, tmpfs, RBAC (Role-Based Access Control), encryption at rest, HashiCorp Vault, External Secrets Operator, Sealed Secrets.
> **Problem:** a `DB_PASSWORD` in a ConfigMap is plain text for anyone who can read ConfigMaps (`-o yaml`, `describe`), and it sits in Git. **Idea:** a **Secret** is **like a ConfigMap, but for sensitive data** (passwords, tokens, keys, certificates): used almost the same way, handled more carefully.

**Analogy:** base64 is like Morse code (**encoding** = rewriting data in another alphabet: anyone can reverse it, no key needed), while **encryption** is a locked safe (scrambled; unreadable without the **key**). Secrets use base64 only so binary data (keys, certificates) fits into YAML text: **it is not protection**.

| | **ConfigMap** | **Secret** |
|---|---|---|
| Stored as | Plain text | base64 (encoding, **not** encryption) |
| `kubectl describe` shows | The values | Only the sizes |
| As a volume | Normal files | Files in **tmpfs** (RAM; never on the node's disk) |
| Extra protection | None | Sent only to nodes that need it; **RBAC** (**Role-Based Access Control**: rules on who may do what); **encryption at rest** in etcd (**off by default**) |

- **Types:**
  - **`Opaque`** (default; any key–value pairs; `kubectl create secret generic`)
  - **`kubernetes.io/tls`** (exactly `tls.crt` + `tls.key`; `kubectl create secret tls`; Ingress TLS)
  - **`kubernetes.io/dockerconfigjson`** (a registry login; `kubectl create secret docker-registry`; for **private images**, listed by name under the Pod spec's `imagePullSecrets:` list, else `ImagePullBackOff`).
- **`data` vs `stringData`** in Secret YAML: values under `data:` must **already be base64** (`DB_PASSWORD: UzNjcjN0IQ==`); values under `stringData:` are plain text the API server encodes for you (write-only: reading back shows everything under `data:`).
  - ⚠️ Plain text under `data:` is a classic bug: `S3cr3t!` is rejected (`illegal base64 data at input byte 6`), but `password` is **accepted** (its letters are valid base64) and the app gets **garbled bytes**.
  - ⚠️ **Use `echo -n`:** `echo 'S3cr3t!' | base64` also encodes the hidden newline (`UzNjcjN0IQo=` instead of `UzNjcjN0IQ==`), so the password gets an invisible newline and logins fail.
- **Using a Secret** = like a ConfigMap: `valueFrom.secretKeyRef` · `envFrom.secretRef` · a volume with `secret:` + ⚠️ **`secretName:`** (not `name:`). Inside the container, values are **already decoded**.
  - **Files beat env vars** for secrets: env vars leak into logs, error reports and child processes.
- **Good practice (examined):** never commit Secret YAML to Git (`.gitignore`; history keeps it forever); limit `get`/`list` with RBAC; turn on encryption at rest.
  - Use an external manager (**HashiCorp Vault**, AWS Secrets Manager) synced in by the **External Secrets Operator**, or **Sealed Secrets** (an encrypted "SealedSecret" that's safe in Git; only the in-cluster controller decrypts it).
- ⚠️ `kubectl apply` keeps a copy of the file in the `last-applied-configuration` annotation, so `stringData` passwords become readable there (`kubectl create secret` doesn't do this).
```yaml
apiVersion: v1                                 # save as app-pod.yaml
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config                   # APP_COLOR, APP_MODE
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secret
          readOnly: true
  volumes:
    - name: secret-vol
      secret:
        secretName: db-secret                  # secretName, not name!
```
### 🛠 Do (~15 min)
```bash
echo -n 'S3cr3t!' | base64                            # UzNjcjN0IQ== (single quotes keep ! safe from zsh)
echo 'S3cr3t!' | base64                               # UzNjcjN0IQo= (newline!)
echo 'UzNjcjN0IQ==' | base64 -d                       # S3cr3t! (-d = decode)
kubectl create secret generic db-secret --from-literal=DB_USER=admin --from-literal=DB_PASSWORD='S3cr3t!'   # kubectl does the base64
kubectl get secret db-secret                                      # Opaque 2
kubectl describe secret db-secret                                 # DB_PASSWORD: 7 bytes, DB_USER: 5 bytes (sizes only)
kubectl get secret db-secret -o yaml                              # data: DB_PASSWORD: UzNjcjN0IQ==, DB_USER: YWRtaW4=
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d   # S3cr3t!: base64 hid nothing
code app-pod.yaml                                     # paste the Pod above, then save
kubectl apply -f app-pod.yaml
kubectl get pod app-pod
kubectl exec app-pod -- env | grep -E 'APP_|DB_'      # APP_COLOR=green (it started after your patch!), APP_MODE=prod, DB_PASSWORD=S3cr3t!
kubectl exec app-pod -- ls /etc/secret                # DB_PASSWORD DB_USER
kubectl exec app-pod -- cat /etc/secret/DB_PASSWORD   # S3cr3t! (decoded)
kubectl exec app-pod -- df -h /etc/secret             # tmpfs … /etc/secret: the files are in RAM (df -h = disk usage, human sizes)
sed 's/name: app-pod/name: broken-pod/; s/key: DB_PASSWORD/key: password/' app-pod.yaml > broken-pod.yaml   # sed = replace text: a copy with a wrong key
kubectl apply -f broken-pod.yaml
kubectl get pod broken-pod                            # STATUS CreateContainerConfigError
kubectl describe pod broken-pod | grep Warning        # Error: couldn't find key password in Secret default/db-secret. Then: kubectl delete pod broken-pod
kubectl create secret docker-registry regcred --docker-server=ghcr.io --docker-username=riya --docker-password=FAKE --dry-run=client -o yaml   # look only: kubernetes.io/dockerconfigjson
kubectl delete pod cfg-demo app-pod                   # clean up (Ctrl + C any port-forward)
kubectl delete configmap app-config app-files
kubectl delete secret db-secret shop-tls
kubectl delete ingress shop                           # keep the ingress add-on: Kubernetes VI uses it
kubectl delete svc web api
kubectl delete deployment web api
rm tls.key
```
### ✅ Check 61
**61.1** A teammate says: "Our Secret YAML files are safe in the Git repo, because the values are base64." What's the best response?
- A) False: base64 is an encoding that anyone can reverse
- B) True: base64 can only be decoded with the cluster's key
- C) True, as long as the Secret's type is set to Opaque
- D) False: base64 is fine, but Git keeps every old version

**61.2** 🐞 A Pod is stuck in `CreateContainerConfigError`. `kubectl describe pod` says `Error: couldn't find key password in Secret default/db-secret`. `kubectl get secret db-secret -o yaml` shows the keys `DB_USER` and `DB_PASSWORD`. What's the fix?
- A) Base64-encode the word `password` in the Pod spec
- B) Change the Secret's type to `kubernetes.io/basic-auth`
- C) Set the Pod's `secretKeyRef` key to `DB_PASSWORD`
- D) Add `imagePullSecrets: db-secret` to the Pod spec

**61.3** (one line) Why should you use `echo -n` when you base64-encode a password by hand?

## ➕ Small but important (Ingress, ConfigMaps, Secrets)
| Item | What it is |
|---|---|
| `--dry-run=client -o yaml` | Generate YAML without creating anything: `kubectl create secret generic db --from-literal=PASS=x --dry-run=client -o yaml > secret.yaml` (also ConfigMaps, Deployments, Services, Ingresses) |
| **`immutable: true`** | Top-level field locking a ConfigMap/Secret: changes are refused (`field is immutable when immutable is set`); delete and re-create instead. Prevents accidents, eases API-server load |
| **`optional: true`** | On `configMapKeyRef`/`secretKeyRef`/`envFrom`: start anyway if the ConfigMap/Secret is missing (no `CreateContainerConfigError`) |
| **`env` beats `envFrom`** | If one variable name comes from both, the `env` value wins |
| **Ingress annotations** | Controller-specific settings in `metadata.annotations`, e.g. `nginx.ingress.kubernetes.io/rewrite-target: /$2` with `path: /api(/\|$)(.*)` + `pathType: ImplementationSpecific` turns `/api/orders` into `/orders`; `…/ssl-redirect: "false"` stops the 308 |
| **404 vs 503 from the controller** | **404**: no rule matched host/path. **503**: a rule matched, but its Service is missing or has no Ready Pods. Check `kubectl describe ingress` (each path's backend + Pod IPs, or `<error: services "api" not found>`) and `kubectl get endpoints <svc>` |
| `kubectl create ingress` | `--rule="shop.example.com/api*=api:80"`: a trailing `*` means `Prefix`, none means `Exact`. Default IngressClass = annotation `ingressclass.kubernetes.io/is-default-class: "true"` |
| **Gateway API** | The newer successor to Ingress, one object per role: **GatewayClass**, **Gateway** (the entry point), **HTTPRoute** (the rules). `ingress-nginx` announced its retirement (maintenance ending March 2026); the Ingress API itself stays |
| **Namespace rule** | Pods use only ConfigMaps/Secrets in **their own** namespace; an Ingress only a TLS Secret in **its own** namespace |

**Quick checks:** (1) Write a command that prints a Secret's YAML (`PASS=x`) without creating it. (2) An Ingress returns `503` for `/api` but `200` for `/`. What do you check first?
*Answers:* (1) `kubectl create secret generic db --from-literal=PASS=x --dry-run=client -o yaml`. (2) That the Service named in the `/api` rule exists and has endpoints: `kubectl describe ingress <name>` and `kubectl get endpoints <service>`.

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory
**Q1.** `curl -H "Host: shop.example.com" http://localhost:8080/api` returns `503 Service Temporarily Unavailable`, while `/` returns 200. `kubectl describe ingress shop` shows `/api   api:80 (<error: services "api" not found>)`. What's the cause?
- A) No rule matches `/api`, so the default backend replies
- B) The Service named in the `/api` rule doesn't exist
- C) TLS isn't set up, so the controller rejects `/api`
- D) pathType Prefix can't match a path such as `/api`

**Q2.** A Pod mounts the ConfigMap `cfg` at `/etc/cfg` and also loads it with `envFrom`. You patch its key `MODE` from `a` to `b`. Two minutes later, what do `cat /etc/cfg/MODE` and `echo $MODE` print inside the container?
- A) `b` and `b`
- B) `a` and `b`
- C) `b` and `a`
- D) `a` and `a`

**Q3.** After you add a `tls:` section to an Ingress, `curl -i -H "Host: shop.example.com" http://localhost:8080/` returns `308 Permanent Redirect`. What's happening?
- A) The controller now redirects plain HTTP to HTTPS
- B) The tls Secret is missing, so NGINX sends you away
- C) The Service's port changed from 80 to port 443
- D) 308 means the page moved temporarily to /tls

**Q4.** A Secret's YAML has `DB_PASSWORD: password` under `data:`. `kubectl apply` succeeds, but the app can't log in to its database. Why?
- A) Secrets need `type: kubernetes.io/tls` to be loaded
- B) Secret env vars always reach the app base64-encoded
- C) The kubelet hides keys whose names contain PASSWORD
- D) `data` wants base64, so `password` decodes to junk

**L1.** Explain Ingress (the resource) vs Ingress controller in contrast form.

**L2.** Explain ConfigMap vs Secret in contrast form, and say why base64 isn't security.
