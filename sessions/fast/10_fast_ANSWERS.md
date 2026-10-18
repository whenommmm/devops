# Session 10 · Kubernetes Troubleshooting · FAST · Answer key

**1.** Pod `web` has two containers, `app` and `sidecar`. `kubectl get pods` shows `web  1/2  CrashLoopBackOff  7`, and the container that keeps crashing is `sidecar`. Which command shows what `sidecar` printed just before its most recent crash?
- A) `kubectl logs web -c app --previous`
- B) `kubectl logs web -c sidecar -f --tail=50`
- C) `kubectl logs web -c sidecar --previous`
- D) `kubectl logs sidecar -c web --previous`

**Correct: C.** You need two things at once: `-c sidecar` to pick the crashing container inside the Pod `web`, and `--previous` to read the dead copy of it rather than the one that just restarted. A has `--previous` but picks the wrong container: `app` is the healthy one and has no crashed copy to show. B picks the right container, but `-f` follows the new, freshly restarted copy live, which may not have printed anything yet. D swaps the names: `kubectl logs` takes the Pod name first and the container after `-c`, and there is no Pod called `sidecar`.

---

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

**Correct: A.** This is the false statement: with CreateContainerConfigError, the kubelet keeps retrying by itself, so as soon as the missing ConfigMap (or Secret, or key) exists, the Pod starts without being deleted. B is true: the container was never created (that is also why RESTARTS is 0), so there is nothing to show in the logs. C is true: `--previous` is exactly the flag for reading a crash-looping container's last dead copy. D is true: OOMKilled means the Out Of Memory killer sent SIGKILL (signal 9), and 128 + 9 = 137.

---

**3.** A new Pod flips between `ErrImagePull` and `ImagePullBackOff`. `kubectl get pods -o wide` shows NODE `minikube` and an IP. The newest event is `Failed to pull image "ngnix:1.27": pull access denied, repository does not exist or may require authorization`. The team only uses public images. What is the root cause?
- A) No node had room, so the Pod was never scheduled
- B) The image name is misspelt, so no such repo exists
- C) The registry wants a login, so add imagePullSecrets
- D) The app crashed at start; `logs --previous` shows it

**Correct: B.** `ngnix` is a typo for `nginx`, and Docker Hub answers "pull access denied … may require authorization" even for repositories that simply don't exist, so the message looks like a login problem when it is really a spelling problem. A is wrong because `-o wide` shows a NODE, so the Pod was placed, and only the download failed (a Pod with no room would be Pending with NODE `<none>`). C is the tempting reading of the message, but a login Secret only helps for a real private repository, and the team uses public images. D is wrong because the image was never downloaded, so the app never ran and there are no logs, previous or current.

---

**4.** Pod `report` stays `Pending` with NODE `<none>`. Its event says `0/3 nodes are available: 3 Insufficient cpu.` Yet `kubectl top nodes` shows every node using only about 10% of its CPU. What explains this?
- A) Other Pods' requests have already reserved that CPU
- B) Its image is still downloading on all three nodes
- C) Its CPU limit is set too low for the app to start up
- D) The nodes carry a taint that the Pod doesn't tolerate

**Correct: A.** The scheduler places Pods by comparing their requests (CPU reserved up front) with each node's unreserved room; it ignores real use, so a node at 10% use is still "full" when other Pods have already requested its CPU. B is wrong because Pending means no node was chosen yet, and an image is pulled only after a node is chosen (that stage is ContainerCreating). C is wrong because the scheduler counts requests, not limits, and a limit can't stop a Pod from being placed. D would produce a different event, `untolerated taint`, not `Insufficient cpu`.

---

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

**Correct: D.** This is the 127.0.0.1 trap: `netstat` shows the app listening only on the loopback address `127.0.0.1:8080`, which accepts connections from inside its own Pod only, so requests arriving from other Pods are refused even though the Service looks perfect. Listening on `0.0.0.0` (all interfaces) fixes it. A is wrong because the endpoints list is not empty (`10.244.0.9:8080`), so the selector already matches the Pod. B is wrong because `wget` printed an IP address and said `Connection refused`, which means the name resolved and the IP was reached; a DNS failure would say `bad address`. C is wrong because `containerPort` is only documentation and opens nothing; the app already uses port 8080, just on the wrong address.

---
