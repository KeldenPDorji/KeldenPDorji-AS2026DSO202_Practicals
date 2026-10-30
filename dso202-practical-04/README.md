# DSO202 - Practical 4

**GitOps with Argo CD: Git as the single source of truth**

Module: DSO202 - Scaling, Orchestration, Monitoring & Observability
Programme: BE in Software Engineering

## Purpose

In Practicals 5 and 6 a person still ran `kubectl apply` or `helm upgrade`,
and nothing checked the cluster against the files afterwards. This practical
installs **Argo CD** inside the cluster, so that the cluster pulls its desired
state from a Git repository and keeps itself equal to it:

1. **Install and register.** Argo CD v3.5.6 runs in the `argocd` namespace.
   The workload is registered declaratively, as an `Application` custom
   resource pointing at a separate GitHub repository.
2. **Experiment 1: scale through Git.** A one-line commit (`replicas: 2` →
   `4`) is the only action, and the cluster follows it.
3. **Experiment 2: self-heal.** A manual `kubectl scale` is reverted by
   Argo CD in about two seconds.
4. **Experiment 3: prune and revert.** Deleting `service.yaml` in Git deletes
   the Service in the cluster, and `git revert` brings it back. The Git log is
   the audit trail.

## Software versions used

| Software | Version |
| --- | --- |
| Operating system | macOS 26.6.2, arm64 |
| Docker Desktop | 29.7.2 |
| kind | v0.32.0 (go1.26.3 darwin/arm64) |
| kubectl | v1.36.3 |
| Kubernetes (cluster) | v1.36.1 (`kindest/node:v1.36.1`) |
| Argo CD | **v3.5.6** |

> **Install Argo CD server-side.** The handout's plain `kubectl apply` fails
> with `metadata.annotations: Too long`. The ApplicationSet CRD is 343,248
> bytes, and client-side apply cannot store more than 262,144 bytes in the
> annotation it writes. Use:
>
> ```bash
> kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.6/manifests/install.yaml
> ```

## Repository layout

```
dso202-practical-04/
├── README.md                    # this file
├── cluster/
│   └── kind-cluster.yaml        # Practical 1 Listing 1: cluster "dso202", 3 nodes
├── gitops-repo/
│   └── manifests/               # seed copy of the GitOps repository's contents
│       ├── deployment.yaml      # webapp-deploy, nginxdemos/hello:plain-text, 2 replicas
│       └── service.yaml         # webapp-svc, ClusterIP port 80
├── argocd/
│   └── argocd-app.yaml          # the Application: repo, path, destination, sync policy
├── evidence/                    # screenshots (.png)
└── report/
    └── practical-04-report.md   # the assessed report
```

The GitOps repository itself is separate:
https://github.com/KeldenPDorji/dso202-gitops-demo (cloned to
`~/Desktop/dso202-gitops-demo`). It holds only `manifests/`. `argocd-app.yaml`
is deliberately kept outside it.

## Rebuild from an empty machine

Run from this directory. The GitOps repository must be public, or Argo CD
needs read credentials for it. To use your own fork, change `repoURL` in
`argocd/argocd-app.yaml`.

```bash
# 0. Tooling and cluster.
export PATH="/opt/homebrew/bin:$PATH"
kind create cluster --config cluster/kind-cluster.yaml
kubectl wait --for=condition=Ready nodes --all --timeout=180s

# 1. Argo CD (server-side: see the note above).
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.6/manifests/install.yaml
kubectl wait --for=condition=Available deployment --all -n argocd --timeout=600s
kubectl rollout status statefulset/argocd-application-controller -n argocd --timeout=300s

# 2. UI: password to the clipboard (never printed), then port-forward.
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d | pbcopy
kubectl port-forward svc/argocd-server -n argocd 8080:443 > /dev/null 2>&1 &
open https://localhost:8080        # user: admin

# 3. Register the application. Argo CD creates demo-app and syncs it.
kubectl apply -f argocd/argocd-app.yaml
kubectl wait application/my-webapp -n argocd --for=jsonpath='{.status.health.status}'=Healthy --timeout=300s
kubectl get applications -n argocd -o wide
```

After any push to the GitOps repository, either wait for the next poll (2 to
3 minutes) or request a refresh, then wait for the new commit:

```bash
kubectl annotate application my-webapp -n argocd argocd.argoproj.io/refresh=normal --overwrite
kubectl wait application/my-webapp -n argocd --for=jsonpath='{.status.sync.revision}'=$(git -C ~/Desktop/dso202-gitops-demo rev-parse HEAD) --timeout=300s
```

## Cleanup

The Application has **no** `resources-finalizer.argocd.argoproj.io`
finalizer, so deleting it leaves the workload running. The `demo-app`
namespace is created by `CreateNamespace=true` but is not managed by Argo CD,
so it is never pruned either. Delete it explicitly, or delete the cluster.

```bash
# 1. Stop the port-forward.
pkill -f "kubectl port-forward svc/argocd-server"

# 2. Remove the application, its namespace and Argo CD.
kubectl delete application my-webapp -n argocd
kubectl delete namespace demo-app argocd

# 3. Delete the cluster (this alone also removes everything above).
kind delete cluster --name dso202
kind get clusters
```

The GitHub repository `dso202-gitops-demo` and its local clone are kept,
because their commit history is the practical's audit trail.

## Known limits of this environment

| On this cluster | In industry |
| --- | --- |
| A push is noticed by polling, or by a manual refresh annotation | A GitHub webhook notifies Argo CD on every push. Polling is only a fallback |
| Commits go straight to `main` | Changes arrive as reviewed pull requests to a protected branch |
| The UI is reached through `kubectl port-forward` with a self-signed certificate | An Ingress with a real certificate and single sign-on (Dex or OIDC). The initial admin Secret is deleted |
| People can still `kubectl scale` a managed Deployment; self-heal reverses it | RBAC gives people read-only access, and only Argo CD can write |
| The `default` AppProject allows any repository and any destination | One AppProject per team, restricting source repositories, destination namespaces and resource kinds |
| One Application, applied by hand | An "app of apps" or ApplicationSet that creates Applications from Git |
