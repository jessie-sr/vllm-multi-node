# **Multi-Node vLLM + Ray Cluster Setup on Vast.ai**

This guide walks you through setting up a **distributed vLLM inference system** using a **Ray cluster** on **Vast.ai** across **multiple GPU-enabled nodes**.

## **1️⃣ Launch Instances on [Vast.ai](https://cloud.vast.ai/create/)**
- Deploy **2 VM instances** (**1 head, 1 worker**) with **at least 16GB GPU memory**.
- Choose the [**Ubuntu 22.04 CLI (VM)**](https://cloud.vast.ai/template/readme/9612e7b1ae3d729a9b1f5984b9f1972c) template.

---

## **2️⃣ Set Up Networking**
### **🔹 SSH into Both VMs**
```bash
ssh -p 18445 root@174.88.252.15 -L 8080:localhost:8080
```

### **🔹 Install [Tailscale](https://login.tailscale.com/admin/machines) for Private Networking**
```bash
sudo tailscale up
```
- This enables a **secure private connection** between nodes.
- Verify connectivity by running:
```bash
ping <tailscale_IP_of_other_node>
```

---

## **3️⃣ Check CUDA & Install Drivers**
### **🔹 Verify CUDA Installation**
```bash
nvcc --version
```
- If `nvcc` is missing, install CUDA:
```bash
sudo apt install -y nvidia-cuda-toolkit
```

---

## **4️⃣ Deploy Ray Cluster**
- Clone the repository to access `run_cluster.sh`:
```bash
git clone <your_repo_link>
cd <repo_directory>
```
- Use **Tailscale IPs** to launch Ray on both nodes.

### **🔹 Start Ray on the Head Node**
```bash
bash run_cluster.sh \
    vllm/vllm-openai \
    100.106.59.63 \  # Head node IP
    --head \
    /home/root/.cache/huggingface \
    -e VLLM_HOST_IP=100.106.59.63 \  # Head node IP
    -e NCCL_SOCKET_IFNAME=tailscale0 \
    -e GLOO_SOCKET_IFNAME=tailscale0
```

### **🔹 Start Ray on the Worker Node**
```bash
bash run_cluster.sh \
    vllm/vllm-openai \
    100.106.59.63 \  # Head node IP
    --worker \
    /home/root/.cache/huggingface \
    -e VLLM_HOST_IP=100.86.62.1 \  # This node's IP
    -e NCCL_SOCKET_IFNAME=tailscale0 \
    -e GLOO_SOCKET_IFNAME=tailscale0
```

---

## **5️⃣ Verify GPU & Enter Container**
- Open a new terminal and SSH into **any node**.
- Verify GPU status:
```bash
nvidia-smi
```
- Enter the **Docker container** running vLLM:
```bash
docker exec -it node /bin/bash
```

---

## **6️⃣ Run vLLM**
Run the following command on the **head node** to start distributed inference:
```bash
vllm serve Qwen/Qwen2.5-7B \
    --tensor-parallel-size 1 \  # Number of GPUs per node
    --pipeline-parallel-size 2  # Number of nodes
```
### **Optional Performance Optimizations**
```bash
vllm serve Qwen/Qwen2.5-7B \
    --tensor-parallel-size 1 \
    --pipeline-parallel-size 2 \
    --gpu-memory-utilization 0.98 \  # Maximize GPU memory usage (default is 90%)
    --max-model-len 13768 \  # Prevent CUDA OOM errors by limiting sequence length
    --enable-prefix-caching \  # Force KV cache reuse
    --num-lookahead-slots 5  # Prefetch tokens for improved efficiency
```

If you see **routing logs and** `"INFO: Application startup complete."`, the cluster is running successfully! 🚀

---

## **7️⃣ Verify Setup**
Run the following **from any node** to check if the model is loaded:
```bash
curl -X GET "http://localhost:8000/v1/models"
```
✅ If you get a **valid JSON response with the model name**, vLLM is running correctly.

---

## **8️⃣ Test Inference with a Long Prompt**
If KV cache is enabled but not utilized, test with **multiple concurrent queries**:
```bash
for i in {1..5}; do
  curl -X POST http://localhost:8000/v1/completions \
       -H "Content-Type: application/json" \
       -d '{
             "model": "Qwen/Qwen2.5-7B",
             "prompt": "Tell me a detailed story about AI and the future.",
             "max_tokens": 512
           }' &
done
```

---

## **Summary**
1️⃣ **Launch GPU instances** on **Vast.ai**.  
2️⃣ **Set up private networking** using **Tailscale**.  
3️⃣ **Deploy a Ray cluster** on multiple nodes.  
4️⃣ **Verify GPU availability** (`nvidia-smi`).  
5️⃣ **Run vLLM** with optimized settings.  
6️⃣ **Ensure the model is loaded** (`curl http://localhost:8000/v1/models`).  
7️⃣ **Send multiple concurrent requests** to fully utilize KV cache.  
