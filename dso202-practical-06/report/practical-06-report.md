# DSO202 - Practical 6 Report

**Helm: Charts, Templates, Values and the Release Lifecycle**

| Field | Detail |
| --- | --- |
| Module | DSO202 - Scaling, Orchestration, Monitoring & Observability |
| Programme | BE in Software Engineering |
| Practical | 6 of 10 |
| Tool | Helm v4.3.0 on kind (Kubernetes IN Docker) |
| Date carried out | 5 October 2026 |
| Repository path | `dso202-practical-06/` |

All screenshots referenced below are in `evidence/`, numbered in the order they
were captured. A suffix `b` marks a frame captured from an unplanned event.

---

## 1. Objective

Practical 5 used Kustomize to produce environment variants of manifests that
a team writes itself. Kustomize left two questions unanswered. Nothing in the
cluster recorded *which version of the application* was installed, and
nothing could remove or roll back a group of objects as one unit. Helm answers
both. A **chart** packages templates and default configuration into one
versioned archive. Each installation is a **release**, and every change to a
release is recorded as a numbered **revision** that can be inspected, rolled
back and uninstalled.

The practical followed Demo Stages 0 to 5 of Unit III 3.2, and then a Stage 6
that exercises the release-lifecycle behaviour described in 3.2.3:

- **Stage 0:** installing Helm 4 and confirming it reaches the cluster;
- **Stage 1:** operating a published chart (podinfo) through its whole
  lifecycle: search, read, install, inspect, upgrade, roll back, uninstall,
  and install from an OCI registry;
- **Stages 2–3:** writing a chart, `webapp`, file by file;
- **Stage 4:** how values from several sources are merged;
- **Stage 5:** four deliberate failures, and which validation tool catches
  each one;
- **Stage 6:** installing the chart for dev and prod, running its tests, and
  observing Helm 4's server-side apply refuse both a drifted field and an
  object Helm does not own.

**Descriptor sections covered:** Unit III - 3.2.1, 3.2.2, 3.2.3.
**Learning outcome:** LO8, and the Final Project item "Create Helm charts for
deploying the application to Kubernetes".

---

## 2. Environment

| Component | Version |
| --- | --- |
| Operating system | macOS 26.6.2 (build 25G83), arm64 |
| Docker Desktop | 29.7.2 |
| kind | v0.32.0 (go1.26.3 darwin/arm64) |
| kubectl | v1.36.3 |
| **Helm** | **v4.3.0** (commit `bec5b06`, KubeClientVersion v1.37) |
| Kubernetes (cluster) | v1.36.1 - `kindest/node:v1.36.1` |
| Node OS / kernel | Debian GNU/Linux 13 (trixie), 7.0.12-linuxkit (arm64) |

### Container images and charts

| Artifact | Used by |
| --- | --- |
| Chart `podinfo/podinfo` 6.15.0 (HTTP repository) | Stage 1 |
| Chart `oci://ghcr.io/stefanprodan/charts/podinfo` 6.15.0 | Stage 1, OCI install |
| Image `ghcr.io/stefanprodan/podinfo:6.15.0` | podinfo Pods |
| Image `nginx:1.30-alpine` | `webapp` chart (from `appVersion`) |
| Image `busybox:1.37` | `webapp` chart test Pod |

### Upgrading from Helm 3

The Homebrew Helm on this machine was **v3.19.0**. This practical depends on
Helm 4 behaviour: server-side apply, the `apply_method` field in the release
record, field-manager conflicts, and the `--rollback-on-failure` and
`--force-conflicts` flags. Helm was therefore upgraded with
`brew upgrade helm` before Stage 0. Helm 3's final feature release was on
9 September 2026, and its security patches end in February 2027.

As in the earlier practicals, `/opt/homebrew/bin` was put first on `PATH` at
the start of each session, so that the Homebrew `kubectl` and `helm` were the
binaries in use.

### Differences from the handout's environment

| Handout (Linux user `student`) | Observed on macOS |
| --- | --- |
| `HELM_REPOSITORY_CONFIG=/home/student/.config/helm/repositories.yaml` | `/Users/keldendrac/Library/Preferences/helm/repositories.yaml` |
| `HELM_REPOSITORY_CACHE=/home/student/.cache/helm/repository` | `/Users/keldendrac/Library/Caches/helm/repository` |
| `HELM_REGISTRY_CONFIG=/home/student/.config/helm/registry/config.json` | `/Users/keldendrac/Library/Preferences/helm/registry/config.json` |
| Helm installed with the `get-helm-4` script | Installed with Homebrew |

Helm follows each operating system's conventions for configuration and cache
directories. The variables mean the same thing on both.

### Repository layout

The handout's lab directory `dso202-helm-lab/` is this repository's
`dso202-practical-06/`. Inside it, `webapp/` is the chart and `environments/`
holds the per-environment values files, as the handout specifies. `site/`
(the Stage 8 umbrella chart) is outside the scope of this practical. Two
additions keep Stage 5 reproducible without editing the chart by hand:

- `checks/` holds the Failure 3 input (`tag-float.yaml`) and
  `values.schema.json`, which was copied into the chart at the point where
  the handout introduces it.
- `outputs/` holds renders and the deliberately broken chart copies used for
  Failures 2 and 4. It is excluded from Git.

---

## 3. Procedure and Observations

### 3.1 Stage 0 - Installing Helm and preparing the cluster

The three-node cluster from Practical 1 (Listing 1) was created from
`cluster/kind-cluster.yaml`. It maps host port 30080 to the cluster, which
the dev release uses in Stage 6. A deterministic wait,
`kubectl wait --for=condition=Ready nodes --all`, preceded every inspection.

**Evidence: `01-stage0-tooling-and-cluster`.** Each tool was checked on its
own, so that any later failure could be traced to one cause:

- `helm version` reports `v4.3.0`, commit `bec5b06ed841…`. This is exactly
  the build the handout was verified against. `KubeClientVersion: "v1.37"` is
  the client library Helm was built with; it is not the cluster version.
- The current context is `kind-dso202`. `control-plane`, `worker-node-1` and
  `worker-node-2` are all `Ready` on v1.36.1.
- The APT check prints `no baltocdn source found`. There is no APT on macOS,
  so no package source can point at the retired mirror.
- `helm env` reports `HELM_MAX_HISTORY="10"` (at most ten revisions are kept
  per release) and `HELM_NAMESPACE="default"`, along with the three
  configuration paths noted in §2.
- `helm list -A` prints only its headings. Helm reached the API server, and no
  release existed yet.

### 3.2 Stage 1 - Consuming a published chart

**Reading before installing. Evidence: `02-stage1-search-and-read-chart`.**
After `helm repo add` and `helm repo update`, the search shows **two version
columns**: `CHART VERSION` is the version of the package and `APP VERSION` is
the version of the software inside it. For podinfo they happen to be equal
(6.15.0, 6.14.1, 6.14.0), but they are independent fields.

`helm show chart` reveals `apiVersion: v1`, the **legacy Helm 2 chart
format**, which Helm 4 still installs. It also shows `kubeVersion: '>=1.23.0-0'`.
`helm show values` shows the configuration interface: `replicaCount: 1`, the
image `ghcr.io/stefanprodan/podinfo:6.15.0`, and `ui.color`/`ui.message`. The
same grep also matched a `docker.io/redis` / `8.8.0` pair. podinfo can
optionally deploy a Redis cache, which is the kind of thing reading a chart
before installing it is meant to reveal.

**An unplanned failure. Evidence: `02b-stage1-install-timeout`.** The first
install failed:

```
Error: INSTALLATION FAILED: resource Deployment/dso202-helm/my-podinfo not ready.
status: InProgress, message: Available: 0/2
context deadline exceeded
```

The chart was not at fault (diagnosis in §5, error 1). The image was still
downloading when `--timeout 3m` expired. After the pull completed, the failed
release was uninstalled and installed again, so that revision 1 of the
release analysed below is a clean `Install complete`.

**The installed release. Evidence: `03-stage1-release-installed`.**
`helm list -n dso202-helm` shows `my-podinfo` at revision 1, `deployed`, chart
`podinfo-6.15.0`. Plain `helm list` prints only headings. A release belongs to
a namespace, and the current namespace was `default`. The Deployment is
`2/2`, as `--set replicaCount=2` requested. The Service `my-podinfo` exposes
`9898/TCP,9999/TCP`.

Through a port-forward, `curl` returned `"version": "6.15.0"` and
`"message": "Hello from DSO202"`. The `--set` value reached the running
application. `helm get values` returned only the two user-supplied values.
`helm get manifest` listed the two objects with their `# Source:`
templates, `service.yaml` and `deployment.yaml`. The manifest is exactly what
Helm sent to the API server, read back from the release record and not from
the live objects.

**The release record. Evidence: `04-stage1-release-record`.** `helm status`
shows `REVISION: 1`, `DESCRIPTION: Install complete`. The record itself is a
Secret, `sh.helm.release.v1.my-podinfo.v1`, of type `helm.sh/release.v1`.
Its labels (`name=my-podinfo,owner=helm,status=deployed,version=1`) let a
release's history be found with a label selector.

Decoding it through its three layers took one pipeline: base64 (Kubernetes),
then base64 (Helm), then gzip:

```json
{"name":"my-podinfo","namespace":"dso202-helm","version":1,"status":"deployed",
 "apply_method":"ssa","values":{"replicaCount":2,"ui":{"message":"Hello from DSO202"}}}
```

Two facts follow. `apply_method: "ssa"` records that Helm 4 created this
release with **server-side apply**. And the user's values are stored in the
cluster in a reversible encoding. Anyone who can read Secrets in
`dso202-helm` can read any value ever passed to this release, including a
password (§4, Q2).

**The values-loss failure. Evidence: `05-stage1-values-loss-and-reuse`.** An
upgrade that supplied only `--set ui.color="#2e7d32"` succeeded, and silently
discarded the install-time values:

| | User values | Replicas |
| --- | --- | --- |
| After the partial upgrade | `{"ui":{"color":"#2e7d32"}}` | **1** |
| After `--reuse-values` + both original values | `{"replicaCount":2,"ui":{"color":"#2e7d32","message":"Hello from DSO202"}}` | 2 |

Production capacity halved without an error. When `helm upgrade` is given
*any* value flag, it starts again from the chart defaults. The defence is to
pass every values file on every upgrade (Stage 4), or to use
`--reset-then-reuse-values`.

**History and rollback. Evidence: `06-stage1-rollback-history`.**
`helm rollback my-podinfo 1` reported `Rollback was a success!`:

| Revision | Status | Description |
| --- | --- | --- |
| 1 | superseded | Install complete |
| 2 | superseded | Upgrade complete (values lost) |
| 3 | superseded | Upgrade complete (`--reuse-values`) |
| 4 | **deployed** | **Rollback to 1** |

The rollback did not return the release to revision 1. It created revision 4,
a copy of revision 1's chart and values, and kept revisions 2 and 3, so the
history records what actually happened, in order. The values after rollback
are revision 1's: `replicaCount: 2` and the message, with no colour.

**Uninstall and OCI. Evidence: `07-stage1-uninstall-and-oci`.** After
`helm uninstall`, `kubectl get all,secrets -n dso202-helm` returned
`No resources found`. Every object *and every release record* was gone. The
namespace was still `Active` (age 12m): `--create-namespace` creates a
namespace without making it part of the release.

The same chart was then read and installed from
`oci://ghcr.io/stefanprodan/charts/podinfo`, with no `helm repo add` or
`helm repo update`. Helm printed what it pulled:

```
Pulled: ghcr.io/stefanprodan/charts/podinfo:6.15.0
Digest: sha256:ff3d3e14728f75476ed4d43c14f80d52d81d36bc16906843463d464c6146f0d8
```

The tag `6.15.0` resolved to one immutable **digest**. The release installed
as revision 1, `deployed`, and was uninstalled again.

### 3.3 Stage 2 - Chart structure

**Evidence: `08-stage2-scaffold-and-webapp-chart`.** `helm create scaffold`
generated the maintainers' reference layout: twelve files, including
`_helpers.tpl`, `hpa.yaml`, `ingress.yaml`, `serviceaccount.yaml`,
`tests/test-connection.yaml` and, new in the Helm 4 scaffold,
**`httproute.yaml`** for the Gateway API. The scaffold was read and removed.

The `webapp` chart was then written file by file, so that every line could be
explained. The listing shows the chart (`.helmignore`, `Chart.yaml`,
`values.yaml`, five templates and one test) beside `environments/dev.yaml`
and `environments/prod.yaml`. The environment files sit **outside** the chart
because they belong to a deployment of the chart, not to the chart itself.
`Chart.yaml` uses `apiVersion: v2` and separates `version: 0.1.0` (the
package) from `appVersion: "1.30-alpine"` (the software). The quotes keep
YAML from turning the version into a number (Stage 5).
`kubeVersion: ">=1.30.0-0"` refuses older clusters, and its `-0` also admits
pre-release builds.

### 3.4 Stage 3 - Rendering the templates

`helm template webapp-dev ./webapp -f environments/dev.yaml -n dso202-dev`
ran every rendering step of an install, without contacting the cluster.

**Evidence: `09-stage3-render-dev`.** The render produced four objects, in
this order:

```
# Source: webapp/templates/configmap.yaml         kind: ConfigMap
# Source: webapp/templates/service.yaml           kind: Service   (NodePort, 30080)
# Source: webapp/templates/deployment.yaml        kind: Deployment (replicas: 1)
# Source: webapp/templates/tests/test-connection.yaml  kind: Pod
```

- **Order follows kind, not file name.** Helm sorts objects into a fixed
  install order, so the ConfigMap exists before the Deployment that mounts
  it.
- **The checksum matches the handout exactly:**
  `checksum/config: 534e2ce7205f6f1d2d13203ea2dbd1ec7235d260e43331302152c76034b4f88b`.
  It is a SHA-256 hash of the rendered ConfigMap, so identical inputs give an
  identical hash on any machine. It is placed on the Pod template so that a
  change to the page content changes the template, and therefore triggers a
  rolling update.
- **The image tag fell back to `appVersion`.** `image.tag` is empty, so
  `default .Chart.AppVersion` produced `nginx:1.30-alpine`.
- **The `nodePort` line appeared** because the type is `NodePort` *and* a port
  was given. Both conditions are in one `if and` in `service.yaml`.

The ConfigMap, rendered alone with `-s templates/configmap.yaml`, shows the
built-in objects at work. The labels carry `helm.sh/chart: webapp-0.1.0`,
`app.kubernetes.io/instance: webapp-dev`, `app.kubernetes.io/managed-by: Helm`
and `dso202/environment: "dev"`. The page says
`Release: webapp-dev in namespace dso202-dev` and
`Chart: webapp-0.1.0, application version 1.30-alpine`. The object is named
`webapp-dev` rather than `webapp-dev-webapp`, because `webapp.fullname`
detected that the release name already contains the chart name.

### 3.5 Stage 4 - Values: precedence and merging

**Evidence: `10-stage4-values-precedence`.** Every output matched the
handout's prediction:

| Command | Output | Rule demonstrated |
| --- | --- | --- |
| `-f dev.yaml --set replicaCount=2` | `replicas: 2` | `--set` overrides every `-f` file |
| `-f dev.yaml -f prod.yaml` (Service) | `type: NodePort`, `nodePort: 30080` | prod.yaml never mentions `service`, so dev's NodePort **leaks into prod** |
| `-f dev.yaml -f prod.yaml` (Deployment) | `replicas: 3`, environment `"prod"` | the rightmost file wins for keys both set |
| `-f prod.yaml -f dev.yaml` | `replicas: 1`, environment `"dev"`, `cpu: 500m`/`100m` | dev.yaml never mentions `resources`, so **prod's CPU leaks into dev** |
| `--set 'podAnnotations.prometheus\.io/scrape=true'` | `prometheus.io/scrape: true` | `--set` converts `true` to a boolean |
| `--set-string …=true` | `prometheus.io/scrape: "true"` | `--set-string` keeps a string |
| `--set resources.limits=null` | only `requests` remain | `null` deletes a key, even a chart default |

The two leak rows are the important result. **Layering environment files is
not switching environments.** Maps merge deeply, so a key the later file does
not mention keeps the earlier file's value. Each environment file must
therefore be applied on its own over the chart defaults. The boolean
annotation renders without complaint, but the API server rejects it at
install time because annotation values must be strings. Rendering success is
not deployment success, which leads into Stage 5.

### 3.6 Stage 5 - Validating charts before installation

Stage 5 is four deliberate failures. **Every error in screenshots 11–14 is the
intended result:** each one shows a check catching, or failing to catch, a
specific class of mistake. Failures 2 and 4 were made in copies of the chart
under `outputs/`, so the real chart was never edited and never needed
restoring. Its two rendered environments were confirmed clean at the end of
the stage (`14`).

**Failure 1 - missing required value. Evidence: `11-stage5-required-value-and-lint`.**
Rendering with no environment file stopped with the chart author's own
message:

```
Error: execution error at (webapp/templates/tests/test-connection.yaml:10:8):
page.environment must be set (dev, staging or prod)
```

The location is the test Pod, the first template to include the helper, not
`_helpers.tpl` itself. The *message*, not the location, identifies the cause.
`helm lint` on the same chart printed the problem only as
`level=WARN msg="missing required values"`, ten times (once for each
evaluation of the helper; see §5), and then reported `1 chart(s) linted, 0 chart(s) failed`
with **`exit code: 0`**. `--strict` made no difference: also `exit code: 0`.
**A CI pipeline relying on `helm lint` alone would pass a chart that cannot
render.**

**Failure 2 - indentation. Evidence: `12-stage5-indentation-and-float-tag`.**
Line 48 of the broken copy reads `{{ toYaml .Values.resources }}`, without
`nindent`. Rendering failed with
`YAML parse error … line 59: mapping values are not allowed in this context`.
`--debug` printed the invalid YAML and showed the cause. Only the first line of
the `toYaml` output inherited the template's indentation; `cpu: 200m`,
`memory: 64Mi` and `requests:` fell back towards column 0. `toYaml` must
always be followed by `| nindent N`.

**Failure 3 - a number where a string was meant (same frame).**
`checks/tag-float.yaml` sets `image.tag: 1.30` without quotes. The render
**succeeded** and produced `image: "nginx:1.3"`. YAML parsed `1.30` as the
number 1.3, and nothing warned. This is the most dangerous failure in the
stage, because the error would only appear in the cluster: as
`ImagePullBackOff`, or, worse, as a successful pull of a different image.

**The fix: a values schema. Evidence: `13-stage5-values-schema`.** With
`values.schema.json` copied into the chart, the same input became an
immediate, precise error, before any template was rendered:

```
- at '/image/tag': got number, want string
```

`--set replicaCount=three` gave `got string, want integer`. A single command
with three bad values got all three reported at once:

```
- at '/service/type': value must be one of 'ClusterIP', 'NodePort'
- at '/page/environment': value must be one of '', 'dev', 'staging', 'prod'
- at '/replicaCount': minimum: got 0, want 1
```

The handout lists the same three violations in a different order. The content
is identical. The schema enforces type and permitted values, while `required`
still enforces presence. `page.environment` accepts `""` in the schema so
that a parent chart's `global.environment` can supply it (3.2.5).

**Failure 4 - a misspelled Kubernetes field. Evidence: `14-stage5-unknown-field-and-ci-loop`.**
The broken copy reads `replica: {{ .Values.replicaCount }}` (line 11). Each
check was run in turn:

| Check | Result |
| --- | --- |
| `helm lint` | `1 chart(s) linted, 0 chart(s) failed`: passed |
| `helm install --dry-run=server` | `STATUS: pending-install`, `DESCRIPTION: Dry run complete`: passed |
| `helm template … \| kubectl apply --dry-run=server` | ConfigMap and Service `created (server dry run)`, then **`strict decoding error: unknown field "spec.replica"`** |

Only the last check, which sends each object to the API server with strict
field validation, caught it. The other two objects passed their dry runs
first. The same pattern in a real install would leave earlier objects applied
before the Deployment was rejected.

**The CI loop (same frame).** The corrected chart passed `helm lint` and
`helm template` for every environment: `dev: ok`, `prod: ok`. The `[INFO]`
line about a missing icon is a recommendation and does not fail the lint.

### 3.7 Stage 6 - Deploying the chart, testing it, and drift

**Install and test. Evidence: `15-stage6-install-and-helm-test`.** Both
releases were installed with `helm upgrade --install … --wait`, which is the
idempotent form used in CI pipelines. `helm list -A` shows `webapp-dev` in
`dso202-dev` and `webapp-prod` in `dso202-prod`, both revision 1, `deployed`,
chart `webapp-0.1.0`, app version `1.30-alpine`. **One chart, two releases,
two environments.**

`curl http://localhost:30080` reached the dev release directly from the Mac,
through the NodePort and kind's port mapping:

```
<h1>DSO202 Helm Demo</h1>
<p>Environment: dev</p>
<p>Release: webapp-dev in namespace dso202-dev</p>
<p>Chart: webapp-0.1.0, application version 1.30-alpine</p>
```

`helm test` ran the chart's test Pod in each namespace. `webapp-dev-test-connection`
and `webapp-prod-test-connection` both reached `Phase: Succeeded`, taking
about 9 seconds each. The test fetches the page through the Service and greps
for that release's own environment, so it proves more than "the Pod is
running". It proves the right content is served behind the right Service in
the right environment.

**Drift and server-side apply. Evidence: `16-stage6-drift-and-ownership`.**
The prod Deployment was changed outside Helm with
`kubectl scale … --replicas=5`. The next ordinary upgrade **refused to
proceed**:

```
Error: UPGRADE FAILED: conflict occurred while applying object
dso202-prod/webapp-prod apps/v1, Kind=Deployment: Apply failed with 1 conflict:
conflict with "kubectl" with subresource "scale" using apps/v1: .spec.replicas
```

The API server records a **field manager** for every field. `kubectl scale`
had become the manager of `.spec.replicas` through the `scale` subresource,
and Helm's server-side apply would not silently overwrite another manager's
field. Helm 3's client-side apply would have reset the replicas to 3 without
a word. Re-running the upgrade with `--force-conflicts` took ownership back,
and the replicas returned to `3`.

`helm history webapp-prod` recorded all of it: revision 1 `superseded`,
revision 2 **`failed`** with the full conflict message as its description,
and revision 3 `deployed`, `Upgrade complete`. A failed upgrade is part of
the history, not a gap in it.

**Ownership refusal (same frame).** A ConfigMap named `webapp-qa` was created
by hand in `dso202-dev`, and a release `webapp-qa` was then installed whose
chart would create an object of that same name. Helm refused before creating
anything:

```
Error: INSTALLATION FAILED: unable to continue with install: ConfigMap "webapp-qa"
in namespace "dso202-dev" exists and cannot be imported into the current release:
invalid ownership metadata; label validation error: missing key
"app.kubernetes.io/managed-by": must be set to "Helm"; annotation validation error:
missing key "meta.helm.sh/release-name": must be set to "webapp-qa"; annotation
validation error: missing key "meta.helm.sh/release-namespace": must be set to "dso202-dev"
```

The message names exactly the three markers Helm adds to every object it
creates: the label `app.kubernetes.io/managed-by: Helm` and the annotations
`meta.helm.sh/release-name` and `meta.helm.sh/release-namespace`. Without
them, Helm treats the object as someone else's and will not take it over.
Doing so deliberately requires `--take-ownership`.

### 3.8 Cleanup

Cleanup is left until after this report is final, because it destroys the
releases the screenshots describe. The procedure in the README uninstalls
`webapp-dev` and `webapp-prod`, then deletes the three namespaces. Deleting
`dso202-dev` also removes the hand-made `webapp-qa` ConfigMap from §3.7, which
no release owns and `helm uninstall` would therefore never remove. Finally it
removes the `podinfo` repository alias and deletes the kind cluster. No
screenshot of cleanup is submitted.

---

## 4. Analysis

The handout's review questions are answered below, grouped by section, with
evidence from this practical where it exists.

### 3.2.1 - Charts, releases and repositories

**Q1. Why does `kubectl apply -f` leave an orphaned object behind when it is
deleted from a manifest file, and how does a release record avoid this?**

`kubectl apply -f` only knows the files it is given in that invocation. An
object deleted from the files is simply not mentioned, and nothing in
`apply` remembers that it was created earlier. So it stays in the cluster,
managed by nothing. Practical 5 saw the same effect with the old generated
ConfigMaps.

A release record stores the **complete manifest of every revision**
(`helm get manifest`, `03`). On upgrade, Helm compares the previous revision's
object list with the new one and deletes objects that have disappeared. On
uninstall it deletes everything in the record. `07` shows the result:
`No resources found` after `helm uninstall`.

**Q2. A password is stored in a values file and the chart is installed.
Identify two places where the password can now be read.**

1. **The release record in the cluster.** `04` decoded
   `sh.helm.release.v1.my-podinfo.v1` with `base64 -d | base64 -d | gunzip`
   and printed the user values in clear. Anyone allowed to read Secrets in
   that namespace can do the same, or simply run `helm get values`.
2. **The values file in version control**, and with it every clone, fork and
   CI checkout of the repository. If it is passed with `--set` instead, it
   also lands in shell history and CI logs.

A third place is the rendered object itself. If the chart puts the password
in a Secret, it is base64-*encoded* there, not encrypted.

**Q3. One situation where Kustomize is more suitable, and one where Helm is.**

Kustomize suits a team's **own** manifests deployed to its own environments,
exactly as in Practical 5. The base stays plain, valid YAML that anyone can
read, and overlays add small, reviewable differences. Helm suits **packaging
software for others** and installing third-party software (podinfo, here).
The consumer needs a versioned artifact with a documented configuration
interface (`values.yaml`), and the operator needs install, upgrade, rollback
and uninstall as a unit.

**Q4. The Secret storing revision 7 of release `payments` in namespace
`finance`.**

`sh.helm.release.v1.payments.v7`, in namespace `finance`, following the
pattern `sh.helm.release.v1.<release>.v<revision>` (`04`).

**Q5. Why did removing Tiller improve Helm's security?**

Tiller ran inside the cluster with broad, usually cluster-admin, permissions,
and acted for whoever could reach it. That bypassed per-user RBAC: a user
with no right to create Deployments could create them through Tiller. Since
Helm 3 the client applies objects with **the user's own kubeconfig
credentials** (step 6 of the install sequence), so RBAC applies to Helm
exactly as it applies to `kubectl`, and nothing Helm-specific runs in the
cluster to attack.

**Q6. A release `web` exists in `team-a` and `team-b`. Why is that permitted,
and which command lists both?**

Releases are namespaced. Each release record is a Secret **in its own
namespace**, so `sh.helm.release.v1.web.v1` can exist in both without
collision. `helm list -A`, or `--all-namespaces`, lists both. `03` shows the
namespacing directly: plain `helm list` showed nothing, because the release
was not in `default`.

**Q7. After `helm upgrade r c --set a=1` following an install that set `b=2`,
what is `b`, and which flag would have preserved it?**

`b` is back to the **chart default** (or absent): the install-time value is
discarded. This is exactly what `05` showed, where `replicaCount: 2` became
`1`. `--reuse-values` would have kept it. `--reset-then-reuse-values` would
also keep it, and is the safer choice when the chart version changes too,
because it starts from the new chart's defaults.

**Q8. Why does `helm rollback r 1` produce revision 4 rather than a return to
revision 1?**

Revisions are an append-only log. A rollback is itself an operation, so it
gets the next number and records a copy of revision 1's chart and values.
Keeping revisions 2 and 3 preserves the full story: what was deployed, when,
and what was undone (`06`, `Rollback to 1`).

**Q9. Two reasons to prefer an OCI registry to an HTTP chart repository.**

1. **One system for images and charts.** Organisations already run a
   container registry with access control, retention and replication, and
   charts reuse all of it. There is no separate `index.yaml` web server to
   operate.
2. **Content addressing.** Each chart version is stored by digest (`07`:
   `sha256:ff3d3e…`). A deployment can pin the digest, which no one can move,
   whereas an HTTP repository's `index.yaml` can be rewritten. OCI also needs
   no `helm repo add` or `update`, so there is no stale local index.

### 3.2.1.5 - Chart structure

**Q10. A chart changes only its default replica count from 1 to 2. Which
field must change, and to what, from 1.4.2?**

`version`, the **chart** version, because a file in the chart changed.
`appVersion` stays the same because the application did not change. A
default that changes runtime behaviour in a backwards-compatible way is a
MINOR change: **1.5.0**. Reusing 1.4.2 would make two different packages
share one version.

**Q11. Why does `templates/_helpers.tpl` never produce a Kubernetes object?**

Files in `templates/` whose names begin with `_` are not rendered as
manifests. They are only parsed, so that the `define` blocks inside them
become named templates available to the other files. `09` shows four objects
from five template files: `_helpers.tpl` produced none.

**Q12. The purpose of `-0` in `kubeVersion: ">=1.30.0-0"`.**

In SemVer, a pre-release such as `1.36.0-rc.1` sorts **below** `1.36.0`, and
a plain constraint like `>=1.30.0` excludes all pre-releases. Managed
Kubernetes services often report versions with suffixes, such as
`v1.36.1-eks-…`. The `-0` lowers the floor to the smallest possible
pre-release, so those clusters satisfy the constraint.

### 3.2.2 - Templates and values

**Q13. Rewrite `{{ quote (upper .Values.name) }}` as a pipeline.**

`{{ .Values.name | upper | quote }}`

**Q14. Inside `{{- range .Values.hosts }}`, why does `{{ .Release.Name }}`
fail, and what is correct?**

`range` rebinds `.` to the current list element, a string such as
`orders.example.com`, which has no `Release` field. `$` always refers to the
root context: `{{ $.Release.Name }}`.

**Q15. Why is `include` used rather than `template` for label blocks?**

`include` is a function that *returns* the rendered text, so it can be piped
into `nindent` to sit at the right depth (`{{- include "webapp.labels" . |
nindent 4 }}` throughout `webapp`). `template` is an action that writes
directly to the output and cannot be piped. `{{ template … | nindent 4 }}`
pipes the context instead, and fails with a type error.

**Q16. `helm install r ./c --set a=1 -f x.yaml`, where `x.yaml` sets `a: 2`.
What is `a`?**

`a = 1`. All `--set`-family flags override every `-f` file, whatever their
position on the command line. Position only matters *within* each group.
`10`'s first row shows `--set replicaCount=2` overriding `dev.yaml`.

**Q17. Why can `-f dev.yaml -f prod.yaml` produce a production release that
exposes a NodePort?**

Maps merge deeply, key by key. `prod.yaml` sets nothing under `service`, so
dev's `type: NodePort` and `nodePort: 30080` survive into the merged values
(`10`, second row). The fix is never to layer one environment's file on
another.

**Q18. A value that `--set` and a YAML file interpret differently.**

`1.30`. In a YAML file it is the float `1.3` (`12`: `image: "nginx:1.3"`).
With `--set image.tag=1.30` it stays the string `"1.30"`, because `--set`
converts only booleans and whole numbers.

### 3.2.2.4 - Validation

**Q19. Why does `helm lint` succeed for a chart that cannot be rendered with
its default values?**

`lint` checks chart structure and attempts a render. When the render stops at
a `required` call, it records a **warning** ("missing required values") rather
than an error, on the reasoning that the chart may be intended to receive
that value at install time. The exit code stays 0, even with `--strict` (`11`).

**Q20. A JSON Schema fragment restricting `service.port` to 80 or 8080.**

```json
"service": {
  "type": "object",
  "properties": {
    "port": { "enum": [80, 8080] }
  }
}
```

**Q21. Why is a misspelled field not caught by `helm template`?**

Templates produce **text**. `helm template` checks that the text is valid
YAML, which `replica: 3` is, but it has no knowledge of the Deployment
schema. Only the API server knows that `spec.replica` does not exist, and
only strict server-side validation reports it (`14`).

### 3.2.3 - Lifecycle and server-side apply

**Q22. What did Stage 6 show about server-side apply that client-side apply
would have hidden?**

Under client-side apply, Helm's three-way merge would have quietly reset
`.spec.replicas` from 5 to 3 at the next upgrade, overwriting whatever made
the change. That is harmless for a stray `kubectl scale`, but harmful if the
other manager is a Horizontal Pod Autoscaler doing its job. Under SSA the
disagreement was made **visible** as a conflict naming the other manager
(`kubectl`, subresource `scale`) and the exact field. Resolving it became a
decision (`--force-conflicts`) recorded in the history as revision 3, rather
than a silent side-effect.

---

## 5. Reflection

### What was difficult

The hardest idea was the **gap between "Helm succeeded" and "the system is
right"**. In this practical that gap appeared four times:

- the partial upgrade returned `has been upgraded` while halving prod capacity
  (`05`);
- the float tag rendered cleanly as the wrong image (`12`);
- `helm lint` returned 0 on a chart that cannot render (`11`);
- `helm install --dry-run=server` returned `Dry run complete` for a Deployment
  the API server rejects (`14`).

Each tool answers a narrower question than its name suggests. Building the
habit of asking *what exactly did this check verify?* was the real work of the
practical.

The second difficulty was keeping the **three places configuration lives**
separate: the chart's defaults, the user-supplied values in the release
record, and the live objects. `helm get values` and `helm get manifest` read
the record, not the cluster. Stage 6 was the first time the record and the
live object were seen to disagree (5 replicas live, 3 in the record), and SSA
was what surfaced the disagreement.

### Errors met, and how they were diagnosed

**1. `INSTALLATION FAILED … not ready. status: InProgress … context deadline exceeded`**
*(`02b-stage1-install-timeout`)*

This was the one unplanned failure. It looked like a broken chart, and it was
not. The diagnosis came from Kubernetes, not Helm. `kubectl get pods` showed
both Pods `ContainerCreating` after 5 minutes, and `kubectl get events` showed
only `Pulling image "ghcr.io/stefanprodan/podinfo:6.15.0"`, with no error
and no `Pulled` event. The image was still downloading. `--timeout 3m`
expired first, so Helm, which had been told to wait for readiness, marked
revision 1 `failed`. Meanwhile the objects stayed in the cluster and the pull
carried on.

The recovery was deterministic, not a retry loop:
`kubectl rollout status … --timeout=20m` to let the pull finish, then
`helm uninstall` and a fresh install. The fresh install took seconds because
the image was now cached on both workers, and it gave a clean revision 1 for
the history analysis.

Two lessons follow. `--wait`'s timeout must cover **first-pull time** on the
slowest network that will run it, not only the application's start-up time.
And `failed` describes Helm's operation, not the workload. Helm 4's
`--rollback-on-failure` would have uninstalled everything at 3 minutes, which
is safer for an upgrade but would have discarded a pull that was about to
succeed.

**2. `helm lint` printed the same warning ten times** *(`11`)*

The repetition looked like a loop in the chart. It is Helm 4 warning once for
**each evaluation** of the `webapp.environment` helper, and there are exactly
ten:

- five label blocks: ConfigMap, Service, Deployment, Pod template and test
  Pod;
- the ConfigMap's page body;
- the test command;
- `NOTES.txt`;
- two more from the Deployment's `checksum/config` annotation, which renders
  the whole ConfigMap again (its labels and its body) in order to hash it.

Counting them confirmed the checksum mechanism works as described: it really
does re-render the other template. The useful conclusion is about CI rather than
Helm. Ten warnings and an exit code of 0 is exactly the output a pipeline
ignores.

**3. The deliberate failures** *(`11`–`14`, `16`)*

These were planned, but reading them still took practice. The `required`
error points at `test-connection.yaml:10:8`, a file that does nothing wrong.
It is just the first template to call the helper. The schema reported three
violations in a different order from the handout, because the schema
validator's output order is not part of its contract. In each case the
*message* identified the cause, while the location and the order did not.

### What would be done differently

**Pin `--timeout` to a measured pull time, or pre-pull.** On a slow network,
`kind load docker-image` (or a local registry mirror) removes first-pull time
from every later install. Error 1 would not have occurred.

**Run the full validation chain in CI from the start.** The table in 3.2.2.4
became concrete in `14`: lint, template and `--dry-run=server` each missed
something that `helm template | kubectl apply --dry-run=server` caught. A
pipeline step that runs that last command for every environment file costs
seconds and closes all four failure classes.

**Never use `--set` for anything that matters.** `05` showed values vanishing,
`10` showed `--set` changing types, and Q2 showed `--set` values ending up in
shell history. Values files in Git, passed with `-f` on every upgrade, avoid
all three.

### What remains unclear

How SSA conflicts should be handled when the other field manager is
legitimate. In Stage 6 it was a stray `kubectl scale`, so `--force-conflicts`
was correct. With an HPA managing `spec.replicas`, forcing the field back on
every upgrade would fight the autoscaler. The likely answer is that the chart
should not set `replicas` at all when autoscaling is enabled. The scaffold's
`hpa.yaml` suggests this, but this practical did not test it.

It is also unclear how teams keep secret values out of the release record in
practice. Section 3.2.8 promises the industry alternatives. Until then, Q2's
answer stands: anything passed to Helm is stored in the cluster.

---

## 6. References

All accessed 5 October 2026.

1. Helm Documentation - *Installing Helm*.
   https://helm.sh/docs/intro/install/
   (Used for the Helm 4 installation methods and the retired APT mirror notice.)
2. Helm Documentation - *Charts* (chart file structure, `Chart.yaml`,
   `apiVersion: v2`, `kubeVersion`).
   https://helm.sh/docs/topics/charts/
3. Helm Documentation - *Chart Template Guide* (built-in objects, values
   files, functions and pipelines, flow control, named templates).
   https://helm.sh/docs/chart_template_guide/
4. Helm Documentation - *Chart Tests*.
   https://helm.sh/docs/topics/chart_tests/
5. Helm Documentation - *Using Helm* (`helm upgrade` value modes,
   `--reuse-values`, `--reset-then-reuse-values`, history and rollback).
   https://helm.sh/docs/intro/using_helm/
6. Helm Documentation - *Registries* (OCI-based chart distribution).
   https://helm.sh/docs/topics/registries/
7. Helm Documentation - *Helm 4 overview and changelog* (server-side apply,
   renamed flags, `--wait` strategies).
   https://helm.sh/docs/
8. Kubernetes Documentation - *Server-Side Apply* (field managers, conflicts,
   `--force-conflicts`).
   https://kubernetes.io/docs/reference/using-api/server-side-apply/
9. Kubernetes Documentation - *Recommended Labels*.
   https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/
10. Semantic Versioning 2.0.0.
    https://semver.org/
    (Used for pre-release ordering and the `-0` suffix.)
11. JSON Schema - draft 2020-12.
    https://json-schema.org/draft/2020-12/schema
12. podinfo - chart repository and source.
    https://github.com/stefanprodan/podinfo
13. DSO202 Unit III 3.2 notes - *Helm*, sections 3.2.0–3.2.3.

---

## Appendix - Evidence index

| File | Stage | What it establishes |
| --- | --- | --- |
| `01-stage0-tooling-and-cluster` | 0 | Helm v4.3.0; context `kind-dso202`; three nodes Ready; no baltocdn source; Helm paths; `helm list -A` empty |
| `02-stage1-search-and-read-chart` | 1 | Chart vs app version columns; podinfo is `apiVersion: v1` (legacy); configuration interface read before install |
| `02b-stage1-install-timeout` | 1 | `--wait` timeout marks a release `failed` while the image is still pulling |
| `03-stage1-release-installed` | 1 | Release `deployed`; namespaced `helm list`; 2/2 replicas; `curl` returns the `--set` message; manifest with `# Source:` |
| `04-stage1-release-record` | 1 | Release Secret and its labels; decoded record shows `apply_method: ssa` and the values in clear |
| `05-stage1-values-loss-and-reuse` | 1 | Partial upgrade discards values (replicas 2→1); `--reuse-values` restores them |
| `06-stage1-rollback-history` | 1 | Rollback creates revision 4 `Rollback to 1`; revisions 2–3 kept |
| `07-stage1-uninstall-and-oci` | 1 | Uninstall removes objects and records; namespace survives; OCI install with digest |
| `08-stage2-scaffold-and-webapp-chart` | 2 | Helm 4 scaffold incl. `httproute.yaml`; `webapp` files; `Chart.yaml` version vs appVersion |
| `09-stage3-render-dev` | 3 | Kind-ordered objects; checksum `534e2ce7…`; tag from appVersion; built-in objects in the ConfigMap |
| `10-stage4-values-precedence` | 4 | `--set` beats `-f`; leaks when layering files; `--set` vs `--set-string`; `null` deletes |
| `11-stage5-required-value-and-lint` | 5 | `required` stops rendering; `helm lint` exits 0 even with `--strict` |
| `12-stage5-indentation-and-float-tag` | 5 | Missing `nindent` → YAML parse error, cause shown by `--debug`; `1.30` → `nginx:1.3` silently |
| `13-stage5-values-schema` | 5 | Schema turns silent failures into precise errors; all violations reported at once |
| `14-stage5-unknown-field-and-ci-loop` | 5 | Only `kubectl apply --dry-run=server` catches `spec.replica`; CI loop `dev: ok`, `prod: ok` |
| `15-stage6-install-and-helm-test` | 6 | One chart, two releases; NodePort `curl` from the host; both `helm test` runs `Succeeded` |
| `16-stage6-drift-and-ownership` | 6 | SSA conflict on drifted `.spec.replicas`; `--force-conflicts`; failed revision in history; ownership refusal naming all three markers |
