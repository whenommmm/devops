# Session 11 · Helm: packaging Kubernetes apps (Oct 16, Session 1)

**Time:** ~90–100 min · **Concepts:** 70 → 73 · **Answers:** `MY_ANSWERS.md` or say "Helm questions" · **Answer key:** `11_helm_ANSWERS.md`

**Before you start:**
```bash
minikube status     # host/kubelet/apiserver: Running (if Stopped: minikube start, wait a minute or two)
kubectl get nodes   # minikube   Ready   control-plane …
helm version        # version.BuildInfo{Version:"v4.3.0", …} (Concept 70 explains this line)
```
> ⚠️ **Reminders.** Never delete the **`minikube`** or **`localstack`** containers. Leave namespaces you didn't create (`argocd`, `ingress-nginx`, `final-app`) alone: `helm list -A` may show a **`final-ticketdesk`** release in `final-app`. Only remove what you create today.

## 🔁 Warm-up (Kubernetes + the weak list)
**W1.** A Deployment reads its settings from a ConfigMap as **environment variables** (`envFrom:`). You edit the ConfigMap with `kubectl edit configmap app-config`, and `kubectl get configmap app-config -o yaml` shows the new value. But the app still prints the **old** value, even 10 minutes later. Why?
- A) Env vars are read only when the container starts
- B) ConfigMap edits need `kubectl apply --force` first
- C) The kubelet syncs ConfigMaps only once every hour
- D) Pods see ConfigMap edits only after a node reboot

**W2.** You run `kubectl create secret generic db-pass --from-literal=password=s3cret` and then apply a Deployment in the namespace **`shop`** whose Pod reads `db-pass`. The Pod is stuck in `CreateContainerConfigError`, and `kubectl describe pod` shows `Error: secret "db-pass" not found`. What's the cause?
- A) The password must be base64-encoded by hand first
- B) Secrets can be read only through a mounted volume
- C) The Secret was made in `default`, not in `shop`
- D) The kubelet caches Secrets until the node reboots

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

## Concept 70 · What Helm is, and why we need it
**Words you'll learn:** Helm, package manager, chart, release, revision, chart repository, Artifact Hub, CNCF, render, Helm client, kubeconfig, Tiller.
- **Problem:** Priya's shop app has 6 YAML files (Deployment, Service, ConfigMap, Secret, Ingress, HPA). Three environments (dev, staging, prod) make **18 files, 90% identical**.
- Fixes drift between the copies, upgrades mean editing many files, and `kubectl rollout undo` rolls back only **one Deployment**, not the whole app.
- **Idea:** a **package manager** (like `apt` or `brew`) installs, upgrades and removes software with one command. **Helm is the package manager for Kubernetes**, a **CNCF** (Cloud Native Computing Foundation) project like Kubernetes.
- One Helm command installs a whole app, upgrades it, rolls it back **as one unit**, or removes it: `helm install web ./mychart`, `helm upgrade`, `helm uninstall web`, `helm repo update`.
- **Chart** = **the package**: a folder of Kubernetes YAML with **blanks**, plus default settings that fill them.
- **Release** = **one installed copy of a chart**, with a name you choose. `shop-dev` and `shop-prod` from one chart are two releases.
- **Revision** = a numbered version of a release. The install is 1; every upgrade **or rollback** adds 1.
- **Chart repository** ("repo") = a server storing charts, like Docker Hub. **Artifact Hub** (`artifacthub.io`) = a search website listing charts from thousands of repos.
- **Analogy: a restaurant.** Chart = a **recipe**; values = the customer's choices ("large, extra spicy"); release = **the dish served at table 5**; revision = each remake of that dish; repo = the cookbook shelf.

| Docker | Helm |
|---|---|
| Image (the packed app; does nothing alone) | **Chart** |
| Container (a running copy of an image) | **Release** (an installed copy of a chart) |
| One image → many containers | One chart → many releases |
| Docker Hub (a registry) | Chart repository (Artifact Hub to search) |
| Image tag `nginx:1.27` | Chart version `mychart-0.1.0` |

```
 You type: helm install web ./mychart
   1. the helm client (on your Mac) reads the templates + default values + your changes
   2. fills in the blanks → plain Kubernetes YAML                      ("rendering")
   3. sends that YAML to the API server (the same door kubectl uses)
   4. saves a record of revision 1 as a Secret in the namespace
 → the controllers create the Deployment, Service, … as usual
```
- To **render** = fill in the blanks to produce the final text. Kubernetes itself knows nothing about charts.
- **Helm is client-only**: nothing extra runs in the cluster. It reads your **kubeconfig** (`~/.kube/config`, which tells kubectl which cluster to use and how), so it has **exactly your kubectl permissions**.
- ⚠️ **Exam-awareness: Tiller.** **Helm v2** (before Nov 2019) had **Tiller**, a server inside the cluster (`kube-system`), usually with **admin rights over the whole cluster**: a big security risk.
- **Helm v3 removed Tiller.** The client talks straight to the API server, and release records moved from ConfigMaps in `kube-system` to **Secrets in the release's namespace**. Tiller in a question = Helm 2.
- Your Mac has **Helm v4** (Nov 2025). Its core commands and chart format are **the same as Helm 3**; a few flags were renamed (➕).

### 🛠 Do (3 min)
```bash
helm version     # ONE section (docker version shows Client + Server), because there is no Helm server
helm list -A     # list = releases; -A = all namespaces. You may see final-ticketdesk (leave it alone)
helm repo list   # repos Helm knows: bitnami https://charts.bitnami.com/bitnami, prometheus-community …
```

### ✅ Check 70
**70.1** Which pairing of Helm ideas with Docker ideas is correct?
- A) Chart ↔ container, release ↔ image
- B) Chart ↔ image, release ↔ container
- C) Chart ↔ registry, release ↔ image
- D) Chart ↔ Dockerfile, release ↔ registry

**70.2** A tutorial says: "Before installing charts, make sure Tiller is running in kube-system." You use Helm v4. What's accurate?
- A) Right; helm install fails until Tiller is ready
- B) Tiller is needed only for charts from repositories
- C) Tiller now runs on your Mac as a background daemon
- D) Tiller was Helm 2's; v3+ talk to the API directly

**70.3** (one line) Your team installs one chart three times in one cluster: `shop-dev`, `shop-staging`, `shop-prod`. How many charts and how many releases is that?

## Concept 71 · Chart structure: what `helm create` gives you
**Words you'll learn:** scaffold, `helm create`, `Chart.yaml`, chart `apiVersion`, `version` (chart version), `appVersion`, SemVer, `type` (application / library), `values.yaml`, `templates/`, named template, `_helpers.tpl`, `NOTES.txt`, `charts/`, dependency (subchart), `.helmignore`, `helm lint`.
- **Problem:** to write your own chart, you need the right files in the right places.
- **Idea:** `helm create mychart` makes a **scaffold**: a ready-made, working starter chart (it deploys nginx) that you then adapt.
- **Analogy:** the scaffolding put up around a building before the real work starts.
```
mychart/
├── Chart.yaml        ← the chart's ID card: name and versions
├── values.yaml       ← the default settings (the "knobs")
├── charts/           ← dependencies: other charts this one needs (empty at first)
├── templates/        ← Kubernetes YAML with blanks
│   ├── deployment.yaml, service.yaml, serviceaccount.yaml
│   ├── ingress.yaml, hpa.yaml, httproute.yaml (new in v4)   ← switched off by default
│   ├── _helpers.tpl  ← reusable named snippets, not a Kubernetes object
│   ├── NOTES.txt     ← the message printed after install
│   └── tests/test-connection.yaml
└── .helmignore       ← files to leave out when packaging
```

| `Chart.yaml` field (as `helm create` writes it) | Meaning |
|---|---|
| `apiVersion: v2` | The **chart format**: v2 = Helm 3 and later (v1 = Helm 2) |
| `name: mychart` / `description: …` | The chart's name (matching the folder) / one line about it |
| `type: application` | Installable (the normal kind). `library` = only reusable snippets; **can't be installed** |
| `version: 0.1.0` | **The chart's own version**. Required; bump it on every chart change |
| `appVersion: "1.16.0"` | **The version of the app inside**. Informational only (quoted to stay text) |

- ⚠️ **Two `apiVersion`s:** `apiVersion: v2` in Chart.yaml is the chart format. `apiVersion: apps/v1` in a Kubernetes manifest is the Kubernetes API group and version.
- **SemVer** (Semantic Versioning) = `MAJOR.MINOR.PATCH`: MAJOR = breaks things for users, MINOR = new features that don't, PATCH = bug fixes only.
- ⚠️ **Classic trap: `version` vs `appVersion`.** The `bitnami/nginx` chart **25.2.1** installs nginx **1.31.6**.
- A new label in a template with the same nginx → bump **`version`** only. A newer nginx → bump **`appVersion`** and also `version`. `helm list` shows both (CHART `mychart-0.1.0`, APP VERSION `1.16.0`).

**`values.yaml`**, the knobs (the important part of the generated file):
```yaml
replicaCount: 1
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""                  # empty, so the template uses the chart's appVersion
service:
  type: ClusterIP
  port: 80
ingress:
  enabled: false
resources: {}              # an empty map: no requests/limits set
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
```
- Every knob is a setting you know from K8s II–V. **A chart turns those settings into knobs**, so users change values instead of editing YAML.
- **`templates/`** holds normal manifests with blanks (Concept 72). Files whose names start with **`_`** are **never turned into Kubernetes objects**.
- `_helpers.tpl` holds **named templates**: small reusable snippets with a name, made with `define` and used with `include` (like a shell function). Examples: `mychart.fullname`, `mychart.labels`.
- **`NOTES.txt`** is a template printed after install and upgrade (usually how to reach the app). `tests/` holds a Pod that runs only on `helm test`.
- **`charts/`** holds **dependencies**, also called **subcharts** (e.g. Redis, listed in Chart.yaml and downloaded here; see ➕).
- **`.helmignore`** works like `.gitignore` and `.dockerignore`: file patterns to leave out when packaging (`.git/`, `*.swp`).

### 🛠 Do (~8 min)
```bash
mkdir -p ~/devops-practice/helm && cd ~/devops-practice/helm   # make the practice folder and move into it
helm create mychart                       # Creating mychart
find mychart | sort                       # every file and folder, alphabetically: the tree above
grep -vE '^(#|$)' mychart/Chart.yaml      # -v = lines NOT matching; ^(#|$) = comment or empty lines → the six real lines
grep -vE '^\s*(#|$)' mychart/values.yaml | head -20   # \s* = indented comments too → replicaCount: 1, image: …
helm lint mychart                         # check without installing: 1 chart(s) linted, 0 chart(s) failed
```
- The single quotes around `'^(#|$)'` stop the shell from touching `$`: **the outer quotes decide!**
- `helm lint` also prints `[INFO] Chart.yaml: icon is recommended`. That's only a suggestion; **0 failed** = the chart is valid.

### ✅ Check 71
**71.1** You add a new label to `templates/deployment.yaml` but still ship the same nginx 1.27 image. Which `Chart.yaml` field should you bump?
- A) `appVersion`, because the app's YAML changed
- B) `apiVersion`, from v2 up to the next number v3
- C) `version`, because the chart itself changed
- D) `appVersion` and `apiVersion`, as a matched pair

**71.2** `values.yaml` has `image.repository: nginx` and `image.tag: ""`. `Chart.yaml` has `version: 0.3.0` and `appVersion: "1.27"`. The template says `image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"`. Which image runs?
- A) `nginx:1.27`
- B) `nginx:0.3.0`
- C) `nginx:latest`
- D) `nginx:v2`

**71.3** (one line) What's special about files in `templates/` whose names start with `_`, like `_helpers.tpl`?

## Concept 72 · Templates and values: filling in the blanks
**Words you'll learn:** template, action (`{{ }}`), Go templating, built-in object, `.Values`, `.Release`, `.Chart`, pipeline (`|`), template function (`default`, `quote`, `upper`, `toYaml`, `nindent`, `required`), `if` / `range`, whitespace trimming (`{{-`), override, `-f` / `--values`, `--set`, precedence, per-environment values file, `helm template`.
- **Problem:** dev wants 1 replica and tag `dev`; prod wants 5 replicas and tag `1.4.2`. Two copies of `deployment.yaml` that differ in two lines = copy-paste drift again.
- **Idea:** a **template** is a file with **blanks** that are filled in later.
- **Analogy: a mail-merge letter** ("Dear ____, …"). One design; `values.yaml` holds the default fill-ins, a `-f` file is a sheet of changes, and `--set` is a last handwritten correction that beats everything.
- In `replicas: {{ .Values.replicaCount }}`, everything inside `{{ }}` is an **action**: an instruction to Helm, not YAML. Rendering replaces it, so `replicaCount: 3` gives `replicas: 3`.
- This style is **Go templating** (from Go, Helm's language). The leading **dot `.`** means "everything available to this template".

| Built-in object | What it holds | Example → result |
|---|---|---|
| `.Values` | values.yaml **merged with your overrides** | `{{ .Values.image.repository }}` → `nginx` |
| `.Release.Name` | the release name you chose | → `web` |
| `.Release.Namespace` / `.Release.Revision` | its namespace / its current revision | → `helm-demo` / `1` |
| `.Chart.Name` / `.Chart.Version` / `.Chart.AppVersion` | Chart.yaml fields, **capitalised** | → `mychart` / `0.1.0` / `1.16.0` |
| `.Capabilities.KubeVersion` | the cluster's Kubernetes version | → `v1.3x.x` |

- ⚠️ **Case matters, and typos are silent.** `.Values.replicacount` is not `.Values.replicaCount`. A missing key renders as **nothing**, with **no error**, and `helm lint` passes too.
- You'd get `replicas:` with an empty value, and Kubernetes uses its default of 1 replica. A favourite debugging question.
- **The pipe `|`** passes a value into a **function** (a small built-in tool that transforms it), like the shell pipe. `{{ .Values.env | upper | quote }}` turns `dev` into `DEV`, then `"DEV"`.

| Function | What it does | Example |
|---|---|---|
| `default X` | use X if the value is empty | `.Values.image.tag \| default .Chart.AppVersion` |
| `quote` / `upper` / `lower` | wrap in quotes / change the case | `dev` → `"dev"` / `DEV` |
| `toYaml` | turn a map or list from values into YAML text | used for `resources:` blocks |
| `nindent N` | start a new line, then indent N spaces | keeps inserted YAML at the right level |
| `include "name" .` | insert a named template from `_helpers.tpl` | `{{ include "mychart.fullname" . }}` |
| `required "msg" X` | **stop with an error** if X is empty | `{{ required "image.tag is required" .Values.image.tag }}` |

**Conditions, loops and a real excerpt** from the generated `templates/deployment.yaml`:
```yaml
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```
- `if … end` renders the lines between **only when** the condition is true. Here `replicas` is left out when autoscaling is on, because the **HPA** (Horizontal Pod Autoscaler) controls the count then.
- That's also why `ingress.yaml` produces nothing while `ingress.enabled: false`. **`range`** is a loop over a list (like `for` in the shell).
- The **dash** in `{{-` (or `-}}`) trims the spaces and newline on that side. `{{/* … */}}` is a template comment.
- `mychart.fullname` = release name + `-` + chart name, so release `web` makes objects named **`web-mychart`** (names are cut at 63 characters, the DNS limit).
- **Overriding values, three ways:** edit `values.yaml` (new defaults for everyone); **`-f my-values.yaml`** (long form `--values`); or **`--set key=value`**.
- A `-f` file holds **only the keys you change**. You can pass several; **the later file wins**.
- `--set`: dots mean nesting (`--set image.tag=1.28-alpine`), commas set several (`--set replicaCount=3,service.type=NodePort`), and `--set-string` forces text (e.g. a tag like `1.10`).
- For list items, quote it: `--set 'ingress.hosts[0].host=shop.local'`, because zsh treats `[ ]` as a filename pattern.
```
 Precedence (checked on your Helm v4):
 LOWEST  chart's values.yaml  <  -f first.yaml  <  -f later.yaml  <  --set  HIGHEST
```
- `--set` wins **even if you type it before `-f`**. Merging is **key by key**: keys you don't mention keep their defaults.
- **One chart, many environments:** `helm install shop-dev ./mychart -f values-dev.yaml -n dev`, and the same with `values-prod.yaml` for prod. Two short values files, easy to review in a PR.
- **`helm template <name> <chart>`** renders the chart and **prints** the YAML. **Nothing is sent to the cluster.** `-s templates/deployment.yaml` (long form `--show-only`) prints one file.
- **`helm lint`** catches syntax errors: a missing `}` gives `[ERROR] … parse error … unexpected "}" in operand` and exit code 1. It **can't** catch a misspelt values key.

### 🛠 Do (~12 min)
`values-prod.yaml` for step 3 (create it next to the chart, not inside it):
```yaml
replicaCount: 5
image:
  tag: "1.28-alpine"
```
```bash
cd ~/devops-practice/helm
helm template web ./mychart -s templates/deployment.yaml | head -40   # web = the release name to pretend with
code mychart/values.yaml                  # change replicaCount: 1 → 2, and under image: tag: "" → "1.27"; save
helm template web ./mychart -s templates/deployment.yaml | grep -E 'replicas|image:'   # replicas: 2, image: "nginx:1.27"
code values-prod.yaml                     # paste the 3 lines above, save
helm template web ./mychart -f values-prod.yaml -s templates/deployment.yaml | grep -E 'replicas|image:'
helm template web ./mychart -f values-prod.yaml --set replicaCount=3 -s templates/deployment.yaml | grep -E 'replicas|image:'
```
- The first render shows `name: web-mychart`, `replicas: 1`, `app.kubernetes.io/managed-by: Helm` and `image: "nginx:1.16.0"`: the tag `""` was empty, so `default .Chart.AppVersion` kicked in.
- With `-f values-prod.yaml`: `replicas: 5` and `image: "nginx:1.28-alpine"`. The repository is still `nginx`, because merging is key by key.
- Adding `--set replicaCount=3`: `replicas: 3` (from `--set`) and still `nginx:1.28-alpine` (from the file).

**Break it on purpose.** In `mychart/templates/deployment.yaml`, delete **one** closing brace so the line reads `replicas: {{ .Values.replicaCount }`. Save, then:
```bash
helm lint mychart              # [ERROR] … unexpected "}" in operand; 1 chart(s) linted, 1 chart(s) failed
echo "exit code: $?"           # exit code: 1
```
Put the brace back. Now change `.Values.replicaCount` to `.Values.replicacount` (lowercase c), save, and run:
```bash
helm lint mychart              # 0 chart(s) failed…
helm template web ./mychart -s templates/deployment.yaml | grep replicas   # …but `replicas:` has NOTHING after it
```
No error anywhere: that's why this bug is sneaky. **Change it back to `replicaCount`** and run `helm lint mychart` again.

### ✅ Check 72
**72.1** `values.yaml` has `replicaCount: 1`. `values-prod.yaml` has `replicaCount: 4`. You run `helm install shop ./mychart --set replicaCount=2 -f values-prod.yaml`. How many replicas are rendered?
- A) 4, because -f comes last on the line
- B) 2, because --set beats any -f file
- C) 1, because the chart's defaults win
- D) An error about conflicting values

**72.2** A template contains `replicas: {{ .Values.replicaCount }}`, but values.yaml says (by mistake) `replicacount: 3`. `helm lint` passes. What does `helm template` print for that line?
- A) `replicas: 3`, as keys ignore case
- B) An error: key replicaCount not found
- C) `replicas:` with nothing after it
- D) `replicas: 0`, the default for ints

**72.3** (one line) What's the difference between `helm template` and `helm install`?

## Concept 73 · Install, upgrade, rollback: deploying an app with Helm
**Words you'll learn:** `helm install`, `helm list`, `helm status`, `helm upgrade`, `--install`, `--reuse-values`, `helm history`, `deployed` / `superseded` / `failed`, `helm rollback`, `helm uninstall`, `--keep-history`, `--wait`, `--timeout`, `--namespace` / `--create-namespace`, `helm repo add / update`, `helm search repo / hub`, `helm show values`, public chart.
```
 helm install web ./mychart           → revision 1  deployed
 helm upgrade web ./mychart --set …   → revision 2  deployed   (1 becomes superseded)
 helm upgrade web ./mychart --set …   → revision 3  deployed   (2 becomes superseded)
 helm rollback web 2                  → revision 4  deployed   = a COPY of revision 2's settings
 helm uninstall web                   → all the release's objects AND its history deleted
```
- **`deployed`** = the live revision. **`superseded`** = replaced by something newer. **`failed`** = Helm knows that revision went wrong.
- **Analogy: a shared document's version history.** Restoring an old version doesn't erase the later ones; it adds a **new** version on top that copies the old one.
- ⚠️ Same with `helm rollback`: revision 3 stays in the history. On your v4, `helm history` showed `4  deployed  Rollback to 2`.

| Command | What it does |
|---|---|
| `helm install <release> <chart>` | Create a release (revision 1). The chart can be a folder, a `.tgz` file, `repo/chart` (`bitnami/nginx`), or `oci://` (➕) |
| `helm list` (short: `helm ls`) | Releases **in the current namespace**. `-n <ns>` for another one, `-A` for all |
| `helm status <release>` | Status, revision and the NOTES |
| `helm upgrade <release> <chart>` | A new chart version or new values → a new revision |
| `helm upgrade --install <release> <chart>` | **Upgrade if it exists, install if it doesn't**: one command for the first and every later deploy (pipelines, next session) |
| `helm history <release>` | Every revision with its STATUS and DESCRIPTION |
| `helm rollback <release> <revision>` | Back to that revision (as a new one). No number (or 0) = the previous revision |
| `helm uninstall <release>` | Delete the objects **and** the history. `--keep-history` keeps the records |
| `helm get values` / `helm get manifest <release>` | The values **you** supplied (`--all` = every value) / the YAML actually sent to the cluster |

- ⚠️ **The `--set` trap:** each `helm upgrade` starts again from the chart's values.yaml **plus only this command's flags**. Earlier `--set` values are **forgotten**.
- Checked on v4: an upgrade with only `--set replicaCount=4` silently put the image back to `1.27`.
- **Fixes:** keep your settings in a values file in Git and pass `-f values-prod.yaml` **on every upgrade** (best). Or add **`--reuse-values`** (start from the last release's values).
- ⚠️ **"deployed" doesn't mean "working".** Helm sends the YAML and returns at once. An upgrade to a non-existent tag still says `STATUS: deployed`.
- Meanwhile one new Pod sits in `ImagePullBackOff`, and the old Pods keep serving (the rolling update keeps them until new ones are Ready).
- **`--wait`** (with `--timeout 2m`) waits until the Pods are Ready. If they aren't, the revision is **`failed`** and the command exits **non-zero**, so a pipeline goes red.
- **`--atomic`** (Helm 3) / **`--rollback-on-failure`** (Helm 4) = roll back automatically if the upgrade fails.
- **Names and namespaces:** a release name must be **unique within its namespace** (else `cannot reuse a name that is still in use`). Use lowercase letters, numbers and dashes.
- `-n` (`--namespace`) picks the namespace, and **`--create-namespace`** creates it if missing. ⚠️ Plain `helm list` shows only the current namespace: "my release disappeared!" usually means it's in another one (use `-A`).
- **Public charts:** `helm repo add bitnami https://charts.bitnami.com/bitnami` remembers a repo under a short name (yours says `already exists … skipping`). `helm repo update` downloads each repo's latest chart list (like `apt update`).
- `helm search repo nginx` searches your repos: `bitnami/nginx  25.2.1  1.31.6` = CHART VERSION, then APP VERSION. `helm search hub nginx` searches Artifact Hub.
- `helm show values bitnami/nginx | head -30` lets you read the knobs **before** installing. Since Aug 2025, Bitnami's free images are limited: a likely cause if its Pod shows `ImagePullBackOff`.
- **Awareness:** normal clusters install the ingress-nginx controller with Helm, but **don't on minikube** (the addon already did). It stopped getting updates in 2026; its successor is the **Gateway API** (a newer way to route outside traffic in), used by Helm v4's `httproute.yaml`.

### 🛠 Do (~20 min): deploy, upgrade, break, roll back, remove
First check that the chart has `replicaCount` (capital C) set to 2 and `tag: "1.27"`, and that `helm lint mychart` passes.
```bash
cd ~/devops-practice/helm
helm install web ./mychart -n helm-demo --create-namespace   # STATUS: deployed, REVISION: 1, then the NOTES (NOTES.txt, rendered)
helm list -n helm-demo                    # web  helm-demo  1  …  deployed  mychart-0.1.0  1.16.0
kubectl get deploy,pods,svc -n helm-demo  # web-mychart 2/2, two Pods Running, service/web-mychart ClusterIP 80/TCP
kubectl get secrets -n helm-demo          # sh.helm.release.v1.web.v1  helm.sh/release.v1 = Helm's record of revision 1
kubectl port-forward -n helm-demo svc/web-mychart 8080:80   # in a 2nd tab (Cmd + T): Mac port 8080 → Service port 80
curl -I http://localhost:8080             # back in tab 1: HTTP/1.1 200 OK, Server: nginx/1.27.x
helm upgrade web ./mychart -n helm-demo --set image.tag=1.28-alpine --set replicaCount=3   # Happy Helming! REVISION: 2
kubectl get deploy web-mychart -n helm-demo -o wide   # -o wide adds IMAGES: 3/3, nginx:1.28-alpine
helm upgrade web ./mychart -n helm-demo --set image.tag=9.99-nope --set replicaCount=3    # a typo'd tag: STATUS: deployed, REVISION: 3 (!)
kubectl get pods -n helm-demo             # ~20 s later: 3 old Pods Running + 1 ImagePullBackOff
helm history web -n helm-demo             # revision 3: deployed   Upgrade complete
helm rollback web 2 -n helm-demo          # Rollback was a success! Happy Helming!
helm history web -n helm-demo             # revision 4: deployed   Rollback to 2; revisions 1–3 superseded
kubectl get pods -n helm-demo             # only 3 Running Pods remain
helm get values web -n helm-demo          # USER-SUPPLIED VALUES: image: tag: 1.28-alpine, replicaCount: 3
helm upgrade web ./mychart -n helm-demo --set replicaCount=4   # revision 5
kubectl get deploy web-mychart -n helm-demo -o wide   # 4/4, but the image is back to nginx:1.27: the --set trap
helm search repo bitnami/nginx            # (optional, ~3 min) a public chart
helm install my-nginx bitnami/nginx -n helm-demo --set service.type=ClusterIP
kubectl get pods -n helm-demo -l app.kubernetes.io/instance=my-nginx   # my-nginx-nginx-… 1/1 Running after ~30 s
helm uninstall my-nginx -n helm-demo      # release "my-nginx" uninstalled
helm uninstall web -n helm-demo           # (Ctrl + C the port-forward first) release "web" uninstalled
helm history web -n helm-demo             # Error: release: not found (the history went too)
kubectl delete namespace helm-demo        # keep the ~/devops-practice/helm folder for revision
```
- **Port-forward:** the Service is `ClusterIP`, which your Mac can't reach directly, so you open a tunnel. It really connects to **one Pod**, so after each upgrade it breaks (`lost connection to pod`): Ctrl + C and run it again.
- **Roll back to a revision you know worked.** Revision 3 was also marked `deployed`, so "the latest deployed" isn't proof it worked.
- `helm list`'s APP VERSION says `1.16.0` (Chart.yaml's claim) even though nginx 1.27 runs: it's informational only. `-l app.kubernetes.io/instance=my-nginx` works because Helm labels everything with the release name.

### ✅ Check 73
**73.1** `helm history shop` shows revisions 1–3, with 3 `deployed`. You run `helm rollback shop 1`. What does `helm history shop` show now?
- A) Revisions 1–3, with revision 1 marked as deployed
- B) Revision 1 only, since 2 and 3 are deleted
- C) Revisions 1–4, with 4 deployed: "Rollback to 1"
- D) Revisions 1–3, with 3 marked "rolled back"

**73.2** You installed with `helm install web ./mychart --set image.tag=1.28`. Later you run `helm upgrade web ./mychart --set replicaCount=3`. The chart's values.yaml has `tag: "1.27"`. Which image runs after the upgrade?
- A) nginx:1.28, because Helm keeps earlier --set values
- B) nginx:1.27, since only this upgrade's flags apply
- C) nginx:1.28, since upgrades change only what's named
- D) No change: upgrade without -f refuses to run at all

**73.3** (one line) A pipeline runs `helm upgrade --install shop ./chart`. What does `--install` add, and why is it handy in CI/CD (Continuous Integration / Continuous Delivery)?

## ➕ Small but important (Helm): read before the quiz
| Item | What it is |
|---|---|
| **Kustomize** | Built into kubectl (`kubectl apply -k <folder>`). Customises YAML **without templates**: a **base** folder of normal YAML plus small per-environment **overlays** of **patches** (only the lines that differ) |
| **Helm vs Kustomize vs `kubectl apply -f`** | Helm: templates + values, charts and repos, numbered revisions, `helm rollback` of everything together. Kustomize: no packaging or releases (roll back by re-applying an older Git commit). Plain kubectl: copy and edit files |
| `--dry-run=client` / `--dry-run=server` | Go through an install or upgrade **without changing anything**, and print the result. `server` also has the API server validate it |
| `--generate-name` | Helm invents the release name (e.g. `mychart-1760600000`) |
| **Chart dependencies** | Listed in Chart.yaml under `dependencies:` (name, version, repository, `condition: redis.enabled`). **`helm dependency update mychart`** downloads them into `charts/` and writes **`Chart.lock`** (the exact versions) |
| `helm package mychart` | Bundles the folder into **`mychart-0.1.0.tgz`** (`name` + `version`; `.tgz` = a gzip-compressed `tar` archive). `helm repo index` makes a repo's `index.yaml` |
| **OCI registries** | **OCI** = **Open Container Initiative**, the standard format of container registries. Charts can live next to images: `helm push mychart-0.1.0.tgz oci://ghcr.io/<you>/charts` |
| **Hooks** | Templates (usually Jobs) with the annotation `"helm.sh/hook": pre-install` run **at a chosen moment**, e.g. a database migration before an upgrade |
| `helm test <release>` | Runs the chart's test Pods (`templates/tests/`). Pass = exit code 0 |
| **Where releases are stored** | **Secrets** of type `helm.sh/release.v1`, named **`sh.helm.release.v1.<release>.v<revision>`**, in the release's namespace. Delete the namespace and the history goes too |
| `--history-max` | Revisions kept per release (default **10**); older ones are deleted |
| `helm show values <chart>` · `helm pull bitnami/nginx --untar` | Read a chart's defaults without installing · download and unpack a chart to read or modify it |
| `--reuse-values` / `--reset-values` | On upgrade: keep the last release's values and merge new ones / go back to the chart's defaults (the normal behaviour) |
| **Helm v4 renames** | `--atomic` → **`--rollback-on-failure`**; `--force` → `--force-replace`; plain `--dry-run` → `--dry-run=client`; `helm list` shows **every status** by default; changes use **server-side apply** |

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory
**Q1.** An engineer runs `helm install shop ./shop -n prod` and gets `Error: INSTALLATION FAILED: release name check failed: cannot reuse a name that is still in use`. Running `helm list` (no flags) shows an empty table. What's the most likely explanation?
- A) The chart folder ./shop has no Chart.yaml inside
- B) A release named shop already exists in prod
- C) kubectl already created a Deployment named shop
- D) Helm needs `helm repo update` before reusing a name

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

**Q4.** A teammate makes four claims about Helm v3 and later. Which one is **NOT** true?
- A) The helm client talks straight to the API server
- B) Release records live as Secrets in its namespace
- C) Tiller must run in kube-system to apply charts
- D) A chart's `version` and `appVersion` can differ

**L1.** Explain **chart vs release** in contrast form, with an example.

**L2.** You set `--set image.tag=2.0` in one upgrade, and the next upgrade (`--set replicaCount=4` only) brings back the old image. Explain why, and give two ways to avoid it.
