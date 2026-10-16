# Retest key · Session 05 (Kubernetes I), set 2 (for Claude's grading; don't open before answering)

W1 A · W2 C · W3 B
46.1 D · 46.2 D · 46.3 one-liner
47.1 C · 47.2 A · 47.3 C
48.1 C · 48.2 A · 48.3 one-liner
49.1 D · 49.2 C · 49.3 one-liner
Q1 B · Q2 D · Q3 A · Q4 B · L1 · L2

Notes:
- W1: `echo "Listening on '$PORT'" 'not "$PORT"'` prints `Listening on '8080' not "$PORT"` (piece 1 has outer double quotes, so it expands; piece 2 has outer single quotes, so it stays literal).
- W2: DNS uses the service name `cache`, not the image name `redis`.
- W3: `down` without -v keeps named volumes; volumes live outside containers.
- 46.1: rescheduling onto healthy nodes (liveness probes restart on the same node; rolling updates and DNS aren't about node failure).
- 46.2: the ReplicaSet controller restores 3 (desired state); a manual delete isn't a new desired state.
- 46.3 model: orchestration = automatically managing containers across many machines; e.g. scheduling, self-healing, scaling, rolling updates, service discovery.
- 47.1: the controller manager (ReplicaSet controller) notices fewer Pods than desired; the scheduler only picks nodes; the kubelet restarts containers in place.
- 47.2: kube-proxy writes the Service routing rules (iptables/IPVS); CoreDNS does names, not routing.
- 47.3: 2 of 3 is a majority (quorum), so etcd keeps working.
- 48.1: after storage, the scheduler acts next.
- 48.2: ContainerCreating + NODE set = stored + scheduled, containers still starting.
- 48.3 model: the node goes NotReady (no kubelet heartbeats); after a timeout (~5 min) its Pods are evicted and the ReplicaSet controller creates replacements, which the scheduler places on healthy nodes.
- 49.1: `-A` / `--all-namespaces`. `-n all` looks for a namespace literally named "all".
- 49.2: kind is case-sensitive: `Pod`.
- 49.3 model: `kubectl create deployment web --image=nginx --dry-run=client -o yaml` (prints the YAML, creates nothing; add `> web.yaml` to save it).
- Q1: the kubelet restarts crashed containers locally per restartPolicy, with no control plane needed. Replacing or moving Pods needs the control plane.
- Q2: the image IS one of the few mutable Pod fields → `pod/web configured`, and the kubelet restarts the container with 1.28 (RESTARTS +1).
- Q3: names are unique per namespace (per kind).
- Q4: Pod → v1 (core group), Deployment → apps/v1.
- L1 model: idempotent = running it again gives the same result; the second apply reports `unchanged`, because the desired state in the file already matches what's stored.
- L2 model: kubectl → kube-apiserver (validates) → etcd (stored) → kube-scheduler (picks node) → kubelet on that node → container runtime starts the container.
