# Answer key · Session 08 · Ingress, ConfigMaps & Secrets

⚠️ Only open after answering.

---

## Warm-up

**W1.** A Service `api` in the namespace `shop` has `type: ClusterIP`, `port: 80`, and `targetPort: 8080`. A Pod in the namespace `web` wants to call it. Which URL works?
- A) `http://api:8080`
- B) `http://api.shop:80`
- C) `http://api.shop:8080`
- D) `http://api:80`

**Correct: B.** Two rules at once: from another namespace you must add the namespace (`api.shop`), and clients call the **Service's `port`** (80), not the `targetPort`. A is wrong twice (no namespace, and 8080 is the Pods' port). C has the right name but the wrong port: the Service doesn't listen on 8080, so it times out. D: the short name `api` is tried as `api.web.svc.cluster.local`, which doesn't exist.

---

**W2.** `kubectl get endpoints web` shows `<none>`, yet `kubectl get pods -l app=web` lists 3 Pods that are Running and Ready, in the same namespace. What's the most likely cause?
- A) The selector has a label those Pods don't have
- B) The targetPort doesn't match the containerPort
- C) kube-proxy hasn't written its iptables rules yet
- D) CoreDNS hasn't made a DNS record for it yet

**Correct: A.** Endpoints are built from the **selector** alone. Ready Pods in the same namespace that aren't listed means the selector asks for something they lack: an extra label, a different value, or different upper/lower case. (Matching `app=web` in your own `-l` doesn't prove the Service's selector is only `app=web`.) B: a wrong targetPort still gives endpoints, just with the wrong port. C: kube-proxy **reads** endpoints; it doesn't create them. D: DNS points the name at the ClusterIP and has nothing to do with Endpoints.

---

**W3.** A shop moves its website to a new domain for good. Which status code should the old URLs return, so browsers and search engines update their saved links?
- A) 302, as the visitor is sent elsewhere
- B) 304, as the old page hasn't changed
- C) 301, as the move is permanent
- D) 307, as the method must be kept

**Correct: C.** **301 Moved Permanently** tells clients the old address is gone for good, so they update their links. A: 302 is **temporary**, so clients keep using the old URL. B: 304 Not Modified is a caching reply ("your saved copy is still fine"), not a redirect at all. D: 307 is also **temporary** (it keeps the method, like 308, but 308 is the permanent one).

---

## Check 58

**58.1** On a fresh cluster, an engineer applies a valid Ingress. `kubectl get ingress` shows no ADDRESS, and requests to the nodes time out. The Pods and Services work fine. What's the most likely cause?
- A) The backend Services must be type LoadBalancer
- B) Every backend Service needs its own NodePort
- C) No Ingress controller is running to act on it
- D) Ingress rules need IP addresses, not host names

**Correct: C.** An Ingress resource is only a list of rules. A plain cluster has no controller, so nothing reads the rules, nothing fills in ADDRESS, and nothing listens for the traffic. A and B: Ingress backends are normally plain ClusterIP Services; only the controller itself needs to be exposed. D: rules use host names (or no host at all).

---

**58.2** Which statement correctly contrasts a Service with an Ingress?
- A) Services route by URL path; an Ingress by port only
- B) Ingress routes by host/path; Services by IP/port
- C) Ingress replaces Services; backends need no Service
- D) Services decrypt TLS; Ingress passes HTTPS untouched

**Correct: B.** An Ingress works at Layer 7 (HTTP), so it can read host names and paths; a Service works at Layer 4, seeing only IPs and ports. A swaps them. C: every Ingress rule points **at** a Service. D also swaps them: the Ingress controller can terminate TLS, while a Service just forwards TCP.

---

## Check 59

**59.1** An Ingress has two rules for host `shop.example.com`: `/api` (Prefix) → `api`, and `/` (Prefix) → `web`. Where do these three requests go: `/api/v1/items`, `/apis`, `/`?
- A) api, api, web
- B) web, web, web
- C) api, web, 404
- D) api, web, web

**Correct: D.** `/api/v1/items` is below `/api`, so it goes to `api` (the longest match wins over `/`). `/apis` is **not** below `/api`, because Prefix compares whole segments (`apis` ≠ `api`), so it falls to `/` → `web`. `/` → `web`. A treats Prefix as a plain letter-by-letter prefix. B ignores the `/api` rule. C forgets that `/` with Prefix catches everything else.

---

**59.2** An Ingress in the namespace `shop` has `tls:` with `secretName: shop-tls`. HTTPS clients get the "Kubernetes Ingress Controller Fake Certificate", and the controller's logs say the Secret can't be found. Yet `kubectl get secret shop-tls -n default` shows it. Why?
- A) The Secret has to be in the Ingress's namespace
- B) TLS Secrets must live in the ingress-nginx namespace
- C) The Secret must be of type Opaque, not TLS
- D) The Secret's name must match the host name exactly

**Correct: A.** An Ingress can only use a TLS Secret from **its own** namespace (`shop`). The copy in `default` is invisible to it, so the controller falls back to its built-in fake certificate. B: the controller looks in the Ingress's namespace, not its own. C: it must be type `kubernetes.io/tls` (with `tls.crt` and `tls.key`). D: any name works; it's referenced by `secretName`.

---

**59.3** (one line) What does "TLS termination at the Ingress" mean? Give one benefit.

**Model answer:** *"TLS termination at the Ingress means the Ingress controller holds the certificate and private key, decrypts the HTTPS traffic there, and forwards plain HTTP to the Pods inside the cluster. One benefit: the apps don't need certificates, so certificates are managed and renewed in one place (and the controller can read the path to route the request)."*

---

## Check 60

**60.1** A Deployment's Pods read `APP_MODE` through `envFrom` from the ConfigMap `app-config`. You change `APP_MODE` to `debug` and `kubectl apply` the ConfigMap. Ten minutes later, the app still sees `prod`. What's the reason, and the fix?
- A) The kubelet refreshes env vars hourly; wait for it
- B) ConfigMaps need `immutable: true` before changes apply
- C) envFrom works only with Secrets; use configMapKeyRef
- D) Env vars are set at start; restart the Pods

**Correct: D.** Environment variables are filled in once, when the container starts, and never refreshed. New Pods read the new value, so run `kubectl rollout restart deployment/<name>`. A: there's no hourly refresh; env vars never change in a running container. B: `immutable: true` does the opposite: it **forbids** changes. C: `envFrom` works with both `configMapRef` and `secretRef`.

---

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

**Correct: B.** ConfigMap values must be strings, and an unquoted `50` is a YAML number, so quote it. A: ConfigMaps have no `spec:`; `data:` sits at the top level. C: `stringData` is a **Secret** field; ConfigMaps don't have it. D: `KEY=VALUE` is env-file syntax, not a YAML key–value pair.

---

**60.3** (one line) Name the three ways a Pod can use a ConfigMap, and say which one picks up changes without a restart.

**Model answer:** *"A Pod can use a ConfigMap as single environment variables (`env` + `valueFrom.configMapKeyRef`), as environment variables for every key (`envFrom.configMapRef`), or as files in a mounted volume. Only the mounted files update automatically (after a short delay), whereas environment variables change only when the container restarts."*

---

## Check 61

**61.1** A teammate says: "Our Secret YAML files are safe in the Git repo, because the values are base64." What's the best response?
- A) False: base64 is an encoding that anyone can reverse
- B) True: base64 can only be decoded with the cluster's key
- C) True, as long as the Secret's type is set to Opaque
- D) False: base64 is fine, but Git keeps every old version

**Correct: A.** base64 is **encoding**, not encryption: `base64 -d` reverses it with no key, so Secret YAML in Git means passwords in Git. B is the misconception itself: there's no key involved. C: the type adds no encryption. D: Git history is a real extra problem, but "base64 is fine" is wrong.

---

**61.2** 🐞 A Pod is stuck in `CreateContainerConfigError`. `kubectl describe pod` says `Error: couldn't find key password in Secret default/db-secret`. `kubectl get secret db-secret -o yaml` shows the keys `DB_USER` and `DB_PASSWORD`. What's the fix?
- A) Base64-encode the word `password` in the Pod spec
- B) Change the Secret's type to `kubernetes.io/basic-auth`
- C) Set the Pod's `secretKeyRef` key to `DB_PASSWORD`
- D) Add `imagePullSecrets: db-secret` to the Pod spec

**Correct: C.** The `key` in `secretKeyRef` must exactly match a key in the Secret, and keys are case-sensitive: `password` ≠ `DB_PASSWORD`. A: the Pod refers to key **names** in plain text; nothing there is encoded. B: changing the type doesn't rename keys. D: `imagePullSecrets` is only for pulling images from private registries.

---

**61.3** (one line) Why should you use `echo -n` when you base64-encode a password by hand?

**Model answer:** *"Plain `echo` adds a newline at the end, and base64 encodes that too, so the Secret would hold the password plus an invisible newline and logins would fail; `echo -n` prints the text without the newline."*

---

## Session quiz

**Q1.** `curl -H "Host: shop.example.com" http://localhost:8080/api` returns `503 Service Temporarily Unavailable`, while `/` returns 200. `kubectl describe ingress shop` shows `/api   api:80 (<error: services "api" not found>)`. What's the cause?
- A) No rule matches `/api`, so the default backend replies
- B) The Service named in the `/api` rule doesn't exist
- C) TLS isn't set up, so the controller rejects `/api`
- D) pathType Prefix can't match a path such as `/api`

**Correct: B.** The `/api` rule **matched**, but its backend Service `api` doesn't exist (the describe output says so), so the controller answers 503. A: no matching rule gives **404**, and here the `/api` rule is clearly listed. C: plain HTTP routing works without TLS (with TLS you'd get a 308, not a 503). D: `Prefix` `/api` matches `/api` itself as well as everything below it.

---

**Q2.** A Pod mounts the ConfigMap `cfg` at `/etc/cfg` and also loads it with `envFrom`. You patch its key `MODE` from `a` to `b`. Two minutes later, what do `cat /etc/cfg/MODE` and `echo $MODE` print inside the container?
- A) `b` and `b`
- B) `a` and `b`
- C) `b` and `a`
- D) `a` and `a`

**Correct: C.** Mounted ConfigMap files are updated automatically after a short delay (about a minute on your minikube), so the file says `b`. Environment variables were fixed when the container started, so `$MODE` is still `a`. A assumes env vars refresh. B has the two behaviours backwards. D forgets that mounted files do update.

---

**Q3.** After you add a `tls:` section to an Ingress, `curl -i -H "Host: shop.example.com" http://localhost:8080/` returns `308 Permanent Redirect`. What's happening?
- A) The controller now redirects plain HTTP to HTTPS
- B) The tls Secret is missing, so NGINX sends you away
- C) The Service's port changed from 80 to port 443
- D) 308 means the page moved temporarily to /tls

**Correct: A.** Once an Ingress has a `tls:` section, the NGINX controller redirects HTTP requests for it to `https://` with a 308 (switch off with `nginx.ingress.kubernetes.io/ssl-redirect: "false"`). B: a missing Secret makes the controller show its fake certificate; the redirect comes from the `tls:` section itself. C: the Service is unchanged; the controller handles port 443. D: 308 is **permanent** (307 is the temporary one), and nothing points to `/tls`.

---

**Q4.** A Secret's YAML has `DB_PASSWORD: password` under `data:`. `kubectl apply` succeeds, but the app can't log in to its database. Why?
- A) Secrets need `type: kubernetes.io/tls` to be loaded
- B) Secret env vars always reach the app base64-encoded
- C) The kubelet hides keys whose names contain PASSWORD
- D) `data` wants base64, so `password` decodes to junk

**Correct: D.** Values under `data:` must already be base64. The letters of `password` happen to be valid base64, so the API server accepts the file, but it decodes them into garbled bytes, and that's what the app receives. (Use `stringData:` for plain text, or encode with `echo -n … | base64`.) A: `Opaque` is the right type for passwords; `kubernetes.io/tls` is only for certificates. B: values are **decoded** before they reach the container. C: no such rule exists.

---

**L1.** Explain Ingress (the resource) vs Ingress controller in contrast form.

**Model answer:** *"An Ingress resource is only a set of routing rules (host and path → Service), whereas an Ingress controller is the actual reverse-proxy program, such as the NGINX Ingress Controller, that runs as Pods, reads those rules, and routes the traffic. Without a controller, an Ingress does nothing."*

---

**L2.** Explain ConfigMap vs Secret in contrast form, and say why base64 isn't security.

**Model answer:** *"A ConfigMap stores non-sensitive configuration as plain text, whereas a Secret stores sensitive data such as passwords base64-encoded, with extra protections (values hidden in `describe`, tmpfs mounts, RBAC, optional encryption at rest). base64 isn't security because it's only an encoding: anyone can reverse it with `base64 -d`, without any key."*

---

## ✍️ Write the command: answers
**C1.** Create a ConfigMap named `app-config` with two keys: `APP_COLOR` set to `blue` and `APP_MODE` set to `prod`.
`kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod`: `create configmap app-config` = make a ConfigMap with that name · `--from-literal=KEY=value` = add one key–value pair typed on the command line · repeat the flag once per key (2 flags = 2 keys, DATA 2). (Alternative: put `APP_COLOR=blue` and `APP_MODE=prod` on two lines of `app.env` and run `kubectl create configmap app-config --from-env-file=app.env`.)

**C2.** Create a TLS Secret named `shop-tls` from the certificate file `tls.crt` and the key file `tls.key`.
`kubectl create secret tls shop-tls --cert=tls.crt --key=tls.key`: `create secret tls` = make a Secret of type `kubernetes.io/tls` · `shop-tls` = its name (the Ingress's `secretName`) · `--cert=tls.crt` = the certificate file (stored as key `tls.crt`) · `--key=tls.key` = the private key file (stored as key `tls.key`).

**C3.** Print the **decoded** value of the key `DB_PASSWORD` stored in the Secret `db-secret`.
`kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d`: `get secret db-secret` = read the Secret · `-o jsonpath='…'` = print only one field (single quotes protect the `{ }`) · `{.data.DB_PASSWORD}` = the base64 value of that key · `|` = pipe it on · `base64 -d` = decode it back to plain text (`S3cr3t!`). (Alternative: `kubectl exec <a-pod-using-it> -- cat /etc/secret/DB_PASSWORD` if the Secret is mounted as a volume.)

**C4.** Without creating anything in the cluster, write the YAML for an Ingress `shop` (IngressClass `nginx`) that sends `shop.example.com/` and every path below it to port 80 of the Service `web`, into a file called `ingress.yaml`.
`kubectl create ingress shop --class=nginx --rule="shop.example.com/*=web:80" --dry-run=client -o yaml > ingress.yaml`: `create ingress shop` = an Ingress named `shop` · `--class=nginx` = sets `ingressClassName: nginx` · `--rule="host/path=service:port"` = one routing rule; the trailing `*` makes it `pathType: Prefix` (no `*` = `Exact`) · `--dry-run=client` = build it locally, send nothing · `-o yaml` = print as YAML · `> ingress.yaml` = save that output to the file.
