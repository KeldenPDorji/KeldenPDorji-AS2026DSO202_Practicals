# DSO202 - Practical 4 Report

**GitOps with Argo CD: Git as the Single Source of Truth**

| Field | Detail |
| --- | --- |
| Module | DSO202 - Scaling, Orchestration, Monitoring & Observability |
| Programme | BE in Software Engineering |
| Practical | 4 of 10 |
| Tool | Argo CD v3.5.6 on kind (Kubernetes IN Docker) |
| Date carried out | 30 October 2026 |
| Repository path | `dso202-practical-04/` |
| GitOps repository | https://github.com/KeldenPDorji/dso202-gitops-demo |

All screenshots referenced below are in `evidence/`, numbered in the order they
were captured. A suffix `b` marks a frame added to complete evidence that the
planned frame did not capture. Clock times in terminal output are local time
(UTC+6). Times from Argo CD and Kubernetes objects end in `Z` and are UTC.

---

## 1. Objective

Practicals 5 and 6 produced manifests with Kustomize and Helm, but in both
cases a person still ran `kubectl apply` or `helm upgrade` from a laptop. After
that command finished, nothing compared the cluster with the files again. A
change made directly in the cluster stayed there, and the files and the
cluster slowly drifted apart.

GitOps removes the person from the deployment path. The desired state is kept
in a Git repository, and an agent **inside** the cluster pulls it and keeps
the cluster equal to it. Git becomes the only place where changes are made.
This practical installs Argo CD as that agent and tests the four OpenGitOps
principles directly:

- **Stage 0:** the three-node `dso202` cluster from Practical 1;
- **Stage 1:** installing Argo CD in its own `argocd` namespace;
- **Stage 2:** reaching the Argo CD UI through a port-forward;
- **Stage 3:** a separate GitHub repository holding only Kubernetes manifests;
- **Stage 4:** registering the application **declaratively**, as an
  `Application` custom resource applied with `kubectl`;
- **Experiment 1:** scaling from 2 to 4 replicas with a Git commit only;
- **Experiment 2:** a "rogue admin" `kubectl scale`, and Argo CD's self-heal;
- **Experiment 3:** deleting a manifest from Git, the automatic prune, and
  restoring it with `git revert`.

**Descriptor sections covered:** GitOps principles; pull-based deployment;
Argo CD architecture, sync status, health status and sync policies.

---

## 2. Environment

| Component | Version |
| --- | --- |
| Operating system | macOS 26.6.2 (build 25G83), arm64 |
| Docker Desktop | 29.7.2 |
| kind | v0.32.0 (go1.26.3 darwin/arm64) |
| kubectl | v1.36.3 (Kustomize v5.8.1) |
| Kubernetes (cluster) | v1.36.1 - `kindest/node:v1.36.1` |
| **Argo CD** | **v3.5.6** - `quay.io/argoproj/argocd:v3.5.6` |
| Git | GitHub, public repository over HTTPS |

### Container images

| Image | Used by |
| --- | --- |
| `quay.io/argoproj/argocd:v3.5.6` | Argo CD server, repo server, application controller and the other controllers |
| `nginxdemos/hello:plain-text` | The `webapp-deploy` Pods. It replies with the name of the Pod that served the request |
| `busybox:1.37` | The short-lived `curl-test` Pod used to call the Service from inside the cluster |

### Differences from the handout

| Handout | This practical | Reason |
| --- | --- | --- |
| `kubectl apply -n argocd -f …/stable/manifests/install.yaml` | `kubectl apply -n argocd --server-side --force-conflicts -f …/v3.5.6/manifests/install.yaml` | The ApplicationSet CRD is too large for client-side apply (§3.2). Pinning `v3.5.6`, which `stable` pointed to on the day, keeps the report reproducible |
| `kubectl wait --for=condition=Ready pods --all` | `kubectl wait --for=condition=Available deployment --all`, then `kubectl rollout status statefulset/argocd-application-controller` | Waits on the controllers that own the Pods, so it cannot pass early on a Pod that has not been created yet |
| Password printed with `base64 -d; echo` | `base64 -d \| pbcopy` | The password goes to the clipboard and never appears in a screenshot |
| Port-forward in a second terminal window | Port-forward as a background job | Same effect, one terminal |
| Repository named `dso202-gitops-demo` in one step and `k8s-gitops-demo` in the next | `dso202-gitops-demo` throughout | The handout's two names would make `repoURL` point at a repository that does not exist |
| Edit on GitHub, then "click Refresh" and "watch the graph" | Commit and push from the terminal, then `kubectl annotate … argocd.argoproj.io/refresh=normal` and `kubectl wait` on the commit hash | The annotation is what the Refresh button sets. Waiting on the exact hash is deterministic and leaves terminal evidence |
| Expected node version `v1.31.0` | `v1.36.1` | The cluster image used in every practical this term |

### Repository layout

```
dso202-practical-04/
├── cluster/kind-cluster.yaml          # Practical 1 Listing 1: cluster "dso202"
├── gitops-repo/manifests/             # seed copy of what was pushed to GitHub
│   ├── deployment.yaml                # webapp-deploy, 2 replicas (handout, verbatim)
│   └── service.yaml                   # webapp-svc, ClusterIP port 80
├── argocd/argocd-app.yaml             # the Application custom resource
├── evidence/                          # screenshots
└── report/practical-04-report.md      # this report
```

The live GitOps repository is a **separate** GitHub repository, cloned to
`~/Desktop/dso202-gitops-demo`, as the handout's "golden rule" requires
(Module 2.1). `gitops-repo/` here is only the starting copy that was pushed to
it. `argocd-app.yaml` is kept **outside** the GitOps repository, which the
handout allows. Argo CD would otherwise be watching the file that defines what
Argo CD watches.

---

## 3. Procedure and Observations

### 3.1 Stage 0 - The cluster

**Evidence: `01-stage0-cluster`.** The context is `kind-dso202`, and
`control-plane`, `worker-node-1` and `worker-node-2` are all `Ready` on
v1.36.1. kubectl v1.36.3 talks to server v1.36.1, inside the supported skew of
one minor version.

The nodes report `AGE 11d`. `kind create cluster` did not build a new cluster.
A `dso202` cluster with the same configuration file already existed from
earlier work, and kind refuses to create a second cluster with the same name.
That cluster was reused. It contained one unrelated namespace from earlier
work (`dso202-a3-helm`) and no namespace called `argocd` or `demo-app`, so
nothing in it could interfere with this practical.

### 3.2 Stage 1 - Installing Argo CD

`kubectl create namespace argocd` was followed by a **server-side** apply of
the pinned install manifest.

**Why the handout's command would fail.** A plain `kubectl apply` is
client-side. It stores a full copy of every object in the annotation
`kubectl.kubernetes.io/last-applied-configuration`, and Kubernetes limits the
total size of an object's annotations to 262,144 bytes (256 KiB). The three
CRDs in the manifest were measured as compact JSON before installing:

| CRD | Size | Fits in the annotation? |
| --- | --- | --- |
| `applications.argoproj.io` | 197,294 bytes | yes |
| `applicationsets.argoproj.io` | **343,248 bytes** | **no** |
| `appprojects.argoproj.io` | 12,097 bytes | yes |

The ApplicationSet CRD is 31 % over the limit, so the handout's command fails
on that CRD with `metadata.annotations: Too long`. The plain command was not
run in this practical. The failure is predicted from the measurement, and the
Argo CD project's own installation instructions use `--server-side
--force-conflicts` for this reason. Server-side apply records ownership in
`managedFields` instead of an annotation, so object size does not matter. The
installed CRD's field managers confirm the method: `kubectl Apply` (the
server-side operation), not `kubectl-client-side-apply`.

**Evidence: `02-stage1-argocd-installed`.** After the deterministic waits,
every Argo CD Pod is `1/1 Running` with 0 restarts, all created 3m53s
earlier:

| Pod | Role |
| --- | --- |
| `argocd-application-controller-0` | **The reconciler.** A StatefulSet, not a Deployment. Compares Git with the cluster and syncs |
| `argocd-repo-server-…` | Clones the repository and renders manifests |
| `argocd-server-…` | Web UI and API |
| `argocd-applicationset-controller-…` | Generates many Applications from templates (not used here) |
| `argocd-dex-server-…` | Single sign-on (not used here) |
| `argocd-notifications-controller-…` | Notifications (not used here) |
| `argocd-redis-…` | Cache shared by the other components |

The handout's three "main components" are the first three rows. The other four
are optional features that the standard install includes anyway.

The three `argoproj.io` CRDs were created at `2026-10-30T02:37:40Z`. These are
the "new words" that the handout's Module 1.4 says Argo CD teaches the API:
`Application`, `ApplicationSet` and `AppProject`. `argocd-server` is a
`ClusterIP` Service on `80/TCP,443/TCP`, so it is not reachable from outside
the cluster, which is why Stage 2 needs a port-forward. The server image is
`quay.io/argoproj/argocd:v3.5.6`.

### 3.3 Stage 2 - The Argo CD UI

The initial password was read from the Secret `argocd-initial-admin-secret`
and piped straight into the macOS clipboard with `pbcopy`. It was never
printed, so it appears in no screenshot. `kubectl port-forward
svc/argocd-server -n argocd 8080:443` ran as a background job, and
`https://localhost:8080` showed the browser's warning about Argo CD's
self-signed certificate. After logging in as `admin`, the UI (v3.5.6)
showed "No applications available to you just yet". This stage has no
screenshot: the empty page proves nothing that `05` does not.

### 3.4 Stage 3 - The GitOps repository

The public repository `dso202-gitops-demo` was created on GitHub with a
README, so that `main` already existed. The clone, outside this practicals
repository, is at `~/Desktop/dso202-gitops-demo`. The two manifests were
copied into `manifests/`, then committed and pushed.

**Evidence: `03-stage3-gitops-repo-pushed`.**

- The log shows `7982284 Initial deployment with 2 replicas` on top of
  GitHub's `b645110 Initial commit`. `HEAD`, `main`, `origin/main` and
  `origin/HEAD` all point to it.
- Git tracks exactly `README.md`, `manifests/deployment.yaml` and
  `manifests/service.yaml`. The repository holds no application code, only
  configuration (Module 2.1).
- `git rev-parse HEAD` and `git ls-remote … HEAD` both print
  `798228430cc7dca922bead5db71edc6c0e1e6af7`. The commit on the laptop is the
  commit on GitHub, which is the state Argo CD will read.

### 3.5 Stage 4 - Registering the application declaratively

`argocd/argocd-app.yaml` was applied with `kubectl apply`. It is the one
`kubectl apply` in this practical. Everything after it was done through Git.
The Application says three things:

- **where the desired state is:** `repoURL` (the GitHub repository),
  `targetRevision: HEAD` and `path: manifests`;
- **where to deploy it:** `https://kubernetes.default.svc` (the cluster
  Argo CD runs in), namespace `demo-app`;
- **how to sync:** `automated` with `prune: true` and `selfHeal: true`, and
  `CreateNamespace=true`.

The controller log shows the sequence. At `02:54:48Z` the new Application was
compared ("spec.source differs"). At `02:54:51Z`, three seconds later, the
first automated sync created `webapp-svc` and `webapp-deploy` in a new
`demo-app` namespace.

**Evidence: `04-stage4-app-registered`.**

- `kubectl get applications -o wide` shows `my-webapp`, `Synced`, `Healthy`,
  revision `798228430cc7…`, project `default`. That revision is
  **character-for-character equal to git HEAD**, printed on the next line.
  This is the central GitOps claim, shown as data: the cluster is running a
  specific commit.
- `webapp-deploy` is `2/2`, and `webapp-svc` is a ClusterIP Service on
  `80/TCP`.
- The two Pods run on **different workers**: `kxdrr` on `worker-node-1`
  (10.244.1.6) and `q7m5r` on `worker-node-2` (10.244.2.5).
- The tracking annotation is
  `argocd.argoproj.io/tracking-id: my-webapp:apps/Deployment:demo-app/webapp-deploy`.
  It has the form `<application>:<group>/<kind>:<namespace>/<name>`. This is
  how Argo CD 3.x recognises objects it owns, and therefore which objects it
  may prune.
- Six requests through `webapp-svc` from a busybox Pod returned
  `Server name: webapp-deploy-6bd9c658d4-kxdrr` and `…-q7m5r`. The Service
  load-balances across both Pods. The lines about recorded container logs
  and the command prompt are kubectl's standard message for an attached Pod.

**Evidence: `05-stage4-argocd-ui-app-tree`.** The UI draws the tree that the
handout describes: `my-webapp` → `webapp-svc`, and `my-webapp` →
`webapp-deploy` → ReplicaSet `webapp-deploy-6bd9c658d4` → two Pods. The header
reads **Healthy**, **Synced to HEAD (7982284)** and "Auto sync is enabled".
The last sync is **Sync OK to 7982284**, at 08:54:51 local time. The UI also
shows the commit's **author** (`KeldenDrac`) and **comment** ("Initial
deployment with 2 replicas"). The deployment record is the Git record, and the
question "who deployed this?" is answered by Git. The health filter counts 5
Healthy resources: the Service, the Deployment, the ReplicaSet and two Pods.

### 3.6 Experiment 1 - Scaling through Git

`replicas: 2` was changed to `replicas: 4` in the clone with `sed`, then
committed as `5a2467b Scale webapp to 4 replicas` (09:00:28 local) and pushed.
No `kubectl` command touched the Deployment.

Argo CD normally finds a new commit by **polling**. The controller log shows
the timer: each periodic refresh is logged as `comparison expired … expiry:
2m0s`, and a random jitter spreads the refreshes out. During the idle period
after Stage 4, the controller refreshed at `02:58:02Z` and `03:00:17Z`, 3m11s
and 2m15s after the previous comparison, which roughly matches the handout's
"every 3 minutes". The second of those refreshes ran **eleven
seconds before** the commit was made. Without intervention, the change would
have waited for the next poll, up to three more minutes.

Instead, the annotation `argocd.argoproj.io/refresh=normal` was set on the
Application. This is exactly what the UI's Refresh button does. The log shows
`normal refresh requested` at `03:00:52Z`, and the automated sync to
`5a2467b…` began **three seconds later**, at `03:00:55Z`. `kubectl wait` was
then held on `.status.sync.revision` until it equalled the new commit hash.

**Evidence: `06-exp1-scale-through-git`.**

- `git show` gives the whole change as a two-line diff:
  `- replicas: 2` / `+ replicas: 4`, under `5a2467b Scale webapp to 4 replicas`.
- `git HEAD: 5a2467b | argo synced: 5a2467b`, and the application is
  `Synced`, `Healthy`.
- There are four Pods, two per worker. The original two, `kxdrr` and
  `q7m5r`, are 8m14s old, and the new two, `pk4qb` and `zgs8p`, are 2m10s
  old. The scheduler spread the new Pods evenly.
- The Deployment's events show the whole history of the ReplicaSet:
  `Scaled up … from 0 to 2`, then `Scaled up … from 2 to 4`.

All four Pods belong to the **same ReplicaSet**, `6bd9c658d4`. Changing
`replicas` does not change the Pod template, and the ReplicaSet name is a hash
of the template, so Kubernetes scaled the existing ReplicaSet instead of
starting a rolling update. The tree in `07` shows the same thing.

**Evidence: `07-exp1-argocd-ui-four-pods`.** The header changed to **Synced to
HEAD (5a2467b)** and **Sync OK to 5a2467b**, with comment "Scale webapp to 4
replicas". The tree has four Pods under one ReplicaSet, and the filter counts
7 Healthy resources.

### 3.7 Experiment 2 - Configuration drift and self-heal

This simulated the handout's "rogue admin" who bypasses Git.

**Evidence: `08-exp2-self-heal`.**

| Local time | Event |
| --- | --- |
| 09:04:28 | `date`, then `kubectl scale deployment webapp-deploy -n demo-app --replicas=1` |
| 09:04:30 (`03:04:30Z`) | Controller log: `Requesting app refresh caused by object update`, then a new sync operation to the **same commit**, `5a2467b` |
| 09:04:31 (`03:04:31Z`) | Sync result: `deployment.apps/webapp-deploy configured`. The operation completed |
| 09:04:35 | `kubectl wait … --for=jsonpath='{.spec.replicas}'=4` had already returned, followed by `date` |

The Deployment's events show both sides of the fight one second apart:
`Scaled down replica set webapp-deploy-6bd9c658d4 from 4 to 1` (14s ago), then
`Scaled up … from 1 to 4` (13s ago). Argo CD's own events for the application
read `Partial sync operation to 5a2467b… succeeded`, followed by
`OutOfSync -> Synced` and `Healthy -> Progressing -> Healthy`. The final state
is `Synced`, `Healthy`.

**Two separate detection paths.** The drift was corrected about two seconds
after it happened. The handout's "within 5 seconds" holds, with room to
spare. This is much faster than the three-minute Git poll in Experiment 1,
because Argo CD learns about the two kinds of change in different ways:

| Change | How Argo CD notices | Observed delay |
| --- | --- | --- |
| A new commit in **Git** | Polling every 2 to 3 minutes, or an explicit refresh (button, annotation or webhook) | 3s after the refresh request; it would have been up to 3 min without it |
| An edit to the **cluster** | A live Kubernetes **watch** on every object it manages: `caused by object update` | about 2s |

**What self-heal restored, and what it did not.** It restored the declared
**count** of Pods, not the same Pods. Three of the four Pods (`kxdrr`, `pk4qb`,
`zgs8p`) were deleted by the scale-down, and three new ones (`86mkf`, `c8t7x`,
`grfdv`) replaced them. Only `q7m5r` survived both events. GitOps restores the
declaration. Any state that existed only inside a deleted Pod is gone. For a
stateless web server this does not matter, but it would for anything that is
not stateless (Practical 2).

**"Partial" sync.** A self-heal sync applies only the resources that are out
of sync. Here that was the Deployment alone (`Tasks (dry-run) [Sync/0 resource
apps/Deployment:demo-app/webapp-deploy obj->obj]`). The Service was not
touched.

### 3.8 Experiment 3 - Prune, then restore with `git revert`

`git rm manifests/service.yaml` was committed as `17ea9c5 Remove webapp-svc`
(09:06:29 local) and pushed, and a refresh was requested. The wait ended once
Argo CD reported the new commit and `Synced`.

**Evidence: `09-exp3-prune`.**

- `git show --stat`: `manifests/service.yaml | 12 ------------`,
  `1 file changed, 12 deletions(-)`. The Service file is the only change.
- `kubectl get svc -n demo-app` prints `No resources found in demo-app
  namespace.` The cluster object was deleted, even though no one ran
  `kubectl delete`.
- The application is `Synced`, `Healthy`.

The third command was meant to print the per-resource result of the sync,
`Service/webapp-svc: Pruned`. It printed **an empty line** instead. This is
not a fault in the experiment. The controller log, read afterwards, explains
it (§5, error 2): two sync operations ran on this commit. The first,
`00007-pugZv`, pruned the Service. The second, `00008-jkLFq`, found nothing
left to do (`Tasks (dry-run) []`, `successfully synced (no more tasks)`).
Because `.status.operationState` keeps **only the most recent operation**, the
empty result had replaced the prune result by the time the command ran.

**Evidence: `09b-exp3-prune-controller-log`.** The prune is recorded in two
places that the next sync cannot overwrite:

1. **The application's sync history** (`.status.history`) holds one entry per
   deployed commit, each with its full hash and time:

   | id | Revision | Deployed at (UTC) | Git commit |
   | --- | --- | --- | --- |
   | 0 | `7982284…` | 02:54:51 | Initial deployment with 2 replicas |
   | 1 | `5a2467b…` | 03:00:55 | Scale webapp to 4 replicas |
   | 2 | `17ea9c5…` | 03:06:37 | Remove webapp-svc |
   | 3 | `fb50266…` | 03:07:20 | Revert "Remove webapp-svc" |

2. **The controller log for sync `00007-pugZv`**, at `03:06:37Z`, shows the
   prune itself. The dry-run plan was `Service:demo-app/webapp-svc obj->nil`
   (live object → nothing) and `Deployment … obj->obj`. Then came
   `Adding resource result, status: 'Pruned', phase: 'Succeeded', message: 'pruned'`,
   `deployment.apps/webapp-deploy unchanged` and
   `successfully synced (all tasks run)`.

Argo CD deleted the Service because the Service carried this application's
tracking ID and no longer appeared in Git. That is the meaning of
`prune: true`. Without it, the Service would have stayed, and the application
would have shown `OutOfSync`.

**Restoring it.** `git revert HEAD --no-edit` created
`fb50266 Revert "Remove webapp-svc"` (09:07:12 local), which was pushed and
refreshed. Sync `00009-RYcNE` planned `Service … nil->obj` and logged
`service/webapp-svc created` at `03:07:20Z`.

**Evidence: `10-exp3-revert-and-audit-trail`.**

- **The audit trail.** `git log --oneline` lists all five commits in order:
  `fb50266` (Revert), `17ea9c5` (Remove), `5a2467b` (Scale), `7982284`
  (Initial deployment), `b645110` (Initial commit). Every change to the
  workload in this practical is a commit, except the deliberate drift in
  Experiment 2, which Argo CD reversed.
- The application is `Synced`, `Healthy`, at revision
  `fb50266258d64eb2daad308cbefeca24dc75a532`, which is HEAD.
- `webapp-svc` exists again, aged 4m13s.
- Six requests through it returned `c8t7x`, `q7m5r` and `86mkf`. The restored
  Service routes to the Pods that self-heal created in Experiment 2.

**The revert restored the declaration, not the object.** The original Service
had ClusterIP `10.96.226.86` (`04`). The restored one has `10.96.251.0`. It is
a new object with a new UID and a new address. Clients that use the DNS name
`webapp-svc.demo-app`, as the busybox Pod did, notice nothing. A client that
had stored the old IP address would now be broken. The deletion was not
undone. Git declared the Service again, and Kubernetes created a new one.

### 3.9 Cleanup

Cleanup is left until after this report is final, because it destroys the
application the screenshots describe. The procedure is in the README. No
screenshot of cleanup is submitted. Two facts found during the practical
shape it:

- The Application has **no finalizer** (`metadata.finalizers` is empty).
  Deleting it leaves `webapp-deploy` and `webapp-svc` running, so the
  `demo-app` namespace must be deleted too, or the cluster deleted.
- The `demo-app` namespace has no tracking ID. Argo CD created it through
  `CreateNamespace=true`, but does not manage it, so it would not be pruned
  either.

The GitHub repository `dso202-gitops-demo` is kept as evidence of the audit
trail.

---

## 4. Analysis

The handout has no review questions. This section checks the handout's model
of GitOps against what the cluster actually did.

### 4.1 The four OpenGitOps principles, tested

| Principle (handout Module 1.3) | Where it was tested | Result |
| --- | --- | --- |
| **Declarative.** The system is described as facts, not steps | `deployment.yaml` says `replicas: 4`. It does not say "add two Pods". In Experiment 2, the same fact also meant "add three Pods" | Holds. One declaration produced different actions depending on the live state, which is only possible because it describes an end state |
| **Versioned and immutable** | `03`, `10`: every state has a commit hash. `05`/`07`: the UI shows the author and comment. `09b`: Argo CD's history is keyed by commit hash | Holds. A state cannot be changed without creating a new hash, and the old one stays in the log |
| **Pulled automatically** | Nothing outside the cluster had credentials for it. GitHub never contacted the cluster. Argo CD fetched from GitHub | Holds. The direction of every connection was cluster → GitHub |
| **Continuously reconciled** | Experiment 2: drift reversed in about 2s. The idle log shows a comparison every 2 to 3 minutes | Holds for the cluster side. The Git side is checked only as often as it is polled (§4.3) |

### 4.2 Push versus pull: where the credentials are

In the push model (Module 1.1), the CI server holds a kubeconfig with write
access to the cluster. In this practical, **no credential to the cluster ever
left the cluster**. The only credential that was needed was Git read access,
and because the repository is public, not even that. The Argo CD controller
used its own ServiceAccount, inside the cluster, to apply the changes.

The direction is reversed, and so is the attack surface. An attacker who
compromised GitHub could still change the cluster, by committing to the
repository. That is why the handout requires branch protection and
pull-request review on the configuration repository (Module 2.1). The attack
must now go through the reviewed, logged path, not around it.

### 4.3 Polling is the slow half of the loop

The two detection paths in §3.7 matter in practice:

- **A bad manual change** (`kubectl scale`, `kubectl edit`) is reversed in
  seconds, because Argo CD watches the objects it manages.
- **A good change in Git** waits up to about three minutes unless something
  tells Argo CD that the repository has changed. In Experiment 1, a poll ran
  eleven seconds before the commit, which shows the cost: without the
  refresh, the change would have waited for the next poll.

In production, the refresh is a **GitHub webhook** to the Argo CD server
(Module 2.3). It does what the annotation did here, without a person. The
polling interval is then only a fallback, for when a webhook is lost.

### 4.4 Argo CD compared with Helm's server-side apply (Practical 6)

Practical 6 drifted a Deployment in the same way, with `kubectl scale
--replicas=5`. Helm 4 **refused** the next upgrade with a field-manager
conflict naming `kubectl` and `.spec.replicas`. Argo CD **silently
overwrote** the same kind of drift. The difference has two causes:

1. **Policy.** `selfHeal: true` is an instruction to overwrite drift. Helm
   has no equivalent standing instruction. Every Helm upgrade is a one-off
   command that a person runs.
2. **Mechanism.** The controller log shows `serverSideApply: false`, and the
   Deployment carries `kubectl.kubernetes.io/last-applied-configuration`.
   Argo CD 3.5 applies with **client-side** apply by default. Its field
   manager on the Deployment is `argocd-controller` with operation `Update`.
   Client-side apply does not raise field-manager conflicts.

The two tools rest on different assumptions. Helm assumes another manager may
own a field legitimately, so it asks. Argo CD with self-heal assumes Git is
always right, so it corrects. The danger is the same as the one at the end of
Practical 6. If a HorizontalPodAutoscaler managed `spec.replicas`, Argo CD
would scale the Deployment back to 4 every time the HPA changed it. The
standard fix is to remove `replicas` from the manifest, or to tell Argo CD to
ignore that field (`spec.ignoreDifferences`).

### 4.5 What `kubectl` still can and cannot do

The handout says that in GitOps, "kubectl becomes a read-only debugging
tool". Experiment 2 shows that this is a **convention, not a control**.
`kubectl scale` succeeded, and the Deployment did run one Pod for about a
second. Self-heal limits how long drift lasts, but does not prevent it. A
change that does damage within that second (deleting a PersistentVolumeClaim,
for example) is not undone by re-applying a manifest. Enforcing "read-only"
needs RBAC that gives people only `get`, `list` and `watch` in managed
namespaces, so that the only identity with write access is Argo CD's.

### 4.6 Rollback by `git revert`

The handout's table (Module 2.2) maps "roll back a broken deployment" to
`git revert`. Experiment 3 confirmed that it works, and showed its exact
meaning. The revert is a **new, forward commit** (`fb50266`), and Argo CD's
history records it as a new deployment (id 3), not a return to id 1. This is
the same append-only idea as `helm rollback` creating revision 4 in
Practical 6. In both tools, undoing something is itself recorded. The revert
also shows that a rollback restores what was **declared**, not the old object
(§3.8, new ClusterIP).

---

## 5. Reflection

### What was difficult

The hardest part was not running Argo CD, but **proving what it did**.
Argo CD reacts in seconds and then shows only the latest state. The handout
asks students to "watch the UI closely" for a flash of `OutOfSync`, which
leaves no evidence. Each experiment therefore needed a measurement that could
be repeated:

- an exact commit hash to wait on (Experiment 1);
- timestamps on both sides of the drift (Experiment 2);
- the controller's own log and history (Experiment 3).

Reading the controller log was new. It turned out to be the most useful
source in the practical, more precise than the UI or the Kubernetes events.

The second difficulty was separating the **two loops**. Before the practical,
"reconciliation every 3 minutes" seemed to apply to everything. The
measurements showed a 3-minute loop for Git and a live watch for the cluster.
Knowing which one a change goes through explains its timing.

### Errors met, and how they were diagnosed

**1. The handout's install command (anticipated, avoided).**
The CRD sizes were measured from the manifest before installing (§3.2), and
the ApplicationSet CRD was found to be over the annotation limit. The install
was therefore done server-side from the start. The guide given before the
install said that *two* CRDs were too large. The measurement for this report
corrected that: only the ApplicationSet CRD exceeds the limit. The
Application CRD fits.

**2. The empty result line in `09`.**
The prune clearly worked: the Service was gone, and the application was
`Synced`. But the command that should have printed `Pruned` printed nothing.
The diagnosis came from the controller log. Searching for `webapp-svc`
found the `Pruned` result in sync `00007`. Listing every operation by
`syncId` then showed a second sync on the same commit, `00008`, with an empty
task list.

The same pairing appears after **every** automated sync in this practical
(see the `Partial sync operation` events in `08`). Immediately after a sync,
the controller re-compares the application before its cache has received the
watch events for the objects it has just changed. It sees those objects as
still out of sync, and self-heal starts a targeted second sync. The second
sync finds nothing to do. It is harmless, but it replaces
`.status.operationState`, which keeps only the latest operation.

The lesson is about evidence: **`.status.operationState` is a snapshot of
the last operation, not a log.** The durable records are
`.status.history` and the controller log, which `09b` captured.

**3. `couldn't attach to pod/curl-test` (first attempt at `10`).**
The first capture of `10` printed `warning: couldn't attach to pod/curl-test,
falling back to streaming logs … no running task found`, and then printed the
output twice. busybox had already pulled, and the six requests finished
before `kubectl run -i` could attach to the container. kubectl fell back to
reading the logs. Nothing was wrong with the Service, but the frame looked
like a failure. The retake runs the Pod without attaching, waits for
`.status.phase=Succeeded`, reads `kubectl logs`, and then deletes the Pod.
Each step waits on a defined state, so there is no race.

**4. The reused cluster.**
`AGE 11d` in `01` showed that `kind create cluster` had not created a new
cluster. Instead of deleting and recreating it, its namespaces were listed to
confirm that nothing could conflict. Only the unrelated `dso202-a3-helm` was
there.

### What would be done differently

**Configure a webhook, or keep the refresh in the procedure.** In a lab
behind NAT, GitHub cannot reach `localhost:8080`. The refresh annotation is
the honest substitute, and it should be written into the procedure rather
than left as an optional click.

**Make deletion explicit.** The Application had no
`resources-finalizer.argocd.argoproj.io` finalizer, so deleting it would
leave the workload running. Whether that is the desired behaviour should be
decided when the Application is written, not discovered at cleanup.

**Enable `ServerSideApply=true`.** It would make Argo CD's field ownership
visible in `managedFields`, the same mechanism Practical 6 used, and it is
needed anyway for large objects such as the CRDs in §3.2.

### What remains unclear

How Argo CD should handle **legitimate** drift, such as an HPA, beyond
`ignoreDifferences`. Ignoring a field means Git no longer describes it, which
weakens the principle that Git is the whole truth. This practical did not
test autoscaling under Argo CD.

How secrets fit is also unclear. The handout points to Sealed Secrets and the
External Secrets Operator (Module 6.2). Both move something other than the
secret itself into Git, but this practical did not use either, so how they
behave under self-heal and prune is still untested.

---

## 6. References

All accessed 30 October 2026.

1. Argo CD Documentation - *Getting Started* (installation, initial admin
   password, port-forward access).
   https://argo-cd.readthedocs.io/en/stable/getting_started/
2. Argo CD Documentation - *Automated Sync Policy* (automated sync, `prune`,
   `selfHeal`).
   https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/
3. Argo CD Documentation - *Sync Options* (`CreateNamespace`,
   `ServerSideApply`).
   https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/
4. Argo CD Documentation - *Resource Tracking* (the `tracking-id`
   annotation).
   https://argo-cd.readthedocs.io/en/stable/user-guide/resource_tracking/
5. Argo CD Documentation - *Git Webhook Configuration* and the reconciliation
   timeout settings (`timeout.reconciliation`, jitter).
   https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/
6. Argo CD Documentation - *App Deletion* (cascading deletion and the
   resources finalizer).
   https://argo-cd.readthedocs.io/en/stable/user-guide/app_deletion/
7. Argo CD - release v3.5.6 install manifest.
   https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.6/manifests/install.yaml
8. OpenGitOps - *GitOps Principles v1.0.0*.
   https://opengitops.dev/
9. Kubernetes Documentation - *Server-Side Apply* (field managers; the
   annotation size limit of client-side apply).
   https://kubernetes.io/docs/reference/using-api/server-side-apply/
10. Kubernetes Documentation - *Custom Resources*.
    https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
11. DSO202 lab guide - *GitOps and Argo CD*, Modules 1 to 6.

---

## Appendix - Evidence index

| File | Stage | What it establishes |
| --- | --- | --- |
| `01-stage0-cluster` | 0 | Context `kind-dso202`; three nodes `Ready` on v1.36.1; client v1.36.3 |
| `02-stage1-argocd-installed` | 1 | Seven Argo CD Pods Running; three `argoproj.io` CRDs; `argocd-server` is ClusterIP 80/443; image `argocd:v3.5.6` |
| `03-stage3-gitops-repo-pushed` | 3 | Commit `7982284` on `main` and `origin/main`; only manifests tracked; local HEAD = GitHub HEAD |
| `04-stage4-app-registered` | 4 | `Synced`/`Healthy`; Argo revision = git HEAD; 2 Pods on both workers; tracking ID; Service balances across both Pods |
| `05-stage4-argocd-ui-app-tree` | 4 | UI tree App → Service, Deployment → ReplicaSet → 2 Pods; Synced to HEAD (7982284); commit author and comment |
| `06-exp1-scale-through-git` | Exp 1 | Two-line Git diff; Argo synced = git HEAD `5a2467b`; 4 Pods, same ReplicaSet; events 0→2→4 |
| `07-exp1-argocd-ui-four-pods` | Exp 1 | UI Synced to `5a2467b`; four Pods under one ReplicaSet |
| `08-exp2-self-heal` | Exp 2 | `kubectl scale` to 1 reverted to 4 within 7s; events 4→1→4 one second apart; `Partial sync` to the same commit |
| `09-exp3-prune` | Exp 3 | `service.yaml` deleted in Git; `webapp-svc` gone from the cluster; app `Synced` |
| `09b-exp3-prune-controller-log` | Exp 3 | Sync history keyed by commit; controller log `obj->nil` and `status: 'Pruned'` for `webapp-svc` |
| `10-exp3-revert-and-audit-trail` | Exp 3 | Five-commit audit trail; Synced at revert `fb50266`; Service recreated; traffic reaches the self-healed Pods |
