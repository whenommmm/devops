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

## ⚡ Must-remember
| Term / command | Meaning |
|---|---|
| `--wait` on Helm | Succeed only when Pods are Ready |
| `helm rollback counter 1` | New revision that copies revision 1 |
| Ingress 404 / 503 | No rule matched / no Ready endpoints |
| HPA `<unknown>` | No CPU request, or no Metrics Server |
| Push vs pull deploy | CI runs Helm / Argo CD syncs from Git |

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
