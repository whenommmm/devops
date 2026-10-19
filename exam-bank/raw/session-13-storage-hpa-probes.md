# Labs · Session 13: Storage, HPA and Probes

## Lab 1 · Create a PVC, mount it in a Pod, write a file, recreate the Pod, prove the file survived
**Answer (what to write in the exam):**
```bash
kubectl apply -f data-pvc.yaml -f data-pod.yaml
kubectl get pvc data-pvc                              # STATUS Bound
kubectl wait --for=condition=Ready pod/data-pod
kubectl exec data-pod -- sh -c 'echo hello-from-pvc > /data/note.txt'
kubectl delete pod data-pod                           # Pod gone, PVC stays
kubectl apply -f data-pod.yaml                        # brand-new Pod, same PVC
kubectl wait --for=condition=Ready pod/data-pod
kubectl exec data-pod -- cat /data/note.txt           # hello-from-pvc
```
`data-pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```
`data-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```
**What each line does:**
- PVC (PersistentVolumeClaim): a request "give me 100Mi of disk". No `storageClassName` → the cluster's **default StorageClass** (minikube: `standard`) creates a PV (PersistentVolume) automatically and binds it.
- `ReadWriteOnce` (RWO): the volume can be mounted read-write by one node at a time.
- `volumes: persistentVolumeClaim: claimName` + `volumeMounts: mountPath: /data`: attach the claimed disk inside the container at `/data`.
- `kubectl wait --for=condition=Ready`: blocks until the Pod is running.
- `delete pod` then `apply` again: the container filesystem is thrown away, but `/data` lives on the PV, not in the container.
**Verify:** `kubectl get pvc,pv` → PVC `Bound` to a PV; the last `cat` prints `hello-from-pvc` from the new Pod.
(If the PVC stays `Pending`, there is no default StorageClass: check `kubectl get storageclass`.)

**🧠 In simple words:** Anything a container writes to its own filesystem dies with the Pod. A PVC gives the Pod a separate disk that outlives it, so a new Pod mounting the same claim sees the old data. In short: PVC = storage that survives Pod restarts and deletes.

## Lab 2 · Add readiness and liveness probes and show how Kubernetes reacts when each one fails
**Answer (what to write in the exam):**
```bash
kubectl apply -f web-probes.yaml && kubectl expose deployment web --port=80
kubectl get pods -l app=web                          # READY 1/1, RESTARTS 0
# A) app NOT READY: readiness path now returns 404
kubectl exec deploy/web -- rm /usr/share/nginx/html/index.html
kubectl get pods -l app=web -w                       # READY 0/1, RESTARTS stays 0
kubectl get endpoints web                            # ENDPOINTS <none>: no traffic sent
kubectl exec deploy/web -- sh -c 'echo ok > /usr/share/nginx/html/index.html'   # back to 1/1
# B) app UNHEALTHY: liveness path now returns 404
kubectl exec deploy/web -- rm /usr/share/nginx/html/50x.html
kubectl get pods -l app=web -w                       # RESTARTS 0 -> 1, then 1/1 again
kubectl describe pod -l app=web | grep -iE 'unhealthy|restarted'
```
`web-probes.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /index.html
              port: 80
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /50x.html
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
```
**What each line does:**
- `readinessProbe httpGet`: every 5 s the kubelet calls `GET /index.html`; 200-399 = ready. Fail → Pod marked **not ready** and **removed from Service endpoints**, but **not restarted**.
- `livenessProbe httpGet`: checks `/50x.html` (a file shipped in the nginx image). 3 failures in a row (`failureThreshold`) → kubelet **kills and restarts** the container; `RESTARTS` goes up.
- `initialDelaySeconds`: wait before the first check so a starting app is not killed.
- Real apps use endpoints like `/ready` and `/healthz`; two nginx files are used here only so each probe can be broken separately.
- After the restart the container starts fresh from the image, so both files are back and the Pod is healthy.
**Verify:** Events show `Readiness probe failed: HTTP probe failed with statuscode: 404` (case A) and `Liveness probe failed ... Container nginx failed liveness probe, will be restarted` (case B).

**🧠 In simple words:** Readiness answers "can this Pod take traffic right now?"; if not, the Service just stops sending it requests. Liveness answers "is this Pod stuck or broken?"; if so, Kubernetes restarts it. In short: readiness failure = removed from traffic, liveness failure = restart.

## Lab 3 · Configure an HPA on a Deployment with CPU requests, generate load, watch it scale
**Answer (what to write in the exam):**
```bash
minikube addons enable metrics-server                # prerequisite 1: metrics
kubectl top pods                                     # works after ~1 min = metrics ready
kubectl apply -f web-hpa-app.yaml                    # Deployment (with CPU request) + Service
kubectl autoscale deployment web --cpu-percent=50 --min=1 --max=5
kubectl get hpa web                                  # TARGETS cpu: 0%/50%
kubectl run load --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://web; done"
kubectl get hpa web -w                               # TARGETS > 50%, REPLICAS 1 -> 2 ... 5
kubectl get deployment web                           # more Pods running
kubectl delete pod load                              # stop load; scales back to 1 after ~5 min
```
`web-hpa-app.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m
            limits:
              cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```
**What each line does:**
- HPA (Horizontal Pod Autoscaler): adds or removes Pod replicas to keep average CPU (Central Processing Unit) usage near a target.
- Prerequisites: (1) **metrics-server** running (HPA reads CPU from it; without it TARGETS shows `<unknown>`), (2) **CPU `requests`** set on the container, because "50%" means 50% of the requested `200m` (0.2 CPU core), (3) room on the node for extra Pods.
- `registry.k8s.io/hpa-example`: Kubernetes' demo PHP page that does CPU-heavy work on every request, so load is easy to create.
- `kubectl autoscale ... --cpu-percent=50 --min=1 --max=5`: creates the HPA (newest kubectl warns this flag is deprecated and also accepts `--cpu=50%`).
- `kubectl run load ... while true; do wget ...`: a busybox Pod that calls the Service non-stop.
- `kubectl get hpa -w`: watch live; HPA re-checks about every 15 s; scale-down waits ~5 min to avoid flapping.
**Verify:** `kubectl get hpa web` → e.g. `cpu: 180%/50%  MINPODS 1  MAXPODS 5  REPLICAS 4`; `kubectl describe hpa web` shows `New size: N; reason: cpu resource utilization above target`.
(Same HPA as YAML: `apiVersion: autoscaling/v2`, `scaleTargetRef: {kind: Deployment, name: web}`, `minReplicas: 1`, `maxReplicas: 5`, metric `cpu` with `averageUtilization: 50`.)

**🧠 In simple words:** The HPA watches how busy the Pods are compared to the CPU they asked for. When the busybox loop overloads them, it adds Pods up to 5; when the load stops, it slowly removes them. In short: metrics-server + CPU requests + HPA = automatic scaling on load.
