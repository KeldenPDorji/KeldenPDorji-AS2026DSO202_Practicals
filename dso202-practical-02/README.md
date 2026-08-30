# DSO202 - Practical 2

**Implementing persistent storage for a stateful application in Kubernetes**

Module: DSO202 - Scaling, Orchestration, Monitoring & Observability
Programme: BE in Software Engineering

## Purpose

Practical 1 built workloads that were disposable: a Pod could be deleted and
recreated at no cost, because nothing inside it was worth keeping. This
repository deals with the opposite case - workloads whose value **is** the data
they hold.

Two bodies of work are covered, in order:

1. **The storage layer.** PersistentVolumes, PersistentVolumeClaims and
   StorageClasses. Static provisioning is done first, so that *matching a claim
   to a volume* is separated from *creating the volume*; dynamic provisioning
   follows, along with the two limits of a laptop provisioner (the requested
   capacity is not enforced, and the volume cannot be expanded).
2. **The controller for stateful workloads.** A Deployment with three replicas
   is deliberately pointed at one volume so that its failure modes can be
   observed, and a StatefulSet is then introduced to supply the four things the
   Deployment could not: a stable name, private storage per ordinal, a stable
   DNS record, and ordered operations.

The practical closes by running **PostgreSQL 18** as a StatefulSet on a
retaining StorageClass, creating a table, inserting rows, deleting the database
Pod, and reading the rows back from its replacement. That database is the same
one the Task Tracker application in Assignment 1 uses, so the configuration
here is directly reusable.

Everything is reproducible from this repository alone: the cluster, the
namespace, the storage and the database can be destroyed and rebuilt with the
command sequences below.

## Software versions used

| Software | Version |
| --- | --- |
| Operating system | macOS 26.5.2, arm64 |
| Docker Desktop | 29.0.1 |
| kind | v0.32.0 (go1.26.3 darwin/arm64) |
| kubectl | v1.36.3 (Kustomize v5.8.1) |
| Kubernetes (cluster) | v1.36.1 (`kindest/node:v1.36.1`, pinned by digest) |

> **Note on the kubectl binary.** Docker Desktop installs its own older
> `kubectl` at `/usr/local/bin/kubectl`, and on this machine `/usr/local/bin`
> precedes `/opt/homebrew/bin` on `PATH`, so the Docker copy shadows the
> current Homebrew binary. Practical 1 was carried out under that skew. For
> this practical the Homebrew binary is put first for the session, so that
> client and cluster are both v1.36:
>
> ```bash
> export PATH="/opt/homebrew/bin:$PATH"
> kubectl version --client        # must report v1.36.3
> ```

### Container images

| Image | Role |
| --- | --- |
| `kindest/node:v1.36.1` | Every cluster node |
| `busybox:1.37` | Volume writers, StatefulSet init container, client Pod |
| `nginx:1.30-alpine` | Web server for the `webnote` StatefulSet |
| `nginx:1.31-alpine` | Rolling-update target (Stage 6 only) |
| `postgres:18-alpine` | The stateful application |

## Repository layout

```
dso202-practical-02/
├── README.md                          # this file
├── cluster/
│   ├── kind-cluster.yaml              # Listing 1  - three nodes, one host mount
│   └── kind-cluster-fallback.yaml     # Listing 1B - used only if node naming fails
├── manifests/
│   ├── 00-namespace.yaml              # Namespace
│   ├── 01-quota-and-limits.yaml       # ResourceQuota (incl. storage caps), LimitRange
│   ├── 02-storageclass-retain.yaml    # StorageClass, reclaimPolicy Retain
│   ├── 03-pv-static.yaml              # PersistentVolume (hostPath, node affinity)
│   ├── 04-pvc-static.yaml             # PersistentVolumeClaim, class "manual"
│   ├── 05-pod-static-writer.yaml      # Pod appending to the static volume
│   ├── 06-pvc-dynamic.yaml            # PersistentVolumeClaim, class "standard"
│   ├── 07-pod-dynamic-writer.yaml     # Pod that triggers dynamic provisioning
│   ├── 08-deployment-shared-pvc.yaml  # ANTI-PATTERN: 3 replicas, 1 volume
│   ├── 09-service-webnote.yaml        # Headless Service
│   ├── 10-statefulset-webnote.yaml    # StatefulSet with volumeClaimTemplates
│   ├── 11-pod-client.yaml             # In-cluster DNS/HTTP client
│   ├── 12-secret-postgres.yaml        # Database credentials
│   ├── 13-service-postgres.yaml       # Headless + ClusterIP Services
│   └── 14-statefulset-postgres.yaml   # PostgreSQL on the retaining class
├── evidence/                          # command output (.txt) and screenshots (.png)
└── report/
    └── practical-02-report.md         # the assessed report
```

Two conventions are deliberate. Every **namespaced** object carries an explicit
`namespace: dso202-practical-02`, so a manifest applies correctly whichever
context is active. The **cluster-scoped** objects - the StorageClass in
`02-` and the PersistentVolume in `03-` - carry no `namespace:` field, because
they do not belong to one.

## Rebuild from an empty machine

Run from this directory (`dso202-practical-02/`).

```bash
# 0. Use the kubectl that matches the cluster, and confirm the tooling.
export PATH="/opt/homebrew/bin:$PATH"
docker info --format '{{.ServerVersion}}'
kind version
kubectl version --client

# 1. Create the host directory BEFORE the cluster.
#    kind binds this into worker-node-1 at creation time; if it does not exist,
#    cluster creation fails in a way that looks like a Kubernetes fault.
mkdir -p /tmp/dso202-p2-storage

# 2. Create the cluster.
kind create cluster --config cluster/kind-cluster.yaml
kubectl get nodes -o wide
docker exec dso202-p2-worker ls -ld /mnt/dso202-static

# 3. Namespace, quota, and the retaining StorageClass.
kubectl apply -f manifests/00-namespace.yaml
kubectl config set-context --current --namespace=dso202-practical-02
kubectl apply -f manifests/01-quota-and-limits.yaml
kubectl apply -f manifests/02-storageclass-retain.yaml

# 4. Static provisioning.
kubectl apply -f manifests/03-pv-static.yaml
kubectl apply -f manifests/04-pvc-static.yaml
kubectl apply -f manifests/05-pod-static-writer.yaml
kubectl wait --for=condition=Ready pod/static-writer --timeout=90s

# 5. Dynamic provisioning.
kubectl apply -f manifests/06-pvc-dynamic.yaml     # stays Pending: correct
kubectl apply -f manifests/07-pod-dynamic-writer.yaml
kubectl wait --for=condition=Ready pod/dynamic-writer --timeout=120s

# 6. The anti-pattern, observed and then removed.
kubectl apply -f manifests/08-deployment-shared-pvc.yaml
kubectl rollout status deployment/shared-writer --timeout=180s
kubectl delete -f manifests/08-deployment-shared-pvc.yaml

# 7. StatefulSet. The headless Service goes first, so the Pods are addressable
#    from the moment they are ready.
kubectl apply -f manifests/09-service-webnote.yaml
kubectl apply -f manifests/10-statefulset-webnote.yaml
kubectl rollout status statefulset/webnote --timeout=300s
kubectl apply -f manifests/11-pod-client.yaml
kubectl wait --for=condition=Ready pod/client --timeout=90s

# 8. PostgreSQL.
kubectl apply -f manifests/12-secret-postgres.yaml
kubectl apply -f manifests/13-service-postgres.yaml
kubectl apply -f manifests/14-statefulset-postgres.yaml
kubectl rollout status statefulset/postgres --timeout=300s
```

Confirm that the running cluster matches this repository:

```bash
kubectl diff -f manifests/ && echo "cluster matches repository"
```

## Cleanup

Cleanup is not housekeeping in this practical - it is where the reclaim
policies are finally understood. **Deleting the workloads does not delete the
storage, and `kubectl delete -f manifests/` never will**, because the claims
generated from a `volumeClaimTemplate` were never in a manifest.

```bash
# 1. Capture evidence first. A logical dump held outside the cluster is a
#    backup; a retained volume is not.
mkdir -p evidence
kubectl get all -o wide                       > evidence/final-state-all.txt
kubectl get pv,pvc,storageclass -o wide       > evidence/final-state-storage.txt
kubectl get statefulset webnote -o yaml       > evidence/final-statefulset-webnote.yaml
kubectl get events --sort-by=.lastTimestamp   > evidence/final-state-events.txt
kubectl exec postgres-0 -- pg_dump -U taskuser -d tasktracker > evidence/tasktracker-dump.sql

# 2. Delete the workloads. Every claim survives this step.
kubectl delete -f manifests/14-statefulset-postgres.yaml
kubectl delete -f manifests/10-statefulset-webnote.yaml
kubectl delete -f manifests/11-pod-client.yaml
kubectl delete -f manifests/05-pod-static-writer.yaml
kubectl get pvc                               # six claims still present

# 3. Delete the claims explicitly. The two reclaim policies now diverge:
#    'standard' volumes (Delete) disappear with their claims; 'dso202-retain'
#    and the static 'manual' volume become Released and keep their data.
kubectl delete pvc --all
kubectl get pv

# 4. Reclaim the Released volumes deliberately. Deleting a PV removes an API
#    object, not a byte of data.
kubectl delete pv pv-web-static
kubectl get pv
kubectl delete pv <name-of-the-released-postgres-volume>

# 5. Reset the context and delete the cluster. Every dynamically provisioned
#    volume goes with the nodes, because it lived inside one.
kubectl config set-context --current --namespace=default
kind delete cluster --name dso202-p2
kind get clusters

# 6. The final asymmetry: the statically provisioned data is still on the HOST,
#    because it never lived inside the cluster at all.
ls -l /tmp/dso202-p2-storage/pv-web-static/

# 7. Only after the report is written and the evidence captured:
rm -rf /tmp/dso202-p2-storage
```

## Known limits of this environment

The provisioner used here (`rancher.io/local-path`) writes directories on the
node filesystem. Four observations therefore differ from a managed cloud
cluster running a real CSI driver:

| On this cluster | On a managed cloud cluster |
| --- | --- |
| A volume lives inside one node, so a Pod using it can only be scheduled there | A network disk detaches and reattaches, so the Pod can move within a zone |
| The requested capacity is recorded and not enforced; a 1Gi claim can fill the node disk | The requested capacity is the size of the provisioned disk, and is enforced |
| Volume expansion is refused (`allowVolumeExpansion: false`) | Expansion is usually permitted; the claim is edited and the filesystem grows |
| Deleting the cluster destroys every dynamically provisioned volume | Deleting the cluster leaves the disks, and their cost, behind |

The most consequential of these appears in Stage 4: because all three replicas
were forced onto one node and `ReadWriteOnce` permits several Pods on the *same*
node, a misconfiguration that would fail loudly in production ran here without
complaint.

> **On the committed Secret.** `manifests/12-secret-postgres.yaml` holds
> credentials in plain text. A Kubernetes Secret is base64-*encoded*, not
> encrypted, and this file in Git is readable by anyone who can read the
> repository. It is committed here only because the cluster is local,
> disposable, and holds nothing of value.
