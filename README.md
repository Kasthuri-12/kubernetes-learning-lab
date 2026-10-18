# Kubernetes Learning Lab

Hands-on labs for learning Kubernetes by doing. Each topic explains a concept in plain words, gives the manifests or charts to apply, the commands to run, the output to expect, and how to clean up. Everything runs on a local minikube cluster unless a topic says it needs a cloud cluster.

## Topics

| # | Topic | What you learn | Where to run |
|---|---|---|---|
| 1 | [Headless Service](topics/01-headless-service) | Why `clusterIP: None` exists, how StatefulSet pods get stable DNS names, and how that differs from a normal Service | minikube |
| 2 | [Cluster cost optimization](topics/02-cost-optimization) | Five ways to cut cluster cost: cost visibility, right-sizing requests and limits, autoscaling, spot instances, cross-zone networking | minikube for visibility, right-sizing and autoscaling; a cloud cluster for spot and cross-zone |
| 3 | [Helm](topics/03-helm) | Charts, repositories, releases and values; install, upgrade and roll back; build your own chart and publish it as a chart repository | minikube |

## Repository layout

```
topics/
├── 01-headless-service/     README + Service and StatefulSet manifests
├── 02-cost-optimization/    README + manifests, one folder per technique
└── 03-helm/                 README with all the commands and a cheat sheet
```

## How to use it

1. Start a local cluster (see Prerequisites below).
2. Open a topic's README and read the concept section first.
3. Run the commands from that topic's folder, and compare your output with what the README says to expect.
4. Run the cleanup commands at the end of the topic before starting the next one.

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
