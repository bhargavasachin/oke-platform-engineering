# CrashLoopBackOff runbook

`CrashLoopBackOff` means the container starts, exits, and the kubelet restarts
it with an exponential backoff (10s, 20s, 40s, ... capped at 5 minutes). It is
a symptom, not a cause. These are the checks I run, in order, before changing
anything.

## First five minutes

**1. Identify the crashing container and its restart count.**

```bash
kubectl -n platform-demo get pods
```

A high restart count on one container tells you where to look. If several
containers in the pod are restarting, suspect the pod spec, not the app.

**2. Read the previous container's logs first.**

```bash
kubectl -n platform-demo logs <pod> -c <container> --previous
```

`--previous` shows the crashed instance. The current container may be a fresh
one that has not logged anything useful yet. If `--previous` is empty, the
container is likely dying before it can log — move to events.

**3. Inspect the terminated state. The exit code names the category.**

```bash
kubectl -n platform-demo describe pod <pod>
```

Look at `lastState.terminated` for the crashed container:

| Exit code / reason | What it usually means |
| --- | --- |
| `1`, reason `Error` | Application startup failure. Read the logs, then check config, env, and args. |
| `137`, reason `OOMKilled` | The container hit its memory limit. This needs a sizing investigation, not just a bigger limit. |
| `139` | Segmentation fault. Suspect the binary or base image, not the manifest. |
| `0`, restarting anyway | The command completed successfully but there is nothing keeping the container alive — wrong `CMD`/`entrypoint`. |

**4. Check what changed recently.**

```bash
kubectl -n platform-demo rollout history deployment/platform-app
```

If the crash started with the latest revision, the fastest mitigation is a
rollback (`rollout undo`), then diagnose without production pressure.

## Common causes, in the order I check them

1. **Bad configuration.** Wrong env var, missing ConfigMap/Secret key, or bad
   command args. `CreateContainerConfigError` is the hard failure version of
   this; `CrashLoopBackOff` is the soft one where the app starts and then dies.
2. **Dependency unreachable at startup.** The app should retry with backoff
   instead of exiting, but when it exits, the runbook still starts here —
   check the dependency before the manifest.
3. **Wrong port.** The container listens on one port while the manifest or
   probes target another.
4. **Liveness probe killing a slow starter.** The classic self-inflicted
   crash loop: the app needs 60s to start, liveness fires at 30s, the kubelet
   kills it, repeat forever. Fix with a `startupProbe`, or a liveness
   `initialDelaySeconds` that respects real startup time. Liveness is for
   "restart will recover this"; it must not punish slow startup.
5. **Resource limits.** `OOMKilled` (exit 137) — compare usage against the
   limit before changing it.
6. **Image or entrypoint.** Wrong `CMD`, a script that exits immediately, or
   a binary that cannot execute in the image.

## Preserve evidence before you fix

Capture `logs --previous` and `describe pod` output before redeploying or
rolling back — a rollback destroys the failing state you need for the
postmortem. Change one variable at a time; a useful diagnosis explains why
the kubelet kept restarting the container, not just how to make the symptom
disappear.

## Mitigation vs. diagnosis

- Crash correlates with a deploy and production is affected: `rollout undo`
  first, diagnose second.
- Workload never ran successfully: fix forward in the manifest; there is no
  good revision to return to.
