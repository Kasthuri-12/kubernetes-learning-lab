# Reducing Kubernetes cluster costs: 5 practical ways

Cluster cost is mostly node cost, and nodes are sized by what pods **request**, not by what they use. Every technique below either shrinks what you reserve, adds capacity only when needed, buys capacity more cheaply, or removes network charges.

| # | Technique | Main idea | Where to run |
|---|---|---|---|
| 1 | [Cost visibility](#1-cost-visibility) | You can't cut what you can't see | minikube for `kubectl top`; a real cluster for cost tools |
| 2 | [Right-sizing](#2-right-sizing-with-requests-and-limits) | Requests match real usage | minikube |
| 3 | [Autoscaling](#3-autoscaling) | Pay for capacity only while load exists | minikube for HPA; a cloud cluster for node autoscaling |
| 4 | [Spot instances](#4-spot-instances) | Run interruption-tolerant work on discounted nodes | Cloud cluster only |
| 5 | [Cross-zone networking](#5-cross-zone-networking-cost) | Keep traffic inside one availability zone | Multi-zone cloud cluster only |

Run commands from `topics/02-cost-optimization`. Check `kubectl config current-context` first. Items marked cloud-only create billable resources, so use a sandbox subscription or account, never a shared cluster.

## 1. Cost visibility

**Why:** teams usually don't know which namespace, team or app drives the bill. Start by measuring.

**What to do**
1. Label namespaces and workloads consistently (`team`, `env`, `cost-center`) so spend can be grouped. See [labelled-namespace.yaml](manifests/01-cost-visibility/labelled-namespace.yaml).
2. Watch real usage against requests with metrics-server.
3. On a real cluster, add a cost tool (OpenCost or Kubecost) and your cloud's cost reports, with cost-allocation tags or labels turned on.

```bash
minikube addons enable metrics-server
kubectl apply -f manifests/01-cost-visibility/labelled-namespace.yaml
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl get pods -A -l team=payments
```

`kubectl top` shows usage only. Allocation by team and dollar figures need a cost tool such as OpenCost, which also needs Prometheus. Follow its current install docs for your cluster.

**Look for:** pods whose requests are far above their usage, idle namespaces, unattached volumes and load balancers.

```bash
kubectl delete -f manifests/01-cost-visibility/labelled-namespace.yaml
```

## 2. Right-sizing with requests and limits

**Why:** the scheduler places pods by their **requests**. Oversized requests leave nodes mostly idle but still billed.

| Setting | Effect |
|---|---|
| `requests` | Reserved capacity. This is what drives node count and cost. |
| `limits.memory` | Hard ceiling. Exceeding it gets the container OOM-killed. |
| `limits.cpu` | Ceiling that throttles the container. Many teams omit it to avoid throttling. |

**Rule of thumb:** set the request near typical (for example p90) usage measured over time, and give memory a limit with headroom above the request.

**Lab:** compare the reservations of an oversized and a right-sized version of the same app.
```bash
minikube addons enable metrics-server
kubectl apply -f manifests/02-right-sizing/01-overprovisioned.yaml
kubectl describe node minikube | grep -A8 "Allocated resources"     # note the CPU and memory requests
kubectl top pods

kubectl apply -f manifests/02-right-sizing/02-rightsized.yaml       # same name, so it updates in place
kubectl rollout status deploy/web-big
kubectl describe node minikube | grep -A8 "Allocated resources"     # requests are much lower
```
Five idle nginx pods reserve 2.5 CPU and 2.5Gi in the first version and 50m and 160Mi in the second.

**Guard rails:** a `LimitRange` supplies defaults to pods that set nothing, and a `ResourceQuota` caps a namespace's total reservations.
```bash
kubectl apply -f manifests/02-right-sizing/03-limitrange-and-quota.yaml
kubectl run nolimits -n sizing-lab --image=nginx:1.27
kubectl get pod nolimits -n sizing-lab -o jsonpath='{.spec.containers[0].resources}{"\n"}'   # defaults were injected
```

**Recommendations from data:** the Vertical Pod Autoscaler in `Off` mode publishes suggested requests without changing anything. See [04-vpa-recommendation.yaml](manifests/02-right-sizing/04-vpa-recommendation.yaml); it needs VPA installed.

```bash
kubectl delete -f manifests/02-right-sizing/01-overprovisioned.yaml
kubectl delete -f manifests/02-right-sizing/03-limitrange-and-quota.yaml    # also removes the nolimits pod
```

## 3. Autoscaling

**Why:** fixed capacity sized for peak load is idle most of the time.

| Layer | Tool | Scales |
|---|---|---|
| Pods | Horizontal Pod Autoscaler (HPA) | Replica count from CPU, memory or custom metrics |
| Pods | KEDA | Replica count from events (queues, schedules), including scale to zero |
| Nodes | Cluster Autoscaler or Karpenter, or the managed option on your cloud | Node count, so unused nodes are removed |

Pod autoscaling saves money only when node autoscaling removes the nodes that empty out.

**Lab (HPA on minikube)**
```bash
minikube addons enable metrics-server
kubectl apply -f manifests/03-autoscaling/01-deployment.yaml -f manifests/03-autoscaling/02-hpa.yaml
kubectl get hpa web-hpa -w                  # wait until TARGETS shows a number instead of <unknown>
```
In a second terminal, generate load:
```bash
kubectl apply -f manifests/03-autoscaling/03-load-generator.yaml
```
Back in the first terminal, REPLICAS should rise toward 5. Stop the load and it falls back to 1 after the stabilization window:
```bash
kubectl delete pod load-generator
```

**Cleanup**
```bash
kubectl delete -f manifests/03-autoscaling/01-deployment.yaml -f manifests/03-autoscaling/02-hpa.yaml
```

**Node autoscaling** needs a cloud cluster. Enable the autoscaler on the node pool with a minimum and maximum, and make sure pods have realistic requests, because the autoscaler adds nodes when requests don't fit.

## 4. Spot instances

**Why:** cloud providers sell spare capacity at a large discount (the amount varies by provider, region and time), and can reclaim it with little notice.

**Good for:** stateless, replicated, restartable work such as web tiers behind multiple replicas, batch jobs and CI runners. **Avoid for:** single-replica services and databases.

**Checklist for running on spot**
- Keep a baseline of on-demand nodes and put only part of the workload on spot.
- Run more than one replica and spread them across nodes.
- Add a PodDisruptionBudget.
- Handle `SIGTERM` and finish quickly. Set `terminationGracePeriodSeconds` to match.
- Tolerate the spot taint and prefer spot nodes with affinity.

| Cloud | Spot node label | Taint |
|---|---|---|
| AKS | `kubernetes.azure.com/scalesetpriority=spot` | `kubernetes.azure.com/scalesetpriority=spot:NoSchedule` (added automatically) |
| EKS (managed node group, capacity type SPOT) | `eks.amazonaws.com/capacityType=SPOT` | none by default |
| GKE | `cloud.google.com/gke-spot=true` | none by default |

**Manifests:** [01-spot-deployment.yaml](manifests/04-spot-instances/01-spot-deployment.yaml) uses the AKS values; change the key and values for another cloud. [02-pdb.yaml](manifests/04-spot-instances/02-pdb.yaml) keeps two replicas up during drains.

**Lab (AKS sandbox only; this creates billable nodes)**
```bash
az aks nodepool add --resource-group <rg> --cluster-name <cluster> --name spotpool \
  --priority Spot --eviction-policy Delete --spot-max-price -1 \
  --enable-cluster-autoscaler --min-count 1 --max-count 3 --node-count 1

kubectl get nodes -L kubernetes.azure.com/scalesetpriority
kubectl apply -f manifests/04-spot-instances/
kubectl get pods -l app=spot-worker -o wide         # replicas land on spot nodes

kubectl delete -f manifests/04-spot-instances/
az aks nodepool delete --resource-group <rg> --cluster-name <cluster> --name spotpool
```
Without a cloud cluster you can still check the manifests parse: `kubectl apply --dry-run=client -f manifests/04-spot-instances/`.

## 5. Cross-zone networking cost

**Why:** cloud providers typically charge per GB for traffic that crosses availability zones. A chatty service calling a backend in another zone pays that charge on every request, and adds latency.

**Two separate goals**
1. **Spread pods across zones** for availability: `topologySpreadConstraints`.
2. **Send traffic to a pod in the caller's own zone** for cost: topology-aware routing.

Doing only the second can overload one zone when pods are uneven, so do both.

**Manifests**
- [01-topology-aware-service.yaml](manifests/05-cross-zone-networking/01-topology-aware-service.yaml): `trafficDistribution: PreferClose` (Kubernetes 1.31 or later). Older clusters use the annotation `service.kubernetes.io/topology-mode: Auto`.
- [02-zone-spread-deployment.yaml](manifests/05-cross-zone-networking/02-zone-spread-deployment.yaml): six replicas spread evenly across zones.
- [03-istio-locality.yaml](manifests/05-cross-zone-networking/03-istio-locality.yaml): the same idea in Istio, with locality load balancing. Needs Istio, and `outlierDetection` must be present.

**Lab (multi-zone cloud cluster)**
```bash
kubectl get nodes -L topology.kubernetes.io/zone       # confirm nodes are in several zones
kubectl apply -f manifests/05-cross-zone-networking/02-zone-spread-deployment.yaml
kubectl apply -f manifests/05-cross-zone-networking/01-topology-aware-service.yaml
kubectl get pods -l app=backend -o wide                # replicas on nodes in different zones
kubectl get endpointslices -l kubernetes.io/service-name=backend -o yaml | grep -i -B2 -A4 hints
```
Endpoints carry zone hints when topology-aware routing is active. minikube has a single zone, so it can't demonstrate the effect. Measure the saving with your cloud's network or data-transfer cost reports.

```bash
kubectl delete -f manifests/05-cross-zone-networking/01-topology-aware-service.yaml -f manifests/05-cross-zone-networking/02-zone-spread-deployment.yaml
```

## Putting it together

Start with 1 and 2, because they cost nothing and make every later step more effective. Then add 3, move tolerant workloads to 4, and fix 5 for chatty multi-zone services.
