# Session 16 · Final DevOps Project & Troubleshooting · FAST (10-min read)
*How one code change reaches users, what breaks at each stop, and how to present it in a demo or viva.*

## 1 · The whole DevOps lifecycle in one picture
DevOps (development + operations) is one loop: code is built, tested, shipped and watched, and what you learn becomes the next task. Every arrow below is automated, like a factory assembly line with inspection points.

```
 Plan → Code (GitHub) → Pull Request + review → merge to main
  ▲                       ▼ push triggers CI
  │   CI: GitHub Actions  test → scans → docker build → Trivy ══ GATE ══
  │                       ▼ clean images only
  │   Registry: ghcr.io/vansh/counter:<commit SHA>
  │                       ▼
  │   CD: helm upgrade from CI (push)  or  Argo CD syncs from Git (pull)
  │                       ▼
  │   Kubernetes: Ingress → Service → Deployment → Pods (+ probes, HPA)
  │                       ▼ /metrics scraped
 Feedback ◄── Monitor: Prometheus → Grafana → Alertmanager
 Underneath: Terraform builds the network and cluster as code
```
- CI (Continuous Integration) builds and tests every change and reports failures; developers, not CI, fix them.
- CD (Continuous Delivery or Deployment) moves the tested image from the registry (the image store) into the cluster.
- The feedback arrow from monitoring back to planning makes this a lifecycle, not a pipeline; drawings without it lose marks.

⚠️ **Exam trap:** In push deployment the CI runner (the job's machine) runs `helm upgrade` with cluster credentials, whereas in pull deployment (GitOps) Argo CD inside the cluster applies whatever Git says; GitHub's runners cannot reach minikube on your Mac, so only pull works there.

## 2 · Code to production, step by step
The counter app is a Flask (Python web framework) page that counts visits in Redis (an in-memory database).

1. **App → GitHub:** branch, commit, open a PR (Pull Request), merge into a protected `main`. Produces reviewed code.
2. **Dockerize:** install dependencies before copying code, run as a non-root user, listen on `0.0.0.0` (every network interface). Produces a tagged image.
3. **CI/CD pipeline (GitHub Actions):** jobs `test` → `build-scan-push` → `deploy`, each on a fresh runner; push and deploy run only on `main`. Produces an image in the registry.
4. **Security scan:** gitleaks (leaked secrets), SAST (Static Application Security Testing, reads source code: Bandit), SCA (Software Composition Analysis, libraries with known vulnerabilities: pip-audit), Trivy (the image). Produces a pass or a failed gate (a non-zero exit stopping the job).
5. **Deploy to Kubernetes (runs containers across machines):** minikube, a one-machine cluster on your Mac, cannot see your Docker images, so run `minikube image load counter:<tag>` first. Produces the image inside minikube.
6. **Helm (package manager for Kubernetes):** one chart holds the Deployment (keeps N Pods, the units that run containers, alive; health probes on `/health`), the Service (stable address, port 80 → targetPort 5000), and Redis with its PVC (PersistentVolumeClaim, a disk request). Produces a release (an installed copy) with numbered revisions.
7. **Ingress (web routing rules):** host `counter.local`, path `/` → Service `counter-web:80`. Produces one entrance for users.
8. **HPA (Horizontal Pod Autoscaler):** keeps average CPU (processor) use at 50% of each Pod's CPU request, with 2 to 5 replicas. Produces extra Pods under load.
9. **Monitoring:** Prometheus scrapes (pulls) `/metrics`, Grafana draws dashboards, Alertmanager alerts. Produces feedback for the next plan.

```bash
helm upgrade --install counter ./charts/counter -n counter \
  --create-namespace --set image.tag=$TAG --wait
```
- `upgrade --install` installs if missing, otherwise upgrades, so re-running is safe.
- `--wait` reports success only once Pods are Ready; without it Helm prints `STATUS: deployed` as soon as Kubernetes accepts the config, even if the new Pods are broken.

⚠️ **Exam trap:** In CI, `${{ github.sha }}` is the full 40-character commit SHA (Secure Hash Algorithm, the commit's unique ID), whereas `git rev-parse --short HEAD` gives 7 characters, so pushing one and deploying the other points at a missing tag. Adding `|| true` to a failing scan forces exit 0, deleting the gate.

## 3 · What breaks where
Method: read the exact error, run one command that shows more, fix one thing, verify. An endpoint is a Ready Pod's address behind a Service; Metrics Server collects the CPU numbers that `kubectl top pods` prints and the HPA uses.

| Layer | Symptom | Likely cause | Check with |
|---|---|---|---|
| Docker build | `COPY failed: file not found` | File outside the build folder, or in `.dockerignore` | `cat .dockerignore` |
| Pipeline | `exit code 1` | A test or scanner failed: the gate worked | `gh run view --log-failed` |
| Image pull | `ImagePullBackOff` | Wrong name or tag, image not loaded into minikube, no registry login | `kubectl describe pod` → Events |
| Pod status | `CrashLoopBackOff` | The app starts, then crashes | `kubectl logs <pod> --previous` |
| Pod status | `Pending` | Requests more CPU than any node has free | `kubectl describe pod` → Events |
| Pod status | `1/1 Running`, `/` gives 500 | Bad config, e.g. `REDIS_HOST` names a missing Service | `kubectl logs deploy/counter-web` |
| Service | No endpoints | Selector does not match Pod labels, or Pods not Ready | `kubectl get pods --show-labels` |
| Service | Endpoints exist, connection refused | `targetPort` is not the app's real port | `kubectl describe svc` |
| Ingress | `404` from nginx (the Ingress controller) | Host or path does not match, or no `ingressClassName` | `kubectl describe ingress` |
| Ingress | `503` from nginx | Rule matched, but no Ready endpoints | `kubectl get endpointslices` |
| HPA | `cpu: <unknown>/50%` | No CPU request, or no Metrics Server (normal for 1–2 minutes) | `kubectl describe hpa` |

- `READY 1/1` means the readiness probe passes; failing readiness shows `0/1` and removes the Pod from endpoints, while failing liveness restarts it.
- During a rolling update (a Deployment's default way of replacing Pods), the old Pods keep serving while a broken new Pod waits.

⚠️ **Exam trap:** A 404 from nginx means no Ingress rule matched, whereas a 503 means a rule matched but its Service has no Ready Pods.

## 4 · Explaining your project in a demo or viva
Examiners check understanding, so tell one story:
1. Draw the lifecycle picture and say one sentence per arrow.
2. Show the PR and the pipeline going green.
3. Run `kubectl get pods,svc,ingress,hpa,pvc -n counter`, then `curl` twice (count 1, then 2).
4. Run a load test, watch the HPA add Pods, open Grafana.
5. Break it: deploy a missing tag `2.0`, show `ImagePullBackOff`, then `helm rollback counter 1`; `helm history` shows revision 3, "Rollback to 1", because a rollback is a new revision.
6. Close: "In the cloud, Terraform (Infrastructure as Code) builds the network and cluster."

- **Scan story:** Bandit's `B104` warning flagged `host="0.0.0.0"`; a container app must listen there, so the fix is a reviewed `# nosec B104` exception on that line.
- **Outage story:** after the `REDIS_HOST` bug, the team added a smoke test (post-deploy check that `/` returns 200), a blameless postmortem (causes, not blame), and a runbook (step-by-step fix guide).

⚠️ **Exam trap:** An Ingress is only routing rules, whereas the Ingress controller (ingress-nginx) is the program that applies them.

## 🎯 Exam checklist (everything that could be asked)
*MCQ = know the fact · CMD = be able to write it from memory.*

| Asked as | Item | What you must know / the exact command |
|---|---|---|
| MCQ | **Lifecycle** · the loop | Plan → Code → Build → Test → Release → Deploy → Operate → Monitor, plus a feedback arrow from Monitor back to Plan; without that arrow it is only a pipeline. |
| MCQ | CI vs Continuous Delivery vs Deployment | CI (Continuous Integration) builds and tests every change and reports failures, which developers fix; Continuous Delivery needs a human approval before production, whereas Continuous Deployment goes there automatically. |
| MCQ | Push vs pull deployment | In push deployment the CI runner runs `helm upgrade` holding cluster credentials, whereas in pull (GitOps) Argo CD inside the cluster syncs from Git; GitHub's runners can't reach minikube, so only pull works there. |
| CMD | **App → GitHub** · branch and PR | `git switch -c feature/welcome` → commit → `gh pr create --fill`: work on a branch and open a PR (Pull Request); a protected `main` accepts merges only via a PR with a green workflow. |
| CMD | Push rejected with `(fetch first)` | `git pull --rebase origin main && git push`: replays your commits on top of your teammate's new ones, then pushes. Never `--force` a shared branch. |
| MCQ | **Dockerize** · Dockerfile rules | `COPY requirements.txt .` and `RUN pip install` go before `COPY . .` (a changed layer rebuilds every later one); run as a non-root `USER appuser`; the app listens on `0.0.0.0`. |
| CMD | Build tagged with the commit SHA | `TAG=$(git rev-parse --short HEAD) && docker build -t counter:$TAG .`: stores the short commit SHA (Secure Hash Algorithm ID), then builds `counter:<sha>` from the current folder (the final `.`). |
| MCQ | SHA tag vs `:latest` | A commit-SHA tag is unique and traces back to its exact code, whereas `:latest` is ambiguous and makes `imagePullPolicy` default to `Always`; any other tag defaults to `IfNotPresent`. |
| MCQ | **CI/CD pipeline** · jobs | Jobs `test` → `build-scan-push` (`needs: test`) → `deploy`, each on a fresh runner (parallel unless `needs:` links them); push and deploy run only `if: github.ref == 'refs/heads/main'`, so PRs are never deployed. |
| MCQ | Full vs short SHA trap | `${{ github.sha }}` is the full 40-character SHA, whereas `git rev-parse --short HEAD` gives 7 characters; pushing one and deploying the other gives `ImagePullBackOff` (tag not found). |
| MCQ | **Security scan** · what each scans | gitleaks = leaked secrets · SAST (Static Application Security Testing, Bandit) = your source code · SCA (Software Composition Analysis, pip-audit) = libraries with known CVEs (Common Vulnerabilities and Exposures) · Trivy = the image · DAST (Dynamic Application Security Testing) = attacks the running app. |
| CMD | Trivy image gate | `trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 counter:$TAG`: exits 1 on fixable HIGH/CRITICAL findings, failing the job before the push; without `--exit-code 1` Trivy exits 0. |
| MCQ | Gate traps | A gate is a step whose non-zero exit stops the job; `\|\| true` forces exit 0 and deletes the gate; `--severity CRITICAL` ignores HIGH, so an image with 3 HIGH and 0 CRITICAL passes. |
| MCQ | Bandit `B104` and pip-audit fixes | B104 flags `host="0.0.0.0"`, but a container app must listen there, so add a reviewed `# nosec B104` on that line; for a pip-audit finding, upgrade to the fixed version. |
| CMD | Tag and push to the registry | `docker tag counter:$TAG ghcr.io/vansh/counter:$TAG && docker push ghcr.io/vansh/counter:$TAG`: adds a registry name (no copy is made), then uploads it; run `docker login ghcr.io` first. |
| CMD | **Deploy to Kubernetes** · load the image | `minikube image load counter:$TAG`: minikube cannot see Colima's Docker images; skip this and Pods show `ErrImagePull` / `ImagePullBackOff`, because Kubernetes searches Docker Hub instead. |
| MCQ | Rolling update vs other strategies | A rolling update (the default) replaces Pods gradually, so old Pods keep serving beside a broken new one; Recreate stops all first; blue-green switches all traffic at once; canary sends a small share of traffic first. |
| CMD | Check the rollout | `kubectl rollout status deploy/counter-web -n counter --timeout=120s`: waits for the rollout; exits 0 on success, or 1 with `timed out waiting for the condition`, turning a CI step red. |
| CMD | **Helm** · install or upgrade | `helm upgrade --install counter ./charts/counter -n counter --create-namespace --set image.tag=$TAG --wait`: installs if missing, otherwise upgrades (safe to re-run); succeeds only once Pods are Ready. |
| MCQ | `--wait` trap | Without `--wait`, Helm prints `STATUS: deployed` as soon as Kubernetes accepts the YAML, even if the new Pod is in `ImagePullBackOff`; `--rollback-on-failure` (Helm 3: `--atomic`) rolls back automatically on failure. |
| MCQ | Helm words | A chart is the package, whereas a release is an installed copy with numbered revisions; `values.yaml` holds defaults and `--set` overrides them; with an HPA on, the template leaves out `replicas`. |
| MCQ | **Ingress** · rules vs controller | An Ingress is only routing rules (host `counter.local`, path `/` → Service `counter-web:80`), whereas the Ingress controller (ingress-nginx) is the program that applies them. |
| CMD | Inspect and test the Ingress | `kubectl describe ingress counter-web -n counter` shows host, path, backend Service and Events; test with `curl -H "Host: counter.local" http://localhost:9080/` after port-forwarding the controller. |
| MCQ | Ingress 404 vs 503 | A 404 from nginx means no rule matched (wrong Host header or path, or no `ingressClassName`), whereas a 503 means a rule matched but its Service has no Ready endpoints. |
| CMD | **HPA** · create and watch | `kubectl autoscale deployment counter-web -n counter --cpu=50% --min=2 --max=5` (older spelling `--cpu-percent=50`), then `kubectl get hpa -n counter -w` to watch replicas change. |
| MCQ | How the HPA scales | HPA = Horizontal Pod Autoscaler; it needs Metrics Server and CPU requests (the target is a % of the request); desired = ceil(current × actual ÷ target), so 2 Pods at 90% with a 50% target → 4. |
| MCQ | HPA shows `cpu: <unknown>/50%` | The Pods have no CPU request or Metrics Server is missing (normal for the first 1–2 minutes); if `kubectl top pods` prints numbers, Metrics Server works, so the CPU request is missing. |
| MCQ | **Monitoring** · the three tools | Prometheus pulls (scrapes) `/metrics`, Grafana draws dashboards, Alertmanager sends alerts; a Prometheus target marked DOWN means it cannot scrape the app, so Grafana shows no data. |
| CMD | **Troubleshooting** · Service endpoints | `kubectl get endpoints counter-web -n counter` (or `get endpointslices`): none means the selector does not match Pod labels (`kubectl get pods -n counter --show-labels`) or Pods aren't Ready; present but refused means a wrong `targetPort`. |
| MCQ | Pod status words | `ImagePullBackOff` = wrong name/tag, image not loaded, or no registry login (`describe` → Events); `CrashLoopBackOff` = app starts, then crashes (`logs --previous`); `Pending` = can't be scheduled, e.g. `Insufficient cpu`. |
| MCQ | Readiness vs liveness | A failing readiness probe shows `0/1` and removes the Pod from the Service's endpoints without a restart, whereas a failing liveness probe restarts the container. |
| MCQ | `1/1 Running` but `/` returns 500 | Probes hit `/health`, which skips Redis, so Pods look healthy; `kubectl logs deploy/counter-web -n counter` shows `Error -2 connecting to cache:6379`: `REDIS_HOST` names a missing Service. |
| CMD | **Final project demo** · show it all | `kubectl get pods,svc,ingress,hpa,pvc -n counter` (pvc = PersistentVolumeClaim), then `curl` twice (visited 1, then 2 times); close with "Terraform builds the VPC (Virtual Private Cloud) and cluster underneath". |
| CMD | Break it, then roll back | `helm history counter -n counter` then `helm rollback counter 1 -n counter`: a rollback is a new revision, so history ends at revision 3, "Rollback to 1". |
| MCQ | Smoke test, postmortem, runbook | A smoke test is a tiny post-deploy check that `/` returns 200; a blameless postmortem records causes and fixes, not blame; a runbook is a step-by-step guide for known problems. |

## 📝 Quiz (5 MCQs; answer in chat as "1 B, 2 D, …")

**1.** The `build-scan-push` job pushes `ghcr.io/vansh/counter:${{ github.sha }}`. The `deploy` job then runs the command below (no `--wait`). Before this, 3 healthy Pods were serving, and the Deployment uses its default rolling update. A few minutes later, what do you see?
```bash
helm upgrade counter ./charts/counter -n counter \
  --set image.tag=$(git rev-parse --short HEAD)
```
- A) Helm says `deployed`; all 3 Pods hit ImagePullBackOff, the site is down
- B) Helm says `deployed`; 1 new Pod in ImagePullBackOff, old ones serve
- C) Helm says `failed`; 1 new Pod in ImagePullBackOff, old ones serve
- D) Helm says `deployed`; 1 new Pod in CrashLoopBackOff, old ones still up

**2.** `curl -H "Host: counter.local" http://localhost:9080/` returns `503` from nginx. `kubectl get pods --show-labels -n counter` shows the web Pods `1/1 Running` with the label `app=counter-web`, and `kubectl get endpointslices -l kubernetes.io/service-name=counter-web -n counter` lists no endpoints. Which single change fixes it?
- A) Add `ingressClassName: nginx` to the Ingress spec
- B) Set the Service's `targetPort` to 5000, the app port
- C) Change the Service selector to `app: counter-web`
- D) Point the readiness probe at `/health` instead

**3.** An HPA created ten minutes ago still shows `TARGETS cpu: <unknown>/50%` and has never added a Pod. `kubectl top pods -n counter` prints CPU and memory numbers for every web Pod. What is the most likely cause?
- A) The web Pods have no CPU request set in the chart
- B) Metrics Server is not installed in the cluster yet
- C) The HPA is still inside its normal warm-up window
- D) The Deployment is already at its maxReplicas count

**4.** Riya's `deploy` job runs `helm upgrade --install` on a GitHub-hosted runner, aimed at the minikube cluster on her Mac, and it times out trying to connect to the cluster. She also wants no cluster credentials stored in GitHub. What should she change?
- A) Store the cluster's credentials as a repository secret
- B) Replace `helm upgrade` with `kubectl apply` in the job
- C) Add `--wait` so Helm keeps retrying until it connects
- D) Run Argo CD in minikube; CI commits the tag to Git

**5.** During your demo, revision 1 of the release `counter` runs a good image. You run `helm upgrade counter ./charts/counter -n counter --reuse-values --set image.tag=2.0` (tag `2.0` does not exist, and there is no `--wait`), then `helm rollback counter 1 -n counter`. Which pair of observations is correct?
- A) upgrade printed `failed`; history ends at revision 3, "Rollback to 1"
- B) upgrade printed `deployed`; history ends at revision 3, "Rollback to 1"
- C) upgrade printed `deployed`; history ends at revision 1, deployed again
- D) upgrade printed `failed`; history ends at revision 2, marked rolled back
