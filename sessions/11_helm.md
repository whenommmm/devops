# Session 11 · Helm: packaging Kubernetes apps (Oct 16, Session 1)

**Time:** ~90–100 min · **Concepts:** 70 → 73 · **Answers:** `MY_ANSWERS.md` or say "Helm questions" · **Answer key:** `11_helm_ANSWERS.md`

**Before you start:** `minikube status` should show `Running` (if not, run `minikube start` and wait a minute or two), `kubectl get nodes` should show `minikube   Ready`, and `helm version` should show `Version:"v4.3.0"` (Concept 70 explains that line).

⚠️ Never delete the **`minikube`** or **`localstack`** containers. Leave the namespaces you didn't create (`argocd`, `ingress-nginx`, `final-app`) alone: `helm list -A` may show a **`final-ticketdesk`** release in `final-app` from earlier work. Only remove what you create today.

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

### The problem
Priya's team runs an online shop on Kubernetes. For **one app** they have 6 YAML files (Deployment, Service, ConfigMap, Secret, Ingress, HPA). They need **three environments** (dev, staging, prod) with slightly different settings: 1 replica in dev but 5 in prod, different image tags. So they copy the folder three times: **18 files, 90% identical**. Then the headaches start:
- **Copy-paste drift:** someone fixes prod's `deployment.yaml` and forgets dev's copy, so the environments quietly differ.
- **Upgrades are manual:** a new version means editing the tag in several files and running `kubectl apply -f` on each one.
- **No "app version" to roll back:** `kubectl rollout undo` (Session 06) undoes **one Deployment**. If version 2 also changed the ConfigMap and the Ingress, no single command puts all of them back together.

### The idea: a package manager for Kubernetes
> **Helm is the package manager for Kubernetes.** Like `apt` or `brew` from the Linux session (one command installs software, keeps track of it, and later upgrades or removes it), one Helm command installs a whole app (all its Kubernetes objects together), upgrades it, rolls it back **as one unit**, or removes it: `helm install`, `helm upgrade`, `helm rollback`, `helm uninstall`. Helm is a **CNCF** (**Cloud Native Computing Foundation**) project, like Kubernetes itself.

### The four core words
- **Chart** = **the package**: a folder of Kubernetes YAML files with **blanks** in them, plus a file of **default settings** that fill the blanks. One chart describes one app.
- **Release** = **one installed copy of a chart** in a cluster, with a name you choose. Install one chart three times (`shop-dev`, `shop-staging`, `shop-prod`) and you have three releases.
- **Revision** = a **numbered version of a release**. The install is revision 1, and every upgrade **or rollback** adds 1. Helm remembers each one, so it can go back.
- **Chart repository** ("repo") = a **server that stores charts** for others to download, like Docker Hub for images. **Artifact Hub** (`artifacthub.io`) is a **search website** listing charts from thousands of repos: it tells you which repo has the chart you want.

| Docker (you know this pattern already) | Helm |
|---|---|
| Image (the packed app; does nothing on its own) | **Chart** |
| Container (a running copy of an image) | **Release** (an installed copy of a chart) |
| One image → many containers | One chart → many releases |
| Docker Hub (a registry) · image tag `nginx:1.27` | Chart repository · chart version `mychart-0.1.0` |

**Analogy: a restaurant.** The chart is a **recipe**, and the settings (values) are the customer's choices ("large, extra spicy"). A release is **the dish served at table 5**; table 9 can order the same recipe differently, which is another release. A revision is each remake of table 5's dish ("less salt, please" = revision 2).

### How Helm works
```
 You type: helm install web ./mychart
   1. helm (on your Mac) reads the chart: templates + default settings + your changes
   2. fills in the blanks → plain Kubernetes YAML                    ("rendering")
   3. sends that YAML to the API server (the same door kubectl uses)
   4. saves a record of revision 1 as a Secret in the release's namespace
 → the controllers create the Deployment, Service, … as usual
```
- To **render** = to fill in the blanks and produce the final text. After rendering it's ordinary Kubernetes YAML: **Kubernetes itself knows nothing about charts**.
- **Helm is client-only:** the `helm` command on your Mac is the whole program, and nothing extra runs inside the cluster. To find the cluster, Helm reads your **kubeconfig**: the file `~/.kube/config` that tells kubectl which cluster to talk to and with which credentials. So Helm has **exactly the same permissions** you have with kubectl.

⚠️ **Exam-awareness: Tiller.** **Helm v2** (before Nov 2019) had a second part called **Tiller**: a server running **inside the cluster** (in `kube-system`) that did the installing for the client. Tiller usually had **admin rights over the whole cluster**, so anyone who could reach it could do anything: a big security risk. **Helm v3 removed Tiller.** The client now talks straight to the API server, and release records moved from ConfigMaps in `kube-system` to **Secrets in the release's own namespace**. If a question mentions Tiller, it's about Helm 2. (Your Mac has **Helm v4**: its commands and chart format are **the same as Helm 3**, so anything written for Helm 3, including exam questions, applies. Two flags were renamed: see ➕.)

### 🛠 Do (~3 min)
```bash
helm version     # ONE line, no "Server" part (docker version showed Client + Server): there is no Helm server
helm list -A     # list = releases; -A = all namespaces. You may see final-ticketdesk (leave it alone)
helm repo list   # repos Helm already knows: bitnami https://charts.bitnami.com/bitnami, prometheus-community …
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

### `helm create`: a ready-made starter chart
To write your **own** chart you need the right files in the right places. Instead of starting from an empty folder, `helm create mychart` makes a **scaffold**: a ready-made starting structure that you then adapt, like the scaffolding put up around a building before the real work starts. It's a complete, working chart that deploys **nginx**:
```
mychart/
├── Chart.yaml        ← the chart's ID card: name and versions
├── values.yaml       ← the default settings (the "knobs")
├── charts/           ← other charts this one depends on (empty at first)
├── templates/        ← Kubernetes YAML with blanks: deployment.yaml, service.yaml, ingress.yaml, hpa.yaml, …
│   ├── _helpers.tpl  ← reusable snippets, not a Kubernetes object
│   ├── NOTES.txt     ← the message printed after install
│   └── tests/        ← a test Pod, run only by `helm test`
└── .helmignore       ← files to leave out when packaging
```

### `Chart.yaml`: the chart's ID card (as `helm create` writes it, comments removed)
```yaml
apiVersion: v2          # the CHART FORMAT: v2 = Helm 3 and later (v1 was Helm 2)
name: mychart           # the chart's name; should match the folder name
description: A Helm chart for Kubernetes   # one line saying what the chart is
type: application       # installable (the normal kind); "library" = only snippets for other charts, can't be installed
version: 0.1.0          # the CHART's own version: required, bump it every time you change the chart
appVersion: "1.16.0"    # the version of the APP inside (nginx): informational only, quoted so YAML keeps it as text
```
⚠️ **Two different `apiVersion`s.** `apiVersion: v2` in **Chart.yaml** is the chart format. `apiVersion: apps/v1` at the top of a **Kubernetes manifest** is the Kubernetes API group and version. Same word, different files, different meaning.

**SemVer** (**Semantic Versioning**) is the `MAJOR.MINOR.PATCH` numbering the chart `version` must follow. In `1.4.2`, **MAJOR** (1) goes up for changes that **break** things for users, **MINOR** (4) for new features that don't, and **PATCH** (2) for bug fixes only.

⚠️ **Classic trap: `version` vs `appVersion`.** `version` describes **the packaging** (the chart: its templates and default values); `appVersion` describes **what's packaged** (the app). Real example: the `bitnami/nginx` chart is **chart version 25.2.1**, and it installs **nginx 1.31.6**. `helm list` shows both (CHART `mychart-0.1.0`, APP VERSION `1.16.0`).
- You add a label to a template but ship the same nginx → bump **`version`** only.
- You ship a newer nginx → bump **`appVersion`**, and also `version`, because the chart's content changed.

### `values.yaml`: the default settings
The list of **knobs** your chart offers (the important part of the generated file). Each knob is a setting you already know from the Kubernetes sessions: **a chart turns those settings into knobs**, so users change values instead of editing YAML.
```yaml
replicaCount: 1          # how many Pods (Session 06)
image:
  repository: nginx      # which image
  tag: ""                # empty, so the template uses the chart's appVersion instead
service:
  type: ClusterIP        # the Service type (Session 07)
  port: 80
ingress:
  enabled: false         # the Ingress template produces nothing while this is false
autoscaling:
  enabled: false         # true = an HPA controls the replica count instead
```

### The rest of the folder
- **`_helpers.tpl`**: Helm **never turns files whose names start with `_` into Kubernetes objects**. This one holds **named templates**: small reusable snippets with a name, created with `define` and inserted elsewhere with `include`. For example, `mychart.fullname` builds the name of every object, and `mychart.labels` builds the standard labels. It's like a shell **function**: define it once, call it many times.
- **`NOTES.txt`** is also a template. Helm fills it in and **prints it after install and upgrade**; it usually tells you how to reach the app.
- **`charts/`** holds **dependencies**, also called **subcharts**: other charts your chart needs. If your app needs Redis, you list Redis in `Chart.yaml`, and Helm downloads the Redis chart into this folder (➕).
- **`.helmignore`** works like `.gitignore` and `.dockerignore`: file patterns (`.git/`, `*.swp`) to leave out when the chart is packaged into one file.

### 🛠 Do (~8 min)
```bash
mkdir -p ~/devops-practice/helm && cd ~/devops-practice/helm   # make the practice folder and move into it
helm create mychart                       # Creating mychart
find mychart | sort                       # every file and folder, alphabetically: the tree above
grep -vE '^(#|$)' mychart/Chart.yaml      # -v = lines NOT matching; ^(#|$) = comment or empty lines → the six real lines
grep -vE '^\s*(#|$)' mychart/values.yaml | head -20   # \s* = indented comments too → replicaCount: 1, image: …
helm lint mychart                         # check the chart without installing: 1 chart(s) linted, 0 chart(s) failed
```
*(The single quotes around `'^(#|$)'` stop the shell from touching `$`: **the outer quotes decide!** `helm lint` also prints `[INFO] Chart.yaml: icon is recommended`. That's only a suggestion; **0 failed** = the chart is valid.)*

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

### The idea: YAML with blanks
Dev wants **1 replica** and image tag `dev`; prod wants **5 replicas** and tag `1.4.2`. Two copies of `deployment.yaml` that differ in two lines would bring back Concept 70's copy-paste drift. The fix is a **template**: a file with **blanks** that are filled in later. **Analogy: a mail-merge letter** ("Dear ________, …"): one design, and each person's name is filled in. In a Helm template, a blank looks like `replicas: {{ .Values.replicaCount }}`.
- Everything between `{{` and `}}` is an **action**: an instruction to Helm, not YAML.
- **Rendering** replaces each action with a value: with `replicaCount: 3` in values.yaml, the line becomes `replicas: 3`.
- This `{{ }}` style is called **Go templating**, because it comes from Go, the programming language Helm is written in.

### Built-in objects: what you can put in the blanks
The leading **dot `.`** means "everything available to this template". From it you reach the **built-in objects** that Helm always provides:

| Object | What it holds | Example → result |
|---|---|---|
| `.Values` | values.yaml **merged with your overrides** | `{{ .Values.image.repository }}` → `nginx` |
| `.Release.Name` / `.Release.Namespace` | the release name you chose / its namespace | → `web` / `helm-demo` |
| `.Chart.Name` / `.Chart.Version` / `.Chart.AppVersion` | fields from Chart.yaml, **capitalised** | → `mychart` / `0.1.0` / `1.16.0` |

⚠️ **Case matters, and typos are silent.** `.Values.replicaCount` and `.Values.replicacount` are **different keys**. If a key doesn't exist, Helm fills the blank with **nothing** and gives **no error**, and `helm lint` passes too. You'd get `replicas:` with an empty value, and Kubernetes then uses its default of 1 replica. A favourite debugging question.

### Pipelines and functions
The **pipe `|`** passes a value into a **function** (a small built-in tool that transforms it), just as the shell pipe sends output into the next command. `{{ .Values.env | upper | quote }}` turns `dev` into `DEV` (`upper` = capital letters; `lower` does the opposite), then into `"DEV"` (`quote` = wrap in double quotes).

| Function | What it does | Example |
|---|---|---|
| `default X` | use X if the value is empty | `.Values.image.tag \| default .Chart.AppVersion` |
| `toYaml` + `nindent N` | turn a map from values into YAML text / start it on a new line, indented N spaces | used for `resources:` blocks |
| `include "name" .` | insert a named template from `_helpers.tpl` | `{{ include "mychart.fullname" . }}` |
| `required "msg" X` | **stop with an error** if X is empty | `{{ required "image.tag is required" .Values.image.tag }}` |

**Conditions, loops and a real excerpt** from the generated `templates/deployment.yaml`:
```yaml
metadata:
  name: {{ include "mychart.fullname" . }}     # release name + "-" + chart name → web-mychart
spec:
  {{- if not .Values.autoscaling.enabled }}    # only when autoscaling is OFF…
  replicas: {{ .Values.replicaCount }}         # …write a fixed replica count
  {{- end }}                                   # end of the if
  # … further down, inside the container:
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```
- **`if … end`** renders the lines between **only when** the condition is true. Here `replicas` is left out when autoscaling is on, because the **HPA** (Horizontal Pod Autoscaler, Session 09) controls the count then. That's also why `ingress.yaml` produces nothing while `ingress.enabled: false`.
- **`range`** is a loop over a list (like `for` in the shell), e.g. one Ingress rule per host.
- The **dash** in `{{-` (or `-}}`) trims the spaces and newline on that side, so the output has no stray blank lines.

### Overriding values: three ways
1. **Edit `values.yaml`** inside the chart: new defaults for everyone.
2. **`-f my-values.yaml`** (long form `--values`): a separate file with **only the keys you change**. You can pass several, and **the later file wins**.
3. **`--set key=value`** on the command line. Dots mean nesting (`--set image.tag=1.28-alpine` sets `tag` inside `image`), and commas set several at once (`--set replicaCount=3,service.type=NodePort`).

**Precedence** (who wins when the same key is set twice), checked on your Helm v4:
```
 LOWEST  chart's values.yaml  <  -f first.yaml  <  -f later.yaml  <  --set  HIGHEST
```
- `--set` wins **even if you type it before `-f`**. Position matters only **between** `-f` files.
- Merging is **key by key**: keys you don't mention keep their defaults. A file containing only `image.tag` changes the tag and leaves `image.repository` alone.
- **One chart, many environments:** `helm install shop-dev ./mychart -f values-dev.yaml -n dev`, then the same chart with `-f values-prod.yaml -n prod`. Concept 70's 18 files become one chart plus two short values files that are easy to review in a PR.

### Checking without installing
- **`helm template <name> <chart>`** renders the chart and **prints the YAML** on your screen. **Nothing is sent to the cluster.** Add `-s templates/deployment.yaml` (long form `--show-only`) to print only one file.
- **`helm lint <chart>`** checks the structure and the **template syntax**: a missing `}` gives `[ERROR] … unexpected "}" in operand` and exit code 1. It **can't** catch a misspelt values key.

### 🛠 Do (~12 min)
`values-prod.yaml`, for the 4th command (create it next to the chart, not inside it):
```yaml
replicaCount: 5          # prod wants 5 Pods
image:
  tag: "1.28-alpine"     # only the tag; the repository keeps the chart's default
```
```bash
cd ~/devops-practice/helm
helm template web ./mychart -s templates/deployment.yaml | head -40   # web = the release name to pretend with (see below)
code mychart/values.yaml                  # change replicaCount: 1 → 2, and under image: tag: "" → "1.27"; save
helm template web ./mychart -s templates/deployment.yaml | grep -E 'replicas|image:'   # replicas: 2, image: "nginx:1.27"
code values-prod.yaml                     # paste the 3 lines above, save
helm template web ./mychart -f values-prod.yaml -s templates/deployment.yaml | grep -E 'replicas|image:'   # replicas: 5, "nginx:1.28-alpine"
helm template web ./mychart -f values-prod.yaml --set replicaCount=3 -s templates/deployment.yaml | grep -E 'replicas|image:'   # replicas: 3, "nginx:1.28-alpine"
```
The first render shows `name: web-mychart`, `replicas: 1`, the label `app.kubernetes.io/managed-by: Helm` and `image: "nginx:1.16.0"` (the tag `""` was empty, so `default .Chart.AppVersion` kicked in). With `-f values-prod.yaml` the repository is still `nginx`, because merging is key by key; `--set replicaCount=3` then beats the file's `5`.

**Break it on purpose.** In `mychart/templates/deployment.yaml`, (1) delete **one** closing brace so the line reads `replicas: {{ .Values.replicaCount }`, run the commands below, and put the brace back. (2) Change `.Values.replicaCount` to `.Values.replicacount` (lowercase c), run them again, then **change it back** and lint once more.
```bash
helm lint mychart; echo "exit code: $?"   # (1) [ERROR] … unexpected "}" in operand, 1 chart(s) failed, exit code: 1  (2) 0 chart(s) failed
helm template web ./mychart -s templates/deployment.yaml | grep replicas   # (2) `replicas:` with NOTHING after it: no error anywhere, sneaky!
```

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

### The life of a release
```
 helm install web ./mychart           → revision 1  deployed
 helm upgrade web ./mychart --set …   → revision 2  deployed   (1 becomes superseded)
 helm upgrade web ./mychart --set …   → revision 3  deployed   (2 becomes superseded)
 helm rollback web 2                  → revision 4  deployed   = a COPY of revision 2's settings
 helm uninstall web                   → all the release's objects AND its history deleted
```
**`deployed`** = the revision that's live now. **`superseded`** = an older revision that a newer one has replaced. **`failed`** = Helm knows that revision went wrong.

⚠️ **A rollback doesn't travel back in time.** It creates a **new revision** (4) that copies an old one (2), and revision 3 stays in the history. **Analogy:** restoring an old version of a shared document doesn't erase the later versions; it adds a new version on top that copies the old one. On your v4, `helm history` showed `4  deployed  Rollback to 2`.

| Command | What it does |
|---|---|
| `helm install <release> <chart>` | Create a release (revision 1). The chart can be a folder (`./mychart`), a packaged `.tgz` file, or `repo/chart` (`bitnami/nginx`) |
| `helm list` (short: `helm ls`) | Releases **in the current namespace**. `-n <ns>` for another one, `-A` for all. (`helm status <release>` = one release's status, revision and NOTES) |
| `helm upgrade <release> <chart>` | A new chart version or new values → a new revision |
| `helm upgrade --install <release> <chart>` | **Upgrade if it exists, install if it doesn't**: one command for the first deploy and every later one (pipelines use it, next session) |
| `helm history <release>` | Every revision with its STATUS and DESCRIPTION |
| `helm rollback <release> <revision>` | Go back to that revision's chart and values, as a new revision. No number = the previous revision |
| `helm uninstall <release>` | Delete the release's objects **and** its history. `--keep-history` keeps the records |
| `helm get values` / `helm get manifest <release>` | The values **you** supplied (`--all` = every value) / the YAML actually sent to the cluster |

### ⚠️ Trap 1: `--set` values are not remembered
Each `helm upgrade` starts again from the **chart's values.yaml plus only the flags you pass this time**, so a value you `--set` in an earlier upgrade is forgotten. Checked on your v4: after one upgrade with `--set image.tag=1.28-alpine`, the next upgrade with only `--set replicaCount=4` silently put the image back to `1.27`. **Fixes:**
- **Best:** keep your settings in a values file (`values-prod.yaml`) in Git, and pass `-f values-prod.yaml` **on every upgrade**.
- Or add **`--reuse-values`**: start from the last release's values, then apply the new flags on top.

### ⚠️ Trap 2: "deployed" doesn't mean "working"
By default Helm **sends the YAML and returns immediately**. It marks the revision `deployed` even if the new Pods then fail: an upgrade to a tag that doesn't exist still says `STATUS: deployed`, while one new Pod sits in `ImagePullBackOff`. The app keeps working only thanks to the **rolling update** (Session 06): Kubernetes keeps the old Pods until the new ones are Ready.
- **`--wait`** (with `--timeout 2m`) makes Helm **wait until the Pods are Ready**. If they aren't Ready in time, the revision is marked **`failed`** and the command exits with a **non-zero exit code**, so a pipeline goes red (next session).
- **`--rollback-on-failure`** (called `--atomic` in Helm 3) = if the upgrade fails, roll back automatically.

### Release names, namespaces and public charts
- A release name must be **unique within its namespace**: installing the same name twice gives `cannot reuse a name that is still in use`. Use lowercase letters, numbers and dashes.
- A release **lives in a namespace**. `-n helm-demo` (long form `--namespace`) chooses it, and **`--create-namespace`** creates it first if it's missing.
- ⚠️ **`helm list` with no flags shows only the current namespace** (usually `default`). "My release disappeared!" usually means it's in another namespace: use `helm list -n <ns>` or `helm list -A`.
- **Public charts:** `helm repo add bitnami https://charts.bitnami.com/bitnami` remembers a repo under a short name (yours says `already exists … skipping`), and `helm repo update` downloads each repo's latest chart list (like `apt update`).
- `helm search repo nginx` searches your repos (`helm search hub nginx` searches Artifact Hub). Its result `bitnami/nginx  25.2.1  1.31.6` = CHART VERSION, then APP VERSION. `helm show values bitnami/nginx` shows a chart's knobs **before** you install it.

### 🛠 Do (~20 min): deploy, upgrade, break, roll back, remove
First check that the chart has `replicaCount: 2` (capital C!) and `tag: "1.27"`, and that `helm lint mychart` passes.
```bash
cd ~/devops-practice/helm
helm install web ./mychart -n helm-demo --create-namespace   # STATUS: deployed, REVISION: 1, then the NOTES (NOTES.txt, filled in)
helm list -n helm-demo                    # web  helm-demo  1  …  deployed  mychart-0.1.0  1.16.0 (APP VERSION = Chart.yaml's claim only)
kubectl get deploy,pods,svc -n helm-demo  # web-mychart 2/2, two Pods Running, service/web-mychart ClusterIP 80/TCP
kubectl get secrets -n helm-demo          # sh.helm.release.v1.web.v1 = Helm's record of revision 1
kubectl port-forward -n helm-demo svc/web-mychart 8080:80   # in a 2nd tab (Cmd + T): Mac port 8080 → Service port 80
curl -I http://localhost:8080             # back in tab 1: HTTP/1.1 200 OK, Server: nginx/1.27.x
```
The Service is `ClusterIP`, which your Mac can't reach directly, so the port-forward opens a tunnel. It really connects to **one Pod**, so after each upgrade it breaks (`lost connection to pod`): press Ctrl + C and run it again.

**Upgrade, break, roll back:**
```bash
helm upgrade web ./mychart -n helm-demo --set image.tag=1.28-alpine --set replicaCount=3   # Happy Helming! REVISION: 2
kubectl get deploy web-mychart -n helm-demo -o wide   # -o wide adds the IMAGES column: 3/3, nginx:1.28-alpine
helm upgrade web ./mychart -n helm-demo --set image.tag=9.99-nope --set replicaCount=3    # a typo'd tag: STATUS: deployed, REVISION: 3 (!)
kubectl get pods -n helm-demo             # ~20 s later: 3 old Pods Running + 1 new Pod in ImagePullBackOff
helm history web -n helm-demo             # revision 3: deployed   Upgrade complete (Helm thinks all is well)
helm rollback web 2 -n helm-demo          # Rollback was a success! Happy Helming!
helm history web -n helm-demo             # revision 4: deployed   Rollback to 2; revisions 1–3 superseded
kubectl get pods -n helm-demo             # only 3 Running Pods remain
helm get values web -n helm-demo          # USER-SUPPLIED VALUES: image: tag: 1.28-alpine, replicaCount: 3
helm upgrade web ./mychart -n helm-demo --set replicaCount=4   # revision 5
kubectl get deploy web-mychart -n helm-demo -o wide   # 4/4, but the image is back to nginx:1.27: the --set trap
```
⚠️ Roll back to a revision you **know worked**. Revision 3 was also marked `deployed`, so "the latest deployed" isn't proof that it worked.

**A public chart (optional, ~3 min), then clean up** (Ctrl + C the port-forward first):
```bash
helm install my-nginx bitnami/nginx -n helm-demo --set service.type=ClusterIP
kubectl get pods -n helm-demo -l app.kubernetes.io/instance=my-nginx   # my-nginx-nginx-… 1/1 Running after ~30 s
helm uninstall my-nginx -n helm-demo      # release "my-nginx" uninstalled
helm uninstall web -n helm-demo           # release "web" uninstalled
helm history web -n helm-demo             # Error: release: not found (the history went too)
kubectl delete namespace helm-demo        # keep the ~/devops-practice/helm folder for revision
```
`-l app.kubernetes.io/instance=my-nginx` works because Helm charts label every object with the release name. Bitnami's free images have been limited since Aug 2025: if its Pod shows `ImagePullBackOff`, that's the likely cause.

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
**Kustomize** is a tool **built into kubectl** (`kubectl apply -k <folder>`) that customises YAML **without templates**. You keep a **base** folder of normal YAML, plus a small **overlay** folder per environment containing **patches**: just the lines that differ.

| | Plain `kubectl apply -f` | **Helm** | **Kustomize** |
|---|---|---|---|
| How you customise | copy and edit files | templates + values | base + overlays (patches), no templates |
| Packaging and sharing | none | charts and repos | none (just folders in Git) |
| Roll back everything together | no (`rollout undo` = one Deployment) | `helm rollback` (numbered revisions) | no (re-apply an older Git commit) |

| Item | What it is |
|---|---|
| `--dry-run=client` / `--dry-run=server` | Go through an install or upgrade **without changing anything**, and print the result. `server` also has the API server validate it |
| **Chart dependencies** | Listed in Chart.yaml under `dependencies:` (name, version, repository). **`helm dependency update mychart`** downloads them into `charts/` and writes **`Chart.lock`**, a file recording the exact versions used |
| `helm package mychart` | Bundles the chart folder into one file, **`mychart-0.1.0.tgz`** (`name` + `version`; `.tgz` = a `tar` archive compressed with gzip) |
| **OCI registries** | **OCI** = **Open Container Initiative**, the standard format of container registries. Charts can be stored next to images: `helm push mychart-0.1.0.tgz oci://ghcr.io/<you>/charts` |
| **Hooks** | Templates (usually Jobs) with the annotation `"helm.sh/hook": pre-install` (or `pre-upgrade`, …) that run **at a chosen moment**, e.g. a database migration before an upgrade |
| `helm test <release>` | Runs the chart's test Pods (`templates/tests/`). Pass = exit code 0 |
| **Where releases are stored** | As **Secrets** of type `helm.sh/release.v1`, named **`sh.helm.release.v1.<release>.v<revision>`**, in the release's namespace. Delete the namespace and the history goes too |
| `--history-max` | How many revisions Helm keeps per release (default **10**); older ones are deleted |
| `--reuse-values` / `--reset-values` | On upgrade: keep the last release's values and merge the new ones / go back to the chart's defaults (the normal behaviour) |
| **Helm v4 vs Helm 3** | `--atomic` is now `--rollback-on-failure`, and plain `--dry-run` is now `--dry-run=client`. Everything else in this session works the same |

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

### ✍️ Write the command
**C1.** Without sending anything to the cluster, print the Kubernetes YAML that the chart in the folder `./mychart` would produce for a release named `web`.

**C2.** Install the chart in `./mychart` as a release named `shop`, into a namespace `dev` that doesn't exist yet, using the settings in the file `values-dev.yaml`.

**C3.** Upgrade the release `web` with the chart in `./mychart`, changing the image tag to `1.28-alpine` and the replica count to `3` right on the command line.

**C4.** Revision 3 of the release `web` is broken. Take the release back to revision 2.
