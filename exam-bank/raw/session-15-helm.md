# Labs · Session 15 · Helm

> Helm = the package manager for Kubernetes (K8s). A **chart** = a folder of K8s YAML templates + default values. A **release** = one installed copy of a chart in the cluster. Every install/upgrade/rollback creates a new numbered **revision**.
> (Mac: `brew install helm`; use minikube as the cluster. Commands are identical.)

## Lab 1 · Create a Helm chart for a web app and explain Chart.yaml, values.yaml, templates/
**Answer (what to write in the exam):**
```bash
helm create web                  # scaffolds a ready-made nginx chart in folder ./web
ls -R web                        # (or: tree web) show the layout
cat web/Chart.yaml web/values.yaml
helm lint ./web                  # check the chart for mistakes
helm template web ./web          # render the final YAML locally, installs nothing
```
Layout and what each part is for:
```
web/
├── Chart.yaml          # chart metadata: name, chart version, app version
├── values.yaml         # DEFAULT settings that fill the template blanks
├── charts/             # sub-charts this chart depends on (empty)
└── templates/          # K8s YAML with {{ .Values.x }} blanks
    ├── deployment.yaml, service.yaml, serviceaccount.yaml, ingress.yaml, hpa.yaml
    ├── _helpers.tpl     # reusable snippets (names, labels); "_" = not rendered as a file
    ├── NOTES.txt        # text printed after install (how to reach the app)
    └── tests/test-connection.yaml   # run with: helm test web
```
`web/Chart.yaml`:
```yaml
apiVersion: v2            # chart format for Helm 3/4
name: web                 # chart name
description: A Helm chart for Kubernetes
type: application
version: 0.1.0            # CHART version: bump when templates/values change
appVersion: "1.16.0"      # version of the APP inside; default image tag
```
`web/values.yaml` (key defaults):
```yaml
replicaCount: 1
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""                 # empty means "use appVersion"
service:
  type: ClusterIP
  port: 80
```
`templates/deployment.yaml` (excerpt showing the blanks):
```yaml
  replicas: {{ .Values.replicaCount }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```
**What each line does:**
- `helm create web`: makes a working starter chart (Deployment + Service for nginx) so you don't write YAML from zero.
- `Chart.yaml`: the chart's ID card. `version` = chart version, `appVersion` = app version (they are different numbers), `apiVersion: v2` = Helm 3+ format.
- `values.yaml`: the default "settings"; users override them with `--set` or `-f my-values.yaml`.
- `templates/`: K8s manifests where `{{ .Values.x }}` is replaced by a value; `{{ .Chart.x }}` / `{{ .Release.Name }}` come from Chart.yaml and the release name.
- `helm lint`: checks syntax and structure. `helm template`: prints the rendered YAML without touching the cluster.

**Verify:** `helm lint ./web` → `1 chart(s) linted, 0 chart(s) failed`; `helm template web ./web | grep -E 'kind:|image:'` → `kind: Service`, `kind: Deployment`, `image: "nginx:1.16.0"`.

**🧠 In simple words:** A chart is a reusable template for an app's K8s objects. Chart.yaml describes the chart, values.yaml holds the default settings, and templates/ holds the YAML with blanks that Helm fills from those values. In short: templates + values = the final K8s YAML, and Chart.yaml says what and which version.

## Lab 2 · Deploy the chart with custom image, replicas and Service port; verify rendered and installed resources
**Answer (what to write in the exam):**
```bash
# 1) In templates/deployment.yaml change  containerPort: {{ .Values.service.port }}
#    to  containerPort: {{ .Values.containerPort }}  and add  containerPort: 80  to values.yaml
helm lint ./web
helm template web ./web --set image.tag=1.27 --set replicaCount=3 --set service.port=8080 \
  | grep -E 'replicas:|image:|port:'                  # check the rendered YAML first
helm install web ./web --set image.tag=1.27 --set replicaCount=3 --set service.port=8080
helm list                                             # release "web", REVISION 1, deployed
helm get values web                                   # only the values YOU overrode
helm get manifest web                                 # exact YAML Helm sent to the cluster
kubectl get all -l app.kubernetes.io/instance=web     # 3 pods, deploy, rs, svc
kubectl rollout status deployment/web
kubectl get endpoints web                             # 3 pod IPs on port 80
kubectl port-forward svc/web 8080:8080                # then in another tab: curl localhost:8080
```
Same thing with a values file (cleaner for many settings): `helm install web ./web -f prod-values.yaml`
```yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.27"
service:
  port: 8080
```
**What each line does:**
- Step 1 (gotcha): the starter chart reuses `service.port` as the container port, but nginx listens on 80; with 8080 the probes fail and pods restart. A separate `containerPort` value fixes it. The Service then maps port 8080 → `targetPort: http` (80).
- `--set key=value`: overrides one value from values.yaml (`image.tag` = nested key). `-f file.yaml` overrides many at once.
- `helm install <release> <chart>`: renders the templates and creates the objects; this is REVISION 1.
- `helm list`: shows releases (name, namespace, revision, status, chart, app version).
- `helm get values` / `helm get manifest`: what was set / what was actually applied.
- `-l app.kubernetes.io/instance=web`: label every chart object gets, so this lists everything from the release.

**Verify:** `helm list` → `web  default  1  deployed  web-0.1.0  1.16.0`; `kubectl get deploy web` → `READY 3/3`; `kubectl get svc web` → `PORT(S) 8080/TCP`; `helm get values web` → `USER-SUPPLIED VALUES:` listing image.tag 1.27, replicaCount 3, service.port 8080.

**🧠 In simple words:** The same chart can be installed with different settings without editing its files: you pass the image tag, replica count and port at install time. `helm template` shows the YAML before installing, and `helm get manifest` plus `kubectl get all` prove what is really running. In short: `--set`/`-f` customise, `template` previews, `install` deploys, `get`/`kubectl` verify.

## Lab 3 · Upgrade a release, inspect history, roll back, verify final status
**Answer (what to write in the exam):**
```bash
helm upgrade web ./web --reuse-values --set image.tag=1.28   # REVISION 2
kubectl rollout status deployment/web
helm history web                                              # 1 superseded, 2 deployed
helm rollback web 1                                           # go back to rev 1 → creates REVISION 3
kubectl rollout status deployment/web
helm history web                                              # 3 deployed "Rollback to 1"
helm status web                                               # STATUS: deployed, REVISION: 3
helm get values web                                           # image.tag is 1.27 again
kubectl get deploy web -o jsonpath='{.spec.template.spec.containers[0].image}'   # nginx:1.27
```
Expected `helm history web` after the rollback:
```
REVISION  STATUS      CHART      APP VERSION  DESCRIPTION
1         superseded  web-0.1.0  1.16.0       Install complete
2         superseded  web-0.1.0  1.16.0       Upgrade complete
3         deployed    web-0.1.0  1.16.0       Rollback to 1
```
**What each line does:**
- `helm upgrade <release> <chart>`: applies new values/chart version to the existing release; makes a new revision. `--reuse-values` keeps the old `--set` values (without it, replicas and port reset to the chart defaults).
- `helm history web`: table of all revisions with status and description.
- `helm rollback web 1`: re-applies revision 1's config. It does NOT delete revision 2; it writes a NEW revision 3 that copies revision 1. (`helm rollback web` with no number = the previous revision.)
- `helm status web`: current state, revision number, last deploy time, and NOTES.txt.
- `helm get values` + `kubectl get deploy … jsonpath`: prove the image really went back to 1.27.

**Verify:** `helm status web` → `STATUS: deployed`, `REVISION: 3`; `kubectl get pods -l app.kubernetes.io/instance=web` → 3 Running pods with image `nginx:1.27`.

**🧠 In simple words:** Helm keeps a numbered record of every change to a release. An upgrade adds a revision; if it goes wrong, `helm rollback` puts back an older revision's settings as a brand-new revision, so history is never lost. In short: upgrade → history → rollback (new revision number) → status to confirm.
