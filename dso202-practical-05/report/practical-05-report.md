# DSO202 — Practical 5 Report

**Environment-Specific Configuration with Kustomize on Kind**

| Field | Detail |
| --- | --- |
| Module | DSO202 — Scaling, Orchestration, Monitoring & Observability |
| Programme | BE in Software Engineering |
| Practical | 5 of 10 |
| Tool | kind (Kubernetes IN Docker), Kustomize built into kubectl |
| Date carried out | 27 September 2026 |
| Repository path | `dso202-practical-05/` |

All screenshots referenced below are in `evidence/`, numbered in the order they
were captured. A suffix `b` marks a supplementary frame taken to show output
that the original frame cut off.

---

## 1. Objective

Earlier practicals wrote one manifest per object and applied it to one
namespace. That works for one environment. A real application runs in several
environments at once, such as development, staging and production. The
objects are almost identical across them. They differ in a handful of values:
the namespace, the replica count, the image tag, the resource budget and the
content.

The naive solution is to copy the manifests into one directory per
environment. It fails slowly. Each copy drifts from the others, a fix made in
one is forgotten in another, and a reviewer comparing two environments has to
read two complete files to find the three lines that differ.

This practical uses **Kustomize** to avoid that. One **base** holds the
Deployment, the Service and the default page exactly once. Each environment is
an **overlay**: a small directory that states only how that environment
differs from the base. Kustomize combines the two at render time and produces
plain Kubernetes YAML. The overlays never contain a copy of the Deployment or
the Service.

The work covered:

- rendering the base and several overlays without touching the cluster;
- comparing two environments by diffing their rendered output;
- deploying through the safe workflow **render → diff → apply → verify**;
- proving the chain by which a content change in a generated ConfigMap causes a
  rollout;
- reading the result of a strategic merge patch;
- writing a new QA overlay with a JSON 6902 patch;
- predicting and then verifying the effect of `namePrefix` on names and
  references (challenge extension).

---

## 2. Environment

| Component | Version |
| --- | --- |
| Operating system | macOS 26.6.2 (build 25G83), arm64 |
| Docker Desktop | 29.7.2 |
| kind | v0.32.0 (go1.26.3 darwin/arm64) |
| kubectl | v1.36.3, with Kustomize v5.8.1 built in |
| Kubernetes (cluster) | v1.36.1 — `kindest/node:v1.36.1`, pinned by digest |
| Node OS / kernel | Debian GNU/Linux 13 (trixie), 7.0.12-linuxkit (arm64) |
| Container runtime | containerd 2.3.1 |

### Container images

| Image | Used by | Version observed at runtime |
| --- | --- | --- |
| `nginx:1.31-alpine` | dev overlay (set by `images:`) | nginx/1.31.6 |
| `nginx:1.30-alpine` | base; staging, prod, qa | nginx/1.30.5 |

The runtime versions were read with `kubectl exec … -- nginx -v` against the
dev and prod Deployments. That confirms the `images:` transformer changed what
actually runs, not only what the YAML says.

### Version-skew correction carried over

As in Practical 2, the Docker Desktop copy of `kubectl` in `/usr/local/bin`
shadows the Homebrew binary. Every session began with:

```bash
export PATH="/opt/homebrew/bin:$PATH"
```

This matters more here than before. `kubectl kustomize` and `kubectl -k` use
the Kustomize version compiled into the binary, and field names such as
`labels.includeTemplates` depend on that version. The version used is the one
reported by `kubectl version --client`: **Kustomize v5.8.1**
(`01-task0-preflight`).

### Repository note

The handout assumes an existing `examples/webapp` repository. None was
supplied, so the base, the four environment overlays and the sandbox overlay
were written for this practical. They follow the structure the handout
specifies. A fifth directory, `examples/mistakes/qa-unescaped-path/`, keeps
the first, broken version of the QA patch as evidence for §5.

The work was carried out in a directory named `dso202-practical-03/`, which
was renamed to `dso202-practical-05/` afterwards to match the module's
practical numbering. The screenshots therefore show `dso202-practical-03` in
the shell prompt, and three identifiers keep their original values so that the
repository matches the evidence: the kind cluster `dso202-p3`, its context
`kind-dso202-p3`, and the label value `app.kubernetes.io/part-of:
dso202-practical-03`.

---

## 3. Procedure and Observations

### 3.1 Task 0 — Pre-flight

A three-node cluster, `dso202-p3`, was created from `cluster/kind-cluster.yaml`.
The node names come from `kubeadmConfigPatches` and match the lab topology.
The cluster was not inspected until every node reported Ready, using a
deterministic wait rather than a fixed delay:

```bash
kubectl wait --for=condition=Ready nodes --all --timeout=180s
```

**Evidence: `01-task0-preflight`.** It shows all three conditions met.
`kubectl cluster-info` reaches a control plane at `https://127.0.0.1:52962`,
the loopback address that `apiServerAddress` sets, which confirms the current
context is the kind cluster (`kind-dso202-p3`) and not a leftover from an
earlier practical. `control-plane`, `worker-node-1` and `worker-node-2` are
all `Ready` on v1.36.1. The client reports `gitVersion: v1.36.3` and
`kustomizeVersion: v5.8.1`. A non-empty Kustomize version is the confirmation
that this binary supports both `kubectl kustomize` and `-k`.

### 3.2 Task 1 — Reading the repository before running it

**Evidence: `02-task1-repo-tree`.** The tree has 8 directories and 20 files.
The base holds four files. Each overlay holds only an `index.html`, a
`kustomization.yaml`, a `namespace.yaml` and, where needed, one patch file.
**No overlay contains a `deployment.yaml` or a `service.yaml`.** That absence
is the whole point of the structure. The three questions set by the handout
are answered in §4 (Q1).

### 3.3 Task 2 — Rendering the base

**Evidence: `03-task2-base-render`.** `kubectl kustomize examples/webapp/base`
emitted three objects: a `ConfigMap`, a `Service` and a `Deployment`. Only two
of them came from files. The ConfigMap is **generated** by the
`configMapGenerator` in `base/kustomization.yaml` from `base/index.html`.

Filtering for `name: web-content` returned **two** lines, both reading
`web-content-448dt2mfcm`:

- the first, at two-space indentation, is the ConfigMap's own
  `metadata.name`;
- the second, deeply indented, is `spec.template.spec.volumes[].configMap.name`
  in the Deployment.

`base/deployment.yaml` names the volume source as plain `web-content`.
Kustomize appended a 10-character hash to the generated object and **rewrote
the reference in the Deployment to match.** Without that rewrite the
Deployment would point at a ConfigMap that does not exist, and its Pods would
never start. The checkpoint question is answered in §4 (Q2).

The base was not applied. It has no namespace of its own and is not an
environment. It is only an input to overlays.

### 3.4 Task 3 — Comparing dev and prod without touching the cluster

Both overlays were rendered to files in `/tmp` and compared with `diff -u`.
No command in this task talks to the API server. The comparison is between
two pieces of text produced locally.

**Evidence: `04-task3-dev-prod-diff`** (Namespace and ConfigMap) and
**`04b-task3-dev-prod-diff-deployment`** (Service and Deployment). Together
they show every difference between the two environments:

| # | Category | dev | prod | Source of the difference |
| --- | --- | --- | --- | --- |
| 1 | Namespace | `webapp-dev` | `webapp-prod` | `namespace:` field + `namespace.yaml` |
| 2 | Environment label | `environment: dev` | `environment: prod` | `labels:` in each overlay |
| 3 | Replica count | `1` | `3` | `replicas:` in each overlay |
| 4 | Image tag | `nginx:1.31-alpine` | `nginx:1.30-alpine` | `images:` in each overlay |
| 5 | Resource limits | 100m / 64Mi | 250m / 128Mi | `prod/patch-resources.yaml` |
| 6 | Memory request | 32Mi | 64Mi | `prod/patch-resources.yaml` |
| 7 | Page content | `DEV v1 - development environment` | `PROD - production environment` | each overlay's `index.html` |
| 8 | ConfigMap name | `web-content-fhfd9gbh6f` | `web-content-9kkk5t2mck` | follows from #7 (see §3.7) |
| 9 | Annotation | absent | `change-policy: approval-required` | `commonAnnotations:` in prod |

Two rows deserve comment. **Row 8 is not an independent setting.** Nobody
chose these two names. They differ because the content differs, and the diff
shows the Deployment's volume reference changing in step
(`04b`, final lines). **Row 9 reaches the Pod template too.** The diff shows
`change-policy` added under `spec.template.metadata.annotations`, so every
production Pod carries it, not only the Deployment object.

The principle stated in the handout holds. Every difference in that table can
be traced to a line or two in the prod overlay, and the reviewer never had to
read a second Deployment.

### 3.5 Task 4 — Deploying dev safely

**Render. Evidence: `05-task4-dev-rendered`** and
**`05b-task4-dev-rendered-deployment`.** The full dev output shows four
objects. The Namespace `webapp-dev` came from the overlay. The ConfigMap
`web-content-fhfd9gbh6f` came from the dev `index.html` (`DEV v1`), which
replaced the base content through `behavior: replace`. The Service `webapp`
and the Deployment `webapp` came from the base and are now in namespace
`webapp-dev`. Frame `05b` shows the parts of the Deployment the overlay
changed: `replicas: 1`, `image: nginx:1.31-alpine`, and the volume now
referencing `web-content-fhfd9gbh6f`. The resources are unchanged from the
base, because dev applies no patch.

**Diff, then apply. Evidence: `06-task4-dev-diff-apply`.** `kubectl diff`
returned:

```
Error from server (NotFound): namespaces "webapp-dev" not found
```

This is not a defect in the overlay (see §5, error 2). `kubectl apply -k` then
created exactly the four objects the render had predicted, with the same
ConfigMap name.

**Verify. Evidence: `07-task4-dev-verify`.** The rollout completed. `get all`
shows one Pod `1/1 Running`, a ClusterIP Service `webapp` on `10.96.84.181`,
a Deployment at `1/1` and one ReplicaSet `webapp-8c98c695d`.
`get configmap` shows `web-content-fhfd9gbh6f` next to the
automatically created `kube-root-ca.crt`.

**The Deployment kept the base name `webapp`.** No `-dev` suffix was needed,
because the namespace is what separates the environments. The same name
appears in all four namespaces in §3.8.

### 3.6 Task 5 — Reaching the application

**Evidence: `08-task5-curl-dev`.** In one terminal,
`kubectl port-forward -n webapp-dev service/webapp 8080:80` forwarded local
port 8080 on both the IPv4 and IPv6 loopback addresses and logged
`Handling connection for 8080` when the request arrived. In the second
terminal, `curl http://127.0.0.1:8080` returned the dev page: title
`webapp - dev`, heading `DEV v1 - development environment`, and
`Environment: dev`. The port-forward was then stopped with Ctrl+C (the `^C`
is visible).

The response proves the whole chain end to end. The overlay's `index.html`
became a ConfigMap, the ConfigMap was mounted at `/usr/share/nginx/html`, and
the Service selected the Pod serving it.

### 3.7 Task 6 — Proving the ConfigMap hash → rollout chain

**Before. Evidence: `09-task6-before-change`.**

| | Value |
| --- | --- |
| ConfigMap | `web-content-fhfd9gbh6f` (age 2m31s) |
| Pod | `webapp-8c98c695d-mfg27` on `worker-node-2`, IP `10.244.2.2` |

**Change, render, apply. Evidence: `10-task6-after-rollout`.** The `<h1>` in
`overlays/dev/index.html` was changed to `DEV v2 - configuration changed`. The
render was checked **before** applying. Both the generated name and the
Deployment's reference had become `web-content-554hm75k72`. The apply output
records exactly what moved:

```
namespace/webapp-dev unchanged
configmap/web-content-554hm75k72 created
service/webapp unchanged
deployment.apps/webapp configured
```

The ConfigMap was **created**, not "configured": to the API server it is a
brand-new object with a new name. The Deployment was **configured**, because
its volume reference changed.

**After.**

| | Before | After |
| --- | --- | --- |
| ConfigMap in use | `web-content-fhfd9gbh6f` | `web-content-554hm75k72` |
| ReplicaSet hash | `8c98c695d` | `7877487cbf` |
| Pod | `webapp-8c98c695d-mfg27` | `webapp-7877487cbf-w8zs5` (age 12s) |
| Pod IP | `10.244.2.2` | `10.244.2.3` |

The Pod was **replaced**, not restarted: it has a new name, a new ReplicaSet
hash and a new IP. The page served by the running Pod was later checked with
`kubectl exec … cat /usr/share/nginx/html/index.html`, and it returned the
`DEV v2` heading.

**The old ConfigMap was not deleted.** After the rollout, `web-content-fhfd9gbh6f`
is still listed, aged 3m21s, beside the new one. `kubectl apply` creates and
updates; it never removes an object that is no longer in the rendered output.
This is useful, because the old ReplicaSet still references the old ConfigMap,
so `kubectl rollout undo` can return to v1 with its content intact. It is also
a cost: every content change leaves one orphaned ConfigMap behind until
someone prunes it.

The explanation of the chain, required by the handout, is in §4 (Q3).

### 3.8 Task 7 — Deploying staging and prod

**Evidence: `11-task7-all-environments`.** Both overlays went through
diff → apply. Both first diffs returned the same `namespaces … not found`
error as dev, for the same reason. The applies created the ConfigMaps
`web-content-gfcm7fg6tb` (staging) and `web-content-9kkk5t2mck` (prod). The
prod name is identical to the one the offline render produced in Task 3,
which shows the render is a reliable preview of the apply. Both rollouts
completed.

The first verification command failed in a revealing way:

```
$ kubectl get deploy -A -l app.kubernetes.io/name=webapp
No resources found
```

The Pod query in the same frame, which used the same label, found all six
Pods. The label was therefore on the Pods but not on the Deployments. This was
an error in the base, diagnosed and fixed in §5 (error 1).

**Evidence: `11b-task7-label-fix-all-environments`.** After the fix, the diff
against prod, now a live namespace, showed a real diff for the first time. It
contained exactly one added line,
`+ app.kubernetes.io/name: webapp`, on each of the Deployment, the ConfigMap
and the Service, and nothing else. All four overlays were re-applied. Each
ConfigMap was reported `configured` **under its unchanged name**, because
labels are not part of the hash (§4, Q2). The label query then succeeded:

| Environment | Namespace | Replicas (ready/desired) | Pod placement |
| --- | --- | --- | --- |
| dev | `webapp-dev` | 1/1 | worker-node-2 |
| staging | `webapp-staging` | 2/2 | worker-node-1, worker-node-2 |
| qa | `webapp-qa` | 2/2 | worker-node-1, worker-node-2 |
| prod | `webapp-prod` | 3/3 | worker-node-1 ×1, worker-node-2 ×2 |

The same frame shows something that matters as much as the counts. **No Pod
was replaced by this apply.** Every Pod name is the same as in `11` and `15`
(for example `webapp-746c666b6-qq98n`), and their ages (8–12 minutes) predate
the fix. A label added to the Deployment's own metadata does not alter the Pod
template, so the Deployment controller had nothing to roll out. Compare §3.7,
where a change that did alter the template replaced the Pod.

### 3.9 Task 8 — What the prod patch changed

**Evidence: `12-task8-prod-patch-merge`.** The frame shows the patch file, the
base's rendered resources, and prod's rendered resources, one after another:

| Field | Base | Patch states | Prod (rendered) | Result |
| --- | --- | --- | --- | --- |
| `limits.cpu` | 100m | 250m | 250m | replaced |
| `limits.memory` | 64Mi | 128Mi | 128Mi | replaced |
| `requests.cpu` | 50m | *(not mentioned)* | **50m** | **kept from base** |
| `requests.memory` | 32Mi | 64Mi | 64Mi | replaced |

`requests.cpu` was left out of the patch on purpose, as a test. It survived
into the rendered output. **The base values were merged, not deleted.** The
three questions of this task are answered in §4 (Q4).

The patch also names its container as `- name: nginx`, not by list position.
The strategic merge uses `name` as the merge key for `containers`, so the
patch reached the right container even though YAML lists are ordered. If the
patch named a container that does not exist, it would **add** a second
container with only a resources block, and the Deployment would then be
rejected. A rendered-output check would catch that before the apply.

### 3.10 Task 9 — The QA overlay

The QA overlay was written to the requirements: namespace `webapp-qa`,
2 replicas, environment label `qa`, its own `index.html`, the same base, and
the annotation `training.example.com/owner: qa-team` added by a
**JSON 6902** patch. No base file was copied.

**The first attempt failed. Evidence: `13-task9-qa-mistake`.** Rendering it
stopped Kustomize with:

```
error: add operation does not apply: doc is missing path:
"/metadata/annotations/training.example.com/owner": missing value
```

The cause and the lesson are in §5 (error 3). The broken version is kept in
`examples/mistakes/qa-unescaped-path/` as evidence.

**The corrected overlay. Evidence: `14-task9-qa-overlay-render`.** The frame
shows the four required files, `kustomization.yaml` and the corrected patch:

```yaml
- op: add
  path: /metadata/annotations/training.example.com~1owner
  value: qa-team
```

The render was checked before anything else, as the handout requires. The
Deployment carries **both** annotations:

```yaml
annotations:
  training.example.com/course: dso202     # from the base
  training.example.com/owner: qa-team     # from the JSON 6902 patch
```

It also carries `environment: qa`, `namespace: webapp-qa` and `replicas: 2`.
The `add` inserted one key into the existing map and did not replace the map.

**Apply and verify. Evidence: `15-task9-qa-applied`.** The diff returned the
familiar `namespaces "webapp-qa" not found`. The apply created four objects,
including `web-content-g5924g4d49`, and the rollout completed. A JSONPath
query for the annotation returned exactly `qa-team`. The full
`kubectl get deployment -o yaml` shows the object **as the API server stored
it**. Both annotations are present, as is
`kubectl.kubernetes.io/last-applied-configuration`, which is the record
`kubectl apply` keeps in order to compute its next three-way merge.
`replicas: 2` is also present.

### 3.11 Challenge extension — `namePrefix` on a sandbox overlay

A sandbox overlay was added with `namePrefix: sandbox-` and namespace
`webapp-sandbox`. It uses the same base and copies no base files. It was
rendered only, never applied.

**Prediction, made before rendering.** The prediction was reasoned from one
rule: a prefix changes object **names**, and Kustomize then updates every
field it knows to be a **reference** to a name.

| Item | Prediction | Observed (`16-challenge-sandbox-prefix`) | Match |
| --- | --- | --- | --- |
| Deployment name | `sandbox-webapp` | `sandbox-webapp` (line 52) | ✓ |
| Service name | `sandbox-webapp` | `sandbox-webapp` (line 33) | ✓ |
| ConfigMap name | `sandbox-web-content-<hash>` | `sandbox-web-content-448dt2mfcm` (line 24) | ✓ |
| Deployment's volume reference to the ConfigMap | rewritten to the prefixed name | `sandbox-web-content-448dt2mfcm` (line 90) | ✓ |
| Namespace object | not prefixed | `webapp-sandbox` (line 6) | ✓ |
| Service selector, Deployment selector, Pod labels | unchanged — label **values** are not names | `app.kubernetes.io/name: webapp` (lines 41, 58, 63) | ✓ |
| Container, port and volume names (`nginx`, `http`, `content`) | unchanged — not object names | unchanged | ✓ |
| Hash suffix | uncertain | **same as base**: `448dt2mfcm` | — |

Every prediction held. The one open question had a clear answer: the hash is
identical to the base's (`03-task2-base-render`). The generator computes the
hash from the ConfigMap's content before the prefix transformer runs, so the
prefix is applied to an already-hashed name.

Two rows show the **reference-aware** half of the transformation. First, the
volume reference on line 90 was rewritten along with the ConfigMap's name. If
it had not been, the sandbox Deployment would reference a `web-content-…`
ConfigMap that does not exist in its namespace. Second, the Service's selector
was **not** rewritten. It matches on a label value, and label values are not
references to object names. Kustomize's rule is not "prefix every string that
looks like `webapp`". It prefixes names, and then only the fields it knows
point at those names.

### 3.12 Task 10 — Cleanup

Cleanup is deliberately left until after this report and its evidence are
final, because deleting the namespaces destroys the live state the
screenshots describe. The procedure, recorded in the README, is
`kubectl delete -k` for each overlay in turn. That removes each namespace
along with everything inside it, including the orphaned ConfigMaps from §3.7,
which `apply` never removed. It is followed by `kind delete cluster --name
dso202-p3`. No screenshot of cleanup is submitted.

---

## 4. Analysis

**Q1. Before building anything: which files exist only once for all
environments, which values differ between environments, and where are those
differences represented?**

*Exist once:* `base/deployment.yaml`, `base/service.yaml`,
`base/kustomization.yaml` and `base/index.html`. The Deployment's container
spec, probe, port, volume mount and selector, and the Service's port mapping,
are written in exactly one place.

*Values that differ:* the namespace, the environment label, the replica count,
the image tag, the page content, the resource budget (prod only) and
annotations (prod and QA).

*Where they are represented:*

| Difference | Represented in |
| --- | --- |
| Namespace | `namespace:` field in each overlay's `kustomization.yaml`, plus that overlay's `namespace.yaml` so the namespace itself is created |
| Environment label | `labels:` in each overlay's `kustomization.yaml` |
| Replicas | `replicas:` in each overlay's `kustomization.yaml` |
| Image tag | `images:` in the dev, staging and prod `kustomization.yaml` |
| Page content | each overlay's `index.html`, pulled in by `configMapGenerator` with `behavior: replace` |
| Resources | `prod/patch-resources.yaml` (strategic merge) |
| Annotations | `commonAnnotations:` in prod; `qa/patch-annotation.yaml` (JSON 6902) |

**Q2. Why does the generated ConfigMap name not exactly equal `web-content`?**

Kustomize appends a suffix computed as a hash of the ConfigMap's content,
whose variable part here is the bytes of `index.html` (`03-task2-base-render`: `web-content-448dt2mfcm`).
The purpose is to turn a content change into a **name** change.

A Deployment does not watch the contents of the ConfigMaps it mounts. If the
generated name were always `web-content`, editing `index.html` and applying
would update the ConfigMap in place. The Deployment's Pod template would be
byte-for-byte unchanged, so the Deployment controller would do nothing, and
the running Pods would go on serving the old content. With the hash, new
content gives a new name, the new name gives a new Pod template, and a new Pod
template is exactly what triggers a rolling update (§3.7). The same mechanism
leaves the previous ConfigMap in place, so a rollback still has its content.

What the hash does **not** cover matters as well. In `11b`, adding a label to
every ConfigMap left all four names unchanged, so labels are not an input.
The hash is taken over the kind, the stem name (`web-content`) and the data. That is why a label-only
change reached the cluster without any rollout.

**Q3. Explain, in your own words, the chain from a file change to a
rollout.** *(Mandatory.)*

1. **The file content changed.** The `<h1>` in `overlays/dev/index.html` went
   from `DEV v1` to `DEV v2`. Nothing else in the repository was touched.
2. **The generated ConfigMap's content changed.** The `configMapGenerator`
   copies the file's bytes into `data.index.html`, so the ConfigMap Kustomize
   emits now holds different data.
3. **The generated ConfigMap's name changed.** The suffix is a hash of that
   data, so `web-content-fhfd9gbh6f` became `web-content-554hm75k72`. This was
   visible in the render before anything was applied (`10`).
4. **The Deployment's reference changed.** Kustomize knows that
   `volumes[].configMap.name` refers to a ConfigMap, and rewrote it to the new
   name in the same render. The `grep` in `10` shows both lines move together.
5. **The Deployment's Pod template changed.** That reference sits inside
   `spec.template`. A change anywhere in the template is a new template, and
   the apply reported `deployment.apps/webapp configured`.
6. **A rollout occurred.** The Deployment controller hashes the template to
   name ReplicaSets. The new template produced a new ReplicaSet
   (`8c98c695d` → `7877487cbf`), which was scaled up while the old one was
   scaled down. The Pod `webapp-8c98c695d-mfg27` was replaced by
   `webapp-7877487cbf-w8zs5` (`09` → `10`).

The weak link, if any step were missing, is step 3. Without the hash the
chain stops there: the ConfigMap updates in place and nothing in the
Deployment changes. `11b` shows that stop happening for real. There, a
label-only change to the same ConfigMaps changed no name, and no Pod was
replaced.

**Q4. (Task 8) Were the base resource values deleted entirely or merged? Which
environment owns the production-specific resource policy? Why is a patch
better than copying `deployment.yaml` into `prod/`?**

*Merged.* A strategic merge patch is a partial object laid over the base:
fields it names are replaced, fields it omits are kept. The evidence is
`requests.cpu`. The patch does not mention it, and it rendered as the base's
`50m` next to the three values the patch did set (`12-task8-prod-patch-merge`).
Nothing was deleted. Deleting a field with a strategic merge would need an
explicit `$patch: delete` directive.

*The prod overlay owns it.* The policy lives in
`overlays/prod/patch-resources.yaml` and nowhere else. Dev, staging and QA
never see it, and the base stays a neutral default that is safe for any
environment.

*Why a patch is better than a copy:*

- **Review.** The patch is 15 lines of YAML that say only "prod gets more memory and
  CPU headroom." A copied Deployment is about 45 lines, and a reviewer must
  diff it against the base to find those 3 values.
- **Drift.** A fix to the base, such as a new probe, a security context or an
  image bump, reaches prod automatically. A copy must be edited by hand, and
  the edit that gets forgotten is how production ends up differing from what
  was tested.
- **Blast radius.** The patch can only change resources. A copy can silently
  change anything, including the selector, which cannot be changed once the
  Deployment exists.

**Q5. Explain strategic merge patching and JSON 6902 patching, and when each is
the right tool.**

| | Strategic merge patch | JSON 6902 patch |
| --- | --- | --- |
| Form | A partial Kubernetes object | An ordered list of operations (`add`, `remove`, `replace`, `move`, `copy`, `test`) |
| Used here | `prod/patch-resources.yaml` | `qa/patch-annotation.yaml` |
| Target | Found from the patch's own `kind` and `metadata.name` | Must be given explicitly (`target:` in the kustomization), because the op list names no object |
| Addressing | By structure; list items matched by a merge key (`containers` by `name`) | By JSON Pointer path (RFC 6901); list items by numeric index |
| Unmentioned fields | Kept | Untouched; only the paths named are affected |
| Removing a field | Needs a special directive (`$patch: delete`) | A plain `remove` op |
| Failure mode | Silent: a mistyped container name *adds* a container | Loud: a wrong path stops the render (`13`) |
| Special characters | None | `/` in a key must be written `~1`, and `~` as `~0` |

The strategic merge is the better default when the change is "these fields
should have these values". It reads like the object it modifies, and it is
robust to list order because it matches containers by name. Its weakness is
that it relies on knowing each field's merge semantics. A wrong merge key does
not fail; it produces a different object than intended.

JSON 6902 is right when the change is an **operation** a merge cannot
express. Removing a field, inserting at a list position, or asserting a value
with `test` before changing it are all examples. It is also right when the
change must be unambiguous. For QA it was the required tool, and its
precision showed on the first attempt: the pointer did not resolve, and the
render failed instead of producing a wrongly shaped Deployment.

In Kustomize v5 both kinds are listed under one `patches:` field. Kustomize
tells them apart by content: a YAML list of operations is JSON 6902, and a
Kubernetes object is a strategic merge. The older `patchesStrategicMerge` and
`patchesJson6902` fields are deprecated.

**Q6. (Challenge) What is the real learning target of `namePrefix`?**

Reference-aware transformation (§3.11). Renaming objects is trivial. Renaming
them and also rewriting every field that refers to them is what makes a
generated name usable, and it is the same machinery that rewrites the
hash-suffixed ConfigMap name in every overlay. The challenge also showed where
that awareness stops. Selectors match label **values**, which Kustomize does
not treat as references. A `sed`-style rename of every `webapp` string would
have broken the selector-to-label link, or accidentally preserved it; neither
would be understood. Kustomize changed exactly the name references and nothing
else.

---

## 5. Reflection

### What was difficult

The hardest idea was that **the name of a generated ConfigMap is an output,
not an input**. Everywhere else in Kubernetes, the author chooses a name and
it stays fixed. Here `web-content` is only a stem, the real name is computed
from the content, and the Deployment file says `web-content` while the
Deployment in the cluster says something else. Until the render in `03` showed
both lines together, it was not obvious how a Deployment could reference an
object whose name nobody had written down.

The second difficulty was keeping track of *which* layer produced a given line
of output. In the rendered prod Deployment, the name comes from the base, the
namespace from the overlay's `namespace:` field, `replicas: 3` from
`replicas:`, the annotation from `commonAnnotations:`, and the resources half
from the base and half from a patch. Reading rendered output with that
question in mind, "which file put this line here?", turned out to be the core
skill of the practical.

### Errors met, and how they were diagnosed

**1. `No resources found` for `kubectl get deploy -A -l app.kubernetes.io/name=webapp`**
*(the Kustomize mistake; `11-task7-all-environments`, fixed in `11b`)*

This was a real mistake in the base, and the rendered output was what
explained it. The Pod query with the same label in the same frame found six
Pods, so the label existed on Pods but not on Deployments. The rendered dev
Deployment in `05b` confirmed it. `metadata.labels` held only
`environment: dev` and `part-of: dso202-practical-03`, while
`app.kubernetes.io/name: webapp` appeared only under `selector.matchLabels`
and `template.metadata.labels`.

The cause was a wrong assumption. `app.kubernetes.io/name` had been written in
`deployment.yaml` as the selector and Pod label, and it was assumed to label
the Deployment itself as well. It does not: a Deployment's own labels, its
selector and its Pod template labels are three separate fields. The base
`labels:` transformer used `includeSelectors: false`, correctly so, because a
selector is immutable once created. It therefore only added what it was
given, which was `part-of`.

The fix was to add `app.kubernetes.io/name: webapp` to the base's `labels:`
pairs, still with `includeSelectors: false`. The fix was checked the same way
the mistake had been found. The rendered output of all four overlays was
compared before and after, and exactly three lines were added per overlay,
none in a selector. The live `kubectl diff` in `11b` then confirmed the same
three one-line additions against prod before the apply.

**What the rendered output exposed, and why it matters.** Nothing about this
mistake was visible at apply time. Every apply succeeded, every Pod ran and
every Service had endpoints. It only surfaced when a tool, here the handout's
own verification query, selected Deployments by label. In production that tool
would be a monitoring rule or a `kubectl rollout restart -l …` script that
silently matched nothing. The rendered YAML was the one place where the gap
between "labelled in the selector" and "labelled on the object" was visible
line by line.

**2. `Error from server (NotFound): namespaces "webapp-dev" not found` from
`kubectl diff`** *(`06`, `11`, `15`)*

It appeared on every first diff of a new environment, and it looked like a
broken overlay. It is not one. `kubectl diff` sends each rendered object to
the API server as a server-side **dry run**. The Namespace's dry-run create
succeeds, but it is not persisted. The next object, the ConfigMap in
`webapp-dev`, is then dry-run into a namespace that still does not exist, and
the API server rejects it. The overlay was correct, and the following apply
created everything as rendered.

The practical consequence is that `kubectl diff` gives no preview of the
**first** deployment of an overlay that creates its own namespace. For a first
deploy the render is the preview, which is why the render step came first
every time. `11b` shows what `diff` gives once the namespace exists: a precise,
three-line answer to "what will this apply change?". (Separately, `diff`
exits with status 1 whenever it finds differences, which is why the handout
appends `|| true`.)

**3. `add operation does not apply: doc is missing path` in the QA JSON 6902
patch** *(`13-task9-qa-mistake`)*

The first QA patch used the path
`/metadata/annotations/training.example.com/owner`. In a JSON Pointer, `/`
separates path segments, so this path means: under `annotations`, find a key
`training.example.com`, and under that add `owner`. No key
`training.example.com` exists, since the real key is the whole string
`training.example.com/owner`, so the `add` had no parent to add into. The fix
is to escape the slash inside the key as `~1`:
`/metadata/annotations/training.example.com~1owner`.

The render caught this **before anything reached the cluster**. That is the
argument for render-first in miniature: the error came from Kustomize on the
laptop, not from a half-applied environment. Its message named the exact path
that failed, which pointed straight at the one character that was wrong. It
also shows how the two patch types fail differently (§4, Q5). A strategic
merge with a comparable mistake would not have failed at all.

One design choice made the fix sufficient. The base Deployment already carries
an annotation (`training.example.com/course`), so `/metadata/annotations`
exists and `add` can insert one key into it. Had the base carried no
annotations, even the escaped path would fail. The patch would instead need to
`add` the whole `/metadata/annotations` map, which would replace, rather than
extend, any annotations a later base change introduced.

### What would be done differently

**Capture long output to a file first, then screenshot a filtered view.**
Screenshots `04` and `05` were both cropped before their most important part,
the Deployment. The supplementary frames `04b` and `05b` could be produced
only because the Task 3 renders happened to still be in `/tmp`. By then the
dev overlay had moved to v2, so a fresh render would have shown the wrong hash.
Writing each render to `evidence/` and screenshotting a `sed -n '/^kind:
Deployment/,$p'` view would have avoided the problem altogether.

**Add a label check to the base before the first apply.** The missing
Deployment label (error 1) would have been caught in Task 2 by one command:
`kubectl kustomize examples/webapp/base | grep -B3 -A6 '^  labels:'`. Checking
that every object carries the labels that tooling will select on is cheap to
do on the base, once, before four environments depend on it.

**Prune on apply, or clean up explicitly.** The orphaned
`web-content-fhfd9gbh6f` from §3.7 is harmless for one change on a laptop, but
a busy environment accumulates one per content change. The options are
`kubectl apply --prune` with an allow-list, a GitOps controller that prunes by
design, or deleting old generated ConfigMaps after a rollout is confirmed.
Each has a cost, since pruning too early removes what `rollout undo` needs.

### What remains unclear

How far the base/overlay split should go before it becomes its own kind of
duplication. Four overlays repeat the same three fields (`namespace:`,
`labels:`, `replicas:`) with different values. Kustomize **components** exist
to share fragments between overlays, but it is not obvious where the line is
between "a small, readable overlay per environment" and an indirection graph
that a reviewer has to evaluate mentally, which is the problem Kustomize set
out to solve.

It is also unclear how the ConfigMap hash behaves under GitOps, where a
controller applies the rendered output continuously. Each content change
leaves an old ConfigMap. Whether those should be garbage-collected by the
controller, kept for a fixed number of revisions to match
`revisionHistoryLimit`, or left for manual cleanup is a policy decision this
practical did not have to make.

---

## 6. References

All accessed 27 September 2026.

1. Kubernetes Documentation — *Declarative Management of Kubernetes Objects
   Using Kustomize*.
   https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
   (Used for bases and overlays, generators, `namePrefix`, and the
   `-k` flag.)
2. Kustomize Documentation — *kustomization.yaml reference*
   (`configMapGenerator`, `labels`, `replicas`, `images`, `patches`,
   `commonAnnotations`).
   https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/
   (Used for field names valid in Kustomize v5, `includeTemplates`, and
   `behavior: replace`.)
3. Kubernetes Documentation — *Update API Objects in Place Using kubectl
   patch*.
   https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/
   (Used for strategic merge semantics, merge keys and `$patch: delete`.)
4. IETF RFC 6902 — *JavaScript Object Notation (JSON) Patch*.
   https://www.rfc-editor.org/rfc/rfc6902
   (Used for the operation set and the requirement that `add` has an existing
   parent.)
5. IETF RFC 6901 — *JavaScript Object Notation (JSON) Pointer*.
   https://www.rfc-editor.org/rfc/rfc6901
   (Used for the `~1` / `~0` escaping rules.)
6. Kubernetes Documentation — *Deployments*.
   https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
   (Used for the Pod-template-change rollout trigger and `pod-template-hash`.)
7. Kubernetes Documentation — *Recommended Labels*.
   https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/
   (Used for the `app.kubernetes.io/*` label set.)
8. Kubernetes Documentation — *kubectl diff* reference.
   https://kubernetes.io/docs/reference/kubectl/generated/kubectl_diff/
   (Used for server-side dry-run behaviour and exit codes.)
9. kind Documentation — *Configuration*.
   https://kind.sigs.k8s.io/docs/user/configuration/
   (Used for `kubeadmConfigPatches` and node naming.)

---

## Appendix — Evidence index

| File | Task | What it establishes |
| --- | --- | --- |
| `01-task0-preflight` | 0 | Cluster created; three nodes Ready on v1.36.1; context reaches kind; client v1.36.3 with Kustomize v5.8.1 |
| `02-task1-repo-tree` | 1 | Base written once; overlays contain no Deployment or Service |
| `03-task2-base-render` | 2 | Three kinds rendered; generated name `web-content-448dt2mfcm`; reference rewritten in the Deployment |
| `04-task3-dev-prod-diff` | 3 | Namespace, label, annotation, content and ConfigMap-name differences |
| `04b-task3-dev-prod-diff-deployment` | 3 | Replicas 1→3, image 1.31→1.30, resources, Pod-template annotation, volume reference |
| `05-task4-dev-rendered` | 4 | Rendered dev Namespace, ConfigMap and Service |
| `05b-task4-dev-rendered-deployment` | 4 | Rendered dev Deployment: replicas, image, rewritten ConfigMap reference; missing metadata name label (§5) |
| `06-task4-dev-diff-apply` | 4 | First-deploy `diff` NotFound; apply created exactly the rendered objects |
| `07-task4-dev-verify` | 4 | Rollout complete; `get all -n webapp-dev`; generated ConfigMap in the cluster |
| `08-task5-curl-dev` | 5 | Port-forward; `curl` returns the dev page |
| `09-task6-before-change` | 6 | ConfigMap `fhfd9gbh6f`; Pod `webapp-8c98c695d-mfg27` |
| `10-task6-after-rollout` | 6 | New hash `554hm75k72` in render; ConfigMap created, Deployment configured; new Pod `webapp-7877487cbf-w8zs5`; old ConfigMap retained |
| `11-task7-all-environments` | 7 | Staging and prod deployed; label query on Deployments returns nothing |
| `11b-task7-label-fix-all-environments` | 7 | Live diff of the label fix; replica counts 1/2/2/3; no Pod replaced |
| `12-task8-prod-patch-merge` | 8 | Patch file; base vs prod resources; `requests.cpu` kept by merge |
| `13-task9-qa-mistake` | 9 | Unescaped JSON Pointer; render fails before reaching the cluster |
| `14-task9-qa-overlay-render` | 9 | QA overlay files; rendered Deployment carries both annotations |
| `15-task9-qa-applied` | 9 | QA applied; annotation `qa-team` read back from the API server |
| `16-challenge-sandbox-prefix` | Challenge | Names and ConfigMap reference prefixed; Namespace, selectors and labels not; hash unchanged |
