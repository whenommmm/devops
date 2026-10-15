# Answer key · Session 11 · Helm

⚠️ Only open after answering.

---

## Warm-up

**W1.** A Deployment reads its settings from a ConfigMap as **environment variables** (`envFrom:`). You edit the ConfigMap with `kubectl edit configmap app-config`, and `kubectl get configmap app-config -o yaml` shows the new value. But the app still prints the **old** value, even 10 minutes later. Why?
- A) Env vars are read only when the container starts
- B) ConfigMap edits need `kubectl apply --force` first
- C) The kubelet syncs ConfigMaps only once every hour
- D) Pods see ConfigMap edits only after a node reboot

**Correct: A.** Environment variables are copied into the container **once, when it starts** (the same as `-e` in Docker). Editing the ConfigMap doesn't change a running process's environment, so you need new Pods: `kubectl rollout restart deployment/<name>`. B: the edit was already saved (you saw it with `kubectl get`), and no "force" is needed. C is a near-miss: ConfigMaps mounted **as a volume** are refreshed automatically after about a minute, but env vars never are. D: a node reboot isn't needed; replacing the Pods is enough.

---

**W2.** You run `kubectl create secret generic db-pass --from-literal=password=s3cret` and then apply a Deployment in the namespace **`shop`** whose Pod reads `db-pass`. The Pod is stuck in `CreateContainerConfigError`, and `kubectl describe pod` shows `Error: secret "db-pass" not found`. What's the cause?
- A) The password must be base64-encoded by hand first
- B) Secrets can be read only through a mounted volume
- C) The Secret was made in `default`, not in `shop`
- D) The kubelet caches Secrets until the node reboots

**Correct: C.** Secrets are **namespaced**, and `kubectl create secret` without `-n` puts it in the current namespace (usually `default`). A Pod in `shop` can only use Secrets in `shop`. Fix: `kubectl create secret generic db-pass --from-literal=password=s3cret -n shop`. A: `--from-literal` encodes for you (you hand-encode only when writing the `data:` field in YAML yourself), and a wrong encoding wouldn't produce "not found". B: Secrets can be used as env vars **or** volumes. D: nothing was ever cached. The Secret simply doesn't exist in that namespace.

---

**W3.** A release named `web` exists in the namespace `demo`. You type:
```bash
REL=web
helm status '$REL' -n demo
```
What happens?
- A) It shows the status of the release named web
- B) It fails: it asks for a release named `$REL`
- C) It fails, since -n must come before `status`
- D) It prints the status of every release in demo

**Correct: B.** **The outer quotes decide:** single quotes make `$REL` literal text, so Helm receives the four characters `$REL` as the release name, and there's no such release. Use `"$REL"` (or `$REL`) instead. A would be right with double quotes. C: flags like `-n` can go anywhere after the subcommand. D describes `helm list -n demo`, not `helm status`.

---

## Check 70

**70.1** Which pairing of Helm ideas with Docker ideas is correct?
- A) Chart ↔ container, release ↔ image
- B) Chart ↔ image, release ↔ container
- C) Chart ↔ registry, release ↔ image
- D) Chart ↔ Dockerfile, release ↔ registry

**Correct: B.** A chart is the **package** that does nothing by itself (like an image). A release is **an installed, running copy** of it (like a container), and one chart can give many releases, just as one image gives many containers. A swaps them. C: the registry matches a **chart repository**, not a chart. D: a Dockerfile is a recipe for building an image, and a release is nothing like a registry.

---

**70.2** A tutorial says: "Before installing charts, make sure Tiller is running in kube-system." You use Helm v4. What's accurate?
- A) Right; helm install fails until Tiller is ready
- B) Tiller is needed only for charts from repositories
- C) Tiller now runs on your Mac as a background daemon
- D) Tiller was Helm 2's; v3+ talk to the API directly

**Correct: D.** Tiller was the in-cluster server component of **Helm 2**. Helm 3 removed it (for security: it usually had cluster-admin rights), and the client talks straight to the API server using your kubeconfig. The tutorial is outdated. A is the Helm 2 situation. B: no Helm 3+ chart needs Tiller, wherever it comes from. C is tempting because the client runs on your Mac, but the client is **not** Tiller and isn't a daemon: it runs only while your command runs.

---

**70.3** (one line) Your team installs one chart three times in one cluster: `shop-dev`, `shop-staging`, `shop-prod`. How many charts and how many releases is that?

**Model answer:** *"One chart and three releases. A chart is the package, whereas each `helm install` with a new name creates a separate release with its own name, values, and revision history."*

---

## Check 71

**71.1** You add a new label to `templates/deployment.yaml` but still ship the same nginx 1.27 image. Which `Chart.yaml` field should you bump?
- A) `appVersion`, because the app's YAML changed
- B) `apiVersion`, from v2 up to the next number v3
- C) `version`, because the chart itself changed
- D) `appVersion` and `apiVersion`, as a matched pair

**Correct: C.** Templates are part of the **chart**, so the chart `version` goes up (e.g., 0.1.0 → 0.1.1). A: `appVersion` is the version of the **app** (nginx), which didn't change. B and D: `apiVersion: v2` is the chart **format** for Helm 3+, and you never bump it for your own changes.

---

**71.2** `values.yaml` has `image.repository: nginx` and `image.tag: ""`. `Chart.yaml` has `version: 0.3.0` and `appVersion: "1.27"`. The template says `image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"`. Which image runs?
- A) `nginx:1.27`
- B) `nginx:0.3.0`
- C) `nginx:latest`
- D) `nginx:v2`

**Correct: A.** `image.tag` is empty, so `default .Chart.AppVersion` supplies `1.27`. B uses the **chart** version, the classic version/appVersion mix-up. C is Docker's rule when you write no tag at all, but here the template always writes a tag. D confuses the chart `apiVersion` with a tag.

---

**71.3** (one line) What's special about files in `templates/` whose names start with `_`, like `_helpers.tpl`?

**Model answer:** *"Helm doesn't render files starting with `_` into Kubernetes objects. They hold named templates (reusable snippets created with `define`) that other templates insert with `include`, such as the chart's full name and standard labels."*

---

## Check 72

**72.1** `values.yaml` has `replicaCount: 1`. `values-prod.yaml` has `replicaCount: 4`. You run `helm install shop ./mychart --set replicaCount=2 -f values-prod.yaml`. How many replicas are rendered?
- A) 4, because -f comes last on the line
- B) 2, because --set beats any -f file
- C) 1, because the chart's defaults win
- D) An error about conflicting values

**Correct: B.** The precedence is: chart values.yaml < `-f` files < `--set`, **whatever order you type them in** (checked on your Helm v4). A is the tempting trap: order matters only **between** several `-f` files (the later one wins), not between `-f` and `--set`. C: the chart's defaults have the **lowest** priority. D: overriding is normal, so there's no conflict error.

---

**72.2** A template contains `replicas: {{ .Values.replicaCount }}`, but values.yaml says (by mistake) `replicacount: 3`. `helm lint` passes. What does `helm template` print for that line?
- A) `replicas: 3`, as keys ignore case
- B) An error: key replicaCount not found
- C) `replicas:` with nothing after it
- D) `replicas: 0`, the default for ints

**Correct: C.** Keys are **case-sensitive**, so `.Values.replicaCount` doesn't exist, and a missing value renders as **nothing**, silently. Kubernetes would then treat the empty `replicas` as unset and use 1. (This was checked on your Mac.) A: case matters. B: Helm only errors on missing values if the template uses `required`. D: Helm doesn't invent a zero. The field is just left empty.

---

**72.3** (one line) What's the difference between `helm template` and `helm install`?

**Model answer:** *"`helm template` renders the chart into plain Kubernetes YAML and prints it locally without contacting the cluster, whereas `helm install` renders it and sends it to the cluster, creating a release (revision 1) that Helm tracks."*

---

## Check 73

**73.1** `helm history shop` shows revisions 1–3, with 3 `deployed`. You run `helm rollback shop 1`. What does `helm history shop` show now?
- A) Revisions 1–3, with revision 1 marked as deployed
- B) Revision 1 only, since 2 and 3 are deleted
- C) Revisions 1–4, with 4 deployed: "Rollback to 1"
- D) Revisions 1–3, with 3 marked "rolled back"

**Correct: C.** A rollback **creates a new revision** (4) that copies revision 1's chart and values. Revisions 1–3 are kept, now `superseded`. A and D imagine the history being edited in place. B: nothing is deleted by a rollback.

---

**73.2** You installed with `helm install web ./mychart --set image.tag=1.28`. Later you run `helm upgrade web ./mychart --set replicaCount=3`. The chart's values.yaml has `tag: "1.27"`. Which image runs after the upgrade?
- A) nginx:1.28, because Helm keeps earlier --set values
- B) nginx:1.27, since only this upgrade's flags apply
- C) nginx:1.28, since upgrades change only what's named
- D) No change: upgrade without -f refuses to run at all

**Correct: B.** Without `--reuse-values`, every upgrade starts from the chart's values.yaml plus **only the flags given this time**, so the earlier `image.tag=1.28` is dropped and the default `1.27` returns. This was checked on your Helm v4. A and C describe what people **expect**, which is exactly why it's a trap. D: `-f` is optional, and the upgrade runs fine.

---

**73.3** (one line) A pipeline runs `helm upgrade --install shop ./chart`. What does `--install` add, and why is it handy in CI/CD (Continuous Integration / Continuous Delivery)?

**Model answer:** *"`--install` makes `helm upgrade` install the release if it doesn't exist yet and upgrade it if it does, so one command works for the very first deployment and for every later one, with no special case in the pipeline."*

---

## Session quiz

**Q1.** An engineer runs `helm install shop ./shop -n prod` and gets `Error: INSTALLATION FAILED: release name check failed: cannot reuse a name that is still in use`. Running `helm list` (no flags) shows an empty table. What's the most likely explanation?
- A) The chart folder ./shop has no Chart.yaml inside
- B) A release named shop already exists in prod
- C) kubectl already created a Deployment named shop
- D) Helm needs `helm repo update` before reusing a name

**Correct: B.** Release names are unique **per namespace**, and the error is about the **release name**. `helm list` with no flags shows only the **current** namespace (usually `default`), so the existing `shop` release in `prod` didn't appear. `helm list -n prod` would show it. A would give `Error: unable to detect chart at …/Chart.yaml: … no such file or directory` instead. C: a Deployment made by kubectl is not a Helm release, and a clash with it gives a different error: `Deployment "…" … exists and cannot be imported into the current release: invalid ownership metadata`. D: `helm repo update` only refreshes chart lists and has nothing to do with release names.

---

**Q2.** Given these files:
```yaml
# chart/values.yaml
replicaCount: 1
image:
  repository: nginx
  tag: "1.27"
```
```yaml
# values-prod.yaml
image:
  tag: "1.28"
```
and a template containing `replicas: {{ .Values.replicaCount }}` and `image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"`, what does `helm template web ./chart -f values-prod.yaml --set replicaCount=3` print for those two lines?
- A) `replicas: 1` and `image: "nginx:1.28"`
- B) `replicas: 3` and `image: "nginx:1.27"`
- C) `replicas: 3` and `image: ":1.28"`
- D) `replicas: 3` and `image: "nginx:1.28"`

**Correct: D.** `--set` gives `replicas: 3`, the `-f` file overrides only `image.tag` to `1.28`, and `image.repository` keeps its default `nginx`, because values are **merged key by key**. A ignores `--set`. B ignores the `-f` file. C assumes the `image:` block in values-prod.yaml **replaces** the whole default block, but Helm merges maps, so the repository survives.

---

**Q3.** `helm history api` shows:
```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         superseded  Upgrade complete
3         deployed    Upgrade complete
```
Revision 2 worked. Revision 3 (run without `--wait`) set a wrong image tag: one new Pod is in `ImagePullBackOff` while the old Pods still serve traffic. Which command restores the working version, and what will the history show afterwards?
- A) `helm rollback api 2`; a new revision 4 appears
- B) `helm rollback api 3`; revision 3 is re-applied
- C) `helm rollback api 2`; revision 3 is deleted
- D) `helm upgrade api --previous`; revision 2 returns

**Correct: A.** Roll back to the last **good** revision (2). Helm records it as a **new revision 4** ("Rollback to 2"), and revision 3 stays in the history as `superseded`. B re-applies the broken settings: revision 3 says `deployed` only because Helm didn't wait for the Pods to become Ready. C has the right command but the wrong result: rollbacks never delete revisions. D: there's no `--previous` flag. (A plain `helm rollback api` with no number would also go to the previous revision, 2.)

---

**Q4.** A teammate makes four claims about Helm v3 and later. Which one is **NOT** true?
- A) The helm client talks straight to the API server
- B) Release records live as Secrets in its namespace
- C) Tiller must run in kube-system to apply charts
- D) A chart's `version` and `appVersion` can differ

**Correct: C.** Tiller existed only in **Helm 2** and was removed in Helm 3. A is true: Helm 3+ is client-only and uses your kubeconfig. B is true: the Secrets are named `sh.helm.release.v1.<release>.v<N>`, in the release's namespace. D is true: e.g., `bitnami/nginx` is chart 25.2.1 with app 1.31.6.

---

**L1.** Explain **chart vs release** in contrast form, with an example.

**Model answer:** *"A chart is a package of templated Kubernetes YAML plus default values that describes an app (like an image), whereas a release is one named, installed copy of that chart in a cluster (like a container). For example, the chart `bitnami/nginx` installed twice gives two releases, `site-a` and `site-b`, each with its own values and revision history."*

---

**L2.** You set `--set image.tag=2.0` in one upgrade, and the next upgrade (`--set replicaCount=4` only) brings back the old image. Explain why, and give two ways to avoid it.

**Model answer:** *"Each `helm upgrade` starts from the chart's values.yaml plus only the values passed in that command, so the earlier `--set image.tag=2.0` isn't remembered and the default tag returns. To avoid it, keep all settings in a values file (e.g., `values-prod.yaml` in Git) and pass `-f values-prod.yaml` on every upgrade, or add `--reuse-values` so the previous release's values are kept and the new flags are merged on top."*
