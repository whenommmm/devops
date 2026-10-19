# Labs · Session 14: Kubernetes Troubleshooting

## Lab 1 · Deploy with a bad image, diagnose ImagePullBackOff (get, describe, events), fix the image
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:doesnotexist   # broken on purpose
kubectl get pods -l app=web                  # STATUS ErrImagePull -> ImagePullBackOff, READY 0/1
kubectl describe pod -l app=web              # Events: Failed to pull image "nginx:doesnotexist": ... not found
kubectl get events --sort-by=.lastTimestamp  # newest events last: Pulling -> Failed -> BackOff
kubectl set image deployment/web nginx=nginx:1.27          # container "nginx" gets a real tag
kubectl rollout status deployment/web        # successfully rolled out
kubectl get pods -l app=web                  # Running, READY 1/1
```
**What each line does:**
- `create deployment ... --image=nginx:doesnotexist`: the tag does not exist on Docker Hub, so the node cannot download the image.
- `get pods`: quick status. `ErrImagePull` = first pull failed; `ImagePullBackOff` = Kubernetes is now waiting longer and longer (back-off) between retries.
- `describe pod`: the **Events** section at the bottom gives the exact reason (`manifest unknown` / `not found` = wrong name or tag; `unauthorized` / `pull access denied` = private registry needs an `imagePullSecret`).
- `get events --sort-by=.lastTimestamp`: all recent cluster events in time order (pull errors, scheduling errors).
- `set image deployment/web nginx=nginx:1.27`: `<container name>=<new image>`; `kubectl create deployment` names the container after the image (`nginx`). This triggers a new rollout.
- `rollout status`: waits until the new Pods are up.
**Verify:** `kubectl get pods -l app=web` → `Running 1/1`; `kubectl describe pod -l app=web | grep Image:` → `nginx:1.27`.
(In a manifest-based setup, fix the `image:` line in the YAML and `kubectl apply -f` again.)

**🧠 In simple words:** The Pod could not start because the node could not download the image. `describe` and `events` tell you why (here: the tag does not exist), and `set image` points the Deployment to a real image, which starts fresh Pods. In short: ImagePullBackOff = wrong or private image; read the Events, fix the image name.

## Lab 2 · Diagnose a CrashLoopBackOff with logs and describe, fix the config, verify recovery
**Answer (what to write in the exam):**
```bash
kubectl apply -f api.yaml
kubectl get pods -l app=api                  # STATUS CrashLoopBackOff, RESTARTS 3, 4, 5...
kubectl logs deploy/api --previous           # ERROR: DB_HOST is not set
kubectl describe pod -l app=api              # Last State: Terminated, Reason: Error, Exit Code: 1
                                             # Events: Back-off restarting failed container
kubectl set env deployment/api DB_HOST=db.default.svc.cluster.local   # add the missing setting
kubectl rollout status deployment/api
kubectl get pods -l app=api                  # Running 1/1, RESTARTS 0 (new Pod)
kubectl logs deploy/api                      # Started, DB=db.default.svc.cluster.local
```
`api.yaml` (broken: needs `DB_HOST` but none is given):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: busybox:1.36
          command: ["sh", "-c"]
          args:
            - 'if [ -z "$DB_HOST" ]; then echo "ERROR: DB_HOST is not set"; exit 1; fi; echo "Started, DB=$DB_HOST"; sleep 3600'
```
**What each line does:**
- CrashLoopBackOff: the container starts, exits with an error, is restarted, crashes again; Kubernetes waits longer between each retry.
- `logs --previous` (`-p`): shows logs of the **last crashed** container (the current one may have no output yet). This is where the root cause usually is.
- `describe pod`: `Last State: Terminated` with the **Exit Code**: `1` = app error, `137` = killed (often `OOMKilled`, out of memory: raise memory limit), `127` = command not found.
- `set env deployment/api DB_HOST=...`: adds the missing environment variable; the Deployment rolls out new Pods with it. (Better long-term: add `env:` to `api.yaml` and `kubectl apply -f api.yaml`.)
**Verify:** `kubectl get pods -l app=api` → `Running`, `RESTARTS 0`, and stays that way; `kubectl logs deploy/api` → `Started, DB=...`.

**🧠 In simple words:** The app kept crashing on startup because a required setting was missing. The previous container's logs said exactly what was wrong and `describe` confirmed it exited with code 1; adding the setting made the new Pods stay up. In short: CrashLoopBackOff = app keeps exiting; read `logs --previous` and the exit code, fix the cause, re-check.

## Lab 3 · Create a Service with a wrong selector, find the problem via labels and endpoints, fix it
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=2   # Pods get label app=web
kubectl apply -f web-svc.yaml                # Service selects app=webapp (typo)
kubectl get endpoints web-svc                # ENDPOINTS <none>   <- the symptom
kubectl describe svc web-svc                 # Selector: app=webapp, Endpoints: (empty)
kubectl get pods --show-labels               # LABELS app=web,...
kubectl get pods -l app=webapp               # No resources found -> selector matches nothing
kubectl edit svc web-svc                     # change selector to app: web, save
kubectl get endpoints web-svc                # 10.244.0.5:80,10.244.0.6:80
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://web-svc
```
`web-svc.yaml` (broken; the fix is `app: web`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
```
**What each line does:**
- A Service finds its Pods **only by labels**: `selector: app: webapp` means "send traffic to Pods labelled `app=webapp`". None exist, so the Service has no endpoints and requests fail.
- `get endpoints web-svc`: the list of Pod IP:port pairs behind the Service; `<none>` is the classic sign of a selector mismatch (or of Pods not being Ready).
- `describe svc`: shows the Service's `Selector`, `TargetPort` and `Endpoints` side by side.
- `get pods --show-labels` / `get pods -l app=webapp`: compare the real Pod labels with what the selector asks for.
- `kubectl edit svc web-svc`: opens the live object in an editor; changing the selector to `app: web` fixes it immediately. (Same fix: correct the YAML and `kubectl apply -f web-svc.yaml`, or `kubectl patch svc web-svc -p '{"spec":{"selector":{"app":"web"}}}'`.)
- Also check `targetPort` equals the container's port (80 for nginx); a wrong port gives endpoints but connection errors.
**Verify:** `kubectl get endpoints web-svc` → two Pod IPs on port 80; the `wget` from the temporary busybox Pod prints the nginx welcome HTML.

**🧠 In simple words:** The Service was looking for Pods with a label that no Pod had, so it pointed at nothing. Comparing the Service selector with the Pods' real labels showed the typo, and fixing the selector filled the endpoints. In short: empty endpoints = selector does not match Pod labels (or Pods not ready).
