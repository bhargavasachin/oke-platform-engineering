# OKE Operations Notes

These notes capture the checks I use when a Kubernetes workload is not behaving as expected. The goal is to find the first failing boundary before changing configuration.

## 1. Pod is not Ready

Start at the workload and move toward the container:

```bash
kubectl -n platform-demo get deployment platform-app
kubectl -n platform-demo get rs
kubectl -n platform-demo get pods -o wide
kubectl -n platform-demo describe pod <pod-name>
kubectl -n platform-demo logs <pod-name> --all-containers
kubectl -n platform-demo get events --sort-by=.lastTimestamp
```

Check the probe path, container port, startup time, and recent events before changing the probe thresholds.

## 2. `CreateContainerConfigError`

This usually means the container could not be constructed from the Pod specification. Inspect the Pod events and then verify referenced ConfigMaps, Secrets, service-account configuration, and volume references.

```bash
kubectl -n platform-demo describe pod <pod-name>
kubectl -n platform-demo get configmap
kubectl -n platform-demo get secret
```

Do not immediately recreate the workload. The event normally identifies the missing or invalid dependency.

## 3. Image pull failure

Check the exact image reference first:

```bash
kubectl -n platform-demo describe pod <pod-name>
```

Look for `ErrImagePull` or `ImagePullBackOff`. Verify the registry hostname, image tag/digest, and the service account or image-pull secret used by the workload.

## 4. Service has no endpoints

A Service can exist while routing to no Pods. Compare the Service selector with Pod labels and inspect the endpoints:

```bash
kubectl -n platform-demo get svc platform-app -o yaml
kubectl -n platform-demo get pods --show-labels
kubectl -n platform-demo get endpoints platform-app
kubectl -n platform-demo get endpointslices
```

If readiness is failing, the Pod may be running but intentionally absent from the ready endpoint set.

## 5. Pod stuck terminating

First inspect the object and recent events. Check whether a finalizer, volume detach, node problem, or application shutdown is preventing completion.

```bash
kubectl -n platform-demo get pod <pod-name> -o yaml
kubectl -n platform-demo describe pod <pod-name>
```

Force deletion should be treated as a recovery action, not the normal solution, because it can bypass graceful shutdown semantics.

## 6. Rollout is stalled

```bash
kubectl -n platform-demo rollout status deployment/platform-app
kubectl -n platform-demo get rs
kubectl -n platform-demo get pods
```

Compare the new ReplicaSet with the previous one. Check scheduling, image availability, readiness, resource requests, and events before rolling back.

## Operating principle

When troubleshooting production workloads, change one variable at a time and preserve evidence from the failing state. A useful diagnosis explains **why** the controller behaved as it did, not just how to make the symptom disappear.
