# Ray DeepSeek vLLM Deployment Architecture

```
                                                                  +-------------------+
                                                                  |                   |
                                                                  |  ConfigMap:       |
                                                                  |  vllm-serve-script|
                                                                  |                   |
                                                                  +--------+----------+
                                                                           |
                                                                           | mounts
                                                                           v
+----------------------+    manages    +----------------------+    uses    +----------------------+
|                      |-------------->|                      |---------->|                      |
|  RayService: vllm    |               |  RayCluster:         |           |  Secret: hf-token    |
|                      |               |  vllm-raycluster     |           |                      |
+----------+-----------+               +----------+-----------+           +----------------------+
           |                                      |
           | creates                              | creates
           |                                      |
           v                                      v
+----------+-----------+               +----------+--------------------+
|                      |               |                               |
|  Service: vllm       |               |  Pod: vllm-raycluster-head    |
|  Service: vllm-head  |<--------------|  (2/2 containers)             |
|  Service: vllm-serve |               |                               |
|                      |               +-------------------------------+
+----------------------+                           |
                                                   | manages
                                                   |
                                                   v
                                      +------------+--------------------+
                                      |                                |
                                      |  Pod: vllm-raycluster-worker   |
                                      |  (GPU worker, 1/1 containers)  |
                                      |                                |
                                      +--------------------------------+
```

## Components Overview

### Core Resources

1. **RayService: vllm**
   - Top-level resource that manages the entire Ray deployment
   - Defines the Serve application configuration
   - Configures the Ray cluster specifications

2. **RayCluster: vllm-raycluster-7scgn**
   - Managed by the RayService
   - Consists of head and worker nodes
   - Status: Ready with 1 available worker

3. **Pods**:
   - **Head Node**: `vllm-raycluster-7scgn-head-9ksmx` (2/2 containers running)
     - Runs Ray head services
     - Hosts the Ray dashboard and Serve controller
   - **Worker Node**: `vllm-raycluster-7scgn-worker-gpu-group-5dpxv` (1/1 containers running)
     - GPU-equipped worker (1 NVIDIA GPU)
     - Runs the actual DeepSeek model inference

### Supporting Resources

4. **Services**:
   - **vllm**: Main service exposing multiple ports (8000, 8080, 6379, 8265, 10001)
   - **vllm-head-svc**: Service for the Ray head node
   - **vllm-serve-svc**: Service specifically for Ray Serve (port 8000)

5. **ConfigMap: vllm-serve-script**
   - Contains the Python code for the vLLM Serve deployment
   - Implements the FastAPI endpoints for model inference
   - Mounted into both head and worker pods

6. **Secret: hf-token**
   - Contains the Hugging Face authentication token
   - Used to download the DeepSeek model

## Application Details

### Model Information
- **Model**: DeepSeek-R1-Distill-Llama-8B
- **Framework**: vLLM 0.7.0
- **API Compatibility**: OpenAI-compatible API

### Endpoints
- **/v1/models**: Lists available models
- **/v1/chat/completions**: Chat completion endpoint (compatible with OpenAI format)

### Deployment Configuration
- **Autoscaling**: Configured with min=1, max=4 replicas
- **Resources per worker**: 12 CPUs, 60GB memory, 1 NVIDIA GPU
- **Head node resources**: 2 CPUs, 12GB memory

### Node Selection
- **Head node**: Runs on CPU nodes with label `NodeGroupType: x86-cpu-karpenter`
- **Worker nodes**: Runs on GPU nodes with label `NodeGroupType: g5-gpu-karpenter`
