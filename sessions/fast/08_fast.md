# Session 08 · Ingress, ConfigMaps & Secrets · FAST (10-min read)
*How web traffic from outside reaches the right app, and how apps get their settings and passwords without baking them into the image.*

## 1 · Ingress and the Ingress controller
Without Ingress, every app you put on the internet needs its own LoadBalancer Service. That means one public IP, one monthly bill and one certificate per app, and no way to route by URL. **Ingress** gives the whole cluster **one entry point** for HTTP/HTTPS traffic and sends each request to the right Service based on its **host name** and **URL path**. It's a **reverse proxy**: a server that sits in front of other servers and forwards each request to the right one. *Analogy:* a shopping mall with one main entrance and an information desk.
- **You always need two parts.** The **Ingress resource** (`kind: Ingress`) only holds the routing rules and does nothing on its own. The **Ingress controller** is the real proxy program (NGINX, Traefik…) running as Pods. It watches every Ingress and follows its rules.
- A plain cluster has **no controller installed**. Your Ingress is accepted and stored, but nothing routes. On minikube you add one with `minikube addons enable ingress`.
- **`ingressClassName: nginx`** says which controller should handle this Ingress. With no class and no default class, no controller picks it up.
- The controller reads the **Host header**, the site name every HTTP request carries (`Host: shop.example.com`). That's how one IP address serves many websites. Without DNS you can set it yourself: `curl -H "Host: shop.example.com" http://localhost:9080/`.
- Ingress works at **Layer 7** (the HTTP request: host, path, headers), whereas a Service works at **Layer 4** (only IP address and port). Ingress is for web traffic only.

⚠️ **Exam trap:** Ingress does **not** replace Services. Every Ingress rule points at a Service, and the controller uses that Service's endpoints to find the Pods.

## 2 · Host and path routing, and TLS
- **Host-based routing:** a rule's `host` is compared with the request's Host header. **Path-based routing:** under one host, different paths go to different Services, called **backends**. When several paths match, the **longest path wins**.
- **`pathType: Prefix`** matches the path and everything below it, comparing **whole segments** (a segment is the part between two `/`s). So `/api` matches `/api/orders` but **not** `/apix`. **`Exact`** matches only that exact path.
- The path reaches the app **unchanged**: the API receives `/api/orders`, not `/orders`.

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shop.example.com
      secretName: shop-tls
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```
- `tls` + `secretName`: serve HTTPS for this host using the certificate stored in the Secret `shop-tls`.
- `host`, `path`, `pathType`: which requests this rule catches.
- `backend.service.name` + `port.number`: the Service, and its port, that receives them.

**TLS termination:** HTTPS is HTTP inside **TLS (Transport Layer Security)** encryption. The **controller** holds the certificate and private key, decrypts each request (it can only read the host and path *after* decrypting), and forwards plain HTTP to the Pods. The apps need no certificates, and renewal happens in one place.
- The certificate lives in a Secret of type **`kubernetes.io/tls`** with exactly two keys, `tls.crt` and `tls.key`. It's made with `kubectl create secret tls`, and it must be in the **same namespace** as the Ingress.
- Once `tls:` is set, plain HTTP gets a **308 Permanent Redirect** to HTTPS. A **self-signed** certificate (signed by itself, not by a CA, a Certificate Authority that browsers trust) makes curl refuse unless you add `-k`.

⚠️ **Exam trap:** **404** from the controller means **no rule matched** the host or path, whereas **503** means a rule matched but its **Service is missing or has no Ready Pods**. You ran into both today.

## 3 · ConfigMaps
Writing settings into the image (e.g. `ENV DB_HOST=…` in a Dockerfile) means rebuilding the image for every environment, so production ends up running an image nobody tested. The **Twelve-Factor App** rule says to keep configuration outside the code. A **ConfigMap** is a Kubernetes object that stores **non-secret settings** as **key–value pairs**, so **one image** runs everywhere with different settings attached. *Analogy:* every restaurant branch uses the same recipe book (the image) but has its own notice board (the ConfigMap).
- Values are always **text**: a single word, or a whole file. A ConfigMap has `data:` instead of `spec:` and holds at most 1 MiB. A Pod can only use ConfigMaps from **its own namespace**.
- Creating one: `kubectl create configmap app-config --from-literal=APP_COLOR=blue` (one key per flag), or `--from-file=settings.properties` (the file name becomes the key).
- **Three ways a Pod uses it:** (a) `env` + `valueFrom.configMapKeyRef` turns **one** key into one variable with a name you choose; (b) `envFrom` + `configMapRef` turns **every** key into a variable with the same name; (c) a `configMap` **volume** + `volumeMounts` turns every key into a **file** in a folder.
- When the ConfigMap changes, **mounted files update by themselves** after about a minute, but **environment variables don't**, because a container reads them only once, when it starts. `kubectl rollout restart deployment/<name>` creates new Pods that read the new values.

⚠️ **Exam traps:** a Pod that refers to a missing ConfigMap or key is stuck in **`CreateContainerConfigError`** (`describe` says `configmap "…" not found`). In YAML, put quotes around numbers and true/false (`MAX_ITEMS: "50"`), because ConfigMap values must be text.

## 4 · Secrets
A **Secret** is like a ConfigMap but for sensitive data: passwords, tokens, keys, certificates. You use it almost the same way; Kubernetes just handles it more carefully.
- Values are stored **base64-encoded**. Encoding only rewrites data in another alphabet, and anyone can reverse it with `base64 -d`, whereas **encryption** needs a key. **base64 is not protection.**
- **How it's handled more carefully:** `describe` shows only sizes; mounted Secret files live in **tmpfs** (memory), never on the node's disk; **RBAC (Role-Based Access Control)** can limit who reads Secrets; **encryption at rest** in etcd is possible but **off by default**.
- **Types:** **`Opaque`** is the default, made with `kubectl create secret generic`. **`kubernetes.io/tls`** holds `tls.crt` + `tls.key`, for Ingress. **`kubernetes.io/dockerconfigjson`** holds a registry login: list it under `imagePullSecrets` to pull private images.
- In YAML, values under **`data:`** must already be base64, whereas values under **`stringData:`** are plain text and the API server encodes them for you.
- **In a Pod:** `secretKeyRef` gives one key, `envFrom` + `secretRef` gives every key, or a volume with `secret:` + **`secretName:`** turns keys into files. Inside the container, the values are already **decoded**.
- When encoding by hand, use `echo -n`, or a hidden newline becomes part of the password.

⚠️ **Exam trap:** a Secret volume uses `secretName:`, whereas a ConfigMap volume uses `name:`. Never commit Secret YAML to Git; real projects use HashiCorp Vault, the External Secrets Operator, or Sealed Secrets.

## 🎯 Exam checklist (everything that could be asked)
*MCQ = know the fact · CMD = be able to write it from memory.*

| Asked as | Item | What you must know / the exact command |
|---|---|---|
| MCQ | **Ingress** · what it is | One entry point for a cluster's HTTP/HTTPS traffic that sends each request to a Service by host name and URL path (a reverse proxy) |
| MCQ | Ingress resource vs controller | The resource only holds the rules, whereas the controller (NGINX, Traefik) is the program that applies them. No controller = nothing routes |
| MCQ | Ingress vs Service | Ingress routes at Layer 7 (host, path), whereas a Service routes at Layer 4 (IP and port). Every Ingress rule still points at a Service |
| MCQ | Why Ingress, not one LoadBalancer per app | One load balancer, one public IP and one place for certificates, plus routing by URL |
| MCQ | `ingressClassName: nginx` | Says which controller handles this Ingress. With no class and no default class, no controller picks it up |
| MCQ | Host header | The site name every HTTP request carries. The controller compares it with each rule's `host`, so one IP can serve many sites |
| MCQ | Ingress `apiVersion` | `networking.k8s.io/v1` |
| CMD | Turn on the controller (minikube) | `minikube addons enable ingress` (installs NGINX into the namespace `ingress-nginx`) |
| CMD | Generate Ingress YAML | `kubectl create ingress shop --class=nginx --rule="shop.example.com/*=web:80" --dry-run=client -o yaml > ingress.yaml` (the `*` makes it `Prefix`) |
| CMD | Ingress YAML key paths | `spec.ingressClassName` · `spec.rules[].host` · `http.paths[].path` + `pathType` · `backend.service.name` + `backend.service.port.number` · `spec.tls[].hosts` + `secretName` |
| CMD | Test a host rule without DNS | `curl -H "Host: shop.example.com" http://localhost:9080/` |
| MCQ | **Routing** · several paths match | The longest path wins: `/api/orders` goes to the `/api` rule, not `/` |
| MCQ | `Prefix` vs `Exact` | Prefix matches the path and everything below it by whole segments (`/api` matches `/api/x`, not `/apix`), whereas Exact matches only that path |
| MCQ | The path reaches the app unchanged | The app receives `/api/orders`, not `/orders`. Removing `/api` needs the `nginx.ingress.kubernetes.io/rewrite-target` annotation |
| MCQ | 404 · 503 · 308 from NGINX | 404 = no rule matched · 503 = a rule matched but its Service is missing or has no Ready Pods · 308 = HTTP sent to HTTPS because `tls:` is set |
| MCQ | `spec.defaultBackend` | A Service that gets every request no rule matches, instead of the controller's 404 |
| MCQ | **TLS** termination | The controller holds the certificate and key, decrypts HTTPS, reads host and path, and sends plain HTTP to the Pods. The Pods need no certificate |
| MCQ | The TLS Secret | Type `kubernetes.io/tls`, exactly two keys `tls.crt` + `tls.key`, in the same namespace as the Ingress |
| CMD | Create the TLS Secret | `kubectl create secret tls shop-tls --cert=tls.crt --key=tls.key` |
| MCQ | SNI (Server Name Indication) | The host name sent in plain text during the TLS handshake, so the controller can choose the right certificate before decrypting |
| MCQ | Self-signed certificate, `curl -k` | Signed by itself, not by a CA (Certificate Authority), so curl refuses it unless you add `-k` (testing only). cert-manager + Let's Encrypt get real ones automatically |
| CMD | Test HTTPS through a tunnel | `curl -k --resolve shop.example.com:9443:127.0.0.1 https://shop.example.com:9443/` |
| MCQ | **ConfigMap** · what it is | Stores non-secret settings as text key–value pairs under `data:` (max 1 MiB), so one image runs everywhere ("config outside the code", Twelve-Factor App) |
| MCQ | ConfigMap scope | A Pod can only use ConfigMaps (and Secrets) from its own namespace |
| CMD | Create from values | `kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod` |
| CMD | Create from a file | `kubectl create configmap app-files --from-file=settings.properties` (the file name becomes the key; `--from-env-file` makes one key per line) |
| CMD | One key → one env var | `env:` → `name: COLOR` + `valueFrom.configMapKeyRef` (`name: app-config`, `key: APP_COLOR`) |
| CMD | Every key → env vars | `envFrom:` → `- configMapRef:` → `name: app-config` |
| CMD | Every key → a file | `volumes:` → `configMap:` → `name: app-config`, plus `volumeMounts:` → `mountPath: /etc/config` |
| MCQ | After you change a ConfigMap | Mounted files update by themselves (about a minute), whereas env vars keep the old value until the container restarts |
| CMD | Make Pods read new env values | `kubectl rollout restart deployment/web` |
| MCQ | `CreateContainerConfigError` | The Pod refers to a ConfigMap, Secret or key that doesn't exist. The container was never created: 0 restarts, no logs. Create it and the kubelet retries by itself |
| MCQ | Numbers and true/false in ConfigMap YAML | Quote them (`"50"`, `"true"`), or the API server rejects the ConfigMap |
| MCQ | Same variable from `env` and `envFrom` | The value from `env` wins |
| MCQ | `immutable: true` vs `optional: true` | immutable locks a ConfigMap or Secret against changes, whereas optional lets the Pod start even if it's missing |
| MCQ | **Secret** vs ConfigMap | A Secret is for sensitive data: base64-encoded, `describe` shows only sizes, mounted as files in tmpfs (memory), access limited by RBAC (Role-Based Access Control) |
| MCQ | base64 | Encoding, not encryption: anyone can reverse it with `base64 -d`, no key needed |
| MCQ | Encryption at rest | Secrets stored encrypted inside etcd. It's off by default |
| MCQ | Secret types | `Opaque` (the default, `generic`) · `kubernetes.io/tls` (Ingress certificates) · `kubernetes.io/dockerconfigjson` (registry login, used through `imagePullSecrets`) |
| CMD | Create a generic Secret | `kubectl create secret generic db-secret --from-literal=DB_USER=admin --from-literal=DB_PASSWORD='S3cr3t!'` |
| CMD | Print one decoded value | `kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' \| base64 -d` |
| CMD | Encode by hand | `echo -n 'S3cr3t!' \| base64` (`-n` stops a hidden newline being encoded too) |
| MCQ | `data:` vs `stringData:` | `data:` needs values already in base64, whereas `stringData:` takes plain text and the API server encodes it |
| CMD | Use a Secret in a Pod | `valueFrom.secretKeyRef` (one key) · `envFrom:` → `- secretRef:` (every key) · volume `secret:` → `secretName: db-secret` (not `name:`) |
| MCQ | Files vs env vars for secrets | Files are safer: env vars leak into logs and child processes, whereas Secret files can be read-only and live in memory |
| MCQ | Secret good practice | Never commit Secret YAML to Git; limit access with RBAC; use HashiCorp Vault, the External Secrets Operator, or Sealed Secrets |
| CMD | Print YAML without creating anything | `kubectl create secret generic db --from-literal=PASS=x --dry-run=client -o yaml` (works for ConfigMaps, Deployments, Ingresses too) |

## 📝 Quiz (answer in chat as "1 B, 2 D, …")

**1.** An Ingress for `shop.example.com` has two Prefix rules: `/api` → Service `orders`, and `/` → Service `web`. `kubectl get svc` lists only `web`. What does `curl -H "Host: shop.example.com" http://localhost:9080/api/v1` return?
- A) 404, since no rule covers the path /api/v1
- B) The web page, since `/` catches the request
- C) 503, as the matched Service doesn't exist
- D) 308, since the controller forces HTTPS first

**2.** A Deployment's Pods get `APP_MODE` through `envFrom` and also mount the same ConfigMap at `/etc/config`. You change `APP_MODE` from `dev` to `prod` with `kubectl patch`. Two minutes later, what does a running container see?
- A) The file says prod; the env var still says dev
- B) Both say prod; the kubelet refreshes them all
- C) Both still say dev until the Pods are recreated
- D) The env var says prod; the file still says dev

**3.** A Secret YAML has the line `DB_PASSWORD: password` under `data:`. `kubectl apply` succeeds, but the app can't log in to its database. Why?
- A) Secrets must be made with create, never apply
- B) The Pod needs imagePullSecrets to read Secrets
- C) The app lacks the key to decrypt the base64
- D) data: needs base64, so it decodes to garbage

**4.** Which statement about TLS on an NGINX Ingress is NOT true?
- A) Its Secret must be in the Ingress's namespace
- B) Backend Pods must each hold the certificate
- C) Plain HTTP to that host gets a 308 redirect
- D) The Secret holds exactly tls.crt and tls.key

**5.** A Pod in namespace `shop` shows `CreateContainerConfigError` with 0 restarts, and `describe` says `configmap "app-config" not found`. Yet `kubectl get configmap app-config` prints it without errors. What's the most likely cause?
- A) It's in `default`; Pods only see their own namespace
- B) The app crashed; `logs --previous` shows the reason
- C) ConfigMaps sync slowly; the kubelet will succeed soon
- D) envFrom works only for Secrets, so use valueFrom
