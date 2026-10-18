# Answer key · Session 11 · Helm · FAST

⚠️ Only open after answering.

---

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

**Correct: C.** Work through the layers from lowest to highest priority. The chart's values.yaml gives 1 replica and `nginx:1.27`. Then `b.yaml` sets 4 replicas and tag `1.28`. Then `a.yaml` (the later `-f` file, so it beats `b.yaml`) sets 2 replicas but says nothing about the image, so the tag stays `1.28` because merging is key by key. Finally `--set replicaCount=3` beats every `-f` file, even though it was typed first, so the result is `replicas: 3` and `image: "nginx:1.28"`. A is the near-miss: it correctly lets the later file `a.yaml` beat `b.yaml`, but forgets that `--set` wins over all files whatever its position on the line. B assumes the later file `a.yaml` replaces everything before it, which would wipe out `b.yaml`'s tag, but merging keeps every key a later file doesn't mention. D lets the first `-f` file win and ignores both `a.yaml` and `--set`.

---

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

**Correct: B.** Count the revisions: the install is revision 1 (tag `1.28`), the bad upgrade is revision 2, the rollback creates a new revision 3 that copies revision 1's chart and values (tag `1.28` again), and the last upgrade is revision 4. That last upgrade starts again from the chart's values.yaml plus only its own flag (`replicaCount=3`), so the earlier `--set image.tag=1.28` is forgotten and the image falls back to the default `nginx:1.27`. A gets the revision number right but assumes Helm remembers earlier `--set` values (it would only with `--reuse-values` or a values file passed again). C has the right image but treats the rollback as going back in time without adding a revision. D imagines the rollback deleting revision 2 and also keeps the forgotten tag.

---

**3.** A teammate makes four claims about Helm 3 and later. Which one is **NOT** true?
- A) One chart installed as `a` and as `b` gives two releases
- B) Each revision is kept as a Secret in the release namespace
- C) Artifact Hub lists charts, but the charts live in repos
- D) Helm can deploy things that your own kubectl user cannot

**Correct: D.** Helm 3 and later are client-only: the `helm` program talks to the API (Application Programming Interface) server using your own kubeconfig, so it has exactly the same permissions you have with kubectl and nothing more. The idea that Helm has extra power comes from Helm 2's Tiller, the in-cluster server with admin rights, which no longer exists. A is true: a chart is the package and each install with a new name is a separate release, just as one image can run as two containers. B is true: Helm stores each revision's record as a Secret (named like `sh.helm.release.v1.a.v1`) in the release's own namespace; it is tempting to doubt it only if you remember Helm 2, which kept records elsewhere. C is true: Artifact Hub is only a search website, and the charts themselves are stored in chart repositories.

---

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

**Correct: A.** A label in a template changes the chart (the packaging), not the app inside, so only the chart's `version` goes up (0.4.0 to 0.4.1, a patch-level change in SemVer). The image tag is empty, so `default .Chart.AppVersion` fills in `appVersion`, which is still `2.1`, giving `shop-api:2.1`. B bumps `appVersion`, which describes the app, and the app did not change; that is the classic version/appVersion mix-up. C gets the bump right but then uses the chart's `version` as the image tag, while the template actually falls back to `.Chart.AppVersion`. D touches `apiVersion`, which is the chart format (`v2` for Helm 3 and later) and is never bumped for your own changes.

---

**5.** An engineer runs `helm upgrade web ./chart -n prod --set image.tag=9.99-nope` (that tag does not exist, and no other flags are used). Helm prints `STATUS: deployed` and `REVISION: 3`. What is the actual state about a minute later?
- A) Helm rolled back on its own to revision 2 when the pull failed
- B) Revision 3 shows `failed` in history, as its Pods never got Ready
- C) All Pods were replaced at once, so the app is down for every user
- D) A new Pod is in ImagePullBackOff while the old Pods keep serving

**Correct: D.** Without `--wait`, Helm sends the YAML and returns at once, so it marks revision 3 `deployed` without checking the Pods. The new Pod cannot download the non-existent image and sits in `ImagePullBackOff`, but the rolling update (replacing Pods a few at a time) keeps the old Pods running until new ones are Ready, so users are still served. A describes `--rollback-on-failure` (called `--atomic` in Helm 3), which was not used. B describes what `--wait --timeout` would do: wait for Ready Pods and mark the revision `failed`; without it, history keeps saying `deployed`. C ignores the rolling update: Kubernetes does not kill the old Pods before the new ones are Ready.

---
