# Dify on AWS Graviton Demo

This repository contains demo materials for running Dify (an LLM application development platform) on AWS Graviton processors, showcasing performance and cost efficiency improvements.

## Architecture Overview

This demo deploys a complete AI application stack on Amazon EKS with multiple node groups optimized for different workloads:

- **Graviton Node Group**: Hosts the Dify application for cost-efficient API serving
- **GPU Node Group (G5)**: Runs GPU-intensive workloads like Stable Diffusion and DeepSeek LLM
- **x86 Node Group**: Handles general-purpose workloads

![Architecture Diagram](./Architectures-drawio/dify-kubernetes-architecture.drawio)

## Prerequisites

- AWS CLI configured with appropriate permissions
- kubectl installed
- Helm v3 installed
- eksctl installed
- Docker installed (for building custom images)

## Deployment Steps

### 1. Create EKS Cluster

```bash
# Create EKS cluster with eksctl
eksctl create cluster \
  --name dify-graviton-demo \
  --region us-west-2 \
  --version 1.28 \
  --without-nodegroup
```

### 2. Install Karpenter for Node Provisioning

Karpenter will dynamically provision the right nodes based on workload requirements:

```bash
# Apply Karpenter installation
kubectl apply -f karpenter/karpenter.yaml

# Create GP3 storage class for persistent volumes
kubectl apply -f karpenter/gp3-storage-class.yaml
```

### 3. Create Node Groups with Karpenter

```bash
# Create Graviton node group for Dify
kubectl apply -f karpenter/gvn-nodepool.yaml

# Create x86 node group for general workloads
kubectl apply -f karpenter/x86-node-pool.yaml

# Create GPU node group for AI workloads
kubectl apply -f karpenter/g5-gpu-karpenter.yaml
```

### 4. Deploy Dify on Graviton Nodes

Dify is deployed using Helm charts with specific configurations for Graviton:

```bash
# Navigate to the Dify chart directory
cd dify-chart

# Install Dify with Graviton-optimized values
helm install dify . \
  --namespace dify \
  --create-namespace \
  --values values.yaml \
  --set nodeSelector.kubernetes\\.io/arch=arm64
```

The Dify deployment includes:
- Web UI
- API server
- Worker processes
- Milvus vector database (deployed with optimized settings for Graviton)

### 5. Deploy Stable Diffusion on GPU Nodes

Stable Diffusion is deployed as a Kubernetes deployment targeting GPU nodes:

```bash
# Deploy Stable Diffusion
kubectl apply -f deploy-Stable-Diffusion/deployment.yaml
```

This deployment:
- Uses NVIDIA GPUs for inference
- Exposes an API endpoint for image generation
- Integrates with Dify for multimodal capabilities

### 6. Deploy DeepSeek LLM with KubeRay on GPU Nodes

DeepSeek is deployed using KubeRay for efficient GPU utilization:

```bash
# Install KubeRay operator
kubectl create namespace ray-system
kubectl apply -k "github.com/ray-project/kuberay/ray-operator/config/default?ref=v1.0.0"

# Deploy DeepSeek with vLLM on Ray
kubectl apply -f vllm-ray-gpu-deepseek/ray-vllm-deepseek.yml

# Deploy Open WebUI for interacting with DeepSeek
kubectl apply -f vllm-ray-gpu-deepseek/open-webui.yaml
```

### 7. Verify Deployments

```bash
# Check node groups
kubectl get nodes -L kubernetes.io/arch,node.kubernetes.io/instance-type

# Verify Dify deployment
kubectl get pods -n dify -o wide

# Check GPU workloads
kubectl get pods -l app=stable-diffusion -o wide
kubectl get rayclusters -n ray-system
```

## Component Details

### Dify Platform

Dify is deployed on Graviton nodes for cost-efficient API serving and application hosting. The deployment includes:

- Frontend UI for building AI applications
- Backend API for handling requests
- Vector database (Milvus) for knowledge retrieval
- Worker processes for background tasks

### Stable Diffusion

Deployed on GPU nodes (G5), this component provides:
- Image generation capabilities
- API endpoint for integration with Dify
- Efficient GPU utilization with optimized container settings

### DeepSeek LLM

Deployed using KubeRay on GPU nodes (G5), this component provides:
- High-performance language model inference
- Efficient GPU memory management with vLLM
- Scalable architecture with Ray clusters
- Web UI for direct interaction

## Performance Comparison

The demo showcases performance and cost benefits of using Graviton processors for the Dify platform:
- Lower cost per request compared to x86 instances
- Comparable or better latency for API serving workloads
- Efficient resource utilization with workload-specific node groups

## Cleanup

To delete all resources created by this demo:

```bash
# Delete applications
helm uninstall dify -n dify
kubectl delete -f deploy-Stable-Diffusion/deployment.yaml
kubectl delete -f vllm-ray-gpu-deepseek/ray-vllm-deepseek.yml
kubectl delete -f vllm-ray-gpu-deepseek/open-webui.yaml

# Delete node groups via Karpenter
kubectl delete -f karpenter/gvn-nodepool.yaml
kubectl delete -f karpenter/x86-node-pool.yaml
kubectl delete -f karpenter/g5-gpu-karpenter.yaml

# Delete Karpenter
kubectl delete -f karpenter/karpenter.yaml

# Delete EKS cluster
eksctl delete cluster --name dify-graviton-demo --region us-west-2
```

## Additional Resources

- [Dify Documentation](https://docs.dify.ai/)
- [AWS Graviton Processors](https://aws.amazon.com/ec2/graviton/)
- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [Karpenter Documentation](https://karpenter.sh/docs/)
