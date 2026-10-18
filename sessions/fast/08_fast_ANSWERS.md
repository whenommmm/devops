# Answer key · Session 08 FAST · Ingress, ConfigMaps & Secrets
⚠️ Only open after answering.

**1.** An Ingress for `shop.example.com` has two Prefix rules: `/api` → Service `orders`, and `/` → Service `web`. `kubectl get svc` lists only `web`. What does `curl -H "Host: shop.example.com" http://localhost:9080/api/v1` return?
- A) 404, since no rule covers the path /api/v1
- B) The web page, since `/` catches the request
- C) 503, as the matched Service doesn't exist
- D) 308, since the controller forces HTTPS first

**Correct: C.** `/api/v1` matches both `/` and `/api`, and the longest match (`/api`) wins. That rule's Service `orders` doesn't exist, so NGINX answers 503. A: a rule did match, so it isn't 404. B: the controller doesn't fall back to `/` when the chosen backend is missing. D: there's no `tls:` section, so there's no redirect.

---

**2.** A Deployment's Pods get `APP_MODE` through `envFrom` and also mount the same ConfigMap at `/etc/config`. You change `APP_MODE` from `dev` to `prod` with `kubectl patch`. Two minutes later, what does a running container see?
- A) The file says prod; the env var still says dev
- B) Both say prod; the kubelet refreshes them all
- C) Both still say dev until the Pods are recreated
- D) The env var says prod; the file still says dev

**Correct: A.** Mounted ConfigMap files update by themselves after about a minute, but environment variables are read only once, when the container starts. B: env vars are never refreshed in a running container. C is half right: the env var stays dev, but the file does update. D swaps the two.

---

**3.** A Secret YAML has the line `DB_PASSWORD: password` under `data:`. `kubectl apply` succeeds, but the app can't log in to its database. Why?
- A) Secrets must be made with create, never apply
- B) The Pod needs imagePullSecrets to read Secrets
- C) The app lacks the key to decrypt the base64
- D) data: needs base64, so it decodes to garbage

**Correct: D.** Values under `data:` must already be base64. The letters of `password` happen to be valid base64, so the API server accepts them, but they decode to garbled bytes. (`stringData:` would have taken the plain text.) A: `apply` works fine for Secrets. B: `imagePullSecrets` is only for pulling private images. C: base64 is encoding, not encryption, so there's no key at all.

---

**4.** Which statement about TLS on an NGINX Ingress is NOT true?
- A) Its Secret must be in the Ingress's namespace
- B) Backend Pods must each hold the certificate
- C) Plain HTTP to that host gets a 308 redirect
- D) The Secret holds exactly tls.crt and tls.key

**Correct: B.** With TLS termination, the controller holds the certificate, decrypts the request, and forwards plain HTTP to the Pods, so the Pods need no certificate. A, C and D are all true.

---

**5.** A Pod in namespace `shop` shows `CreateContainerConfigError` with 0 restarts, and `describe` says `configmap "app-config" not found`. Yet `kubectl get configmap app-config` prints it without errors. What's the most likely cause?
- A) It's in `default`; Pods only see their own namespace
- B) The app crashed; `logs --previous` shows the reason
- C) ConfigMaps sync slowly; the kubelet will succeed soon
- D) envFrom works only for Secrets, so use valueFrom

**Correct: A.** `kubectl get` without `-n` looks in `default`, so the ConfigMap exists there, but a Pod can only use ConfigMaps from its own namespace (`shop`). B: with `CreateContainerConfigError` the container was never created, so there's no crash and no logs. C: the kubelet does retry, but it will keep failing until the ConfigMap exists in `shop`. D: `envFrom` works with both ConfigMaps (`configMapRef`) and Secrets (`secretRef`).
