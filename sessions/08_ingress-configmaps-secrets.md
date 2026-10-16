# Session 08 · Kubernetes IV: Ingress, ConfigMaps & Secrets (Oct 14, Session 2)

**Time:** ~90–100 min · **Concepts:** 58 → 61 · **Answers:** `MY_ANSWERS.md` or say "Ingress questions" · **Answer key:** `08_ingress-configmaps-secrets_ANSWERS.md`

**Before you start:**
```bash
minikube status                       # everything Running?
kubectl get deploy,svc                # leftovers from last session?
minikube addons list | grep ingress   # Ingress add-on on?
mkdir -p ~/devops-practice/k8s-ingress-config && cd ~/devops-practice/k8s-ingress-config
```
**What each line does:**
- `minikube status`: checks that your local cluster is up: `host`, `kubelet` and `apiserver` should all say `Running` (if not: `minikube start`).
- `kubectl get deploy,svc`: lists Deployments and Services (the comma = both kinds at once). If old `web` practice objects are left, delete them: `kubectl delete deployment web` and `kubectl delete svc web web-np web-demo`.
- `minikube addons list | grep ingress`: lists minikube's optional extras (addons); `|` sends that list into `grep`, which keeps only the lines containing `ingress`. It shows whether the Ingress add-on (the NGINX Ingress controller) is switched on.
- `mkdir -p … && cd …`: makes this session's practice folder (`-p` = also create parent folders, no error if it exists) and moves into it (`&&` = only if the first part worked).

> ⚠️ Never delete the `minikube` or `localstack` containers, and don't delete anything in the `ingress-nginx`, `kube-system`, `argocd` or `final-app` namespaces. If `kubectl get ingress -A` shows Ingresses elsewhere, leave them alone. Work only in `default`. (Delete those leftovers because this session reuses the names `web` and `api`.)

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

### The problem, and the idea
Riya's website (`web`), API (`api`) and admin panel (`admin`) must be reachable from the internet. Using Services alone, each would be a **LoadBalancer** Service (one that asks the cloud for its own load balancer and public IP). In a cloud that means **3 load balancers** (3 public IPs, **3 monthly bills**), **3 places** to renew HTTPS certificates, and **no way** to send `shop.example.com/api/…` to the API, because a LoadBalancer only sees IP addresses and ports, never URLs.

A **reverse proxy** is a server that sits in front of other servers, receives every request, and forwards each one to the right server. Kubernetes' version is **Ingress** (*ingress* means "the way in"; its opposite, *egress*, means "the way out"):

> **Ingress** lets **one entry point** receive all the HTTP/HTTPS traffic for a cluster and **route each request to the right Service**, based on the **host name** and the **URL path**.

**Analogy: a shopping mall.** Instead of every shop having its own street entrance and guard, the mall has **one main entrance** with an **information desk**: "Shoes? Second floor."

### The two parts (you always need both)
1. **The Ingress resource** (`kind: Ingress`): a YAML object that holds **routing rules only**, like "host `shop.example.com`, path `/api` → Service `api`, port 80". On its own, it **does nothing**. *In the mall:* the **directory board**.
2. **The Ingress controller**: the **actual reverse-proxy program**, running as Pods. It **watches** every Ingress and reconfigures itself to follow the rules. Examples: the **NGINX Ingress Controller** (`ingress-nginx`, the one minikube installs), Traefik, HAProxy. *In the mall:* the **desk staff** who read the board and send people the right way.

⚠️ **A plain Kubernetes cluster comes with no Ingress controller.** Your Ingress objects are accepted and stored, but **nothing happens**: no ADDRESS, no routing. "My Ingress does nothing" → first check that a controller is running.
```
 Browser ──► ONE entry point ──► Ingress controller Pod (NGINX: reads every Ingress, looks at the Host header + path)
                                   ├─ shop.example.com/        ──► Service web    ──► web Pods
                                   ├─ shop.example.com/api/…   ──► Service api    ──► api Pods
                                   └─ admin.example.com        ──► Service admin  ──► admin Pods
```
The controller itself is reached through **one** Service: in a cloud, a single LoadBalancer (one bill instead of three); on your minikube, a NodePort Service called `ingress-nginx-controller`.

### Layer 4 vs Layer 7
The **OSI model** (**Open Systems Interconnection**) describes networking as 7 stacked layers. **Layer 4 (L4)**, transport, is TCP/UDP: IP addresses and **ports**. **Layer 7 (L7)**, application, is the actual **HTTP** request: host name, URL path, headers, cookies.

| | **Service** (kube-proxy) | **Ingress** (controller) |
|---|---|---|
| Works at · routes by | L4 · IP address and port only | L7 · host name and URL path |
| HTTPS | Passes it through untouched | Can decrypt it ("terminate" TLS: the encrypted connection ends here) |
| Works for | Any TCP/UDP app (databases, DNS…) | Web traffic only |

⚠️ **Ingress doesn't replace Services.** Every Ingress rule points at a **Service**, and the controller uses that Service's endpoints to find the Pods.

### IngressClass, the Host header, and your minikube
- **IngressClass:** a cluster can run more than one controller, and an **IngressClass** is a small object that names one. On your cluster, the class **`nginx`** belongs to the controller `k8s.io/ingress-nginx`, and **`spec.ingressClassName: nginx`** in an Ingress says "this one is for NGINX". One class can be the **default** (it takes Ingresses with no class); with no default and no `ingressClassName`, **no controller picks your Ingress up**. **Always write `ingressClassName`.**
- **Host header:** every HTTP request carries the name of the site the client wants (`Host: shop.example.com`), filled in by the browser from the URL. It lets **one IP address serve many websites** (**virtual hosting**), and it's exactly what the controller reads. Without real DNS, curl can set it: `curl -H "Host: shop.example.com" http://localhost:8080/` (`-H` = add or replace a header).
- **On your minikube:** `minikube addons enable ingress` installs the NGINX controller into the namespace `ingress-nginx` (an **addon** = an optional extra that minikube installs for you). As with NodePorts, the node lives inside Colima's VM (your Mac has no route to it), so this session **port-forwards to the controller** and uses `curl -H "Host: …"`. *(Untested alternative: `minikube tunnel`, which asks for your Mac password since ports 80/443 are privileged, then `curl -H "Host: shop.example.com" http://127.0.0.1/`.)*

### 🛠 Do (~15 min)
**`kicbase/echo-server`** is a tiny web server that replies with **which Pod answered** and **the exact request it received**: perfect for seeing where the Ingress sent you.
```bash
minikube addons list | grep ingress   # enabled?
kubectl get pods,svc,ingressclass -n ingress-nginx   # the controller + its class
kubectl create deployment web --image=kicbase/echo-server:1.0 --port=8080
kubectl expose deployment web --port=80 --target-port=8080                  # Service web: 80 → 8080
kubectl create deployment api --image=kicbase/echo-server:1.0 --port=8080
kubectl expose deployment api --port=80 --target-port=8080
kubectl create ingress shop --class=nginx --rule="shop.example.com/*=web:80" --dry-run=client -o yaml > ingress.yaml   # only WRITES the YAML
kubectl apply -f ingress.yaml
kubectl get ingress                   # shop  nginx  shop.example.com …
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80   # in Tab 2; leave it running
curl -H "Host: shop.example.com" http://localhost:8080/   # served by web-…
curl -i http://localhost:8080/        # 404 Not Found
```
**What each line does:**
- `minikube addons list | grep ingress`: should show `│ ingress │ minikube │ enabled ✅ │`. If it says disabled, run `minikube addons enable ingress` and wait 1–2 minutes.
- `kubectl get pods,svc,ingressclass -n ingress-nginx`: three kinds at once in the namespace `ingress-nginx` (`-n` = namespace): the controller Pod `1/1 Running` (the "admission" Pods `Completed` are finished setup jobs), its NodePort Service `80:31119,443:32766`, and the IngressClass `nginx` marked `(default)`.
- `kubectl create deployment web --image=kicbase/echo-server:1.0 --port=8080`: a Deployment running the echo server; `--image` = the image; `--port` = sets `containerPort: 8080` (where the echo server listens).
- `kubectl expose deployment web --port=80 --target-port=8080`: a ClusterIP Service `web`. `--port` = the Service's own port (80); `--target-port` = the Pods' port, where the app really listens (8080).
- The two `api` lines: the same pair again, for a second app called `api`.
- `kubectl create ingress shop --class=nginx --rule="shop.example.com/*=web:80" … > ingress.yaml`: writes (doesn't create) an Ingress named `shop`. `--class` = the IngressClass; `--rule` = `host/path=service:port`, and the `*` at the end means pathType `Prefix`; `--dry-run=client` = pretend only; `-o yaml` = print as YAML; `>` = save into the file. Open it to see the nesting: rules → host → paths → backend.
- `kubectl apply -f ingress.yaml`: creates the Ingress from the file (`-f` = file).
- `kubectl get ingress`: `shop  nginx  shop.example.com  192.168.49.2  80`. The ADDRESS can take about a minute to appear: the controller is picking the Ingress up.
- `kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80`: (in Tab 2, Cmd + T) carries your Mac's port 8080 to the controller's port 80. It keeps the terminal busy; leave it running.
- `curl -H "Host: shop.example.com" http://localhost:8080/`: sends a request whose Host header is `shop.example.com` (`-H` = add or replace a header), as a browser would. The echo server replies `Request served by web-…` and shows the request it got, including `X-Forwarded-For: 127.0.0.1`: reverse proxies add these `X-Forwarded-…` headers so the app can still learn the original client and host.
- `curl -i http://localhost:8080/`: `-i` = also show the response headers. Now the Host is `localhost`, which matches no rule, so the controller itself answers `HTTP/1.1 404 Not Found`.

*Leave the port-forward running for the routing and TLS steps below.*

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

### Routing by host and by path
Through **one** controller and **one** IP, Riya wants `shop.example.com` → `web`, `shop.example.com/api/…` → `api`, `api.example.com` → `api` too (for the mobile app), and everything on **HTTPS**.
- **Host-based routing:** each rule can have a **`host`**, which the controller compares with the request's Host header. In real DNS both names would point at the **same** IP; the Host header alone tells them apart. A rule **without** a `host` matches every host (wildcards like `*.example.com` work too).
- **Path-based routing:** under each host, `paths` send different URL paths to different Services (each target Service is called a **backend**). When several paths match, the **longest (most specific) one wins**: `/api/orders` matches both `/` and `/api`, so it goes to `/api`.

Every path needs a **`pathType`**:

| `pathType` | How it matches | `path: /api` matches | Doesn't match |
|---|---|---|---|
| **`Prefix`** | The path and everything **below** it, compared **segment by segment** (a **segment** is a part between `/`s) | `/api`, `/api/`, `/api/orders` | `/apix`, `/apis` |
| **`Exact`** | Only that exact path | `/api` | `/api/`, `/api/orders` |
| **`ImplementationSpecific`** | Up to the controller (NGINX uses it for regex paths) | depends | depends |

⚠️ **The path reaches the backend unchanged:** the API receives `/api/orders`, not `/orders`. If the app expects `/orders`, you need the `rewrite-target` annotation (➕ section).

**When the controller answers by itself:**
- No rule matches the host/path → **404 Not Found**, unless `spec.defaultBackend` names a Service to receive everything else (e.g. a friendly "page not found" app).
- A rule matches, but its Service doesn't exist or has no Ready Pods → **503 Service Temporarily Unavailable**.
- TLS is set up for that host, and the request used `http://` → **308 Permanent Redirect** to `https://` (below).

### The worked example: two hosts, two Services, TLS
```yaml
apiVersion: networking.k8s.io/v1        # Ingress lives in the networking API group
kind: Ingress
metadata:
  name: shop
spec:
  ingressClassName: nginx               # handled by the NGINX controller
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
                name: api               # the backend Service…
                port:
                  number: 80            # …and its port
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
          - path: /                     # every path → api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

### TLS termination
**HTTPS** is HTTP inside **TLS** encryption: the server proves who it is with a **certificate**, signed by a **CA** (**Certificate Authority**, an organisation browsers trust), and keeps a matching **private key** secret. **TLS termination** means **the place where the encrypted connection ends** (gets decrypted). With Ingress, the **controller** holds the certificate and key: it decrypts each request, reads the host and path (possible only **after** decrypting), and forwards plain HTTP to the Pods.
```
 Browser ══ HTTPS (encrypted) ══► Ingress controller (holds cert + key; decrypts here) ── plain HTTP ──► Service ──► Pods
```
**Benefits:** the apps need no certificates at all, certificates are renewed in **one** place, and the controller can route by path. **Analogy:** the mall's security desk opens sealed parcels at the entrance, reads the labels, and sends the contents to the right shop on the internal trolley.
- **Where the certificate lives:** in a **Secret of type `kubernetes.io/tls`** with exactly two keys, **`tls.crt`** (the certificate) and **`tls.key`** (the private key), made with `kubectl create secret tls`. It must be in the **same namespace as the Ingress**, and valid for the names in `tls.hosts`: its **CN** (**Common Name**, the main name) and its **SAN** list (**Subject Alternative Name**: every extra name).
- **SNI (Server Name Indication):** the Host header is **inside** the encryption, so how does the controller pick a certificate **before** decrypting? The client sends the host name in plain text during the TLS handshake: that's SNI. If it matches no `tls.hosts`, NGINX shows its built-in **"Kubernetes Ingress Controller Fake Certificate"**. So HTTPS tests use `curl --resolve`, which puts the real name in both SNI and the Host header while connecting to `127.0.0.1`.
- **The 308 redirect:** once an Ingress has `tls:`, NGINX answers its plain-HTTP requests with **`308 Permanent Redirect`** to `https://…`. 308 is permanent like 301, but guarantees the method stays the same (a POST stays a POST); 307 is the temporary version, like 302. Switch it off with the annotation `nginx.ingress.kubernetes.io/ssl-redirect: "false"`.
- **Self-signed certificate:** signed by its own key instead of a trusted CA. Fine for practice, but browsers and curl warn about it; curl's **`-k`** (`--insecure`) skips the check. **Testing only!**
- **cert-manager** (awareness): a controller that automatically gets real certificates from a CA such as **Let's Encrypt** (free and automated), stores them in the TLS Secret and renews them. You add an annotation like `cert-manager.io/cluster-issuer: letsencrypt-prod` to the Ingress.

### 🛠 Do (~15 min)
```bash
openssl req -x509 -nodes -newkey rsa:2048 -days 365 -keyout tls.key -out tls.crt \
  -subj "/CN=shop.example.com" -addext "subjectAltName=DNS:shop.example.com,DNS:api.example.com"
```
*`-x509` = a finished **self-signed** certificate (not a request for a CA) · `-nodes` = no password on the key · `-newkey rsa:2048` = a new RSA key (the most common type) · `-days 365` = valid for a year · `-keyout`/`-out` = where to save · `-subj` = the CN · `-addext` = the SAN list · `\` = the command continues on the next line. After some progress dots, `ls` shows `tls.crt` and `tls.key`.*
```bash
kubectl create secret tls shop-tls --cert=tls.crt --key=tls.key   # secret/shop-tls created
kubectl get secret shop-tls           # TYPE kubernetes.io/tls, DATA 2
kubectl describe secret shop-tls      # sizes only
code ingress.yaml                     # replace everything with the worked example above, then save
kubectl apply -f ingress.yaml         # shop configured (= updated)
kubectl get ingress shop              # PORTS 80, 443
kubectl describe ingress shop         # TLS + each rule's Pods
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80 8443:443   # in Tab 2: Ctrl + C the old one first
curl -k --resolve shop.example.com:8443:127.0.0.1 https://shop.example.com:8443/api/orders   # api-…
curl -k --resolve shop.example.com:8443:127.0.0.1 https://shop.example.com:8443/apix         # web-…
curl -k --resolve api.example.com:8443:127.0.0.1 https://api.example.com:8443/               # api-…
curl -i -H "Host: shop.example.com" http://localhost:8080/   # 308 Permanent Redirect
curl --resolve shop.example.com:8443:127.0.0.1 https://shop.example.com:8443/   # no -k → certificate error
curl -kv --resolve shop.example.com:8443:127.0.0.1 https://shop.example.com:8443/ 2>&1 | grep "subject:"   # CN=shop.example.com
```
**What each line does:**
- `kubectl create secret tls shop-tls --cert=tls.crt --key=tls.key`: stores your certificate (`--cert`) and private key (`--key`) in a Secret of type `kubernetes.io/tls` named `shop-tls`.
- `kubectl get secret shop-tls`: TYPE `kubernetes.io/tls`, DATA `2` (the two keys `tls.crt` and `tls.key`).
- `kubectl describe secret shop-tls`: `tls.crt: 1168 bytes`, `tls.key: 1704 bytes`: sizes only, as `describe` never prints a Secret's values.
- `code ingress.yaml`: open the file, replace everything with the worked example above, and save.
- `kubectl apply -f ingress.yaml`: `ingress.networking.k8s.io/shop configured` (= the existing Ingress was updated).
- `kubectl get ingress shop`: HOSTS `shop.example.com,api.example.com` and PORTS `80, 443` (443 because of `tls:`).
- `kubectl describe ingress shop`: shows `TLS: shop-tls terminates …` and each rule with the Pod it reaches, e.g. `/api  api:80 (10.244.0.154:8080)`.
- `kubectl port-forward … 8080:80 8443:443`: (in Tab 2: press Ctrl + C on the old port-forward first) two ports now: Mac 8080 → HTTP (80), and Mac 8443 → HTTPS (443).
- `curl -k --resolve shop.example.com:8443:127.0.0.1 https://shop.example.com:8443/api/orders`: `--resolve name:port:127.0.0.1` = "for this name and port, connect to 127.0.0.1" (no DNS needed, and SNI and the Host header carry the real name); `-k` = accept the self-signed certificate. Answered by `api-…` with `GET /api/orders`: the full path reaches the app.
- `… https://shop.example.com:8443/apix`: answered by `web-…`, because `/apix` isn't below `/api` (Prefix compares whole segments).
- `curl -k --resolve api.example.com:8443:127.0.0.1 https://api.example.com:8443/`: answered by `api-…`: host-based routing.
- `curl -i -H "Host: shop.example.com" http://localhost:8080/`: plain HTTP now gets `HTTP/1.1 308 Permanent Redirect` with `Location: https://shop.example.com`.
- `curl --resolve … https://shop.example.com:8443/` (no `-k`): curl checks the certificate and refuses: `curl: (60) SSL certificate problem: self signed certificate`.
- `curl -kv … 2>&1 | grep "subject:"`: `-v` = verbose: prints the TLS details on stderr (the error output); `2>&1` sends stderr into the pipe too, so `grep` can filter it. `subject: CN=shop.example.com` proves the controller presented **your** certificate.

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

### The problem, and the idea
Riya's app needs settings: a banner colour, a mode (`dev` or `prod`), a database host. She **hard-codes** them, writing them straight into the image with `ENV` in her Dockerfile, where they can't change without a rebuild. When production needs `DB_HOST=db-prod`, she rebuilds, so production now runs an image she **never tested** (breaking the promise "the same box everywhere"), and every change means rebuild → push → redeploy.

The **Twelve-Factor App** is a famous list of 12 rules for building cloud apps. Factor III, "Config", says: **store configuration in the environment, not in the code.** Then **one image** runs in dev, testing and production; only the configuration attached to it changes. **Analogy: a restaurant chain.** Every branch uses the **same recipe book** (the image), but has its own **notice board** (the ConfigMap) for local opening hours and specials. Changing the board doesn't mean reprinting the book.

### What a ConfigMap is
> A **ConfigMap** is a Kubernetes object that stores **non-secret configuration** as **key–value pairs**.

- Values are always **text**: a short word (`blue`), or a **whole file** (the key is the file name, the value is its contents).
- It has **`data:`** instead of `spec:`. Pods can only use ConfigMaps from **their own** namespace, and the limit is **1 MiB** (**mebibyte** = 1,048,576 bytes): it's for small settings.
- **Creating one:** from values typed on the command line (`--from-literal=APP_COLOR=blue`), from a file (`--from-file=settings.properties`: the file name becomes the key), from an env file (`--from-env-file=app.env`: each `KEY=VALUE` line becomes its own key), or from YAML (`kind: ConfigMap` with `KEY: value` lines under `data:`; `settings.properties: |` holds a whole file, as `|` keeps the line breaks).
- ⚠️ **In YAML, quote numbers and booleans.** `MAX_ITEMS: 50` is a YAML **number**, and the API server rejects it: `json: cannot unmarshal number into Go struct field ConfigMap.data of type string`. Write `"50"`. The same goes for `true`/`false`, and for `value:` under a container's `env:`.

### Three ways a Pod can use it
```yaml
apiVersion: v1
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
  volumes:                                     # the volumes mounted above
    - name: config-vol
      configMap:
        name: app-config
    - name: files-vol
      configMap:
        name: app-files
```

| Way | The app sees | When the ConfigMap changes |
|---|---|---|
| (a) `env` + `valueFrom.configMapKeyRef`: **one** key → a variable you name | `$COLOR` | **Nothing changes** until the container restarts: env vars are set once, at start |
| (b) `envFrom` + `configMapRef`: **every** key → a same-named variable | `$APP_COLOR`, `$APP_MODE` | The same: nothing until a restart |
| (c) a `configMap` **volume** + `volumeMounts`: every key → a **file** in `mountPath` | `/etc/config/APP_COLOR` | The files **update automatically** after a delay (about a minute on minikube). Good for whole config files |

⚠️ **Common traps:**
- A Pod that refers to a ConfigMap (or key) that **doesn't exist** is stuck in **`CreateContainerConfigError`**, and `kubectl describe pod` says e.g. `configmap "app-confg" not found`. (`optional: true` on the reference allows it to be missing.)
- You **can't add env vars to a running Pod**: most of a Pod's spec is fixed once created (`pod updates may not change fields other than …`). Delete and re-create it. In a Deployment, editing the template simply triggers a rolling update.
- To make a Deployment's Pods pick up a changed ConfigMap in their env vars, run **`kubectl rollout restart deployment/<name>`**: the new Pods read the new values.

### 🛠 Do (~15 min)
```bash
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod   # configmap/app-config created
kubectl get configmap app-config -o yaml                       # APP_COLOR: blue, APP_MODE: prod
printf 'log_level=info\ncache=true\n' > settings.properties    # a two-line file
kubectl create configmap app-files --from-file=settings.properties
kubectl describe configmap app-files                           # one key, two lines
kubectl create configmap demo --from-literal=APP_COLOR=blue --dry-run=client -o yaml   # prints YAML only
code cfg-demo.yaml                                             # paste the Pod above, then save
kubectl apply -f cfg-demo.yaml && kubectl get pod cfg-demo     # wait until Running
kubectl exec cfg-demo -- env | grep -E 'COLOR|APP_'            # COLOR=blue, APP_COLOR=blue, APP_MODE=prod
kubectl exec cfg-demo -- ls /etc/config                        # APP_COLOR  APP_MODE
kubectl exec cfg-demo -- cat /etc/config/APP_COLOR             # blue
kubectl exec cfg-demo -- cat /etc/app/settings.properties      # the two lines
kubectl patch configmap app-config -p '{"data":{"APP_COLOR":"green"}}'
kubectl exec cfg-demo -- cat /etc/config/APP_COLOR             # blue, a minute later green
kubectl exec cfg-demo -- env | grep APP_COLOR                  # STILL blue
```
**What each line does:**
- `kubectl create configmap app-config --from-literal=… --from-literal=…`: creates a ConfigMap from values typed on the command line; each `--from-literal=KEY=value` adds one key. Prints `configmap/app-config created`.
- `kubectl get configmap app-config -o yaml`: shows it as YAML (`-o yaml`): `data:` holds `APP_COLOR: blue` and `APP_MODE: prod` in plain, readable text.
- `printf 'log_level=info\ncache=true\n' > settings.properties`: `printf` writes text, `\n` = a new line, and `>` saves it into the file: a two-line file.
- `kubectl create configmap app-files --from-file=settings.properties`: a ConfigMap from a file: the file name becomes the key, its contents the value.
- `kubectl describe configmap app-files`: one key, `settings.properties`, holding both lines.
- `kubectl create configmap demo … --dry-run=client -o yaml`: only pretends (`--dry-run=client`) and prints ready-made YAML; add `> demo.yaml` to save it.
- `code cfg-demo.yaml`: paste the Pod above, then save.
- `kubectl apply -f cfg-demo.yaml && kubectl get pod cfg-demo`: creates the Pod (`-f` = file), then shows its status (`&&` = only if the first part worked). Repeat the `get` until it says `Running`.
- `kubectl exec cfg-demo -- env | grep -E 'COLOR|APP_'`: `exec` = run one command inside the Pod (everything after `--`); `env` lists its environment variables, and `grep -E 'A|B'` keeps lines with A or B. You should see `COLOR=blue` (way a), `APP_COLOR=blue` and `APP_MODE=prod` (way b).
- `kubectl exec cfg-demo -- ls /etc/config`: `APP_COLOR  APP_MODE`: one file per key (way c).
- `kubectl exec cfg-demo -- cat /etc/config/APP_COLOR`: prints `blue`. Your prompt may follow it on the same line, as the value has no newline.
- `kubectl exec cfg-demo -- cat /etc/app/settings.properties`: the two lines of the file-based ConfigMap.
- `kubectl patch configmap app-config -p '{"data":{"APP_COLOR":"green"}}'`: `patch` = change only the fields you list; `-p` = the change, written as JSON (in single quotes so the shell leaves it alone). `APP_COLOR` becomes `green`.
- `kubectl exec cfg-demo -- cat /etc/config/APP_COLOR`: `blue` at first; about a minute later, `green`.
- `kubectl exec cfg-demo -- env | grep APP_COLOR`: `blue`, and **still** `blue` a minute later.

*The **file** updated by itself; the **env var** was fixed when the container started.*

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

### What a Secret is
Riya puts `DB_PASSWORD` into her ConfigMap. Now anyone allowed to read ConfigMaps sees it in plain text (`-o yaml`, `describe`), and it sits in her Git repository. Passwords, API tokens, keys and certificates need **more careful handling**:

> A **Secret** is like a ConfigMap, but for **sensitive data**: passwords, tokens, keys, certificates. You use it almost the same way; Kubernetes just treats it more carefully.

| | **ConfigMap** | **Secret** |
|---|---|---|
| Stored as | Plain text | base64 (encoding, **not** encryption: below) |
| `kubectl describe` shows | The values | Only the sizes |
| As a volume | Normal files | Files in **tmpfs** (RAM), never written to the node's disk |
| Extra protection | None | Sent only to nodes that need it; access limited by **RBAC**; can be **encrypted at rest** |

- **RBAC** (**Role-Based Access Control**) = rules that say which users and programs may do what, for example who may read Secrets.
- **Encryption at rest** = Secrets are stored encrypted in **etcd** (the cluster's database). **It's off by default.**

### base64 is ENCODING, not encryption
- **Encoding** = rewriting data in another alphabet so it can travel safely as text. **Anyone can reverse it, no key needed**, like Morse code.
- **Encryption** = scrambling data with a **key**, so it can't be read without that key, like a locked safe.

Secrets use base64 only so that **binary** data (keys, certificates) fits into a YAML text file. **It is not protection.** ⚠️ When you encode by hand, **use `echo -n`** (`-n` = no newline at the end): `echo -n 'S3cr3t!' | base64` gives `UzNjcjN0IQ==`, but `echo 'S3cr3t!' | base64` gives `UzNjcjN0IQo=`, because the hidden newline was encoded too. The password then ends in an invisible newline, and logins fail.

### Secret types, `data` vs `stringData`

| Type | Holds | Created with | Used for |
|---|---|---|---|
| **`Opaque`** (the default) | Any key–value pairs | `kubectl create secret generic` | Passwords, API tokens |
| **`kubernetes.io/tls`** | Exactly `tls.crt` + `tls.key` | `kubectl create secret tls` | Ingress TLS (the controller's HTTPS certificate) |
| **`kubernetes.io/dockerconfigjson`** | A registry login | `kubectl create secret docker-registry` | Pulling **private images**: list it under the Pod spec's `imagePullSecrets:`, or the Pod gets `ImagePullBackOff` |

`kubectl create secret` does the base64 for you. In **Secret YAML** there are two sections:
- Values under **`data:`** must **already be base64** (`DB_PASSWORD: UzNjcjN0IQ==`).
- Values under **`stringData:`** are plain text, and the API server encodes them for you. It's write-only: reading the Secret back shows everything under `data:`.
- ⚠️ **Plain text under `data:`** is a classic bug. `DB_PASSWORD: S3cr3t!` is rejected (`illegal base64 data at input byte 6`). Worse, `DB_PASSWORD: password` is **accepted**, because those letters happen to be valid base64, and the app receives **garbled bytes** instead of the password.

### Using a Secret in a Pod
Exactly like a ConfigMap, with "secret" in the names: `valueFrom.secretKeyRef` (one key), `envFrom.secretRef` (every key), or a volume with `secret:` + ⚠️ **`secretName:`** (not `name:`). Inside the container, the values are **already decoded**.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]      # just stay alive for an hour
      envFrom:
        - configMapRef:
            name: app-config                   # APP_COLOR, APP_MODE
      env:
        - name: DB_PASSWORD                    # one key from the Secret → one variable
          valueFrom:
            secretKeyRef:
              name: db-secret                  # which Secret
              key: DB_PASSWORD                 # which key in it
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secret               # every key → a file here
          readOnly: true
  volumes:
    - name: secret-vol
      secret:
        secretName: db-secret                  # secretName, not name!
```
**Files are safer than env vars for secrets:** env vars leak into logs and error reports, and child processes inherit them; mounted files can be read-only and live in RAM.

**Good practice (awareness, but examined):**
- **Never commit Secret YAML to Git:** base64 is readable by anyone, and Git keeps every old version forever. Keep such files in `.gitignore`.
- **Limit access with RBAC** (few people and programs should be able to `get` or `list` Secrets), and **turn on encryption at rest**.
- **Use an external secret manager** for real projects: **HashiCorp Vault** or AWS Secrets Manager, synced into the cluster by the **External Secrets Operator**. Or **Sealed Secrets**: you encrypt a Secret into a "SealedSecret" that's safe to commit, and only a controller inside the cluster can decrypt it.
- `kubectl apply` keeps a copy of the applied file in the `last-applied-configuration` annotation, so a `stringData` password applied this way can be read there. `kubectl create secret` doesn't do this.

### 🛠 Do (~15 min)
```bash
echo -n 'S3cr3t!' | base64            # UzNjcjN0IQ==
echo 'S3cr3t!' | base64               # UzNjcjN0IQo= (different!)
echo 'UzNjcjN0IQ==' | base64 -d       # S3cr3t!
kubectl create secret generic db-secret --from-literal=DB_USER=admin --from-literal=DB_PASSWORD='S3cr3t!'
kubectl describe secret db-secret     # sizes only
kubectl get secret db-secret -o yaml  # the base64 values
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d   # S3cr3t!
code app-pod.yaml                     # paste the Pod above, then save
kubectl apply -f app-pod.yaml
kubectl exec app-pod -- env | grep -E 'APP_|DB_'      # APP_COLOR=green, …, DB_PASSWORD=S3cr3t!
kubectl exec app-pod -- cat /etc/secret/DB_PASSWORD   # S3cr3t!
kubectl exec app-pod -- df -h /etc/secret             # tmpfs
sed 's/name: app-pod/name: broken-pod/; s/key: DB_PASSWORD/key: password/' app-pod.yaml > broken-pod.yaml   # a copy with a wrong key
kubectl apply -f broken-pod.yaml
kubectl get pod broken-pod                            # CreateContainerConfigError
kubectl describe pod broken-pod | grep Warning        # couldn't find key password …
kubectl delete pod broken-pod
kubectl create secret docker-registry regcred --docker-server=ghcr.io --docker-username=riya --docker-password=FAKE --dry-run=client -o yaml   # look only
```
**What each line does:**
- `echo -n 'S3cr3t!' | base64`: encodes the password; `-n` = no newline at the end. Gives `UzNjcjN0IQ==`. The single quotes keep the `!` safe from zsh.
- `echo 'S3cr3t!' | base64`: the same without `-n` gives `UzNjcjN0IQo=`: the hidden newline was encoded too.
- `echo 'UzNjcjN0IQ==' | base64 -d`: `-d` = decode. Gives back `S3cr3t!`, no key needed.
- `kubectl create secret generic db-secret --from-literal=… --from-literal=…`: creates an `Opaque` Secret (`generic`) with two keys, `DB_USER` and `DB_PASSWORD`; kubectl does the base64 for you.
- `kubectl describe secret db-secret`: `Type: Opaque`, `DB_PASSWORD: 7 bytes`, `DB_USER: 5 bytes`: sizes only.
- `kubectl get secret db-secret -o yaml`: but `-o yaml` shows the stored values: `DB_PASSWORD: UzNjcjN0IQ==`, `DB_USER: YWRtaW4=`.
- `kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d`: `-o jsonpath` = print just one field; decoding it gives `S3cr3t!`. base64 hid nothing.
- `code app-pod.yaml` + `kubectl apply -f app-pod.yaml`: paste the Pod above, save, and create it.
- `kubectl exec app-pod -- env | grep -E 'APP_|DB_'`: `APP_COLOR=green` (this Pod started after your patch, so it read the new value), `APP_MODE=prod`, and `DB_PASSWORD=S3cr3t!`, already decoded.
- `kubectl exec app-pod -- cat /etc/secret/DB_PASSWORD`: `S3cr3t!`, decoded, read from the mounted file (`ls /etc/secret` would show one file per key).
- `kubectl exec app-pod -- df -h /etc/secret`: `df -h` = show disk usage in human-readable sizes. The filesystem is `tmpfs`: the secret files live in RAM.
- `sed '…' app-pod.yaml > broken-pod.yaml`: `sed` replaces text (`s/old/new/`; `;` separates two replacements). The copy is renamed `broken-pod` and asks for a key `password` that doesn't exist.
- `kubectl apply -f broken-pod.yaml` + `kubectl get pod broken-pod`: STATUS `CreateContainerConfigError`.
- `kubectl describe pod broken-pod | grep Warning`: keeps only the warning line: `… couldn't find key password in Secret default/db-secret`.
- `kubectl delete pod broken-pod`: removes it.
- `kubectl create secret docker-registry regcred … --dry-run=client -o yaml`: look only: prints a registry-login Secret (`--docker-server`/`--docker-username`/`--docker-password` = the registry and its login) of type `kubernetes.io/dockerconfigjson`, without creating it.

**Clean up the whole session** (press Ctrl + C on any port-forward first):
```bash
kubectl delete pod cfg-demo app-pod
kubectl delete configmap app-config app-files
kubectl delete secret db-secret shop-tls
kubectl delete ingress shop           # keep the ingress add-on
kubectl delete svc web api
kubectl delete deployment web api
rm tls.key
```
**What each line does:**
- `kubectl delete pod/configmap/secret …`: deletes the two Pods, the two ConfigMaps and the two Secrets; you can list several names of one kind in a single command.
- `kubectl delete ingress shop`: removes the Ingress rules. Keep the ingress add-on itself: a later session uses it.
- `kubectl delete svc web api` + `kubectl delete deployment web api`: removes both apps and their Services.
- `rm tls.key`: deletes the private key file from your Mac, as private keys shouldn't be left lying around (`tls.crt` is public and harmless).

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

## ➕ Small but important

| Item | What it means |
|---|---|
| `--dry-run=client -o yaml` | Generate YAML without creating anything: `kubectl create secret generic db --from-literal=PASS=x --dry-run=client -o yaml > secret.yaml`. Works for ConfigMaps, Deployments, Services, Ingresses too |
| **`immutable: true`** | A top-level field that **locks** a ConfigMap or Secret: any change is refused (`field is immutable when immutable is set`), so you delete and re-create it instead. Prevents accidents and eases the API server's load |
| **`optional: true`** | On `configMapKeyRef`/`secretKeyRef`/`envFrom`: start anyway if the ConfigMap/Secret is missing (no `CreateContainerConfigError`) |
| **`env` beats `envFrom`** | If the same variable name comes from both, the value in `env` wins |
| **Ingress annotations** | Controller-specific settings in `metadata.annotations`. E.g. `nginx.ingress.kubernetes.io/rewrite-target: /$2` with `path: /api(/\|$)(.*)` and `pathType: ImplementationSpecific` turns `/api/orders` into `/orders` for the backend |
| **404 vs 503 from the controller** | **404**: no rule matched the host/path. **503**: a rule matched, but its Service is missing or has no Ready Pods. Check `kubectl describe ingress` (each path's backend and Pod IPs, or `<error: services "api" not found>`) and `kubectl get endpoints <svc>` |
| `kubectl create ingress` | `--rule="shop.example.com/api*=api:80"`: a `*` at the end of the path means `Prefix`; without it, `Exact`. The default IngressClass carries the annotation `ingressclass.kubernetes.io/is-default-class: "true"` |
| **Gateway API** | The newer successor to Ingress, with one object per role: **GatewayClass**, **Gateway** (the entry point), **HTTPRoute** (the rules). The `ingress-nginx` project announced its retirement (maintenance ending March 2026); the Ingress API itself stays |

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

### ✍️ Write the command
**C1.** Create a ConfigMap named `app-config` with two keys: `APP_COLOR` set to `blue` and `APP_MODE` set to `prod`.

**C2.** Create a TLS Secret named `shop-tls` from the certificate file `tls.crt` and the key file `tls.key`.

**C3.** Print the **decoded** value of the key `DB_PASSWORD` stored in the Secret `db-secret`.

**C4.** Without creating anything in the cluster, write the YAML for an Ingress `shop` (IngressClass `nginx`) that sends `shop.example.com/` and every path below it to port 80 of the Service `web`, into a file called `ingress.yaml`.
