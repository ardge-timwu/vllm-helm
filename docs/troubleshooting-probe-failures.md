# Troubleshooting Kubernetes Probe Failures

This guide explains how to identify if a pod was killed by a failed health probe and how to diagnose probe-related issues.

## Quick Answer

**Yes**, you can use `kubectl get events` to catch probe failures for deleted pods, **but events are only retained for ~1 hour** by default.

```bash
kubectl get events -n <namespace> | grep -E "(Unhealthy|probe|Killing)"
```

## Ways to Identify Probe Failures

### 1. Pod Events (Most Reliable)

```bash
kubectl describe pod <pod-name>
```

Look for events like:

**Liveness probe failure:**
```
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Warning  Unhealthy  2m    kubelet            Liveness probe failed: Get "http://10.1.2.3:8000/health": dial tcp 10.1.2.3:8000: connect: connection refused
  Warning  Unhealthy  1m    kubelet            Liveness probe failed: HTTP probe failed with statuscode: 503
  Normal   Killing    1m    kubelet            Container vllm failed liveness probe, will be restarted
```

**Startup probe failure:**
```
Events:
  Warning  Unhealthy  10m   kubelet            Startup probe failed: Get "http://10.1.2.3:8000/health": context deadline exceeded
  Normal   Killing    10m   kubelet            Container vllm failed startup probe, will be restarted
```

**Readiness probe failure** (does NOT kill, just removes from service):
```
Events:
  Warning  Unhealthy  5m    kubelet            Readiness probe failed: HTTP probe failed with statuscode: 503
```

### 2. Pod Status and Restart Count

```bash
kubectl get pod <pod-name>
```

```
NAME                    READY   STATUS    RESTARTS   AGE
vllm-7d8f9c5b6d-xyz12   0/1     Running   3          10m
                                          ↑
                                   Restarts from probe failures
```

Check restart reason:
```bash
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
```

Output might be:
```
Error
```

And the exit code:
```bash
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
```

For liveness probe kills, typically:
```
137  (SIGKILL from kubelet)
```

### 3. Container Logs

```bash
kubectl logs <pod-name> --previous
```

The logs themselves won't show "killed by probe", but you can see if the application was healthy before termination. Look for the last entries before restart.

### 4. Pod Conditions

```bash
kubectl get pod <pod-name> -o yaml | grep -A 10 "conditions:"
```

```yaml
conditions:
- lastProbeTime: null
  lastTransitionTime: "2025-01-08T10:30:00Z"
  status: "False"
  type: Ready              # Pod not ready (readiness probe failed)
- lastProbeTime: null
  lastTransitionTime: "2025-01-08T10:35:00Z"
  status: "True"
  type: ContainersReady
```

### 5. Real-time Event Monitoring

Watch events in real-time:
```bash
kubectl get events --watch --field-selector involvedObject.name=<pod-name>
```

Or for all pods:
```bash
kubectl get events --watch | grep -E "(Unhealthy|Killing|probe)"
```

## Event Retention and Limitations

### Important: Events are Ephemeral

**Events are typically retained for only 1 hour** (default `--event-ttl` on kube-apiserver)

```bash
# Events from pods deleted 30 minutes ago - still visible ✓
kubectl get events -n <namespace> --field-selector involvedObject.name=deleted-pod-name

# Events from pods deleted 2 hours ago - gone ✗
kubectl get events -n <namespace> --field-selector involvedObject.name=old-deleted-pod-name
# (returns nothing)
```

### Checking Events for Deleted Pods

```bash
# Show all warning events (includes probe failures)
kubectl get events -n <namespace> --field-selector type=Warning --sort-by='.lastTimestamp'

# Filter for probe-related events
kubectl get events -n <namespace> | grep -E "(Unhealthy|probe|Killing)"

# Watch events in real-time
kubectl get events -n <namespace> --watch | grep -E "(Unhealthy|probe)"
```

## Quick Diagnostic Script

Save this as `check-probe-failures.sh`:

```bash
#!/bin/bash

if [ -z "$1" ]; then
  echo "Usage: $0 <pod-name> [namespace]"
  exit 1
fi

POD_NAME="$1"
NAMESPACE="${2:-default}"

echo "=== Pod Status ==="
kubectl get pod $POD_NAME -n $NAMESPACE

echo -e "\n=== Restart Count and Reason ==="
kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.status.containerStatuses[0].restartCount}{"\n"}'
kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}{"\n"}'

echo -e "\n=== Recent Events ==="
kubectl describe pod $POD_NAME -n $NAMESPACE | grep -A 10 "Events:"

echo -e "\n=== Probe Failure Events ==="
kubectl get events -n $NAMESPACE --field-selector involvedObject.name=$POD_NAME | grep -E "(Unhealthy|probe|Killing)"

echo -e "\n=== Previous Container Logs (last 20 lines) ==="
kubectl logs $POD_NAME -n $NAMESPACE --previous --tail=20 2>&1
```

Usage:
```bash
chmod +x check-probe-failures.sh
./check-probe-failures.sh my-vllm-pod my-namespace
```

## Namespace-wide Probe Failure Summary

Find all probe failures in a namespace:

```bash
#!/bin/bash
NAMESPACE="${1:-default}"

echo "=== All Warning Events (last hour) ==="
kubectl get events -n $NAMESPACE --field-selector type=Warning --sort-by='.lastTimestamp'

echo -e "\n=== Probe Failures ==="
kubectl get events -n $NAMESPACE --field-selector type=Warning | grep -E "(Unhealthy|probe|Killing)"

echo -e "\n=== Pod Restart Summary ==="
kubectl get pods -n $NAMESPACE -o custom-columns=\
NAME:.metadata.name,\
RESTARTS:.status.containerStatuses[0].restartCount,\
LAST_RESTART:.status.containerStatuses[0].lastState.terminated.finishedAt
```

## Probe Failure Types Summary

| Probe Type | Kills Pod? | Event Message | Status Change |
|------------|-----------|---------------|---------------|
| **startupProbe** | ✅ Yes | `failed startup probe, will be restarted` | RESTARTS++ |
| **livenessProbe** | ✅ Yes | `failed liveness probe, will be restarted` | RESTARTS++ |
| **readinessProbe** | ❌ No | `Readiness probe failed` | READY: 0/1 |

**Key indicator**: Look for `"Killing"` events with `"failed liveness probe"` or `"failed startup probe"` in the message.

## Long-term Monitoring Solutions

For production environments, events should be exported to persistent storage since they only last ~1 hour.

### Option 1: kubernetes-event-exporter

Deploy an event exporter to send events to external storage:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: event-exporter-config
  namespace: monitoring
data:
  config.yaml: |
    logLevel: info
    logFormat: json
    receivers:
    - name: "elasticsearch"
      elasticsearch:
        hosts:
        - "http://elasticsearch:9200"
        index: "kubernetes-events"

    route:
      routes:
      - match:
        - receiver: "elasticsearch"
          labels:
            reason: "Unhealthy"  # Catches all probe failures
      - match:
        - receiver: "elasticsearch"
          labels:
            reason: "Killing"
```

### Option 2: Prometheus + AlertManager

Set up alerts for probe failures:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: vllm-probe-alerts
spec:
  groups:
  - name: probe_failures
    interval: 30s
    rules:
    - alert: HighProbeFaiureRate
      expr: |
        rate(kube_pod_container_status_restarts_total{namespace="vllm"}[5m]) > 0.1
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High pod restart rate in {{ $labels.namespace }}"
        description: "Pod {{ $labels.pod }} is restarting frequently (likely probe failures)"
```

### Option 3: Fluentd/Fluent Bit

Configure log forwarding for events:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [INPUT]
        Name              kubernetes_events
        Tag               kube.events
        Kube_URL          https://kubernetes.default.svc:443

    [FILTER]
        Name    grep
        Match   kube.events
        Regex   reason (Unhealthy|Killing|BackOff)

    [OUTPUT]
        Name  es
        Match kube.events
        Host  elasticsearch
        Port  9200
        Index kubernetes-events
```

## Common Probe Failure Patterns

### Pattern 1: Model Loading Timeout (Startup Probe)

```
Warning  Unhealthy  10m  kubelet  Startup probe failed: Get "http://10.1.2.3:8000/health": dial tcp: i/o timeout
Normal   Killing    10m  kubelet  Container vllm failed startup probe, will be restarted
```

**Solution**: Increase `slowStart.maxStartupSeconds`:
```yaml
slowStart:
  enabled: true
  maxStartupSeconds: 1800  # 30 minutes for large models
```

### Pattern 2: CUDA OOM Crash (Liveness Probe)

```
Warning  Unhealthy  2m  kubelet  Liveness probe failed: Get "http://10.1.2.3:8000/health": EOF
Normal   Killing    2m  kubelet  Container vllm failed liveness probe, will be restarted
```

**Solution**: Increase memory limits or reduce model size:
```yaml
resources:
  limits:
    memory: 32Gi  # Increase based on model requirements
```

### Pattern 3: Service Degradation (Readiness Probe)

```
Warning  Unhealthy  5m  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 503
```

**Note**: This does NOT restart the pod, only removes it from service endpoints.

**Solution**: Investigate application logs for performance issues.

## Best Practices

1. **Monitor events in real-time** during deployments:
   ```bash
   kubectl get events -n <namespace> --watch
   ```

2. **Set up persistent event storage** for production clusters (event exporters)

3. **Configure alerts** for high restart rates (indicates probe failures)

4. **Adjust probe settings** based on your model's actual startup time

5. **Use `slowStart.maxStartupSeconds`** instead of disabling probes

6. **Check events regularly** within the 1-hour retention window

## Related Documentation

- [Health Probes Configuration](../README.md#health-probes-and-startup-time)
- [Resource Management](../README.md#resource-management)
- [slowStart Configuration](../README.md#simple-configuration-with-slowstart-recommended)
