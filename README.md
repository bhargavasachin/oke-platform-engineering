# OKE Platform Engineering

A public reference implementation of Kubernetes platform patterns for OCI OKE: workload packaging, safe health checks, resource controls, scaling, and operational troubleshooting.

> **Portfolio note:** This repository is based on patterns from cloud/platform engineering work. It is not an export of an employer's internal platform and contains no internal names, credentials, or proprietary configuration.

## What this demonstrates

- Kubernetes workload structure that is easy to operate
- Readiness and liveness probes with deliberate failure behavior
- CPU/memory requests and limits
- Rolling updates and rollback-friendly deployment settings
- PodDisruptionBudget for voluntary disruption
- Horizontal Pod Autoscaling
- Helm-based packaging
- Operational troubleshooting patterns

## Layout

```text
.
├── manifests/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── pdb.yaml
│   └── hpa.yaml
├── helm/
│   └── platform-app/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── docs/
│   └── troubleshooting.md
└── README.md
```

## Production-minded details

A readiness failure removes a pod from Service endpoints without necessarily restarting it. Liveness is reserved for conditions where restarting the container is expected to recover the process. This distinction prevents a slow dependency from turning into a restart loop.

Resource requests are used for scheduling and capacity planning; limits are treated as an explicit workload constraint rather than a substitute for sizing. HPA is meaningful only when the workload has requests and the cluster exposes the required metrics.

The examples use placeholders such as `YOUR_REGISTRY/your-image:tag`. No registry credentials are included.

## Apply the example

The manifests are intentionally generic and can be used as a starting point for an OKE workload:

```bash
kubectl apply -f manifests/namespace.yaml
kubectl apply -f manifests/deployment.yaml
kubectl apply -f manifests/service.yaml
kubectl apply -f manifests/pdb.yaml
kubectl apply -f manifests/hpa.yaml
```

Inspect rollout and health:

```bash
kubectl -n platform-demo rollout status deployment/platform-app
kubectl -n platform-demo get pods
kubectl -n platform-demo get endpoints
```

## Troubleshooting mindset

Start with the symptom and move down the Kubernetes control path: Deployment -> ReplicaSet -> Pod -> container state -> events -> logs -> Service -> endpoints -> ingress/load balancer. Avoid changing multiple variables at once; the goal is to identify the first failing boundary.
