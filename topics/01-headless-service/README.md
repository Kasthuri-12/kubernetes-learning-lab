# Headless Service

A normal Service gets one virtual IP (ClusterIP) and load-balances across its pods. A headless Service (`clusterIP: None`) has no virtual IP: DNS returns the pod IPs directly, and each StatefulSet pod gets its own stable DNS name.

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
mysql-statefulset-0.my-db-headless-service.default.svc.cluster.local
```

Use it when clients must reach a specific pod, such as a database primary versus its replicas, or members of a cluster that discover each other by name.

| | ClusterIP Service | Headless Service |
|---|---|---|
| `clusterIP` | an allocated virtual IP | `None` |
| DNS lookup of the service name | one IP | the IPs of all ready pods |
| Per-pod DNS names | no | yes, with a StatefulSet |
| Load balancing | by kube-proxy | by the client |

## Manifests

| File | Purpose |
|---|---|
| [01-headless-service.yaml](manifests/01-headless-service.yaml) | The headless Service |
| [02-statefulset.yaml](manifests/02-statefulset.yaml) | 3 MySQL pods, each with its own 1Gi volume. `serviceName` must match the Service name. |
| [03-clusterip-service.yaml](manifests/03-clusterip-service.yaml) | A normal Service over the same pods, for comparison |

The three MySQL pods are independent instances. This lab shows naming and DNS, not MySQL replication.

## Lab

Run from this folder (`topics/01-headless-service`).

**1. Create the password Secret and the resources:**
```bash
kubectl create secret generic mysql-root --from-literal=password='choose-a-lab-password'
kubectl apply -f manifests/01-headless-service.yaml -f manifests/02-statefulset.yaml
kubectl get pods -w          # -0, then -1, then -2 start in order; Ctrl+C when all are Running
kubectl get pvc              # one volume per pod: data-mysql-statefulset-0, -1, -2
```

**2. Look up a single pod by name:**
```bash
kubectl run -it --rm --restart=Never --image=busybox:1.36 dns-test -- \
  nslookup mysql-statefulset-0.my-db-headless-service.default.svc.cluster.local
```
Expected: one address, the IP of pod `-0` (compare with `kubectl get pod mysql-statefulset-0 -o wide`).

**3. Look up the headless Service name:**
```bash
kubectl run -it --rm --restart=Never --image=busybox:1.36 dns-test -- nslookup my-db-headless-service
```
Expected: three addresses, one per pod.

**4. Compare with a normal Service:**
```bash
kubectl apply -f manifests/03-clusterip-service.yaml
kubectl run -it --rm --restart=Never --image=busybox:1.36 dns-test -- nslookup my-db-clusterip-service
kubectl get svc
```
Expected: one address, the ClusterIP. The headless Service shows `CLUSTER-IP: None`.

**5. See stable identity.** Delete a pod and watch it return with the same name and the same volume:
```bash
kubectl delete pod mysql-statefulset-1
kubectl get pods -w
```
A Deployment pod would come back with a new random name.

## Cleanup
```bash
kubectl delete -f manifests/
kubectl delete pvc -l app=mysql       # volumes are not removed with the StatefulSet
kubectl delete secret mysql-root
```

## Notes
- The original gist uses `mysql:latest` and a plaintext password in the manifest. Here the image is pinned and the password comes from a Secret that is not stored in the repo.
- Pod DNS records only exist for ready pods. Set `publishNotReadyAddresses: true` on the Service if peers must find each other before they are ready.
