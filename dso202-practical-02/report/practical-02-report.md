# DSO202 — Practical 2 Report

**Implementing Persistent Storage for a Stateful Application in Kubernetes**

| Field | Detail |
| --- | --- |
| Module | DSO202 — Scaling, Orchestration, Monitoring & Observability |
| Programme | BE in Software Engineering |
| Practical | 2 of 10 |
| Tool | kind (Kubernetes IN Docker) |
| Date carried out | 26 August 2026 |
| Repository path | `dso202-practical-02/` |

All screenshots referenced below are in `evidence/`, numbered in the order they
were captured.

---

## 1. Objective

Practical 1 built objects that were disposable. A Pod could be deleted and
recreated at no cost because nothing inside it was worth keeping. This
practical deals with the opposite case: workloads whose value *is* the data
they hold, and which therefore need storage whose lifetime is independent of
any Pod.

Two separate bodies of work were needed, and they were done in order.

The first is the **storage layer** — PersistentVolumes, PersistentVolumeClaims
and StorageClasses. These objects answer where bytes are written, and what
happens to those bytes when the Pod, the claim, or the whole workload
disappears. Static provisioning was done first, deliberately, because it
separates two ideas that dynamic provisioning presents together: *matching a
claim to a volume*, and *creating the volume in the first place*.

The second is the **controller built for stateful workloads** — the
StatefulSet. A Deployment treats its Pods as interchangeable copies. A database
replica is not interchangeable with another, because each one owns a specific
disk. Before introducing the StatefulSet, a Deployment with three replicas was
pointed at a single volume so that the failure modes of the wrong tool could be
observed at first hand.

The practical closed by running PostgreSQL 18 as a StatefulSet on a retaining
StorageClass, creating a table, inserting rows, deleting the database Pod, and
reading the rows back through its replacement.

**Descriptor sections covered:** Unit I — 1.2.5, 1.4.1, 1.4.2, 1.4.3, 1.5.3;
Unit II — 2.1.1, 2.1.2, 2.1.3, 2.1.4.
**Learning outcomes:** LO4 primarily; LO1, LO2 and LO3 reinforced.

---

## 2. Environment

| Component | Version |
| --- | --- |
| Operating system | macOS 26.5.2 (build 25F84), arm64 |
| Docker Desktop | 29.0.1 |
| kind | v0.32.0 (go1.26.3 darwin/arm64) |
| kubectl | v1.36.3 (Kustomize v5.8.1) |
| Kubernetes (cluster) | v1.36.1 — `kindest/node:v1.36.1`, pinned by digest |
| Node OS / kernel | Debian GNU/Linux 13 (trixie), 6.12.54-linuxkit (arm64) |
| Container runtime | containerd 2.3.1 |
| Storage provisioner | `rancher.io/local-path` (installed by kind) |

### Container images

| Image | Role | Version observed at runtime |
| --- | --- | --- |
| `busybox:1.37` | Volume writers, init container, client Pod | — |
| `nginx:1.30-alpine` | `webnote` StatefulSet | — |
| `nginx:1.31-alpine` | Rolling-update target (Stage 6) | — |
| `postgres:18-alpine` | The stateful application | PostgreSQL 18.6 on aarch64-unknown-linux-musl |

### Version-skew correction carried over from Practical 1

Practical 1 was carried out with a two-minor version skew, because Docker
Desktop installs its own `kubectl` at `/usr/local/bin/kubectl` and that
directory precedes `/opt/homebrew/bin` on this machine's `PATH`. The Docker
copy (v1.34.1) therefore shadowed the current Homebrew binary.

For this practical the shadowing was corrected at the start of every session:

```bash
export PATH="/opt/homebrew/bin:$PATH"
kubectl version --client        # v1.36.3
```

Client and cluster were then both v1.36, inside the supported skew window.

### One environment difference worth recording

The guide shows host files owned by `root root`. On Docker Desktop for macOS
they appear as `keldendrac wheel`, because VirtioFS maps container UIDs onto
the host user. This is visible in `04-stage2-released-and-data-intact` and
`18-stage8-final-asymmetry`, and is a file-sharing behaviour, not a Kubernetes
one.

---

## 3. Procedure and Observations

### 3.1 Stage 1 — Cluster and the storage landscape

A three-node cluster was created from `cluster/kind-cluster.yaml`, with
`/tmp/dso202-p2-storage` bind-mounted into the first worker at
`/mnt/dso202-static`. The host directory was created **before** the cluster,
because kind binds it at creation time.

**Evidence: `01-stage1-cluster-nodes-mount`.** The creation output includes the
line `Installing StorageClass`, which is the moment kind adds the storage
provisioner as a cluster add-on in the same way it installs a CNI plugin — a
cluster built by hand with `kubeadm` has neither. All three nodes report
`Ready` on v1.36.1 under the names set by the `kubeadmConfigPatches`
(`control-plane`, `worker-node-1`, `worker-node-2`), and the `-L` output
confirms the `dso202/node-index` labels of `1` and `2` on the workers. Those
labels are what `03-pv-static.yaml` selects on. The Docker container names
(`dso202-p2-control-plane`, `dso202-p2-worker`, `dso202-p2-worker2`) differ
from the Node object names, and the mapping between them was needed for every
later `docker exec`. `/mnt/dso202-static` exists inside `dso202-p2-worker`.

**Evidence: `02-stage1-storage-landscape`.** Two StorageClasses exist. Both name
the same provisioner, `rancher.io/local-path`; only their policies differ:

| Class | Reclaim policy | Binding mode | Expansion |
| --- | --- | --- | --- |
| `dso202-retain` | Retain | WaitForFirstConsumer | false |
| `standard` (default) | Delete | WaitForFirstConsumer | false |

Each of those four columns is exercised later: `WaitForFirstConsumer` produces
the Pending claim in Stage 3, `allowVolumeExpansion: false` produces the resize
rejection in the same stage, and the two reclaim policies diverge visibly in
Stage 8. The provisioner Pod runs `1/1` in its own `local-path-storage`
namespace, and its ConfigMap gives the authoritative answer for where volumes
are written: `/var/local-path-provisioner`. The ResourceQuota shows the three
storage constraints added for this practical — a cap on the number of claims
(12), a cap on total requested storage (20Gi), and two per-class caps
(`standard` 12Gi of storage; `dso202-retain` 4 claims).

*This stage shows that dynamic provisioning is performed by an identifiable Pod
writing to an identifiable path, not by the control plane itself.*

### 3.2 Stage 2 — Static provisioning and the meaning of Retain

**Evidence: `03-stage2-static-bind-and-placement`.** The PersistentVolume was
created first and reported `Available`. The claim then bound **immediately**,
at an age of 9 seconds — no waiting, unlike Stage 3. The reason is in the same
frame: `kubectl get storageclass manual` returns
`Error from server (NotFound)`. No StorageClass object named `manual` exists,
so no provisioner and no binding mode were involved. The name is only a
matching label between volume and claim, and the control plane simply found an
`Available` PV whose class name, capacity and access modes satisfied the
request.

The Pod then landed on `worker-node-1` even though `05-pod-static-writer.yaml`
contains no `nodeSelector` and no affinity rules anywhere. The volume placed it
there: the directory exists on one node only, so the PV declares node affinity
for it, and the scheduler had no choice. **A volume that constrains scheduling
is the defining property of node-local storage**, and it is the mechanism
behind the failure in Stage 4.

**Evidence: `04-stage2-released-and-data-intact`.** The Pod was deleted and
recreated; `ledger.txt` then held two lines, the first written by a Pod that no
longer existed. The claim was then deleted, and the PV moved to phase
**`Released`** — not `Available` — with the `CLAIM` column still naming
`dso202-practical-02/pvc-web-static`, a claim that no longer exists. This is
the Retain contract working as designed: Kubernetes will not hand a volume that
may hold one workload's data to the next claim that comes along.

The same frame reads the file **from the host**, showing both lines still
present, then deletes the PersistentVolume and lists the directory again —
`ledger.txt` is still there, 128 bytes. *Deleting a PersistentVolume deleted an
entry in the Kubernetes API. It did not delete a single byte.* This screenshot
also serves as the host-side read for §3.2 generally, since `03` was cropped
before its final `cat`.

**Evidence: `05-stage2-ledger-three-lines`.** All three objects — volume, claim
and Pod — were recreated from scratch, and the file holds **three** timestamped
lines. A brand-new PV object, a brand-new claim and a brand-new Pod adopted
data written before any of them existed.

### 3.3 Stage 3 — Dynamic provisioning, and two uncomfortable truths

**Evidence: `06-stage3-pending-then-bound`.** The claim alone reported
`Pending`, and `kubectl describe` gave the reason rather than leaving it to be
guessed:

```
Normal  WaitForFirstConsumer  ...  persistentvolume-controller
        waiting for first consumer to be created before binding
```

This claim was not broken. The `standard` class uses `WaitForFirstConsumer`, so
the control plane refuses to choose storage until it knows which node the Pod
will run on — choosing earlier would risk creating a volume on a node where the
Pod cannot be scheduled. A Pending claim on such a class, with no Pod referring
to it, is correct behaviour.

Applying the Pod bound the claim within seconds. Three details in the same
frame are worth recording: the PersistentVolume was created automatically and
**nobody wrote a manifest for it**; its name is `pvc-` followed by the claim's
UID (`pvc-7edfd19c-0520-4f8c-92f3-710f98ff2cf7`), which is how a dynamically
provisioned volume is recognised at a glance; and its reclaim policy is
`Delete`, inherited from the class and not from anything in the claim. The
directory on the node encodes all three identities:
`pvc-7edfd19c-…_dso202-practical-02_dynamic-data`.

**Evidence: `07-stage3-capacity-resize-reclaim`.** Two limits of this
environment were demonstrated.

*The requested size is not enforced.* The claim asked for 1Gi;
`df -h /data` inside the container reports a 452.1G filesystem with 372.0G
available. This provisioner records the requested capacity in the API and
ignores it when creating storage. Nothing prevents this Pod from filling the
node disk and evicting unrelated Pods, and a laptop cluster therefore cannot be
used to test how an application behaves when its volume is full.

*The volume cannot grow.* The patch was rejected by the API server:

```
Error from server (Forbidden): persistentvolumeclaims "dynamic-data" is forbidden:
only dynamically provisioned pvc can be resized and the storageclass that
provisions the pvc must support resize
```

The rejection comes from the API server, not the provisioner, and is caused by
`allowVolumeExpansion: false` on the class. The valuable part is that it failed
loudly. The choice of class is a decision about the future of a workload, taken
before any data exists.

Deleting the claim then removed the volume object *and* the directory on the
node — `kubectl get pv` shows only the static volume remaining, and the
`docker exec … ls` prints nothing. One field, `reclaimPolicy` on the
StorageClass, produced the entire difference from Stage 2, and it was chosen by
whoever created the class rather than by whoever wrote the claim.

### 3.4 Stage 4 — Why a Deployment cannot own state

**This stage is a demonstration of failure, not of success.** A Deployment with
three replicas was deliberately pointed at one claim, which is something that
must never be done in production.

**Evidence: `08-stage4-three-observations`.** All three observations are in one
frame.

**Observation 1 — the volume dictated placement.** All three replicas were
scheduled onto `worker-node-2`, although the Deployment expresses no node
preference at all. The claim bound to a volume that exists on one node, so no
other node could accept these Pods. The scheduler's freedom was removed by a
storage decision.

**Observation 2 — there is one set of data, not three.** All three replicas
appended to the *same* `visitors.log` on the *same* volume:

```
2026-08-26T16:29:46Z pod=shared-writer-6f4b987c7b-pzbwb node=worker-node-2
2026-08-26T16:29:46Z pod=shared-writer-6f4b987c7b-4x9t7 node=worker-node-2
2026-08-26T16:29:47Z pod=shared-writer-6f4b987c7b-p9fbd node=worker-node-2
```

A stateless web server can tolerate that. A database cannot, because two
database processes writing to the same data directory corrupt it. Nothing in a
Deployment specification can give each replica its own volume, because the
claim is named once in the Pod template and every replica uses that template.

**Observation 3 — no replica has an identity that survives.** After deleting
all three, the replacements came back as `-8f57s`, `-dbqd7` and `-pp6zb`.
Every name is new. There is no way for an application, a monitoring system or
another Pod to refer to "the first replica" and mean the same process before
and after a restart — and replicated databases require exactly that, because
members must find each other by a name that outlasts any individual Pod.

**A fourth failure that this cluster hid.** On a managed cloud cluster this
experiment normally produces a multi-attach error, because the volume is a
network disk attached to one node and any replica scheduled elsewhere never
starts. Here all three replicas were forced onto the same node, and
`ReadWriteOnce` permits multiple Pods on **one node** — it does not mean one
Pod — so the misconfiguration ran without complaint. A configuration that
appears to work locally and fails in production is exactly why the local
cluster is a teaching environment and not a substitute for one.

### 3.5 Stage 5 — StatefulSets and stable identity

**Evidence: `09-stage5-headless-service`.** The headless Service was applied
*before* the StatefulSet, so that the Pods were addressable from the moment
they became ready. `CLUSTER-IP` reads `None`: no virtual address was allocated
and no load balancing will occur. The Service exists purely to publish DNS
records.

Ordered creation was captured during the recreate in
`14-stage6-delete-sts-keep-data`, where the watch shows `webnote-0` reaching
`1/1 Running` before `webnote-1` is created at all, and `webnote-1` reaching
`1/1` before `webnote-2` appears. That is the `OrderedReady` Pod management
policy, and it is what a database whose members must join an existing cluster
one at a time depends on.

**Evidence: `09b-stage5-claims-and-placement`.** One `volumeClaimTemplate`
produced three separate claims bound to three **different** volumes:

| Claim | Volume |
| --- | --- |
| `content-webnote-0` | `pvc-797204ba-a658-4715-a269-4c3f26e43b90` |
| `content-webnote-1` | `pvc-2bfce558-15a9-4d4e-961a-2109219e0575` |
| `content-webnote-2` | `pvc-a932b403-a100-49b3-9ec6-cfc7bb093323` |

The naming rule is fixed: `<template>-<statefulset>-<ordinal>`. The claims are
selectable with `-l app=webnote` only because the labels were written inside
`volumeClaimTemplates[].metadata`; generated claims inherit nothing else.

The Pods were spread across **both** workers (`webnote-0` on `worker-node-1`,
`webnote-1` and `webnote-2` on `worker-node-2`). Unlike Stage 4, placement was
free, because each ordinal has storage of its own.

**Evidence: `10-stage5-dns-and-private-volumes`.** One name resolved to three
addresses:

```
Name: webnote.dso202-practical-02.svc.cluster.local  → 10.244.1.6
Name: webnote.dso202-practical-02.svc.cluster.local  → 10.244.2.16
Name: webnote.dso202-practical-02.svc.cluster.local  → 10.244.2.14
```

A ClusterIP Service would have returned a single virtual address instead,
hiding the individual Pods entirely. (The busybox `nslookup` in this image
prints the addresses without the per-Pod FQDN labels shown in the guide; the
individual names were confirmed instead by fetching each Pod directly.)

A line was then appended by hand to `webnote-0`'s page only. `webnote-0`
returns it; `webnote-1` does not. Three replicas of one workload, three
different files — precisely what Stage 4 could not achieve.

**Evidence: `11-stage5-identity-survives-deletion`.** `webnote-1` was deleted
and its replacement examined. Four facts, all in one frame:

1. **The name is unchanged** — still `webnote-1`.
2. **The claim was reattached, not recreated** — `content-webnote-1` is still
   bound to the same volume `pvc-2bfce558-…`, and its age went 20m → 21m,
   spanning the deletion.
3. **The `created:` line still carries its original timestamp**,
   `2026-08-26T16:32:53Z`, so the file was not regenerated; a second
   `started: 2026-08-26T16:53:42Z` was appended.
4. **The IP address changed**, `10.244.2.14` → `10.244.2.17`.

Point 4 is exactly why an application must be configured with the DNS name and
never with an address.

### 3.6 Stage 6 — Scaling, retention and ordered updates

**Evidence: `12-stage6-scale-retain-and-timestamp`.** Scaling to four replicas
created a fourth claim from the same template — scaling a StatefulSet *up*
creates storage, and the namespace quota is what stands between a mistyped
`--replicas=400` and a full disk.

After scaling **down** to two, four claims remained for two Pods:

```
content-webnote-0  Bound  ...  24m
content-webnote-1  Bound  ...  24m
content-webnote-2  Bound  ...  23m
content-webnote-3  Bound  ...  2m23s
```

The data of the removed replicas was kept because `whenScaled: Retain` is set
(and is also the default). This is a safety decision, not an oversight: scaling
down is frequently a reaction to a problem, and destroying data during an
incident is unrecoverable. The cost is that unused claims accumulate and must
be removed deliberately — which Stage 8 does.

Scaling back to three brought `webnote-2` back with its **original**
`created: 2026-08-26T16:33:20Z` and a new `started: 2026-08-26T16:57:22Z`.
Ordinal 2 was deleted and recreated, and it reclaimed the volume that belonged
to ordinal 2 *by name*. **This is the single most important behaviour in the
practical: identity is what links a replica to its data, and the ordinal is the
identity.**

**Evidence: `12b-stage6-descending-termination`.** Scaling from three to one
shows termination in strictly descending ordinal order, one Pod at a time:
`webnote-2` reached `Completed` before `webnote-1` even began terminating. The
highest ordinal is always removed first, so the members that remain are always
`0` through `n-1`. A database that designates ordinal 0 as its initial primary
depends on this.

**The partitioned rolling update.** `manifests/10-statefulset-webnote.yaml` was
edited — not patched imperatively — to set
`updateStrategy.rollingUpdate.partition: 2` and change the image from
`nginx:1.30-alpine` to `nginx:1.31-alpine`. Applying it updated **only ordinal
2**, leaving ordinals 0 and 1 on the old image, because `partition: 2`
instructs the controller to update Pods whose ordinal is greater than or equal
to 2. `kubectl rollout status` reported
`partitioned roll out complete: 1 new pods have been updated...`. `partition`
was then returned to `0` in the same file and reapplied, at which point
`rollout status` reported `3 new pods have been updated...` and the remaining
Pods were updated in descending order, one at a time. This is how a new version
is tried on one member of a stateful set before the rest is committed to it,
and it has no equivalent in a Deployment.

> **Evidence gap, stated honestly.** The intermediate frame — the moment when
> only `webnote-2` carried `nginx:1.31-alpine` — was not captured. The first
> screenshot attempt caught only the completed rollout, and a second attempt
> was applied before the manifest edit had been made, so it recorded no change
> at all. Rather than submit a screenshot that shows something other than what
> it claims, the misleading file was deleted. The change itself is evidenced by
> the manifest and its git history. `14-stage6-delete-sts-keep-data` confirms
> the end state: all Pods running `nginx:1.31-alpine`.

**Evidence: `14-stage6-delete-sts-keep-data`.** The StatefulSet itself was
deleted. Every Pod went (`No resources found`), and **every claim remained** —
`wc -l` returns `4` — because `whenDeleted: Retain` is set. On this cluster,
deleting a StatefulSet is a recoverable mistake.

Recreating it brought the workload back with its data intact. `webnote-1` still
serves `created: 2026-08-26T16:32:53Z` — the timestamp written back in Stage 5,
before the scale-up, the scale-down, the rolling update to a new nginx version,
and the deletion of the controller itself — with one further `started:` line
appended for each of those events:

```
<h2>webnote-1</h2>
created: 2026-08-26T16:32:53Z on worker-node-2
started: 2026-08-26T16:32:53Z on worker-node-2
started: 2026-08-26T16:53:42Z on worker-node-2
started: 2026-08-26T17:35:05Z on worker-node-2
started: 2026-08-26T17:39:42Z on worker-node-2
started: 2026-08-26T17:43:21Z on worker-node-2
```

That file is the clearest single piece of evidence in the practical: the volume
outlived every object that ever used it.

### 3.7 Stage 7 — A real stateful application: PostgreSQL

**Evidence: `15-stage7-postgres-ready-and-storage`.**

*The Secret.* `postgres-credentials` was created with three keys, and the
username was recovered in plain text with a single command:

```
kubectl get secret postgres-credentials -o jsonpath='{.data.POSTGRES_USER}' | base64 -d
taskuser
```

A Secret keeps credentials out of the workload manifest and out of container
images. **It is base64-encoded, not encrypted.** Restricting who may read
Secrets is RBAC (Unit II 2.3); keeping credentials out of Git entirely is
Unit III tooling.

*Two Services, both needed.* `postgres-headless` has `CLUSTER-IP None` and
gives the database Pod its stable per-Pod DNS name; it is what `serviceName`
points at. `postgres` is an ordinary ClusterIP Service on `10.96.184.125`, and
is the name an application places in its connection string so that it need not
know how many replicas exist or which ordinal is currently primary.

*Readiness.* The Pod reached `1/1 Running`, and the log ends with
`database system is ready to accept connections`. The gap between the container
starting and the Pod becoming `1/1` is the readiness probe doing its work — and
during that window no Service would have routed a connection to it. A database
without a readiness probe is added to its Service the moment its process
starts, which produces connection errors in the application while
initialisation is still running.

*Storage.* `data-postgres-0` is `Bound`, **2Gi**, on the **`dso202-retain`**
class, and its PersistentVolume carries `RECLAIM POLICY Retain` — inherited
from the class, not from the claim. This is the standard production choice for
a database: an accidental `kubectl delete pvc` must not be able to destroy the
data. The corresponding cost appears in Stage 8.

*The mount path.* `echo "$PGDATA"` returns `/var/lib/postgresql/18/docker`
while the volume is mounted one level above at `/var/lib/postgresql`, where
`ls` shows a single entry, `18`. From PostgreSQL 18 onward the official image
places its data in a version-numbered subdirectory. The general rule behind
this matters for every database: a freshly provisioned volume may contain
entries the storage driver placed there, and `initdb` refuses to initialise a
directory that is not empty.

**Evidence: `16-stage7-rows-survive-and-dns`.** This is the test the whole
practical builds towards.

```
CREATE TABLE
INSERT 0 3

 id |        title         | done
----+----------------------+------
  1 | Complete Practical 2 | f
  2 | Read Unit II notes   | f
  3 | Draft the report     | f
(3 rows)
```

The Pod was then **deleted**, its replacement waited for, and the table
queried again:

```
pod "postgres-0" deleted
pod/postgres-0 condition met

 count
-------
     3
(1 row)
```

**Three rows, written by a process that no longer exists, read back through a
Pod that did not exist when they were written.**

The restart log confirms what happened at the storage layer:

```
PostgreSQL Database directory appears to contain a database; Skipping initialization
LOG:  starting PostgreSQL 18.6 on aarch64-unknown-linux-musl
LOG:  database system was shut down at 2026-08-26 17:48:22 UTC
```

There is no `init process complete` line this time, because the data directory
already existed and initialisation was skipped. The clean-shutdown message —
rather than a recovery message — is the result of
`terminationGracePeriodSeconds: 60`, which gave the database time to finish
writing and close its files. Too short a grace period means the process is
killed and must replay its write-ahead log on the next start.

Both Service names resolve, and they resolve to different things:

| Name | Resolves to | Used by |
| --- | --- | --- |
| `postgres.dso202-practical-02.svc.cluster.local` | `10.96.184.125` (ClusterIP) | An application's connection string |
| `postgres-0.postgres-headless.dso202-practical-02.svc.cluster.local` | `10.244.2.27` (Pod IP) | A backup job or replication peer needing one specific instance |

### 3.8 Stage 8 — Cleanup, and the cost of Retain

Evidence was captured before anything was deleted:
`final-state-all.txt`, `final-state-storage.txt`,
`final-statefulset-webnote.yaml`, `final-state-events.txt`, and
`tasktracker-dump.sql` (2,245 bytes, containing the `tasks` table definition
and `setval('public.tasks_id_seq', 3, true)`). **The dump is the backup; the
retained volume is not**, because a single mistaken command can destroy a
volume and its data together.

**Evidence: `17-stage8-reclaim-policies-diverge`.** Every workload was deleted —
both StatefulSets, the client Pod and the static writer — and `kubectl get pods`
returned `No resources found`. **Six claims survived**, holding storage for
workloads that no longer existed. Deleting a StatefulSet does not delete the
claims it generated, and `kubectl delete -f manifests/` never will, because
those claims were never in a manifest.

`kubectl delete pvc --all` then made the two reclaim policies diverge visibly:

| Volume | Class | Policy | Outcome |
| --- | --- | --- | --- |
| `pvc-797204ba…` | `standard` | Delete | gone with its claim |
| `pvc-2bfce558…` | `standard` | Delete | gone with its claim |
| `pvc-a932b403…` | `standard` | Delete | gone with its claim |
| `pvc-c72f3117…` | `standard` | Delete | gone with its claim |
| `pv-web-static` | `manual` | Retain | **`Released`**, data intact |
| `pvc-d27e24a6…` | `dso202-retain` | Retain | **`Released`**, data intact |

Four volumes and their data were destroyed; two remain, still naming their
deleted claims. Whoever chose the class made that decision before any data
existed.

**Evidence: `18-stage8-final-asymmetry`.** The static PV was deleted, leaving
only the released PostgreSQL volume. The cluster was then destroyed:

```
Deleting cluster "dso202-p2" ...
Deleted nodes: ["dso202-p2-control-plane" "dso202-p2-worker" "dso202-p2-worker2"]
No kind clusters found.
```

`docker ps` shows no container at all. Every dynamically provisioned volume
went with the nodes, **including the PostgreSQL data directory**, because it
lived inside a node. And yet:

```
$ ls -l /tmp/dso202-p2-storage/pv-web-static/
-rw-r--r--  1 keldendrac  wheel  192 Aug 26 22:26 ledger.txt

$ cat /tmp/dso202-p2-storage/pv-web-static/ledger.txt
2026-08-26T16:24:17Z start pod=static-writer node=worker-node-1
2026-08-26T16:25:07Z start pod=static-writer node=worker-node-1
2026-08-26T16:26:26Z start pod=static-writer node=worker-node-1
```

The cluster is gone, the nodes are gone, and the statically provisioned data is
still on the host, because it never lived inside the cluster at all. **This is
the sharpest available illustration of what a PersistentVolume object is: a
description of storage, and never the storage itself.**

One command in this frame produced no output:
`docker exec dso202-p2-worker ls /var/local-path-provisioner`. That is not a
failure — `postgres-0` had been scheduled onto **worker-node-2** (its Pod IP
was `10.244.2.27`), so the released directory was on `dso202-p2-worker2` and
the command interrogated the wrong node. The general point it was meant to make
still holds and is a cloud-cost consideration: storage released by Kubernetes
remains occupied until somebody removes it.

---

## 4. Analysis

**Q1. The claim in Stage 3 was Pending immediately after creation, while the
claim in Stage 2 bound at once. Name the single field responsible and explain
the reasoning behind its design.**

The field is `volumeBindingMode` on the StorageClass, set to
`WaitForFirstConsumer` (`02-stage1-storage-landscape`).

The Stage 2 claim named `storageClassName: manual`, and **no StorageClass
object of that name exists** — `03-stage2-static-bind-and-placement` shows the
`NotFound` error. With no class object there is no binding mode to apply, so
the control plane matched the claim against `Available` volumes immediately and
bound it within seconds.

The Stage 3 claim named `standard`, whose binding mode is
`WaitForFirstConsumer`, so binding was deferred until a Pod existed
(`06-stage3-pending-then-bound`). The design reasoning is that this provisioner
creates node-local storage. If the volume were created as soon as the claim
appeared, it could land on a node the Pod cannot be scheduled to — because of
taints, resource pressure or affinity rules — and the Pod would then be
permanently unschedulable with a `volume node affinity conflict`. Deferring
until the scheduler has picked a node guarantees the volume is created where
the Pod will actually run. The cost is that a Pending claim with no consumer is
*correct*, and is routinely misdiagnosed as a broken provisioner.

**Q2. After the claim was deleted, the Stage 2 data survived and the Stage 3
data did not. State which object carried the deciding field, and who in a real
organisation would have chosen its value.**

Two different objects carried it, which is itself the point.

For the **static** volume the field is `persistentVolumeReclaimPolicy: Retain`,
written directly on the PersistentVolume in `03-pv-static.yaml`. For the
**dynamic** volume the field is `reclaimPolicy` on the StorageClass — the
`standard` class sets `Delete`, and every PV it creates inherits it. Nothing in
the claim can override this: `06-stage3-pending-then-bound` shows the
auto-created PV carrying `Delete` although the claim never mentions a policy.

In a real organisation both are chosen by a **platform or cluster
administrator**, not by the developer who writes the claim. PersistentVolumes
and StorageClasses are cluster-scoped, so creating them is an administrator's
task by design. The consequence is uncomfortable: a developer deleting a claim
may destroy data, or may leave expensive cloud storage billing indefinitely,
according to a decision somebody else made weeks earlier before any data
existed.

**Q3. In Stage 4 all three replicas were scheduled onto one node although the
Deployment expressed no node preference. Explain the mechanism, and state what
would have happened on a managed cloud cluster using a zonal disk.**

The mechanism is a chain. The Deployment names one PVC in its Pod template, so
all three replicas use the same claim. That claim is on the `standard` class,
whose provisioner creates a directory on **one specific node** — the node the
first Pod was scheduled to. The resulting PersistentVolume declares node
affinity for that node. From then on the scheduler cannot place any Pod using
that claim anywhere else, so all three replicas landed on `worker-node-2`
(`08-stage4-three-observations`). The storage decision removed the scheduler's
freedom entirely.

On a managed cloud cluster with a zonal disk (EBS, Persistent Disk), the volume
is a network device attachable to **one node at a time**. The first replica
would start; the others, scheduled onto different nodes, would remain
`ContainerCreating` indefinitely with a **multi-attach error** in their events
— `Volume is already exclusively attached to one node and can't be attached to
another`. The misconfiguration would fail loudly there. Locally it succeeded
silently, because all three Pods were forced onto one node and `ReadWriteOnce`
means one *node*, not one *Pod*.

**Q4. Give the fully qualified DNS name of the second replica of the webnote
StatefulSet, and name every object that must exist for it to resolve.**

```
webnote-1.webnote.dso202-practical-02.svc.cluster.local
```

Following the pattern `<pod>.<service>.<namespace>.svc.cluster.local`. Note
that the *second replica* is ordinal 1, since ordinals start at 0.

Every one of the following must exist:

1. **The Pod `webnote-1`** — and it must be **Ready**, because
   `publishNotReadyAddresses` is `false` (the default), so unready Pods are
   withheld from DNS.
2. **The headless Service `webnote`** — it must have `clusterIP: None`. A
   normal ClusterIP Service publishes one virtual address and no per-Pod
   records at all.
3. **A matching selector** — the Service's `selector: app=webnote` must match
   the Pod's labels, or the Pod is not an endpoint.
4. **The StatefulSet's `serviceName: webnote`** — this field must name that
   exact Service. Without it the per-Pod records are never created, even though
   the Pods are healthy.
5. **The namespace `dso202-practical-02`**, which forms the third label.
6. **The EndpointSlice** for the Service, maintained by the endpoint
   controller, from which the DNS records are derived.
7. **The cluster DNS server** (CoreDNS, at `10.96.0.10` here), and the Pod's
   `/etc/resolv.conf` pointing at it.

**Q5. The StatefulSet was scaled from four replicas to two and back to three.
Describe what happened to the claims at each step, name the two governing
fields, and state their defaults.**

| Step | Pods | Claims | What happened |
| --- | --- | --- | --- |
| Scale to 4 | 4 | 4 | A fourth claim, `content-webnote-3`, was **created** from the template |
| Scale to 2 | 2 | **4** | Two Pods were deleted; **no claim was touched** |
| Scale to 3 | 3 | 4 | `webnote-2` was recreated and **reattached its existing claim by name** |

The evidence is in `12-stage6-scale-retain-and-timestamp`: four claims listed
while only two Pods ran, and `webnote-2` returning with its **original**
`created: 2026-08-26T16:33:20Z`.

The two fields are both under `persistentVolumeClaimRetentionPolicy`:

- **`whenScaled`** — governs claims of Pods removed by a scale-down. Default
  **`Retain`**.
- **`whenDeleted`** — governs claims when the StatefulSet itself is deleted.
  Default **`Retain`**.

Both were set explicitly to `Retain` in `10-statefulset-webnote.yaml`, matching
the defaults, because a reader should see a safety decision rather than assume
it. Setting either to `Delete` makes an ordinary operational action destroy
data.

**Q6. Listing 16 mounts the volume at `/var/lib/postgresql` rather than at the
data directory. Explain why, and describe the failure that mounting at the data
directory would produce on a volume that is not empty.**

From PostgreSQL 18 the official image places its data directory at
`/var/lib/postgresql/18/docker` and declares `/var/lib/postgresql` as the
volume location. Mounting at the parent therefore puts the data inside the
volume while leaving the data directory itself one level down — confirmed in
`15-stage7-postgres-ready-and-storage`, where `$PGDATA` is
`/var/lib/postgresql/18/docker` and `ls /var/lib/postgresql` shows only `18`.

The general rule matters more than the version detail. `initdb` **refuses to
initialise a directory that is not empty**, and a freshly provisioned volume is
not guaranteed to be empty — a storage driver may leave entries behind, most
famously `lost+found` on a formatted ext4 volume. Mounting the volume directly
onto the data directory therefore makes the database's first start depend on
which storage backend is underneath, which is the worst possible property: it
succeeds on some and fails on others.

The failure is specific and diagnosable. The container exits during
initialisation with a message that the data directory is not empty, the Pod
enters **`CrashLoopBackOff`**, and — the detail that makes it confusing — it
only ever fails on the *first* start, because a successfully initialised volume
never hits the check again. It is read with
`kubectl logs postgres-0 --previous`. For PostgreSQL 17 and earlier, whose data
directory is `/var/lib/postgresql/data`, the equivalent fix is to mount at that
path and set `subPath: pgdata` on the mount.

**Q7. State two things a StatefulSet does not provide for a database, and name
the mechanism or software category that provides each in production.**

**It does not replicate data.** Three replicas produce three independent
volumes containing three unrelated sets of data — demonstrated directly in
`10-stage5-dns-and-private-volumes`, where a line written into `webnote-0` was
absent from `webnote-1`. Replication is a property of the *application*, not of
the controller, and this is why `14-statefulset-postgres.yaml` sets
`replicas: 1`: a second replica of that manifest would be a second, empty,
unrelated database. In production, replication and the leader election that
goes with it are handled by an **Operator** — a controller that understands one
specific database, such as CloudNativePG or Zalando's Postgres Operator
(Unit II 2.4).

**It does not perform backup.** A volume that survives Pod deletion is not a
backup, because one mistaken command destroys the volume and the data together
— and Stage 8 showed four volumes vanish with a single `kubectl delete pvc
--all`. Nothing a StatefulSet does protects against a dropped table, since the
drop is faithfully persisted. Production backup is a **logical dump or
snapshot held outside the cluster**: `pg_dump` on a CronJob shipping to object
storage, continuous WAL archiving for point-in-time recovery, or
`VolumeSnapshot` objects driven by a CSI driver. `tasktracker-dump.sql` is the
practical's example.

**Q8. After the claims were deleted in Stage 8, two PersistentVolumes reported
`Released`. Explain why the phase was not `Available`, and state what an
administrator must do to return that storage to service.**

`Released` means the claim is gone but the volume **has not been reclaimed**,
and the volume still holds the previous workload's data. Kubernetes will not
move it to `Available`, because doing so would allow the next claim that
happens to match on class, capacity and access modes to bind to it — and that
claim might belong to an entirely different team or application. Handing one
workload's data to another silently is precisely what the Retain policy exists
to prevent. `17-stage8-reclaim-policies-diverge` shows both volumes still
naming their deleted claims in the `CLAIM` column, which is the record of the
association that blocks rebinding.

The consequence is that a `Released` volume is invisible to the binding
process: a new claim of the right size will stay `Pending` forever even though
a suitable volume is plainly listed.

To return the storage to service the administrator must act deliberately, and
must decide about the data first:

1. **Recover or destroy the data**, according to whether it is still needed.
2. Then either **delete the PV object and recreate it** pointing at the same
   backing storage — which is what `18-stage8-final-asymmetry` does, deleting
   `pv-web-static` and leaving `ledger.txt` untouched on the host — **or**
   clear the stale binding in place with
   `kubectl patch pv <name> -p '{"spec":{"claimRef": null}}'`, which returns
   the volume to `Available` with its data still on it.
3. **Delete the underlying directory or disk separately** if the data is not
   wanted. Removing the API object removes nothing from the node, which is the
   difference between a deleted cluster and a continuing cloud invoice.

---

## 5. Reflection

### What was difficult

The hardest idea was not any single command but the **separation between a
Kubernetes object and the storage it describes**. Deleting `pv-web-static` and
then finding `ledger.txt` still on the host made that concrete in a way no
definition had. Related to it, the three-way split of responsibility took time
to hold in mind at once: the developer writes the claim, the administrator
writes the class, and the *class* decides whether the developer's data survives
the developer's own `kubectl delete`.

The second difficulty was accepting that a `Pending` claim can be correct.
Every instinct said something was broken, and the fix was to stop guessing and
run `kubectl describe pvc` — the event said `waiting for first consumer to be
created before binding`, which is a statement of policy, not an error.

### Errors met, and how they were diagnosed

**1. `zsh: no matches found: custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image`**

This looked at first like a malformed `kubectl` flag. It is not a `kubectl`
error at all — nothing was ever sent to the API server. The clue is the prefix
`zsh:`. The shell tried to expand `[0]` as a glob character class, found no
matching filename, and refused to run the command. The fix is to quote the
argument:

```bash
kubectl get pods -l app=webnote -o 'custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image'
```

The generalisable lesson is to read the *prefix* of an error message before its
text. Any `jsonpath` or `custom-columns` expression containing `[0]` needs
quoting in zsh, and the same applies to `kubectl explain ...[0]` paths.

**2. `Error from server (NotFound): pods "webnote-2" not found`**

Running `kubectl scale statefulset webnote --replicas=3` followed immediately
by `kubectl wait --for=condition=Ready pod/webnote-2` failed. The Pod genuinely
did not exist yet — under `podManagementPolicy: OrderedReady` the controller
brings up `webnote-1` and waits for it to be Ready before creating `webnote-2`,
so there was nothing to wait *for* at the instant `wait` ran. This is a race
between the command and the controller, not a fault.

The fix was to wait on the **controller** rather than on a named Pod:

```bash
kubectl rollout status statefulset/webnote --timeout=300s
```

`rollout status` tracks the StatefulSet's own progress and tolerates Pods that
do not exist yet. The wider lesson is that `kubectl wait` requires its target
to already exist, which makes it a poor tool for anything a controller creates
in sequence.

**3. Nodes captured as `NotReady`**

The first attempt at the Stage 1 screenshot ran `kubectl get nodes` about 13
seconds after `kind create cluster` returned, and all three nodes showed
`NotReady`. Nothing was wrong — the CNI plugin had not finished installing, and
a node without a working pod network is correctly reported as not ready.
`kind create cluster` returns when the control plane answers, not when the
cluster is fully converged. Inserting a deterministic wait fixed it:

```bash
kubectl wait --for=condition=Ready nodes --all --timeout=120s
```

**4. `docker exec dso202-p2-worker ls /var/local-path-provisioner` printed nothing**

In Stage 8 this was expected to list the released PostgreSQL directory. The
diagnosis was to check where the Pod had actually run: `postgres-0` had a Pod IP
of `10.244.2.27`, which is in `worker-node-2`'s range, so the directory was on
`dso202-p2-worker2`. The command had interrogated the wrong node. Node-local
storage exists on exactly one node, and the node must be read from the Pod
rather than assumed.

### What would be done differently

**Capture evidence in smaller frames.** The one genuine gap in this submission
is the intermediate step of the partitioned rolling update. Two attempts failed
for different reasons — the first captured only the completed rollout, and the
second applied the manifest before the edit had been made, recording no change
at all. Both failures share a cause: trying to capture a multi-step sequence in
one screenshot, taken after the fact, instead of one screenshot per observable
state as it happened. Screenshots would be taken *at* each state next time, not
reconstructed at the end.

**Redirect output to files before screenshotting.** The `.txt` artefacts
captured in Stage 8 proved more reliable than any screenshot: they cannot be
cropped, they do not wrap, and they can be re-read afterwards. Running each
significant command through `tee evidence/<name>.txt` would have made the
screenshots a convenience rather than the primary record — and would have made
the partition gap recoverable.

**Fix `PATH` permanently rather than per-session.** The Docker Desktop
`kubectl` shadowing the Homebrew one was carried over from Practical 1 and
worked around with an `export` in each shell. One line in `~/.zshrc` would
remove the whole class of problem.

### What remains unclear

How a `Released` volume is handled at scale. Clearing `spec.claimRef` by hand
is obviously fine for two volumes on a laptop, but a production cluster
retiring hundreds of retained volumes cannot be operated that way, and it is
not clear what the real practice is — whether it is a controller, a scheduled
reconciliation job, or simply an accepted operational cost that pushes teams
towards `Delete` plus disciplined snapshots.

Relatedly, the boundary between what a StatefulSet handles and what needs an
Operator is clear in the two extreme cases — plain storage on one side,
failover and resharding on the other — but not in the middle. It is not obvious
where a simple primary/replica PostgreSQL pair falls, or how much of that can
honestly be assembled from a StatefulSet, a headless Service and an init
container before an Operator becomes the correct answer rather than an
indulgence.

---

## 6. References

All accessed 26 August 2026.

1. Kubernetes Documentation — *Persistent Volumes*.
   https://kubernetes.io/docs/concepts/storage/persistent-volumes/
   (Used for reclaim policies, PV phases, and the meaning of `Released`.)
2. Kubernetes Documentation — *Storage Classes*.
   https://kubernetes.io/docs/concepts/storage/storage-classes/
   (Used for `volumeBindingMode`, `allowVolumeExpansion` and provisioner names.)
3. Kubernetes Documentation — *StatefulSets*.
   https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
   (Used for the four guarantees, `podManagementPolicy` and
   `persistentVolumeClaimRetentionPolicy`.)
4. Kubernetes Documentation — *StatefulSet Basics* and
   *Update a StatefulSet*.
   https://kubernetes.io/docs/tasks/run-application/tune-statefulset-rolling-update/
   (Used for the `partition` field and partitioned rollouts.)
5. Kubernetes Documentation — *DNS for Services and Pods*.
   https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
   (Used for headless Services and the per-Pod FQDN pattern.)
6. Kubernetes Documentation — *Resource Quotas*.
   https://kubernetes.io/docs/concepts/policy/resource-quotas/
   (Used for the per-StorageClass quota key format.)
7. kind Documentation — *Configuration*.
   https://kind.sigs.k8s.io/docs/user/configuration/
   (Used for `extraMounts`, `kubeadmConfigPatches` and node naming.)
8. rancher/local-path-provisioner — README.
   https://github.com/rancher/local-path-provisioner
   (Used to confirm the node path and that requested capacity is not enforced.)
9. Docker Hub — *postgres* official image documentation.
   https://hub.docker.com/_/postgres
   (Used for `PGDATA`, the PostgreSQL 18 data-directory change, and the
   `POSTGRES_*` environment variables.)
10. PostgreSQL 18 Documentation — *pg_dump* and *Server Start-up*.
    https://www.postgresql.org/docs/18/
    (Used for the dump format and the shutdown/recovery log messages.)
11. `kubectl explain` against the running v1.36.1 API server, for
    `persistentvolumeclaim.spec`, `statefulset.spec.updateStrategy.rollingUpdate`
    and `statefulset.spec.persistentVolumeClaimRetentionPolicy`. The schema from
    the live API server is authoritative for the version in use.

---

## Appendix — Evidence index

| File | Stage | What it establishes |
| --- | --- | --- |
| `01-stage1-cluster-nodes-mount` | 1 | Cluster created, `Installing StorageClass`, three nodes Ready, node-index labels, host mount inside `dso202-p2-worker` |
| `02-stage1-storage-landscape` | 1 | Both StorageClasses and their four policy columns, provisioner Pod, node path, storage quotas |
| `03-stage2-static-bind-and-placement` | 2 | Immediate binding with no provisioner; `manual` class does not exist; volume dictated Pod placement |
| `04-stage2-released-and-data-intact` | 2 | PV phase `Released` with stale claim; deleting a PV deletes no data |
| `05-stage2-ledger-three-lines` | 2 | Three lines; new PV, claim and Pod adopted pre-existing data |
| `06-stage3-pending-then-bound` | 3 | `WaitForFirstConsumer` event; auto-created PV with `Delete` policy; node directory |
| `07-stage3-capacity-resize-reclaim` | 3 | Requested capacity not enforced; resize `Forbidden`; Delete reclaim observed |
| `08-stage4-three-observations` | 4 | One node, one shared log file, regenerated Pod names |
| `09-stage5-headless-service` | 5 | `CLUSTER-IP None` |
| `09b-stage5-claims-and-placement` | 5 | Three separate claims and volumes; placement free across both workers |
| `10-stage5-dns-and-private-volumes` | 5 | One name → three addresses; per-Pod private content |
| `11-stage5-identity-survives-deletion` | 5 | Name kept, claim reattached, `created:` preserved, IP changed |
| `12-stage6-scale-retain-and-timestamp` | 6 | Four claims for two Pods; returning `created:` timestamp |
| `12b-stage6-descending-termination` | 6 | Descending ordinal termination, one at a time |
| `14-stage6-delete-sts-keep-data` | 5, 6 | Ordered creation; controller deleted with all claims retained; data intact across the whole stage |
| `15-stage7-postgres-ready-and-storage` | 7 | Secret decoded; two Services; readiness; retaining class; `PGDATA` one level below the mount |
| `16-stage7-rows-survive-and-dns` | 7 | `count = 3` after Pod deletion; initialisation skipped; both Service names resolve |
| `17-stage8-reclaim-policies-diverge` | 8 | Six claims survive workload deletion; four volumes gone, two `Released` |
| `18-stage8-final-asymmetry` | 8 | Cluster and containers gone; host data survives |
| `final-state-all.txt` | 8 | All objects before cleanup |
| `final-state-storage.txt` | 8 | PVs, PVCs and StorageClasses before cleanup |
| `final-statefulset-webnote.yaml` | 8 | The StatefulSet as stored by the API server |
| `final-state-events.txt` | 8 | Namespace event history |
| `tasktracker-dump.sql` | 8 | Logical backup of the `tasks` table, held outside the cluster |

*No screenshot numbered 13 is submitted; see the evidence gap noted in §3.6.*
