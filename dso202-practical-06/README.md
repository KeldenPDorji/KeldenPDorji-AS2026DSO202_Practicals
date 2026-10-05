# DSO202 - Practical 6

**Helm: charts, templates, values and the release lifecycle (Unit III 3.2)**

Module: DSO202 - Scaling, Orchestration, Monitoring & Observability
Programme: BE in Software Engineering

## Purpose

Kustomize (Practical 5) produces environment variants of manifests, but
nothing in the cluster records what was installed, and nothing removes or
rolls back a group of objects as one unit. This practical uses **Helm 4**:

1. **Consuming a chart** - podinfo is found, read, installed, inspected,
   upgraded (including the values-loss failure), rolled back, uninstalled,
   and installed again from an OCI registry.
2. **Writing a chart** - `webapp`, an nginx page whose content comes from
   values, built file by file with the industry conventions: recommended
   labels, name truncation, a ConfigMap checksum on the Pod template, and a
   `helm test` Pod.
3. **Values and validation** - merge precedence, and four deliberate failures
   that show which check (`required`, `values.schema.json`, `helm lint`,
   `helm template`, server-side dry runs) catches what.
4. **Lifecycle under Helm 4** - one chart deployed as dev and prod releases,
   tested, then drifted with `kubectl` to show server-side apply conflicts and
   Helm's refusal to adopt objects it does not own.

## Software versions used

| Software | Version |
| --- | --- |
| Operating system | macOS 26.6.2, arm64 |
| Docker Desktop | 29.7.2 |
| kind | v0.32.0 (go1.26.3 darwin/arm64) |
| kubectl | v1.36.3 |
| Helm | **v4.3.0** |
| Kubernetes (cluster) | v1.36.1 (`kindest/node:v1.36.1`) |

> **Helm 4 is required.** Helm 3 uses client-side apply, so the release record
> has no `apply_method` and drift is overwritten silently instead of raising a
> conflict. On macOS:
>
> ```bash
> export PATH="/opt/homebrew/bin:$PATH"
> brew upgrade helm
> helm version        # must report v4.x
> ```

## Repository layout

The handout's lab directory `dso202-helm-lab/` is this directory.

```
dso202-practical-06/
├── README.md                      # this file
├── .gitignore                     # excludes outputs/
├── cluster/
│   └── kind-cluster.yaml          # Practical 1 Listing 1: 3 nodes, host port 30080
├── webapp/                        # the chart (Stages 2-5)
│   ├── Chart.yaml                 # version 0.1.0, appVersion "1.30-alpine"
│   ├── values.yaml                # documented defaults: the chart's interface
│   ├── values.schema.json         # types and permitted values (added in Stage 5)
│   ├── .helmignore
│   └── templates/
│       ├── _helpers.tpl           # name, fullname, chart, labels, environment
│       ├── configmap.yaml         # the HTML page, built from values
│       ├── deployment.yaml        # nginx + checksum/config annotation
│       ├── service.yaml           # ClusterIP or NodePort
│       ├── NOTES.txt              # printed after install/upgrade
│       └── tests/
│           └── test-connection.yaml   # helm test: fetches the page, checks the environment
├── environments/                  # per-environment values, outside the chart
│   ├── dev.yaml                   # 1 replica, NodePort 30080
│   └── prod.yaml                  # 3 replicas, larger resources
├── checks/                        # Stage 5 inputs
│   ├── tag-float.yaml             # unquoted image tag (Failure 3)
│   └── values.schema.json         # source copied into webapp/ in Stage 5
├── outputs/                       # renders and deliberately broken chart copies (git-ignored)
├── evidence/                      # screenshots (.png)
└── report/
    └── practical-06-report.md     # the assessed report
```

## Rebuild from an empty machine

Run from this directory.

```bash
# 0. Tooling and cluster.
export PATH="/opt/homebrew/bin:$PATH"
helm version
kind create cluster --config cluster/kind-cluster.yaml
kubectl wait --for=condition=Ready nodes --all --timeout=180s

# 1. Validate the chart for every environment before touching the cluster.
for env in dev prod; do
  helm lint ./webapp -f environments/$env.yaml &&
  helm template webapp-$env ./webapp -f environments/$env.yaml -n dso202-$env --skip-tests \
    | kubectl apply --dry-run=server -f - > /dev/null &&
  echo "$env: ok"
done
# The server-side dry run needs the namespaces to exist; create them first
# (kubectl create namespace dso202-dev dso202-prod) or run step 2 once.

# 2. Install (idempotent - the same command upgrades later).
#    A first install on a slow network can exceed 3m while images pull;
#    raise --timeout rather than retrying.
helm upgrade --install webapp-dev  ./webapp -f environments/dev.yaml  -n dso202-dev  --create-namespace --wait --timeout 5m
helm upgrade --install webapp-prod ./webapp -f environments/prod.yaml -n dso202-prod --create-namespace --wait --timeout 5m

# 3. Verify.
helm list -A
curl -s http://localhost:30080
helm test webapp-dev  -n dso202-dev
helm test webapp-prod -n dso202-prod
```

Always pass the environment file on every upgrade. An upgrade given only a
`--set` flag starts again from the chart defaults and silently discards the
rest.

## Cleanup

`helm uninstall` removes every object a release created and every release
record. It does **not** remove the namespace (`--create-namespace` does not
make it part of the release), or objects no release owns, such as the
hand-made `webapp-qa` ConfigMap from Stage 6. Deleting the namespaces takes
care of both.

```bash
# 1. Remove the releases.
helm uninstall webapp-dev  -n dso202-dev
helm uninstall webapp-prod -n dso202-prod
helm list -A                                   # headings only

# 2. Remove the namespaces (dso202-helm is left over from Stage 1).
kubectl delete namespace dso202-dev dso202-prod dso202-helm

# 3. Remove the podinfo repository alias and local scratch output.
helm repo remove podinfo
rm -rf outputs

# 4. Delete the cluster.
kind delete cluster --name dso202
kind get clusters
```

## Known limits of this environment

| On this cluster | In industry |
| --- | --- |
| Releases installed by hand with `helm upgrade --install` | A CI pipeline or a GitOps controller (Argo CD, Flux) runs Helm from Git |
| Values files hold only non-secret settings | Secrets come from an external store (External Secrets, Sealed Secrets, SOPS), never from values |
| The chart is used from a directory | Charts are packaged (`helm package`) and pushed to an OCI registry, then pinned by version or digest |
| dev and prod share one cluster, separated by namespace | Production runs on its own cluster with separate credentials |
