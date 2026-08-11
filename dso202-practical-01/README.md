# DSO202 - Practical 1

**Setting up a local Kubernetes cluster with kind, and deploying first workloads**

Module: DSO202 - Scaling, Orchestration, Monitoring & Observability
Programme: BE in Software Engineering

## Purpose

This repository builds a three-node Kubernetes cluster on a single laptop using
**kind** (Kubernetes IN Docker), and creates five categories of Kubernetes object
on it: a Namespace with a ResourceQuota and LimitRange, a bare Pod, a Deployment,
and two Services (ClusterIP and NodePort).

The application is deliberately trivial - a static nginx web server - so that
attention stays on the Kubernetes objects rather than on application code.

Everything here is reproducible: the cluster and every object can be destroyed
and rebuilt from this repository alone. The laptop holds no state that matters.

## Software versions used

| Software | Version |
| --- | --- |
| Operating system | macOS 26.5.2 (build 25F84), arm64 |
| Docker Desktop | 29.0.1 |
| kind | v0.32.0 (go1.26.3 darwin/arm64) |
| kubectl | v1.34.1 (Kustomize v5.7.1) - see note below |
| Kubernetes (cluster) | v1.36.1 (`kindest/node:v1.36.1`) |

> **Known version skew.** `kubectl` is supported within one minor version of the
> cluster it addresses. This practical was carried out with kubectl v1.34.1
> against a v1.36.1 cluster - a two-minor skew, outside the supported window.
> Every operation succeeded because only stable core APIs (`v1`, `apps/v1`,
> `discovery.k8s.io/v1`) were used, but the combination is not one the project
> supports and should not be relied on.
>
> The cause is a macOS PATH ordering issue: Docker Desktop installs its own
> older `kubectl` at `/usr/local/bin/kubectl`, which shadows the current
> Homebrew binary at `/opt/homebrew/bin/kubectl` (v1.36.3 on this machine).
> Verify with `kubectl version --client`; if it reports below v1.35, put
> Homebrew first before creating the cluster:
>
> ```bash
> export PATH="/opt/homebrew/bin:$PATH"
> ```
>
> To make it permanent: `echo 'export PATH="/opt/homebrew/bin:$PATH"' >> ~/.zshrc`

## Repository structure

```
dso202-practical-01/
├── README.md                        # this file
├── cluster/
│   └── kind-cluster.yaml            # Listing 1 - three-node cluster definition
├── manifests/
│   ├── 00-namespace.yaml            # Listing 2 - namespace
│   ├── 01-quota-and-limits.yaml     # Listing 3 - ResourceQuota + LimitRange
│   ├── 02-pod-web.yaml              # Listing 4 - bare Pod
│   ├── 03-deployment-web.yaml       # Listing 5 - Deployment (3 replicas)
│   ├── 04-service-clusterip.yaml    # Listing 6 - ClusterIP Service
│   ├── 05-service-nodeport.yaml     # Listing 7 - NodePort Service
│   └── 06-pod-client.yaml           # Listing 8 - in-cluster client Pod
├── evidence/                        # command output captured as .txt
└── report/
    └── practical-01-report.md       # the assessed report
```

Manifests are numbered because `kubectl apply -f manifests/` applies a directory
in **lexical order**. The namespace must exist before the quota, and the quota
before any Pod, or admission fails.

## Rebuilding from an empty machine

Prerequisites: Docker Desktop, kind ≥ 0.32.0 and kubectl ≥ 1.35 installed and on
`PATH`, with Docker running and at least 4 GB memory, 2 CPUs and 15 GB free disk
available to it.

```bash
# 0. Verify tooling before starting. All three must print a version.
docker info --format '{{.ServerVersion}} {{.OperatingSystem}}'
kind version
kubectl version --client

# 1. Create the three-node cluster (~1-3 min; first run pulls a ~1 GB image).
kind create cluster --config cluster/kind-cluster.yaml

# 2. Confirm the cluster exists and kubectl is pointed at it.
kind get clusters
kubectl config current-context      # -> kind-dso202
kubectl get nodes                   # -> control-plane, worker-node-1, worker-node-2

# 3. Create every object in one command.
kubectl apply -f manifests/

# 4. Make the namespace the default for this context, so -n can be omitted.
kubectl config set-context --current --namespace=dso202-practical

# 5. Wait for the workload to become ready.
kubectl rollout status deployment/web-deployment
kubectl wait --for=condition=Ready pod/web-pod pod/client-pod --timeout=90s

# 6. Verify the application answers from outside the cluster.
curl -s http://localhost:30080 | grep -o '<title>.*</title>'
```

## Cleanup

```bash
# Delete the workload objects, leaving the namespace, quota and limit range.
kubectl delete -f manifests/06-pod-client.yaml
kubectl delete -f manifests/05-service-nodeport.yaml
kubectl delete -f manifests/04-service-clusterip.yaml
kubectl delete -f manifests/03-deployment-web.yaml
kubectl delete -f manifests/02-pod-web.yaml
kubectl get all                     # -> No resources found

# Reset the default namespace so later work is not silently placed here.
kubectl config set-context --current --namespace=default

# Delete the cluster and confirm nothing is left running.
kind delete cluster --name dso202
kind get clusters                   # -> No kind clusters found.
docker ps                           # -> no kindest/node containers

# Optional: reclaim the ~1 GB node image. Keeping it makes the next
# cluster creation far faster, so remove it only if disk space is short.
docker image ls | grep kindest
# docker image rm kindest/node:v1.36.1
```

## Design notes

**Two names per node.** The Docker container name is chosen by kind and always
follows `<cluster-name>-control-plane` / `-worker` / `-worker2`; it cannot be
configured. The Kubernetes Node object name is the name the kubelet registers
with the API server, and `cluster/kind-cluster.yaml` patches the generated
kubeadm configuration to set it. This is why `docker ps` shows
`dso202-control-plane` while `kubectl get nodes` shows `control-plane`.

**The `managed-by` label.** `web-pod` (Listing 4) and the Deployment's Pods
(Listing 5) both carry `app=web`, so that Stage 4's label-selector exercises
work. They are distinguished by `managed-by`: `declarative` for the hand-written
Pod, `deployment` for the Pods the ReplicaSet creates. Both Services select
`app=web,managed-by=deployment`, so the unmanaged Pod is deliberately excluded
from the load-balanced backend set and the EndpointSlice lists exactly three
addresses.

**Named container port.** Both Services use `targetPort: http` - the *name* of
the container port - rather than the number 80, so that changing the container
port does not require editing the Services.

**Pinned `nodePort: 30080`.** Normally it is better to omit `nodePort` and let
the cluster allocate a free one. It is pinned here only because
`cluster/kind-cluster.yaml` publishes that exact port from the control-plane
container to the host, which is what makes `curl http://localhost:30080` work.

## References

- kind - Quick Start: https://kind.sigs.k8s.io/docs/user/quick-start/
- kind - Configuration: https://kind.sigs.k8s.io/docs/user/configuration/
- Kubernetes - Namespaces: https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/
- Kubernetes - Resource Quotas: https://kubernetes.io/docs/concepts/policy/resource-quotas/
- Kubernetes - Limit Ranges: https://kubernetes.io/docs/concepts/policy/limit-range/
- Kubernetes - Deployments: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Kubernetes - Service: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes - EndpointSlices: https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/

