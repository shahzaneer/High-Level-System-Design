# Troubleshooting: Pod & Container Issues

This covers the most common pod failure states and how to diagnose them. The general approach: check status, check events, check logs, check probes, check resources.

## CrashLoopBackOff

Pod starts, crashes, K8s restarts it, crashes again → exponential backoff.

```bash
# 1. Check what happened
kubectl describe pod <pod-name>
# Look for:
#   State: Waiting (CrashLoopBackOff)
#   Last State: Terminated (Exit Code: 1)
#   Events: "Back-off restarting failed container"

# 2. Check logs (include previous instance)
kubectl logs <pod>                          # Current instance
kubectl logs <pod> --previous               # Previous crashed instance
kubectl logs <pod> -c <container-name>      # Specific container

# 3. Common causes and fixes
```

| Cause | Symptom | Fix |
|-------|---------|-----|
| App panic/crash | Exit code 1, error in logs | Fix application bug |
| OOMKilled | Exit code 137 | Increase memory limits or fix memory leak |
| Wrong command/args | Exit code 127 (command not found) | Fix command in pod spec |
| Missing config/secret | Error mounting volume | Check ConfigMap/Secret exists |
| Probe failure | "Liveness probe failed" | Fix health endpoint |
| Signal handling | Exit code 143 (SIGTERM) | App must handle graceful shutdown |

## ImagePullBackOff

```bash
kubectl describe pod <pod> | grep -A5 "Events:"
# Common causes:
# - Wrong image name/tag
# - Registry authentication failure
# - Network cannot reach registry
# - Image doesn't exist

# Fixes:
kubectl get pod <pod> -o yaml | grep image:    # Verify image name
kubectl get secret regcred -n production       # Check registry credentials
kubectl describe pod <pod> | grep "Failed to pull image"
```

## Pending (Unschedulable)

```bash
kubectl describe pod <pod> | grep -A10 "Events:"
# Common causes:
# - Insufficient CPU/memory on any node
# - Node affinity/selector can't be satisfied
# - Taints that pod doesn't tolerate
# - PVC not bound (waiting for volume)

kubectl get nodes                         # Any nodes available?
kubectl describe nodes | grep -A5 "Allocated resources"  # Resource capacity
kubectl get pvc                           # Volumes bound?
```

## OOMKilled (Out of Memory)

```bash
kubectl describe pod <pod> | grep "OOMKilled"
# State: Terminated, Reason: OOMKilled, Exit Code: 137

# Check memory usage trend
kubectl top pod <pod>
kubectl top pod <pod> --containers

# Fix: increase memory limits, fix memory leak, or add swap (not recommended)
```

## CreateContainerConfigError / CreateContainerError

```bash
kubectl describe pod <pod> | grep -A10 "Events:"
# Causes: missing ConfigMap/Secret, invalid command, permission issues
kubectl get configmap <name>              # Does it exist?
kubectl get secret <name>                 # Does it exist?
```

## Debug Commands Reference

```bash
# Quick health check
kubectl get pods -n production
kubectl get pods -n production -o wide
kubectl get pods -n production --sort-by=.status.startTime

# Filter by status
kubectl get pods --field-selector=status.phase=Running
kubectl get pods --field-selector=status.phase=Pending

# Detailed investigation
kubectl describe pod <pod>                # Events, state, conditions
kubectl get events --sort-by=.lastTimestamp | tail -20
kubectl logs -f <pod>                     # Follow logs
kubectl exec -it <pod> -- /bin/sh         # Shell into container

# Ephemeral debug container (for distroless images)
kubectl debug -it <pod> --image=busybox --target=<container-name>
```

## Imperative vs Declarative

Troubleshooting is 100% imperative—you're investigating live state. The fixes are declarative: update YAML and apply.
