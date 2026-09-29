# Kubernetes troubleshooting notes

## CreateContainerConfigError

Start with:

```bash
kubectl -n platform-demo describe pod <pod>
kubectl -n platform-demo get events --sort-by=.lastTimestamp
```

Check referenced ConfigMaps, Secrets, service accounts, volumes and environment variables before changing the Deployment.

## Readiness probe failures

A readiness failure normally means the pod should not receive traffic. Check the application endpoint from inside the pod, then inspect logs and events. Do not automatically increase probe delays; first determine whether the application is actually becoming ready.

```bash
kubectl -n platform-demo logs <pod> --previous
kubectl -n platform-demo exec <pod> -- wget -qO- http://127.0.0.1:8080/ready
kubectl -n platform-demo get endpoints platform-app
```

## Pods stuck in Terminating

Check the pod description and finalizers first. Also determine whether a node is reachable and whether the workload has a long termination grace period or a preStop hook. Force deletion should be a last resort because it can bypass normal application shutdown.

## Image pull failures

Check the image name/tag, registry reachability and image-pull credentials. `describe pod` usually shows the first useful event. Avoid changing application configuration when the container image has not been pulled successfully.

## CPU or memory pressure

Compare requested resources with actual usage and node capacity. Requests influence scheduling; limits constrain the container. A pod that is repeatedly OOM-killed needs a memory-sizing investigation rather than simply increasing the replica count.
