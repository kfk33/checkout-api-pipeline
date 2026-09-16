# Runbook: Responding to an OOMKilled `checkout-api` Pod

## Scope

Use this runbook when `checkout-api` pods in the `checkout-api` namespace are restarting, unavailable, or reported as `OOMKilled`.

Known-good baseline from the deployment manifest:

- Deployment: `checkout-api`
- Namespace: `checkout-api`
- Container: `checkout-api`
- Memory request: `128Mi`
- Memory limit: `128Mi`

## Diagnosis

### 1. Check pod health and restart counts

```bash
kubectl get pods -n checkout-api -l app=checkout-api -o wide
kubectl get deployment checkout-api -n checkout-api
```

Look for a rising `RESTARTS` count, `CrashLoopBackOff`, or pods that are not `Ready`.

### 2. Identify the affected pod

```bash
POD=$(kubectl get pods -n checkout-api -l app=checkout-api \
  -o jsonpath='{.items[0].metadata.name}')
echo "$POD"
```

If more than one pod is present, select the pod with the highest restart count instead.

### 3. Confirm OOMKilled and inspect recent events

```bash
kubectl describe pod "$POD" -n checkout-api
kubectl get pod "$POD" -n checkout-api \
  -o jsonpath='{range .status.containerStatuses[*]}{.name}{" lastState="}{.lastState.terminated.reason}{" exitCode="}{.lastState.terminated.exitCode}{"\n"}{end}'
kubectl get events -n checkout-api --sort-by=.lastTimestamp | tail -20
```

Confirm that the `checkout-api` container's last terminated state has `reason=OOMKilled` (normally exit code `137`). In `describe` output, also check the container's memory limit and whether the pod was evicted for a node-level resource problem. An `OOMKilled` container usually indicates that the container exceeded its own memory limit; `Evicted` indicates node pressure and requires a different investigation.

### 4. Check current resource settings and usage

```bash
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{" requests="}{.resources.requests.memory}{" limits="}{.resources.limits.memory}{"\n"}{end}'
kubectl top pod "$POD" -n checkout-api --containers
```

`kubectl top` requires Metrics Server. Treat a missing metrics result as a tooling limitation, not evidence that memory use is low. Compare the configured limit with recent usage and with the known-good `128Mi` baseline. A limit such as `8Mi` is too low for this workload, even at idle.

### 5. Check the rollout and recent configuration

```bash
kubectl rollout history deployment/checkout-api -n checkout-api
kubectl get deployment checkout-api -n checkout-api -o yaml
```

Look for a recent resource-limit change or another pod-template change that caused the restart loop.

## Resolution

### 1. Restore the known-good memory limit

If the diagnosis confirms that the limit was reduced below the workload's requirement, restore the deployment manifest to `128Mi` for both request and limit, then apply it:

```bash
kubectl -n checkout-api patch deployment checkout-api --type='strategic' \
  -p='{"spec":{"template":{"spec":{"containers":[{"name":"checkout-api","resources":{"requests":{"memory":"128Mi"},"limits":{"memory":"128Mi"}}}]}}}}'
```

For a controlled change through the repository, update `checkout-api-deployment.yaml` and apply it instead:

```bash
kubectl apply -f checkout-api-pipeline/checkout-api-deployment.yaml
```

Do not increase the limit blindly if current usage indicates a genuine application memory leak; preserve evidence and escalate that case for application investigation.

### 2. Watch the rollout

```bash
kubectl rollout status deployment/checkout-api -n checkout-api --timeout=120s
kubectl get pods -n checkout-api -l app=checkout-api -w
```

Stop watching with `Ctrl-C` after the replacement pod is `Running` and `Ready`.

### 3. Verify stability and service health

```bash
kubectl get pods -n checkout-api -l app=checkout-api
kubectl get deployment checkout-api -n checkout-api
kubectl describe pod "$POD" -n checkout-api
kubectl logs deployment/checkout-api -n checkout-api --tail=100
```

Because the deployment rollout replaces the pod, refresh `POD` before inspecting the replacement:

```bash
POD=$(kubectl get pods -n checkout-api -l app=checkout-api \
  -o jsonpath='{.items[0].metadata.name}')
kubectl get pod "$POD" -n checkout-api \
  -o jsonpath='{range .status.containerStatuses[*]}{.name}{" ready="}{.ready}{" restartCount="}{.restartCount}{"\n"}{end}'
```

Confirm that the pod is `Ready`, the restart count is no longer increasing, and requests are succeeding. For this service, the container health endpoint is `/health` on port `8000`; test it through the service or an appropriate local port-forward if needed:

```bash
kubectl port-forward -n checkout-api pod/"$POD" 8000:8000
curl --fail http://127.0.0.1:8000/health
```

### 4. Close out the incident

Record the affected pod, the observed `OOMKilled` evidence, the resource value that caused the incident, the applied fix, and the final restart and readiness status. Follow up by validating resource-limit changes against observed memory usage before deployment.
