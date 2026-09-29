# Upgrade and rollout notes

Kubernetes changes should be treated as a sequence of observable steps rather than a single deployment command.

## Before the change

- Confirm the target image or chart version.
- Check resource requests against available node capacity.
- Review PodDisruptionBudget settings.
- Confirm readiness checks represent application readiness rather than merely process startup.
- Verify the rollback version is known and available.

## During rollout

Watch the Deployment and ReplicaSets first:

```bash
kubectl -n platform-demo rollout status deployment/platform-app
kubectl -n platform-demo get rs
kubectl -n platform-demo get pods -o wide
```

If progress stops, inspect events and the affected Pod before changing the Deployment:

```bash
kubectl -n platform-demo describe pod <pod>
kubectl -n platform-demo get events --sort-by=.lastTimestamp
kubectl -n platform-demo logs <pod> --previous
```

## Rollback

Rollback should be used when the new version cannot reach the expected healthy state or introduces a regression that cannot be safely resolved in place.

```bash
kubectl -n platform-demo rollout history deployment/platform-app
kubectl -n platform-demo rollout undo deployment/platform-app
kubectl -n platform-demo rollout status deployment/platform-app
```

After rollback, verify Service endpoints and application health. A successful Deployment rollout alone does not prove that the application is behaving correctly for users.

## What to record

For a real production change, capture the version deployed, start/end time, observed symptoms, health signals, rollback decision if applicable, and the follow-up action. This makes the next incident easier to diagnose instead of relying on memory.
