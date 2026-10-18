# Session 16 · Final DevOps Project & Troubleshooting · FAST · Answer key

**1.** The `build-scan-push` job pushes `ghcr.io/vansh/counter:${{ github.sha }}`. The `deploy` job then runs the command below (no `--wait`). Before this, 3 healthy Pods were serving, and the Deployment uses its default rolling update. A few minutes later, what do you see?
```bash
helm upgrade counter ./charts/counter -n counter \
  --set image.tag=$(git rev-parse --short HEAD)
```
- A) Helm says `deployed`; all 3 Pods hit ImagePullBackOff, the site is down
- B) Helm says `deployed`; 1 new Pod in ImagePullBackOff, old ones serve
- C) Helm says `failed`; 1 new Pod in ImagePullBackOff, old ones serve
- D) Helm says `deployed`; 1 new Pod in CrashLoopBackOff, old ones still up

**Correct: B.** CI pushed the image under the full 40-character SHA (`github.sha`), but the deploy asks for the 7-character short SHA, a tag that does not exist in the registry, so the new Pod cannot download its image and shows `ImagePullBackOff`. Without `--wait`, Helm prints `STATUS: deployed` as soon as Kubernetes accepts the YAML, and the rolling update keeps the old Pods serving while the broken new Pod waits. A is wrong because the default rolling update starts one new Pod first and keeps the old Pods serving until it is Ready, which never happens here, so the site stays up. C is wrong because only `--wait` makes Helm wait for Ready Pods and report a failure; without it Helm says `deployed`. D is wrong because `CrashLoopBackOff` means the image downloaded and the app started and crashed, but here the image cannot even be downloaded.

---

**2.** `curl -H "Host: counter.local" http://localhost:9080/` returns `503` from nginx. `kubectl get pods --show-labels -n counter` shows the web Pods `1/1 Running` with the label `app=counter-web`, and `kubectl get endpointslices -l kubernetes.io/service-name=counter-web -n counter` lists no endpoints. Which single change fixes it?
- A) Add `ingressClassName: nginx` to the Ingress spec
- B) Set the Service's `targetPort` to 5000, the app port
- C) Change the Service selector to `app: counter-web`
- D) Point the readiness probe at `/health` instead

**Correct: C.** A 503 from nginx means an Ingress rule matched but its Service has no Ready endpoints. The Pods are `1/1`, so they are Ready; the only other reason for empty endpoints is that the Service's selector matches none of the Pods' labels, so setting the selector to `app: counter-web` fills the endpoints. A is wrong because a missing `ingressClassName` means no rule is applied at all, which gives a 404, not a 503. B is wrong because a wrong `targetPort` still produces endpoints (the symptom would be "endpoints exist, connection refused"). D is wrong because `1/1` already shows the readiness probe is passing, so the probe is not what keeps the Pods out of the endpoints.

---

**3.** An HPA created ten minutes ago still shows `TARGETS cpu: <unknown>/50%` and has never added a Pod. `kubectl top pods -n counter` prints CPU and memory numbers for every web Pod. What is the most likely cause?
- A) The web Pods have no CPU request set in the chart
- B) Metrics Server is not installed in the cluster yet
- C) The HPA is still inside its normal warm-up window
- D) The Deployment is already at its maxReplicas count

**Correct: A.** The HPA's target is "50% of each Pod's CPU request", so if the chart sets no CPU request there is nothing to divide by and the HPA shows `<unknown>`. B is wrong because `kubectl top pods` prints numbers only when Metrics Server is running and collecting them, so it is clearly installed. C is wrong because `<unknown>` is normal only for the first 1–2 minutes, not after ten. D is wrong because a Deployment at its maximum would still show a real percentage (for example `90%/50%`); `<unknown>` means the HPA has no usable number at all.

---

**4.** Riya's `deploy` job runs `helm upgrade --install` on a GitHub-hosted runner, aimed at the minikube cluster on her Mac, and it times out trying to connect to the cluster. She also wants no cluster credentials stored in GitHub. What should she change?
- A) Store the cluster's credentials as a repository secret
- B) Replace `helm upgrade` with `kubectl apply` in the job
- C) Add `--wait` so Helm keeps retrying until it connects
- D) Run Argo CD in minikube; CI commits the tag to Git

**Correct: D.** This is the pull (GitOps) style: Argo CD runs inside the cluster and reaches out to Git, so CI only commits the new tag and never needs to reach the cluster or hold its credentials. GitHub's runners cannot reach minikube on a Mac, which is why the push-style job times out. A is wrong on both counts: it stores the very credentials she wants to avoid, and the runner still cannot reach her Mac. B is wrong because `kubectl apply` from the runner is still push style, so it hits the same unreachable cluster. C is wrong because `--wait` makes Helm wait for Pods to become Ready after it has connected; it does not help a runner that cannot reach the cluster at all.

---

**5.** During your demo, revision 1 of the release `counter` runs a good image. You run `helm upgrade counter ./charts/counter -n counter --reuse-values --set image.tag=2.0` (tag `2.0` does not exist, and there is no `--wait`), then `helm rollback counter 1 -n counter`. Which pair of observations is correct?
- A) upgrade printed `failed`; history ends at revision 3, "Rollback to 1"
- B) upgrade printed `deployed`; history ends at revision 3, "Rollback to 1"
- C) upgrade printed `deployed`; history ends at revision 1, deployed again
- D) upgrade printed `failed`; history ends at revision 2, marked rolled back

**Correct: B.** Without `--wait`, Helm reports `deployed` as soon as Kubernetes accepts the YAML, even though the new Pod sits in `ImagePullBackOff`. A rollback never rewinds the history; it creates a new revision that copies the old one, so revision 1 (install) and revision 2 (bad tag) are followed by revision 3, "Rollback to 1". A gets the history right but the upgrade message wrong, since Helm only reports a failure when it waits for Ready Pods. C gets the message right but the history wrong, because revision numbers only ever go up. D is wrong on both halves: the upgrade said `deployed`, and the latest entry is the new revision 3, not revision 2.

---
