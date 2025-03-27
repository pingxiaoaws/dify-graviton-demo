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


## Deployment Steps

### 1. Create EKS Cluster

```bash
# Create EKS cluster with eksctl
eksctl create cluster \
  --name dify-graviton-demo \
  --region us-west-2 \
  --version 1.28 \
  --without-nodegroup

# Create storage class
kubectl apply -f efssc.yaml
kubectl apply -f g3-storage-class.yaml
```
After you created cluster, please install Amazon EFS CSI Driver from the add-on catalog.


### 2. Install Karpenter for Node Provisioning

Karpenter will dynamically provision the right nodes based on workload requirements:

```bash
# Apply Karpenter installation
kubectl apply -f karpenter/karpenter.yaml

# Create GP3 storage class for persistent volumes
kubectl apply -f karpenter/gp3-storage-class.yaml

# Create Graviton node group for Dify
kubectl apply -f karpenter/gvn-nodepool.yaml 

# Create x86 node group for general workloads
kubectl apply -f karpenter/x86-node-pool.yaml

# Create GPU node group for AI workloads
kubectl apply -f karpenter/g5-gpu-karpenter.yaml

#EKS Auto-mode Create Graviton node group for Dify
kubectl apply -f karpenter/gvn-nodepool-automode.yaml 

#EKS Auto-mode Create x86 node group for general workloads
kubectl apply -f karpenter/x86-node-pool-automode.yaml

#EKS Auto-mode Create GPU node group for AI workloads
kubectl apply -f karpenter/g5-gpu-karpenter-automode.yaml


```

### 3. Install nvidia-device-plugin (skip this if EKS automode)
```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

helm install nvidia-device-plugin nvidia/nvidia-device-plugin --version 0.17.1 --values dify-graviton-demo/vllm-ray-gpu-deepseek/nvidia-device-plugin-values.yaml --namespace gpu-operator --create-namespace
```


### 4. Deploy Milvus on Graviton Nodes
```bash
kubectl create namespace milvus
helm install milvus-graviton -n milvus -f graviton-values-smallsize.yaml .
```

### 5. Deploy Dify on Graviton Nodes

Dify is deployed using Helm charts with specific configurations for Graviton:

```bash
# Navigate to the Dify chart directory
cd dify-chart

# Install Dify with Graviton-optimized values
helm install dify-graviton . \
  --namespace dify \
  --create-namespace \
  -f milvus-graviton-values.yaml
 # Sample output
NAME: dify-graviton
LAST DEPLOYED: Wed Mar 26 19:59:37 2025
NAMESPACE: dify
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace dify -l "app.kubernetes.io/name=dify,app.kubernetes.io/instance=dify-graviton,component=proxy" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace dify $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace dify port-forward $POD_NAME 8080:$CONTAINER_PORT 
```


The Dify deployment includes:
- Web UI
- API server
- Worker processes
- Milvus vector database (deployed with optimized settings for Graviton)

### 6. Deploy Stable Diffusion on GPU Nodes

Stable Diffusion is deployed as a Kubernetes deployment targeting GPU nodes:

```bash
# Deploy Stable Diffusion
kubectl apply -f deploy-Stable-Diffusion/deployment.yaml
```

This deployment:
- Uses NVIDIA GPUs for inference
- Exposes an API endpoint for image generation
- Integrates with Dify for multimodal capabilities


### 7. Deploy DeepSeek LLM with KubeRay on GPU Nodes
DeepSeek is deployed using KubeRay for efficient GPU utilization:

```bash
cat > kuberay-operator-values.yaml << 'EOF'
batchScheduler:
  enabled: true
EOF

# 添加 KubeRay Helm 仓库
helm repo add kuberay https://ray-project.github.io/kuberay-helm/

# 更新 Helm 仓库
helm repo update

# 创建 values 文件
cat > kuberay-operator-values.yaml << 'EOF'
batchScheduler:
  enabled: true
EOF

# 安装 KubeRay Operator
helm install kuberay-operator kuberay/kuberay-operator \
  --version 1.1.1 \
  --values kuberay-operator-values.yaml \
  --namespace kuberay-operator \
  --create-namespace

# Install nvidia-device-plugin （no need for EKS automode）
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

helm install nvidia-device-plugin nvidia/nvidia-device-plugin --version 0.17.1 --values dify-graviton-demo/vllm-ray-gpu-deepseek/nvidia-de


# Install KubeRay operator
helm repo add volcano https://volcano-sh.github.io/helm-charts
helm repo update
helm install volcano volcano/volcano --namespace volcano-system 
--create-namespace

helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm install kuberay-operator kuberay/kuberay-operator \
  --version 1.1.1 \
  --values kuberay-operator-values.yaml \
  --namespace kuberay-operator \
  --create-namespace

# Deploy DeepSeek with vLLM on Ray
export HUGGING_FACE_HUB_TOKEN=$(echo -n "Your-Hugging-Face-Hub-Token-Value" | base64)
envsubst < ray-vllm-deepseek.yml | kubectl apply -f -

# Deploy Open WebUI for interacting with DeepSeek
kubectl apply -f vllm-ray-gpu-deepseek/open-webui.yaml
```

### 8. Verify Deployments

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
