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

## 🎯 Exam checklist (everything that could be asked)
*MCQ = know the fact · CMD = be able to write it from memory.*

| Asked as | Item | What you must know / the exact command |
|---|---|---|
| MCQ | **Toolkit** · the method | Work in a fixed order: get → describe → logs → exec → events. A symptom is what you see (`CrashLoopBackOff`), whereas the root cause is why it happens; you fix the cause |
| CMD | `kubectl get` | `kubectl get pods -o wide` shows READY, STATUS and RESTARTS plus each Pod's IP (Internet Protocol) address and NODE; `--show-labels` adds a LABELS column |
| MCQ | STATUS words `Running` · `Completed` · `Error` | `Running` = the main process runs (not necessarily ready) · `Completed` = it exited with code 0 · `Error` = it exited with a non-zero code |
| CMD | `kubectl describe` | `kubectl describe pod web` shows each container's Last State (how it ended last time: Reason + Exit Code) and, at the bottom, the Events; read the `Warning` lines first |
| CMD | Events of the whole namespace | `kubectl get events --sort-by=.lastTimestamp` lists every event sorted by time, newest at the bottom. Events are kept only about 1 hour |
| CMD | Logs of the dead container | `kubectl logs web-7d4b9 --previous` shows what the previous, dead copy printed; run it first for a crash loop, as the new copy may have printed nothing yet |
| CMD | Logs of one container, live | `kubectl logs web-7d4b9 -c log-shipper -f`: `-c` picks one container in a multi-container Pod (otherwise you get the default one), `-f` follows new lines live |
| MCQ | `logs` vs `describe` · `--previous` vs `-c` | `logs` shows what the app printed, whereas `describe` shows how the container ended plus the Events. `--previous` = an earlier run of the same container, whereas `-c` = a different container |
| CMD | Shell inside a Pod | `kubectl exec -it api-5f6c8 -- sh` opens an interactive shell (`-i` = interactive, `-t` = a terminal; what follows `--` runs inside the container) |
| MCQ | **CrashLoopBackOff** · meaning | The main process keeps exiting and Kubernetes keeps restarting it; the kubelet (the agent on each node) waits longer before each restart, up to 5 minutes. It is a symptom, not the cause |
| MCQ | `restartPolicy` trap | `Always` (the default, and the only value a Deployment allows) restarts after any exit, even code 0, so a Deployment whose command simply finishes still ends in CrashLoopBackOff |
| MCQ | Exit codes 1 · 126 · 127 | 1 = a general app error (`DB_HOST is not set`) · 126 = the file was found but isn't executable · 127 = command not found (`sh: pyhton: not found`) |
| MCQ | The 128 + N rule | A process ended by signal N exits with 128 + N: 137 = SIGKILL 9 (killed by force) · 143 = SIGTERM 15 (a polite stop, normal on delete or rollout) · 139 = SIGSEGV 11 (bad memory access) |
| MCQ | Exit Code 137 with Reason `OOMKilled` | Seen in `describe`'s Last State: the container went over its memory limit and the OOM (Out Of Memory) killer sent SIGKILL. Raise the limit or fix the memory leak |
| MCQ | `CreateContainerConfigError` vs CrashLoopBackOff | A ConfigMap, Secret or key it needs is missing (named in the `describe` Events), so the container is never created: 0 restarts, no logs. CrashLoopBackOff instead has rising RESTARTS and logs |
| CMD | Fix a missing ConfigMap | `kubectl create configmap crash-settings --from-literal=db_host=postgres`; the kubelet retries by itself, so the stuck Pod starts without being deleted |
| MCQ | **Image pull errors** · `ErrImagePull` vs `ImagePullBackOff` | `ErrImagePull` = the image download just failed, whereas `ImagePullBackOff` = the kubelet is waiting before retrying: one problem. The Pod already has a NODE; no logs, so read the `Failed` event |
| MCQ | Pull error messages | `nginx:1.277 … not found` = that tag doesn't exist, whereas `pull access denied, repository does not exist or may require authorization` most often = a misspelt name (`ngnix`): Docker Hub says it even for missing repositories |
| CMD | Private-registry login | Under the Pod's `spec:` add `imagePullSecrets:` → `- name: regcred` (a Secret of type `docker-registry`); needed only for a real private repository |
| MCQ | ImagePullBackOff vs CrashLoopBackOff | ImagePullBackOff means nothing ever ran, whereas CrashLoopBackOff means the image did run and its process keeps exiting |
| MCQ | **Pending** · meaning | The scheduler (which picks a node for each Pod) hasn't found a node: `-o wide` shows NODE `<none>`, and the event `FailedScheduling … 0/1 nodes are available: <reason>` gives the reason after the colon |
| MCQ | `Insufficient cpu` / `Insufficient memory` | The Pod's requests (CPU or memory it reserves up front) fit no node's unreserved room; lower them. The scheduler counts requests, not real use or limits, so a node at 10% use can be full |
| MCQ | `didn't match Pod's node affinity/selector` · `persistentvolumeclaim "x" not found` | The Pod's `nodeSelector` asks for a node label that no node has · the PVC (PersistentVolumeClaim, the Pod's storage request) doesn't exist |
| CMD | Label a node | `kubectl label node minikube disktype=ssd` adds the label that a Pod's `nodeSelector: disktype: ssd` needs; `kubectl label node minikube disktype-` removes it |
| MCQ | Taint vs toleration | A taint is a "keep out" mark on a node (event: `untolerated taint`), whereas a toleration in the Pod's `spec.tolerations` lets that Pod past the mark but doesn't force it there |
| MCQ | Pending vs ContainerCreating | Pending means no node has been chosen yet, whereas ContainerCreating means a node was chosen and is still pulling the image or mounting volumes |
| CMD | **Service & DNS** · test from another Pod | `kubectl exec client -- wget -qO- -T 3 http://shop` (`-qO-` = print the page quietly, `-T 3` = give up after 3 s); the error words show which layer failed |
| CMD | Check a Service's endpoints | `kubectl describe svc shop` shows its Selector, TargetPort and Endpoints; compare the selector with `kubectl get pods --show-labels` |
| MCQ | Empty endpoints | Endpoints = the Pod IP:port pairs a Service forwards to. Empty when the selector matches no labels (case-sensitive: `Web` ≠ `web`), the Pods aren't Ready, or they're in another namespace |
| MCQ | `targetPort` vs `containerPort` | The app must listen on the `targetPort` shown in the endpoints, or you get `Connection refused`, whereas `containerPort` in the Pod spec is only documentation and opens nothing |
| MCQ | The 127.0.0.1 trap | An app bound to 127.0.0.1 (loopback) accepts only its own Pod: endpoints look fine and a test inside works, yet other Pods get `Connection refused`. It must listen on `0.0.0.0` |
| CMD | See which address the app listens on | `kubectl exec shop-7c9d -- netstat -tln` lists listening TCP ports; `127.0.0.1:8080 LISTEN` reveals the loopback-only bind |
| MCQ | CoreDNS and short names | CoreDNS, the cluster's DNS (Domain Name System) server, resolves a short name like `api` only in the same namespace; from elsewhere use `api.shop` or the FQDN (Fully Qualified Domain Name) `api.shop.svc.cluster.local` |
| CMD | Test a DNS name | `kubectl exec client -- nslookup api.shop` shows whether the name resolves; `nslookup kubernetes.default` checks that DNS works at all |
| MCQ | `bad address` · `Connection refused` · timeout | `bad address` = DNS found no such name (wrong name or namespace, CoreDNS down) · `Connection refused` = the IP was reached but nothing listens · timeout = packets dropped, e.g. by a NetworkPolicy (a firewall rule for Pods; `kubectl get networkpolicy`) |

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
