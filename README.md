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

### How It Works

When `init_drop_cache: true`, the cache drop command runs **before every vLLM start**, including after container restarts. This is important for:
- **CUDA OOM recovery**: If vLLM crashes due to CUDA out of memory, the cache will be dropped before restart
- **Consistent benchmarking**: Every startup gets a clean cache state
- **No init container limitations**: Unlike init containers which only run once, this runs on every container start

### Security Context Setup

The cache drop operation requires:
- `privileged: true` - to write to `/proc/sys/vm/drop_caches`
- `hostPID: true` - to access host process namespace

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
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  volumes:
  - '*'
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
4. You understand the security implications of hostPID access

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

## Storage Configuration

### Using an Existing PVC

If you already have a PersistentVolumeClaim (PVC) with your models, you can use it directly instead of creating a new one:

```bash
helm install my-vllm ardge-timwu/vllm-helm \
  --set persistence.enabled=true \
  --set persistence.existingClaim=my-existing-pvc
```

Or in your values.yaml:

```yaml
persistence:
  enabled: true
  existingClaim: "my-existing-pvc"
```

This is useful when:
- Sharing models across multiple vLLM deployments
- Using pre-populated model caches
- Managing PVCs independently of Helm releases

### Using HostPath Volumes

By default, if no `existingClaim` is specified, the chart creates a PersistentVolumeClaim (PVC) that expects dynamic storage provisioning. However, you can also use a HostPath volume to mount a directory from the host node. This is useful for:

- Development and testing environments
- Pre-loading models directly on the node filesystem
- Accessing existing model files on worker nodes
- Avoiding network storage overhead

#### Step 1: Create a HostPath PersistentVolume

First, create a PersistentVolume (PV) that uses HostPath storage. Save this as `hostpath-pv.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: vllm-models-pv
  labels:
    type: local
    app: vllm
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: vllm-hostpath
  hostPath:
    path: /data/vllm-models  # Change this to your desired path
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - your-node-name  # Change this to your actual node name
```

Apply the PersistentVolume:

```bash
kubectl apply -f hostpath-pv.yaml
```

#### Step 2: Configure values.yaml

Update your `values.yaml` (or use `--set` flags) to use the HostPath storage class:

```yaml
persistence:
  enabled: true
  storageClass: "vllm-hostpath"  # Match the storageClassName from PV
  accessMode: ReadWriteOnce
  size: 50Gi
  annotations: {}

# If using HostPath, you'll want to schedule the pod on the same node
nodeSelector:
  kubernetes.io/hostname: your-node-name  # Must match the node in PV
```

#### Step 3: Install the Chart

```bash
helm install my-vllm ardge-timwu/vllm-helm -f values.yaml
```

Or using command-line flags:

```bash
helm install my-vllm ardge-timwu/vllm-helm \
  --set persistence.enabled=true \
  --set persistence.storageClass=vllm-hostpath \
  --set persistence.size=50Gi \
  --set nodeSelector."kubernetes\.io/hostname"=your-node-name
```

#### Important Considerations

1. **Node Affinity**: HostPath volumes are node-specific. The pod must be scheduled on the same node where the path exists. Use `nodeSelector` or `nodeAffinity` to ensure this.

2. **Directory Permissions**: Ensure the directory on the host has appropriate permissions. The vLLM container typically runs as root, so the directory should be readable/writable:
   ```bash
   sudo mkdir -p /data/vllm-models
   sudo chmod 755 /data/vllm-models
   ```

3. **Pre-loading Models**: If you want to pre-load models into the HostPath directory:
   ```bash
   # Models are cached in /root/.cache inside the container
   # which maps to the HostPath directory
   sudo mkdir -p /data/vllm-models/huggingface/hub
   # Place your model files in this directory
   ```

4. **Multi-node Clusters**: HostPath volumes are not suitable for multi-replica deployments across different nodes. For multi-node deployments, consider using:
   - NFS or other network storage
   - Dynamic provisioners (AWS EBS, GCE PD, etc.)
   - Shared filesystem solutions (CephFS, GlusterFS)

5. **Reclaim Policy**: The example uses `Retain` policy, which means the data persists even after the PV is deleted. Change to `Delete` if you want automatic cleanup.

#### Alternative: Direct HostPath Mount (Without PV/PVC)

For simpler setups, you can modify the deployment to use HostPath directly without PV/PVC. Create a custom values file:

```yaml
# Disable the PVC
persistence:
  enabled: false

# Add this in your deployment by creating a custom template
# Note: This requires modifying the chart templates
```

However, using PV/PVC is the recommended Kubernetes approach as it provides better abstraction and lifecycle management.

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
| `resources` | Pod resource requests and limits | `{}` (not set) |
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.existingClaim` | Use existing PVC instead of creating new one | `""` (create new) |
| `persistence.size` | Storage size (when creating new PVC) | `50Gi` |
| `persistence.storageClass` | Storage class name (when creating new PVC) | `""` |
| `slowStart.enabled` | Simple health probe configuration | `true` |
| `slowStart.maxStartupSeconds` | Max time for container to start serving | `600` (10 min) |
| `startupProbe.*` | Advanced manual probe config (if slowStart disabled) | See values.yaml |
| `livenessProbe.*` | Advanced manual probe config (if slowStart disabled) | See values.yaml |
| `readinessProbe.*` | Advanced manual probe config (if slowStart disabled) | See values.yaml |
| `init_drop_cache` | Enable drop-cache init container (requires privileged pod) | `false` |

## Health Probes and Startup Time

### Understanding Model Loading

When vLLM starts, it needs to:
1. Download the model from HuggingFace (if not cached)
2. Load the model into GPU memory
3. Compile CUDA graphs for optimized inference

This process can take several minutes, especially for large models.

### Simple Configuration with `slowStart` (Recommended)

**Just specify how long your model takes to start**, and all three probes will be configured automatically:

```bash
# For a model that takes up to 30 minutes to start
helm install my-vllm ardge-timwu/vllm-helm \
  --set slowStart.maxStartupSeconds=1800
```

Or in values.yaml:

```yaml
slowStart:
  enabled: true
  maxStartupSeconds: 1800  # 30 minutes
```

**Common configurations by model size:**

```bash
# Small models (< 7B parameters) - 5 minutes
helm install my-vllm ardge-timwu/vllm-helm \
  --set slowStart.maxStartupSeconds=300

# Medium models (7B-13B) - 10 minutes (default)
helm install my-vllm ardge-timwu/vllm-helm \
  --set slowStart.maxStartupSeconds=600

# Large models (13B-70B) - 30 minutes
helm install my-vllm ardge-timwu/vllm-helm \
  --set slowStart.maxStartupSeconds=1800

# Huge models (70B+) - 60 minutes
helm install my-vllm ardge-timwu/vllm-helm \
  --set slowStart.maxStartupSeconds=3600
```

**What `slowStart` does automatically:**
- Configures `startupProbe` to allow the specified startup time
- Sets up `livenessProbe` for crash detection (starts after startup succeeds)
- Sets up `readinessProbe` to control traffic routing (starts after startup succeeds)
- All probes check `/health` endpoint
- No need to calculate `failureThreshold` or other probe parameters

### Advanced: Manual Probe Configuration

If you need fine-grained control, disable `slowStart` and configure probes manually:

```yaml
slowStart:
  enabled: false

startupProbe:
  enabled: true
  failureThreshold: 120  # Custom calculation
  periodSeconds: 10

livenessProbe:
  enabled: true
  periodSeconds: 30

readinessProbe:
  enabled: true
  periodSeconds: 10
```

### ⚠️ Don't Disable Probes (Anti-Pattern)

**Bad practice** (found in some deployments):
```yaml
# DON'T DO THIS - disables health monitoring entirely
livenessProbe:
  enabled: false
readinessProbe:
  enabled: false
```

**Why this is bad:**
- No health monitoring in production
- Container won't be restarted if vLLM crashes
- Service continues routing traffic to failed pods
- Can't detect when pod is ready to serve traffic

**Instead, use startupProbe** as shown above, or increase `failureThreshold`:

```yaml
# BETTER - keeps monitoring, extends startup time
startupProbe:
  enabled: true
  failureThreshold: 180  # 30 minutes for extremely large models
```

### Legacy Approach (Without Startup Probe)

If you can't use startup probes (Kubernetes < 1.16), increase `initialDelaySeconds`:

```yaml
livenessProbe:
  enabled: true
  initialDelaySeconds: 600  # 10 minutes
readinessProbe:
  enabled: true
  initialDelaySeconds: 600
```

To disable startup probe and use legacy timing:

```bash
helm install my-vllm ardge-timwu/vllm-helm \
  --set startupProbe.enabled=false
```

## Resource Management

### Memory Requirements

By default, this chart does **not** set resource limits or requests. This gives Kubernetes maximum flexibility but means:
- Pods can use all available node memory
- Risk of OOMKilled if the model is too large for available memory
- No memory-based scheduling guarantees

**Important**: vLLM model memory requirements vary greatly:
- Small models (< 1B parameters): 4-8 GB
- Medium models (7B parameters): 16-32 GB
- Large models (13B+ parameters): 32-80+ GB

### Setting Resource Limits

To prevent OOMKilled errors and ensure proper scheduling, set memory limits based on your model:

```bash
# For a 7B parameter model
helm install my-vllm ardge-timwu/vllm-helm \
  --set resources.limits.memory=24Gi \
  --set resources.requests.memory=16Gi \
  --set resources.limits.nvidia.com/gpu=1
```

Or in values.yaml:

```yaml
resources:
  limits:
    cpu: 4000m
    memory: 24Gi
    nvidia.com/gpu: "1"
  requests:
    cpu: 2000m
    memory: 16Gi
    nvidia.com/gpu: "1"
```

### Calculating Memory Requirements

A rough formula for estimating memory needs:
```
Memory (GB) = (Model Parameters × Bytes per Parameter × 1.2) / 1,000,000,000

For FP16: Bytes per Parameter = 2
For FP32: Bytes per Parameter = 4
The 1.2 factor accounts for overhead
```

Example for a 7B FP16 model:
```
(7,000,000,000 × 2 × 1.2) / 1,000,000,000 = ~17 GB minimum
Recommend setting limit to 24-32 GB for safety
```

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

### With Resource Limits

```bash
# Recommended for production to prevent OOMKilled
helm install vllm ardge-timwu/vllm-helm \
  --set vllm.model="meta-llama/Llama-2-7b-hf" \
  --set resources.limits.memory=24Gi \
  --set resources.requests.memory=16Gi \
  --set resources.limits.nvidia.com/gpu=1
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

### With Existing PVC

```bash
helm install vllm ardge-timwu/vllm-helm \
  --set persistence.existingClaim=my-models-pvc
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
- Enable `hostPID` for the pod
- Set the container as privileged
- Run `sync; echo 3 > /proc/sys/vm/drop_caches` before every vLLM start (including restarts)
- Clear system caches before vLLM startup (useful for performance benchmarking and CUDA OOM recovery)

**Benefits over init containers**:
- Cache drop runs on every container restart, not just initial pod creation
- Helps with CUDA OOM recovery by clearing caches before restart attempts
- Ensures consistent cache state for benchmarking across restarts

**Note**: Multiple releases can run simultaneously on the same node since this does not use `hostNetwork`.

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
