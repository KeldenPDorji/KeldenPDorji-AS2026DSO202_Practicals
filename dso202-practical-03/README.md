# DSO202 - Practical 3

**Environment-specific configuration with Kustomize on kind**

Module: DSO202 - Scaling, Orchestration, Monitoring & Observability
Programme: BE in Software Engineering

## Purpose

Practicals 1 and 2 applied one manifest per object to one namespace. This
practical deploys the same application to several environments - dev,
staging, prod and a QA environment written as part of the work - **without
copying the base Deployment or Service**.

A single **base** holds the Deployment, the Service and a default page. Each
environment is an **overlay** that states only what differs: namespace,
environment label, replica count, image tag, page content, and - through
patches - resources (prod, strategic merge) and an ownership annotation (QA,
JSON 6902). Every change goes through **render -> diff -> apply -> verify**.

The practical also demonstrates the generated-ConfigMap hash: a change to a
page changes the ConfigMap's name, which changes the Deployment's Pod template,
which triggers a rollout.

## Software versions used

| Software | Version |
| --- | --- |
| Operating system | macOS 26.6.2, arm64 |
| Docker Desktop | 29.7.2 |
| kind | v0.32.0 (go1.26.3 darwin/arm64) |
| kubectl | v1.36.3 (Kustomize v5.8.1) |
| Kubernetes (cluster) | v1.36.1 (`kindest/node:v1.36.1`, pinned by digest) |

> **kubectl on this machine.** Docker Desktop's older `kubectl` in
> `/usr/local/bin` shadows the Homebrew binary. Put Homebrew first for the
> session; the Kustomize version compiled into kubectl decides which
> `kustomization.yaml` fields are understood:
>
> ```bash
> export PATH="/opt/homebrew/bin:$PATH"
> kubectl version --client        # must report v1.36.3 / Kustomize v5.8.1
> ```

### Container images

| Image | Role |
| --- | --- |
| `kindest/node:v1.36.1` | Every cluster node |
| `nginx:1.30-alpine` | Base image; staging, prod, qa |
| `nginx:1.31-alpine` | dev overlay (set by `images:`) |

## Repository layout

```
dso202-practical-03/
├── README.md                              # this file
├── cluster/
│   └── kind-cluster.yaml                  # three nodes: control-plane, worker-node-1, worker-node-2
├── examples/
│   ├── webapp/
│   │   ├── base/                          # written once, shared by every environment
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── index.html                 # default page
│   │   │   └── kustomization.yaml         # labels + configMapGenerator (web-content)
│   │   └── overlays/
│   │       ├── dev/                       # 1 replica, nginx 1.31, own page
│   │       ├── staging/                   # 2 replicas, own page
│   │       ├── prod/                      # 3 replicas, annotation, patch-resources.yaml (SMP)
│   │       ├── qa/                        # 2 replicas, patch-annotation.yaml (JSON 6902)
│   │       └── sandbox/                   # challenge: namePrefix "sandbox-" (render only)
│   └── mistakes/
│       └── qa-unescaped-path/             # DELIBERATELY BROKEN first QA patch; never applied
├── evidence/                              # screenshots (.png)
└── report/
    └── practical-03-report.md             # the assessed report
```

Two conventions are deliberate. **No overlay contains a Deployment or a
Service** - only the fields that differ. And no base object carries a
`namespace:`; each overlay sets it with `namespace:` and creates it with its
own `namespace.yaml`.

## Rebuild from an empty machine

Run from this directory (`dso202-practical-03/`).

```bash
# 0. Tooling.
export PATH="/opt/homebrew/bin:$PATH"
docker info --format '{{.ServerVersion}}'
kind version
kubectl version --client

# 1. Cluster.
kind create cluster --config cluster/kind-cluster.yaml
kubectl wait --for=condition=Ready nodes --all --timeout=180s

# 2. Render before anything touches the cluster.
kubectl kustomize examples/webapp/base
kubectl kustomize examples/webapp/overlays/dev

# 3. Each environment: diff, apply, wait.
#    The FIRST diff of an overlay reports 'namespaces "..." not found' - the
#    namespace does not exist yet, so the server-side dry run cannot place the
#    namespaced objects. The render in step 2 is the preview for a first deploy.
for env in dev staging prod qa; do
  kubectl diff -k examples/webapp/overlays/$env || true
  kubectl apply -k examples/webapp/overlays/$env
  kubectl rollout status deployment/webapp -n webapp-$env --timeout=120s
done

# 4. Verify.
kubectl get deploy -A -l app.kubernetes.io/name=webapp
kubectl port-forward -n webapp-dev service/webapp 8080:80   # then: curl http://127.0.0.1:8080
```

Confirm that the running cluster matches this repository (no output means no
drift):

```bash
for env in dev staging prod qa; do kubectl diff -k examples/webapp/overlays/$env; done
```

The sandbox overlay is for rendering only:

```bash
kubectl kustomize examples/webapp/overlays/sandbox
```

## Cleanup

`kubectl apply` never removes a generated ConfigMap that is no longer
rendered, so each content change leaves the previous `web-content-<hash>`
behind. Deleting the namespace removes them together with everything else.

```bash
# 1. Delete each environment. Each overlay includes its Namespace, so this
#    removes the namespace and every object in it, orphaned ConfigMaps included.
kubectl delete -k examples/webapp/overlays/dev
kubectl delete -k examples/webapp/overlays/staging
kubectl delete -k examples/webapp/overlays/prod
kubectl delete -k examples/webapp/overlays/qa

# 2. Verify nothing is left.
kubectl get ns | grep 'webapp-' || true

# 3. Delete the cluster.
kind delete cluster --name dso202-p3
kind get clusters
```

## Known limits of this environment

| On this cluster | In a real multi-environment setup |
| --- | --- |
| All environments share one cluster, separated only by namespace | Prod usually runs on its own cluster, with separate credentials |
| `kubectl apply -k` is run by hand from a laptop | A GitOps controller (Argo CD, Flux) renders and applies each overlay from Git |
| Old generated ConfigMaps are never pruned | The controller prunes, or a retention policy is set |
| The app is reached with `kubectl port-forward` | Each environment has its own Ingress and DNS name |
