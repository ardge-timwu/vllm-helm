# vLLM Helm Chart

This Helm chart deploys vLLM (Very Large Language Model) inference server on Kubernetes.

## Introduction

vLLM is a high-throughput and memory-efficient inference and serving engine for Large Language Models (LLMs). This chart provides an easy way to deploy vLLM as a Kubernetes service for scalable AI inference.

> **Attribution**: This Helm chart is based on the [vLLM project](https://github.com/vllm-project/vllm) by the vLLM team. We extend our gratitude to the vLLM project contributors for their excellent work on the core inference engine.

> **v0.11.0 Highlights**: This release features major improvements including V1 engine as the only engine, FULL_AND_PIECEWISE as the default CUDA graph mode for better performance, and support for new model architectures like DeepSeek-V3.2-Exp, Qwen3-VL series, and more.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+
- GPU-enabled nodes with NVIDIA drivers and CUDA support
- NVIDIA device plugin installed in your cluster

## Privileged Pods Requirements

When enabling the `init_drop_cache` feature, the pod requires privileged access to drop system caches. This requires additional cluster-side configuration:

### Security Context Setup

The drop-cache init container requires:
- `privileged: true` - to write to `/proc/sys/vm/drop_caches`
- `hostPID: true` - to access host process namespace
- `hostNetwork: true` - to access host network namespace

### Kubernetes 1.25+ (Pod Security Standards)

If your cluster uses Pod Security Standards (PSS), you need to configure the namespace to allow privileged pods:

```bash
# Label the namespace to use privileged Pod Security Standard
kubectl label namespace <your-namespace> \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged
```

Or configure namespace-specific exceptions:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: vllm-namespace
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

### Kubernetes 1.21-1.24 (PodSecurityPolicy - Deprecated)

If your cluster still uses PodSecurityPolicy, create a policy allowing privileged pods:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: vllm-privileged
spec:
  privileged: true
  hostPID: true
  hostNetwork: true
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  volumes:
  - '*'
  hostPorts:
  - min: 0
    max: 65535
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: vllm-privileged-psp
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames: ['vllm-privileged']
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vllm-privileged-psp
roleRef:
  kind: ClusterRole
  name: vllm-privileged-psp
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: <vllm-service-account-name>
  namespace: <your-namespace>
```

### Security Considerations

**Warning**: Enabling `init_drop_cache: true` grants the pod significant privileges that could be a security risk. Only enable this feature when:

1. You need to drop system caches for performance benchmarking or testing
2. You trust all users who can deploy to the namespace
3. Your cluster security policy allows privileged pods
4. You understand the security implications of hostPID and hostNetwork access

**Recommendation**: Use this feature only in dedicated namespaces for performance testing, not in production environments with untrusted workloads.

## Installing the Chart

To install the chart with the release name `my-vllm`:

```bash
helm install my-vllm ardge-timwu/vllm-helm
```

The command deploys vLLM on the Kubernetes cluster in the default configuration. The [Parameters](#parameters) section lists the parameters that can be configured during installation.

## Uninstalling the Chart

To uninstall/delete the `my-vllm` deployment:

```bash
helm delete my-vllm
```

## Configuration

The following table lists the configurable parameters of the vLLM chart and their default values.

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | vLLM image repository | `vllm/vllm-openai` |
| `image.tag` | vLLM image tag | `latest` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `service.type` | Kubernetes service type | `ClusterIP` |
| `service.port` | Service port | `8000` |
| `vllm.model` | Model to load | `microsoft/DialoGPT-medium` |
| `vllm.host` | Host to bind to | `0.0.0.0` |
| `vllm.port` | Port to bind to | `8000` |
| `vllm.gpuMemoryUtilization` | GPU memory utilization | `0.9` |
| `vllm.maxModelLen` | Maximum model length | `2048` |
| `vllm.cudaGraphMode` | CUDA graph mode (v0.11.0+) | `FULL_AND_PIECEWISE` |
| `vllm.asyncScheduling` | Async scheduling (disabled in v0.11.0) | `false` |
| `resources.limits.nvidia.com/gpu` | GPU resource limit | `1` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.size` | Storage size | `50Gi` |
| `init_drop_cache` | Enable drop-cache init container (requires privileged pod) | `false` |

## Examples

### Basic Installation

```bash
helm install vllm ardge-timwu/vllm-helm
```

### Custom Model Configuration

```bash
helm install vllm ardge-timwu/vllm-helm \
  --set vllm.model="microsoft/DialoGPT-large" \
  --set vllm.maxModelLen=4096
```

### GPU Configuration

```bash
helm install vllm ardge-timwu/vllm-helm \
  --set resources.limits.nvidia.com/gpu=2 \
  --set vllm.gpuMemoryUtilization=0.8
```

### v0.11.0 Performance Optimization

```bash
helm install vllm ardge-timwu/vllm-helm \
  --set vllm.cudaGraphMode=FULL_AND_PIECEWISE \
  --set vllm.asyncScheduling=false
```

### With Ingress

```bash
helm install vllm ardge-timwu/vllm-helm \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=vllm.example.com
```

### With Cache Dropping (Privileged Mode)

**Prerequisites**: Ensure your namespace is configured for privileged pods (see [Privileged Pods Requirements](#privileged-pods-requirements))

```bash
# First, configure the namespace for privileged pods (K8s 1.25+)
kubectl label namespace <your-namespace> \
  pod-security.kubernetes.io/enforce=privileged

# Then install with cache dropping enabled
helm install vllm ardge-timwu/vllm-helm \
  --set init_drop_cache=true
```

This configuration will:
- Enable `hostPID` and `hostNetwork` for the pod
- Add an init container that runs `sync; echo 3 > /proc/sys/vm/drop_caches`
- Clear system caches before starting vLLM (useful for performance benchmarking)

## API Usage

Once deployed, vLLM provides a REST API compatible with OpenAI's API:

```bash
# Health check
curl http://your-service-url/health

# Text completion
curl -X POST "http://your-service-url/v1/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "microsoft/DialoGPT-medium",
    "prompt": "Hello, world!",
    "max_tokens": 50
  }'
```

## Troubleshooting

### GPU Issues

If you encounter GPU-related issues:

1. Verify GPU nodes are available:
   ```bash
   kubectl get nodes -l accelerator=nvidia-tesla-k80
   ```

2. Check if the NVIDIA device plugin is running:
   ```bash
   kubectl get pods -n kube-system | grep nvidia
   ```

3. Verify GPU resources in the pod:
   ```bash
   kubectl describe pod <pod-name>
   ```

### Memory Issues

If you encounter out-of-memory errors:

1. Increase the GPU memory utilization:
   ```bash
   helm upgrade vllm ./charts/vllm --set vllm.gpuMemoryUtilization=0.7
   ```

2. Reduce the maximum model length:
   ```bash
   helm upgrade vllm ./charts/vllm --set vllm.maxModelLen=1024
   ```

## Attribution and Credits

This Helm chart is built upon the excellent work of the [vLLM project](https://github.com/vllm-project/vllm) team. We acknowledge and thank the vLLM contributors for their groundbreaking work in high-performance LLM inference.

- **vLLM Project**: https://github.com/vllm-project/vllm
- **vLLM Documentation**: https://docs.vllm.ai/
- **Original vLLM Helm Chart**: https://github.com/vllm-project/vllm/tree/main/examples/online_serving/chart-helm

## License

This chart is licensed under the Apache 2.0 License, following the same license as the original vLLM project.
