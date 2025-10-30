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

## Installing the Chart

To install the chart with the release name `my-vllm`:

```bash
helm install my-vllm arktec-quant-charts/vllm
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

## Examples

### Basic Installation

```bash
helm install vllm arktec-quant-charts/vllm
```

### Custom Model Configuration

```bash
helm install vllm arktec-quant-charts/vllm \
  --set vllm.model="microsoft/DialoGPT-large" \
  --set vllm.maxModelLen=4096
```

### GPU Configuration

```bash
helm install vllm arktec-quant-charts/vllm \
  --set resources.limits.nvidia.com/gpu=2 \
  --set vllm.gpuMemoryUtilization=0.8
```

### v0.11.0 Performance Optimization

```bash
helm install vllm arktec-quant-charts/vllm \
  --set vllm.cudaGraphMode=FULL_AND_PIECEWISE \
  --set vllm.asyncScheduling=false
```

### With Ingress

```bash
helm install vllm arktec-quant-charts/vllm \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=vllm.example.com
```

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
