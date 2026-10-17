# Kubernetes Learning Lab

Short, hands-on Kubernetes topics. Each topic has a README that explains the concept, the manifests to apply, the commands to run, and what you should see.

## Topics

| # | Topic | What it covers | Where to run |
|---|---|---|---|
| 1 | [Headless Service](topics/01-headless-service) | `clusterIP: None`, StatefulSet pod DNS, comparison with a normal Service | minikube |
| 2 | [Cluster cost optimization](topics/02-cost-optimization) | Cost visibility, right-sizing, autoscaling, spot instances, cross-zone networking cost | Mixed: see the table in the topic README |
| 3 | [Helm](topics/03-helm) | Charts, repositories, releases, values, upgrade and rollback, building and publishing your own chart | minikube |

## Status

Nothing in this repo has been run end to end yet. Update this table as you verify each topic.

| Topic | Status |
|---|---|
| 1. Headless Service | Not yet run |
| 2.1 Cost visibility | Not yet run |
| 2.2 Right-sizing | Not yet run |
| 2.3 Autoscaling | Not yet run |
| 2.4 Spot instances | Not yet run (needs a cloud cluster) |
| 2.5 Cross-zone networking | Not yet run (needs a multi-zone cluster) |
| 3. Helm | Not yet run |

## Prerequisites

`docker`, `minikube`, `kubectl`, and `helm` for topic 3. Start a local cluster:

```bash
unset DOCKER_DEFAULT_PLATFORM      # an amd64 default breaks the minikube node image on Apple Silicon
minikube start --cpus=4 --memory=4096
kubectl config current-context     # must be: minikube
```

Check the context before every `apply` or `delete`. Never run these labs against a shared or production cluster.

## Credits

The headless Service topic is based on the [headless-service-demo gist](https://gist.github.com/iam-veeramalla/d3dbcc2f1fc626a211d7c4dbfef28570) by iam-veeramalla. The manifests were rewritten here (pinned image, Secret for the password, persistent volumes). The cost optimization topic follows the five-way outline of the tutorial it is based on. The Helm topic is based on [helm-zero-to-hero](https://github.com/iam-veeramalla/helm-zero-to-hero) (Apache-2.0).
