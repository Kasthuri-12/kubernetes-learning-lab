# Helm

Helm is the package manager for Kubernetes, like APT or YUM on Linux. Instead of applying many YAML files by hand, you install one **chart** and customize it with **values**.

| Term | Meaning | Linux analogy |
|---|---|---|
| Chart | A package of templated Kubernetes manifests | a `.deb` file |
| Repository | A place that hosts packaged charts | an APT repo |
| Release | One installed instance of a chart in a cluster | an installed package |
| Values | Settings that fill in the chart's templates | `/etc` config files |

A chart is a folder:

```
mychart/
├── Chart.yaml        # name, chart version, app version
├── values.yaml       # default values
├── templates/        # Kubernetes manifests with {{ .Values.x }} placeholders
└── charts/           # dependencies
```

Why it helps: one command installs a whole app, per-environment differences live in values files, and every change is a numbered revision you can roll back.

## Prerequisites

```bash
unset DOCKER_DEFAULT_PLATFORM
minikube start --cpus=4 --memory=4096
kubectl config current-context        # must be: minikube
helm version
```

## Part A: Install a public chart

The tutorial this topic is based on uses `bitnami/nginx`. Bitnami changed how it distributes images in 2025, so image pulls from that chart may fail. This lab uses `podinfo`, a small demo app, to practice the same commands.

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm repo update
helm repo list
helm search repo podinfo

helm show values podinfo/podinfo | head -40       # what you can customize
helm template my-podinfo podinfo/podinfo | head   # render the YAML without installing

helm install my-podinfo podinfo/podinfo
helm list
kubectl get pods
helm status my-podinfo
```

Upgrade with a changed value, then look at the revisions:

```bash
helm upgrade my-podinfo podinfo/podinfo --set replicaCount=2
kubectl get pods
helm history my-podinfo
helm get values my-podinfo
```

Roll back and remove:

```bash
helm rollback my-podinfo 1
helm history my-podinfo               # rollback creates a new revision (3)
helm uninstall my-podinfo
```

Pin a chart version for repeatable installs: `helm install my-podinfo podinfo/podinfo --version <x.y.z>` (list versions with `helm search repo podinfo --versions`).

## Part B: Create your own chart

Scenario: two services, `payments` and `shipping`.

```bash
mkdir helm-lab && cd helm-lab
helm create payments
helm create shipping
tree payments        # or: find payments -type f
```

`helm create` generates a working nginx chart. Check it before installing:

```bash
helm lint payments
helm template payments ./payments | head -60
```

Change behaviour through values instead of editing templates. Create one values file per environment:

```yaml
# payments/values-dev.yaml
replicaCount: 1
resources:
  requests:
    cpu: 50m
    memory: 64Mi
```
```yaml
# payments/values-prod.yaml
replicaCount: 3
resources:
  requests:
    cpu: 200m
    memory: 256Mi
```

Install, upgrade and roll back:

```bash
helm install payments-dev ./payments -f payments/values-dev.yaml
kubectl get pods -l app.kubernetes.io/instance=payments-dev

helm upgrade payments-dev ./payments -f payments/values-prod.yaml    # replicas 1 -> 3
helm history payments-dev
helm rollback payments-dev 1

helm upgrade --install shipping-dev ./shipping      # install if missing, upgrade if present
helm list
```

To customize the message or image of a service, add keys to `values.yaml` and reference them in `templates/deployment.yaml` as `{{ .Values.appMessage }}`. Run `helm template` after each edit to see the result before installing.

## Part C: Publish your charts as a Helm repository

A Helm repository is a web location holding packaged charts (`.tgz`) and an `index.yaml`. GitHub Pages works for this.

```bash
cd helm-lab
helm package payments shipping          # creates payments-0.1.0.tgz and shipping-0.1.0.tgz
helm repo index . --url https://<your-user>.github.io/helm-repo
```

Then:
1. Create a GitHub repository named `helm-repo`.
2. Commit the `.tgz` files and `index.yaml`, and push to `main`. Use a credential helper such as `gh auth login`. Don't put a token in the remote URL.
3. In the repo settings, enable GitHub Pages from the `main` branch.

Use it like any other chart repository:
```bash
helm repo add myrepo https://<your-user>.github.io/helm-repo
helm repo update
helm search repo myrepo
helm install payments-service myrepo/payments
```

## Cleanup

```bash
helm uninstall payments-dev shipping-dev
helm repo remove podinfo
minikube stop
```

## Command cheat sheet

| Task | Command |
|---|---|
| Add or refresh a repo | `helm repo add <name> <url>`, `helm repo update` |
| Find charts | `helm search repo <keyword>` |
| See default values | `helm show values <chart>` |
| Render without installing | `helm template <release> <chart>` |
| Check a chart for problems | `helm lint <chart>` |
| Install | `helm install <release> <chart> -f values.yaml --set key=value` |
| Install or upgrade | `helm upgrade --install <release> <chart>` |
| List and inspect | `helm list`, `helm status <release>`, `helm get values <release>` |
| History and rollback | `helm history <release>`, `helm rollback <release> <revision>` |
| Remove | `helm uninstall <release>` |
| Package and index | `helm package <chart>`, `helm repo index . --url <url>` |

## Credits

Based on [iam-veeramalla/helm-zero-to-hero](https://github.com/iam-veeramalla/helm-zero-to-hero) (Apache-2.0).
