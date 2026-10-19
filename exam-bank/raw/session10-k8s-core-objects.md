# Labs · Session 10 · Kubernetes core objects (Pods, Deployments, ReplicaSets, rollouts)

## Lab 1 · Create an Nginx Pod and inspect its status, labels, IP address and logs
**Answer (what to write in the exam):**
```bash
kubectl run nginx --image=nginx:1.27 --port=80 --labels="app=nginx,tier=web"
kubectl get pod nginx                          # STATUS Running, READY 1/1
kubectl get pod nginx -o wide --show-labels    # IP, NODE, LABELS
kubectl get pod nginx -o jsonpath='{.status.podIP}{"\n"}'   # only the IP
kubectl describe pod nginx                     # full details + Events
kubectl logs nginx                             # container output
kubectl label pod nginx env=dev                # add one more label
kubectl get pods -l app=nginx --show-labels    # select Pods by label
```
Same Pod as a manifest (`kubectl apply -f nginx-pod.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
    tier: web
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```
**What each line does:**
- `kubectl run nginx --image=nginx:1.27`: creates ONE bare Pod named `nginx`; `--port=80` documents the container port; `--labels` attaches key=value tags.
- `get pod nginx`: STATUS goes Pending → ContainerCreating → Running; READY `1/1` = 1 of 1 containers ready.
- `-o wide --show-labels`: adds IP (the Pod's internal IP, Internet Protocol address), NODE, and a LABELS column `app=nginx,tier=web`.
- `-o jsonpath='{.status.podIP}'`: prints just one field from the Pod object.
- `describe pod`: node, IP, labels, container state, image, and **Events** (Scheduled, Pulling, Pulled, Created, Started).
- `logs nginx`: nginx startup lines ("Configuration complete; ready for start up"); add `-f` to follow, `--previous` for a crashed container.
- `label` / `get -l app=nginx`: add a label, then use a *label selector* to find Pods that match it.

**Verify:** `kubectl get pod nginx -o wide --show-labels` → `nginx 1/1 Running 0 … 10.244.0.x minikube … app=nginx,tier=web`.

**🧠 In simple words:** A Pod is the smallest thing Kubernetes runs: one or more containers sharing one IP. Labels are tags that let you find and select Pods later (Services and Deployments use them). In short: run → get -o wide --show-labels (status, IP, labels) → describe (Events) → logs.

## Lab 2 · Create a Deployment with 2 replicas, scale to 5, scale back to 2, verify each step
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=2   # or: kubectl apply -f web-deploy.yaml
kubectl rollout status deployment/web
kubectl get deployment web             # READY 2/2
kubectl scale deployment web --replicas=5
kubectl rollout status deployment/web
kubectl get deployment web             # READY 5/5
kubectl get rs -l app=web              # DESIRED 5  CURRENT 5  READY 5
kubectl scale deployment web --replicas=2
kubectl get deployment web             # READY 2/2
kubectl get pods -l app=web            # exactly 2 Pods left (3 Terminating, then gone)
```
Manifest (`web-deploy.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
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
```
**What each line does:**
- `create deployment … --replicas=2`: Deployment → creates a ReplicaSet → which creates 2 Pods labelled `app=web`.
- `spec.replicas`: desired Pod count; `selector.matchLabels` must match `template.metadata.labels` (that is how the ReplicaSet finds its Pods).
- `scale deployment web --replicas=5`: changes `spec.replicas`; the ReplicaSet controller starts 3 more Pods. Scaling to 2 deletes 3.
- `get deployment`: READY `x/y`, UP-TO-DATE, AVAILABLE. `get rs`: DESIRED / CURRENT / READY of the ReplicaSet.
- Declarative way: edit `replicas: 5` in the file and `kubectl apply -f web-deploy.yaml` again.

**Verify:** `kubectl get deploy web -o jsonpath='{.spec.replicas} {.status.readyReplicas}{"\n"}'` → `2 2`, then `5 5`, then `2 2`.

**🧠 In simple words:** You never create or delete Pods by hand; you change one number (`replicas`) and the ReplicaSet controller adds or removes Pods until reality matches. If a Pod dies, it is replaced automatically. In short: create with 2 → scale to 5 → scale to 2, checking READY each time.

## Lab 3 · Deploy v1, rolling-update to v2, verify, roll back to v1
**Answer (what to write in the exam):**
```bash
kubectl create deployment web --image=nginx:1.26 --replicas=3          # version 1
kubectl annotate deployment web kubernetes.io/change-cause="v1 nginx:1.26"
kubectl rollout status deployment/web
kubectl set image deployment/web nginx=nginx:1.27                      # version 2
kubectl annotate deployment web kubernetes.io/change-cause="v2 nginx:1.27" --overwrite
kubectl rollout status deployment/web                                  # wait for v2
kubectl rollout history deployment/web                                 # REVISION 1 and 2
kubectl get rs -l app=web                                              # new RS 3 Pods, old RS 0
kubectl rollout undo deployment/web                                    # back to v1 (or --to-revision=1)
kubectl rollout status deployment/web
kubectl get deployment web -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```
**What each line does:**
- `create deployment … nginx:1.26`: version 1; the container is auto-named `nginx` (from the image name).
- `annotate … kubernetes.io/change-cause`: text shown in the CHANGE-CAUSE column of `rollout history` (replaces the old deprecated `--record` flag).
- `set image deployment/web nginx=nginx:1.27`: `<container-name>=<new-image>`; changing the Pod template starts a **rolling update**: a new ReplicaSet scales up while the old one scales down (default maxSurge 25%, maxUnavailable 25%), so the app stays up.
- `rollout status`: waits until "successfully rolled out"; `rollout history`: lists revisions.
- `get rs`: two ReplicaSets: the new one has 3 Pods, the old one 0 (kept for rollback).
- `rollout undo`: scales the previous ReplicaSet back up (it becomes revision 3); `--to-revision=1` picks an exact one.

**Verify:** after `set image` the jsonpath prints `nginx:1.27`; after `undo` it prints `nginx:1.26`; `kubectl rollout history deployment/web` shows revisions 2 and 3 (revision 1 was re-used as 3).

**🧠 In simple words:** A Deployment upgrades Pods a few at a time, so users never see downtime, and it keeps the old ReplicaSet so you can go back with one command. In short: set image → rollout status → rollout history → rollout undo.
