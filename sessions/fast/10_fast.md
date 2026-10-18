# Session 10 · Kubernetes Troubleshooting · FAST (10-min read)
*How to find out why a Pod or Service is broken: read the STATUS word, then run the command it points to.*

## 1 · The toolkit: get → describe → logs → exec → events
A fixed order of commands beats guessing. A **symptom** is what you see (`CrashLoopBackOff`), and the **root cause** is why it happens (a missing setting); you fix the cause.
- `kubectl get pods` shows READY, STATUS and RESTARTS, `-o wide` adds each Pod's IP (Internet Protocol) address and NODE, and `--show-labels` adds its labels.
- `kubectl describe pod <p>` shows each container's **Last State** (how it ended last time, with a Reason and an Exit Code) and, at the bottom, the **Events**: a time-ordered list of what went wrong; read the `Warning` lines first.
- `kubectl logs <p>` shows what the app printed, `-f` follows new lines live, and `-c <container>` picks one container in a multi-container Pod (otherwise you get the default one).
- `kubectl logs <p> --previous` shows what the previous, dead copy of the container printed.
- `kubectl exec -it <p> -- sh` opens a shell inside the container, and `kubectl get events --sort-by=.lastTimestamp` lists the namespace's events, newest at the bottom.

⚠️ **Exam trap:** `kubectl logs` shows what the app printed, whereas `kubectl describe` shows how the container ended plus the Events. `--previous` shows an earlier run of the same container, whereas `-c` picks a different container.

## 2 · CrashLoopBackOff, exit codes and CreateContainerConfigError
**CrashLoopBackOff** means the container's main process keeps exiting and Kubernetes keeps restarting it, while the kubelet (the agent on each node that runs containers) waits longer before each restart, up to 5 minutes. Like redialling a friend whose phone is off, the waiting tells you nothing about why: it is a symptom, and the cause is in the app or its settings.
- Run `kubectl logs <p> --previous` first, because the new container may not have printed anything yet.
- **Exit code 1** is a general app error, such as `DB_HOST is not set`.
- **Exit code 127** means "command not found", for example `sh: pyhton: not found`.
- **Exit code 137** is 128 + 9, where 9 is SIGKILL (the "stop right now" signal); with Reason **OOMKilled**, the container went over its memory limit and the Linux OOM (Out Of Memory) killer stopped it, so you raise the limit or fix the leak.
- **CreateContainerConfigError** means a ConfigMap or Secret (objects that hold settings), or a key inside one, that the container needs is missing, so the container is never created: RESTARTS stays 0 and there are no logs.
- Once you create the missing object, the kubelet retries by itself.

⚠️ **Exam trap:** CrashLoopBackOff has rising RESTARTS and logs from the dead container, whereas CreateContainerConfigError has 0 restarts and no logs at all.

## 3 · ErrImagePull and ImagePullBackOff
Before a container starts, the kubelet must pull (download) its image from a registry such as Docker Hub. **ErrImagePull** means the pull just failed, and **ImagePullBackOff** means the kubelet is waiting before retrying; it is one problem.
- The Pod was already placed on a node (`-o wide` shows a NODE), and only the download failed.
- The container never started, so there are no logs; read the `Failed` event in `describe`.
- `not found` after a tag such as `nginx:1.277` means that tag doesn't exist.
- `pull access denied, repository does not exist or may require authorization` most often means a misspelt name (`ngnix`), because Docker Hub says this even for repositories that don't exist.
- Only a real private repository needs a login Secret listed under the Pod's `imagePullSecrets`.

⚠️ **Exam trap:** ImagePullBackOff means nothing ever ran, whereas CrashLoopBackOff means the image did run and its process keeps exiting.

## 4 · Pending Pods
A **Pending** Pod was accepted, but the **scheduler** (the part that picks a node for each Pod) hasn't found a node for it, so `-o wide` shows NODE `<none>`. The `describe` event `FailedScheduling … 0/1 nodes are available: <reason>` gives the reason after the colon.
- `Insufficient cpu` or `Insufficient memory` means the Pod's **requests** (the CPU, Central Processing Unit, or memory it reserves up front) fit no node's unreserved room, so you lower them.
- The scheduler counts requests, not real use and not limits, so a node at 10% CPU use is still full if other Pods reserved it all.
- `didn't match Pod's node affinity/selector` means the Pod's `nodeSelector` asks for a node label (such as `disktype: ssd`) that no node has.
- `persistentvolumeclaim "x" not found` means the Pod's storage request (a PVC, PersistentVolumeClaim) doesn't exist.
- `untolerated taint` means the node carries a "keep out" mark (a taint) that the Pod has no permission (a toleration) to pass.

⚠️ **Exam trap:** Pending means no node has been chosen yet, whereas ContainerCreating means a node was chosen and is still pulling the image.

## 5 · Service & DNS troubleshooting
Pods can be `1/1 Running` while their Service fails, because the path is broken: name → DNS (Domain Name System, which turns names into IP addresses) → endpoints → Pod port → app. Test from another Pod with `kubectl exec <client> -- wget -qO- -T 3 http://<service>`.
- **Endpoints** (`kubectl describe svc <s>`) are the Pod IP:port pairs the Service forwards to; they are empty when the selector matches no Pod labels (case-sensitive: `Web` is not `web`), when the Pods aren't Ready (their "can you take traffic?" check fails), or when the Pods are in another namespace.
- If the app doesn't listen on the `targetPort` shown in the endpoints, nothing answers; `containerPort` in the Pod spec is only documentation.
- **The 127.0.0.1 trap:** an app bound to 127.0.0.1 (loopback) accepts connections only from inside its own Pod, so endpoints look fine and `netstat -tln` inside shows `127.0.0.1:<port>`, yet other Pods get `Connection refused`; it must listen on `0.0.0.0` (all interfaces).
- CoreDNS, the cluster's DNS server, resolves a short name like `api` only within the same namespace, so from elsewhere use `api.shop` (or `api.shop.svc.cluster.local`); test names with `nslookup`.
- `bad address` means DNS found no such name, `Connection refused` means the IP was reached but nothing listens, and a timeout means packets were dropped (by a NetworkPolicy, a firewall rule for Pods).

⚠️ **Exam trap:** Empty endpoints point to a selector, readiness or namespace problem, whereas endpoints plus `Connection refused` point to the port or a 127.0.0.1 bind.

## ⚡ Must-remember

| STATUS / symptom | Likely cause | Command to check |
|---|---|---|
| `CrashLoopBackOff` | App keeps exiting (1 = app error, 127 = command not found) | `kubectl logs <p> --previous` |
| `OOMKilled`, Exit Code 137 | Over the memory limit, killed by SIGKILL (128 + 9) | `kubectl describe pod <p>` (Last State) |
| `CreateContainerConfigError`, 0 restarts | Missing ConfigMap, Secret or key | `kubectl describe pod <p>` (Events) |
| `ErrImagePull` / `ImagePullBackOff` | Wrong tag, misspelt name, private repo | `kubectl describe pod <p>` (Failed event) |
| `Pending`, NODE `<none>` | Requests too big, nodeSelector, missing PVC, taint | `kubectl describe pod <p>` (FailedScheduling) |
| Service `Endpoints: <none>` | Selector ≠ labels, not Ready, other namespace | `describe svc` + `get pods --show-labels` |
| Endpoints fine, `Connection refused` | Wrong targetPort, or app on 127.0.0.1 | `kubectl exec <p> -- netstat -tln` |
| `bad address` | Wrong name or namespace, CoreDNS down | `kubectl exec <client> -- nslookup <svc>` |
| Timeout | Dropped by a NetworkPolicy or firewall | `kubectl get networkpolicy` |

## 📝 Quiz (5 MCQs; answer in chat as "1 B, 2 D, …")

**1.** Pod `web` has two containers, `app` and `sidecar`. `kubectl get pods` shows `web  1/2  CrashLoopBackOff  7`, and the container that keeps crashing is `sidecar`. Which command shows what `sidecar` printed just before its most recent crash?
- A) `kubectl logs web -c app --previous`
- B) `kubectl logs web -c sidecar -f --tail=50`
- C) `kubectl logs web -c sidecar --previous`
- D) `kubectl logs sidecar -c web --previous`

**2.** An engineer runs `kubectl get pods`:
```
NAME    READY   STATUS                       RESTARTS
pay-a   0/1     CreateContainerConfigError   0
pay-b   0/1     CrashLoopBackOff             6
pay-c   0/1     OOMKilled                    2
```
Which statement is NOT true?
- A) pay-a must be deleted by hand once its ConfigMap exists
- B) pay-a has no logs, since its container was never created
- C) `logs pay-b --previous` shows what its dead copy printed
- D) pay-c's Last State shows Exit Code 137 (128 + SIGKILL 9)

**3.** A new Pod flips between `ErrImagePull` and `ImagePullBackOff`. `kubectl get pods -o wide` shows NODE `minikube` and an IP. The newest event is `Failed to pull image "ngnix:1.27": pull access denied, repository does not exist or may require authorization`. The team only uses public images. What is the root cause?
- A) No node had room, so the Pod was never scheduled
- B) The image name is misspelt, so no such repo exists
- C) The registry wants a login, so add imagePullSecrets
- D) The app crashed at start; `logs --previous` shows it

**4.** Pod `report` stays `Pending` with NODE `<none>`. Its event says `0/3 nodes are available: 3 Insufficient cpu.` Yet `kubectl top nodes` shows every node using only about 10% of its CPU. What explains this?
- A) Other Pods' requests have already reserved that CPU
- B) Its image is still downloading on all three nodes
- C) Its CPU limit is set too low for the app to start up
- D) The nodes carry a taint that the Pod doesn't tolerate

**5.** An engineer runs three checks:
```
$ kubectl describe svc shop | grep -E "Selector|Endpoints"
Selector:   app=shop
Endpoints:  10.244.0.9:8080
$ kubectl exec client -- wget -qO- -T 3 http://shop
wget: can't connect to remote host (10.96.41.7): Connection refused
$ kubectl exec shop-7c9d -- netstat -tln
tcp   0   0 127.0.0.1:8080   0.0.0.0:*   LISTEN
```
What fixes it?
- A) Fix the selector so the Service picks up the Pod
- B) Fix the Service name, since DNS can't resolve shop
- C) Add containerPort 8080 so the Pod opens that port
- D) Make the app listen on 0.0.0.0, not on 127.0.0.1
