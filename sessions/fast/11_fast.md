# Session 11 · Helm · FAST (10-min read)
*Helm is the package manager for Kubernetes: one command installs, upgrades or rolls back a whole app as one unit.*

## 1 · What is Helm? Chart, release, repository
One app needs several YAML (YAML Ain't Markup Language) files, and three environments (dev, staging, prod) mean three near-identical copies that drift apart; `kubectl rollout undo` rolls back only one Deployment, not the whole app. **Helm**, the **package manager for Kubernetes** (like `brew` on a Mac), manages the whole app as one unit. Analogy: a chart is a recipe, a release is one dish cooked from it, and a repository is a shop selling recipe books.
- A **chart** is the package: a folder of Kubernetes YAML files with blanks, plus a file of default settings that fill them. Like a Docker image, it does nothing on its own.
- A **release** is one installed copy of a chart, with a name you choose. Installing one chart as `shop-dev` and `shop-prod` gives one chart and two releases (like one image, two containers).
- A **revision** is a numbered version of a release: the install is revision 1, and every upgrade or rollback adds 1.
- A **chart repository** ("repo") is a server storing charts for download, like Docker Hub. **Artifact Hub** (artifacthub.io) is only a search website that tells you which repo has a chart.
- Helm is **client-only**: the `helm` program on your Mac **renders** the chart (fills the blanks to produce plain Kubernetes YAML) and sends it to the API (Application Programming Interface) server using your **kubeconfig** (`~/.kube/config`), so it has exactly your kubectl permissions.
- Each revision's record is saved as a **Secret** in the release's **namespace** (a named folder inside the cluster), e.g. `sh.helm.release.v1.web.v1`.

⚠️ **Exam trap:** Helm 2 had **Tiller**, an in-cluster server (in `kube-system`) with admin rights that did the installing, whereas Helm 3 and later have no Tiller, so `helm version` shows no "Server" part.

## 2 · Chart structure and Chart.yaml
`helm create mychart` builds a **scaffold**: a working starter chart (it deploys nginx) that you adapt instead of starting from nothing.
- `Chart.yaml` is the chart's ID card (name and versions), and `values.yaml` holds the default settings, the "knobs" users can change.
- `templates/` holds the YAML files with blanks. Files starting with `_` (like `_helpers.tpl`, which holds reusable snippets) never become Kubernetes objects, and `NOTES.txt` is printed after install.
- `charts/` holds dependencies (other charts yours needs), and `.helmignore` lists files to leave out when packaging, like `.gitignore`.

```yaml
apiVersion: v2          # chart format: v2 = Helm 3 and later
name: mychart
type: application       # installable ("library" = snippets only)
version: 0.1.0          # the CHART's own version
appVersion: "1.16.0"    # the version of the APP inside (nginx)
```
- `version` follows **SemVer** (Semantic Versioning, `MAJOR.MINOR.PATCH`); bump it whenever the chart's templates or default values change.
- `appVersion` is only a label; `helm list` shows both versions side by side.

⚠️ **Exam trap:** `version` describes the packaging (the chart), whereas `appVersion` describes what is packaged (the app), so a template-only change bumps `version` only. `apiVersion: v2` in Chart.yaml is the chart format, whereas `apiVersion: apps/v1` in a manifest is a Kubernetes API.

## 3 · values.yaml and templates: filling in the blanks
Dev wants 1 replica, prod wants 5, and two copies of `deployment.yaml` would drift. A **template** is one YAML file with blanks filled from the values, like a mail-merge letter ("Dear ____").
```yaml
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```
- Everything inside `{{ }}` is an **action** (an instruction to Helm, not YAML) that rendering replaces with a value.
- `.Values` is values.yaml merged with your overrides, `.Release.Name` is the release name, and `.Chart.AppVersion` is `appVersion` from Chart.yaml.
- The pipe `|` passes a value into a function: `default .Chart.AppVersion` uses the app version when `image.tag` is empty, so `tag: ""` gives `nginx:1.16.0`.
- Keys are case-sensitive and typos are silent: `.Values.replicacount` (small c) renders an empty `replicas:` with no error.
- `helm lint mychart` checks the chart's structure and template syntax (a missing `}` fails it) but cannot catch a misspelt values key.

**Overriding values** (lowest to highest priority):
```
values.yaml  <  -f first.yaml  <  -f later.yaml  <  --set key=value
```
- `-f` (`--values`) adds a file holding only the keys you change; `--set image.tag=1.28` sets one key (dots mean nesting).
- `--set` wins even when typed before `-f`; order matters only between two `-f` files (the later one wins).
- Merging is key by key: a file that sets only `image.tag` leaves `image.repository` at its default.

⚠️ **Exam trap:** `helm template web ./mychart` only renders the YAML and prints it on your Mac (**nothing** reaches the cluster), whereas `helm install` sends it to the cluster and creates a release.

## 4 · Install, upgrade and rollback: deploying an app
```
helm install web ./mychart -n demo --create-namespace   → revision 1, deployed
helm upgrade web ./mychart --set image.tag=1.28         → revision 2 (1 becomes superseded)
helm upgrade web ./mychart --set image.tag=9.99-nope    → revision 3
helm rollback web 2                                     → revision 4, a COPY of revision 2
helm uninstall web                                      → objects AND history deleted
```
- `-n demo` picks the namespace and `--create-namespace` creates it; a release name must be unique within its namespace.
- `deployed` means the live revision, `superseded` means an older one that was replaced, and `failed` means Helm knows it went wrong.
- `helm upgrade --install` upgrades the release if it exists and installs it if not (one command for every deploy).
- `helm history web` lists every revision, and `helm list` shows releases in the **current namespace only** (`-A` for all).
- `helm repo add bitnami https://charts.bitnami.com/bitnami` remembers a public repo and `helm repo update` refreshes its list; then `helm install my-nginx bitnami/nginx`.

**Three traps:**
- **A rollback is a new revision.** `helm rollback web 2` creates revision 4, a copy of revision 2's chart and values; revision 3 stays in the history.
- **`--set` values are not remembered.** Each upgrade starts again from the chart's values.yaml plus only this command's flags, so a later `--set replicaCount=4` silently puts the image tag back to the default. Fix: pass `-f values-prod.yaml` on every upgrade, or add `--reuse-values` (start from the last release's values).
- **"deployed" does not mean "working".** Without `--wait`, Helm sends the YAML and returns at once, so a bad image tag still shows `deployed` while a new Pod sits in `ImagePullBackOff` (its image cannot be downloaded), and the rolling update (replacing Pods a few at a time) keeps the old Pods serving. `--wait --timeout 2m` waits for Ready Pods; if they never come, the revision is `failed` and Helm exits non-zero.

⚠️ **Exam trap:** `--wait` only marks a bad upgrade `failed`, whereas `--rollback-on-failure` (called `--atomic` in Helm 3) also rolls it back automatically.

## ⚡ Must-remember
| Term / command | Meaning |
|---|---|
| Chart / release / repo | Package / one installed named copy / server storing charts |
| `version` vs `appVersion` | Chart's own SemVer version / version of the app inside |
| `helm template` | Render and print YAML; nothing reaches the cluster |
| `helm lint` | Check structure and template syntax; misses misspelt keys |
| Precedence | values.yaml < `-f` (later file wins) < `--set` |
| `helm rollback web 2` | New revision that copies revision 2 |
| `--reuse-values` | Upgrade starts from the last release's values |
| `--wait --timeout 2m` | Wait for Ready Pods; else `failed` and non-zero exit |
| `helm list` | Current namespace only; `-A` for all namespaces |
| Tiller | Helm 2's in-cluster server; gone since Helm 3 |

## 📝 Quiz (5 MCQs; answer in chat as "1 B, 2 D, …")

**1.** Given these files:
```yaml
# chart/values.yaml
replicaCount: 1
image:
  repository: nginx
  tag: "1.27"
```
```yaml
# a.yaml
replicaCount: 2
```
```yaml
# b.yaml
replicaCount: 4
image:
  tag: "1.28"
```
The template has `replicas: {{ .Values.replicaCount }}` and `image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"`. What does this command print for those two lines?
```bash
helm template web ./chart --set replicaCount=3 -f b.yaml -f a.yaml
```
- A) `replicas: 2` and `image: "nginx:1.28"`
- B) `replicas: 3` and `image: "nginx:1.27"`
- C) `replicas: 3` and `image: "nginx:1.28"`
- D) `replicas: 4` and `image: "nginx:1.28"`

**2.** The chart's values.yaml sets `image.repository` to `nginx` and `image.tag` to `"1.27"`. These commands run in order (none use `--reuse-values`):
```bash
helm install api ./chart --set image.tag=1.28
helm upgrade api ./chart --set image.tag=9.99-nope
helm rollback api 1
helm upgrade api ./chart --set replicaCount=3
```
Afterwards, what is the live revision number in `helm history api`, and which image runs?
- A) Revision 4, running image `nginx:1.28`
- B) Revision 4, running image `nginx:1.27`
- C) Revision 3, running image `nginx:1.27`
- D) Revision 2, running image `nginx:1.28`

**3.** A teammate makes four claims about Helm 3 and later. Which one is **NOT** true?
- A) One chart installed as `a` and as `b` gives two releases
- B) Each revision is kept as a Secret in the release namespace
- C) Artifact Hub lists charts, but the charts live in repos
- D) Helm can deploy things that your own kubectl user cannot

**4.** A chart has this `Chart.yaml`, and its values.yaml sets `image.repository` to `shop-api` and `image.tag` to `""` (empty):
```yaml
apiVersion: v2
name: shop
version: 0.4.0
appVersion: "2.1"
```
The template line is `image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"`. You add one new label to `templates/service.yaml` and ship the same app. Which Chart.yaml change is right, and which image runs after the upgrade?
- A) Bump version to 0.4.1; image `shop-api:2.1`
- B) Bump appVersion to 2.2; image `shop-api:2.2`
- C) Bump version to 0.4.1; image `shop-api:0.4.1`
- D) Bump apiVersion to v3; image `shop-api:2.1`

**5.** An engineer runs `helm upgrade web ./chart -n prod --set image.tag=9.99-nope` (that tag does not exist, and no other flags are used). Helm prints `STATUS: deployed` and `REVISION: 3`. What is the actual state about a minute later?
- A) Helm rolled back on its own to revision 2 when the pull failed
- B) Revision 3 shows `failed` in history, as its Pods never got Ready
- C) All Pods were replaced at once, so the app is down for every user
- D) A new Pod is in ImagePullBackOff while the old Pods keep serving
