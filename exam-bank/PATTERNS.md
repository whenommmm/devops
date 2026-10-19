# The 8 lab patterns (≈90% of all 57 labs)

**Every lab answer = 1. Do it → 2. Verify it (`get` / `ps` / `curl` / `logs`) → 3. One line saying what you saw.** Verification earns marks even if a flag is slightly off.

| Topic | The skeleton to remember |
|---|---|
| **Docker run** | `docker run -d --name X -p HOST:CONTAINER image` → `docker ps` → `curl` → `docker logs X` |
| **Docker build** | Dockerfile = **FROM · WORKDIR · COPY · CMD** → `docker build -t name:1.0 .` → run → curl (nginx serves `/usr/share/nginx/html`) |
| **Docker data / network** | `-v volname:/path` (data survives) · `docker network create net` + `--network net` (talk by name) · `docker compose up -d / ps / logs / down` |
| **K8s create** | `kubectl create deployment X --image=nginx --replicas=2` · `kubectl expose deployment X --port=80 --target-port=80 [--type=NodePort]` |
| **K8s check (always the same 4)** | `get` → `describe` → `logs` → `exec` (+ `get endpoints` for Services) |
| **K8s change** | `scale --replicas=5` · `set image deployment/X nginx=nginx:1.27` → `rollout status / history / undo` |
| **K8s config** | `kubectl create configmap X --from-literal=K=V` · `kubectl create secret generic X --from-literal=K=V` · `kubectl autoscale deployment X --cpu-percent=50 --min=1 --max=5` |
| **Tool lifecycles** | **Helm:** create → lint → template → install → upgrade → history → rollback · **Terraform:** fmt → init → validate → plan → apply → output → destroy · **Git:** init → add → commit → remote add → push · switch -c → merge (conflict: edit → add → commit) |

**GitHub Actions order:** `on` → `jobs` → `runs-on` → `steps` → `uses` / `run`. Add `needs:` to make jobs wait, and `${{ secrets.X }}` for passwords.

**Don't memorise YAML:** `kubectl create deployment web --image=nginx --dry-run=client -o yaml > web.yaml` prints ready-made YAML (works for deployment, service, configmap, secret, ingress). Forgot a field? `kubectl explain deployment.spec`.

**On your Mac:** use host port 9080 instead of 8080 (8080 is taken). In the exam, 8080 is fine.
