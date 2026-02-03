---
name: gpu-workload
description: |
  GPU 工作負載管理技能。使用此技能來：
  - 安裝和配置 NVIDIA GPU Operator
  - 配置 DRA (Dynamic Resource Allocation)
  - 部署 GPU 推理服務
  - 監控 GPU 使用狀況
  專為 DGX Spark GB10 統一記憶體架構優化。
triggers:
  - gpu operator
  - nvidia kubernetes
  - dra driver
  - gpu workload
  - 推理服務
  - AI 部署
---

# GPU 工作負載管理技能

## DGX Spark 特殊考量

```
DGX Spark GB10 使用統一記憶體架構 (UMA)：
- 128GB DRAM 由 CPU 和 GPU 共享
- nvidia-smi 顯示 "Memory-Usage: Not Supported" 是正常的
- 需要 GPU Operator 25.10.0+ 才能正常運作
```

## 安裝 GPU Operator

```bash
# 添加 NVIDIA Helm repo
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# 安裝 GPU Operator (DGX Spark 配置)
# 注意：DGX Spark 已預裝驅動，設定 driver.enabled=false
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace \
  --set driver.enabled=false \
  --set toolkit.enabled=true \
  --set devicePlugin.version=v0.17.4 \
  --set migManager.enabled=false \
  --set dcgmExporter.enabled=true

# 等待部署完成
kubectl wait --for=condition=ready pod \
  -l app=nvidia-device-plugin-daemonset \
  -n gpu-operator --timeout=300s
```

## 驗證 GPU 可用

```bash
# 檢查 GPU 節點標籤
kubectl get nodes -o json | jq '.items[].metadata.labels | with_entries(select(.key | startswith("nvidia.com")))'

# 檢查 GPU 資源
kubectl describe node | grep -A5 "Capacity:" | grep nvidia

# 測試 GPU Pod
kubectl run gpu-test --rm -it --restart=Never \
  --image=nvcr.io/nvidia/cuda:12.0-base-ubuntu22.04 \
  --limits=nvidia.com/gpu=1 \
  -- nvidia-smi
```

## 安裝 DRA Driver

```bash
# DRA 需要 Kubernetes 1.32+ 且啟用 DynamicResourceAllocation

# 安裝 NVIDIA DRA Driver
helm install nvidia-dra-driver-gpu nvidia/nvidia-dra-driver-gpu \
  --version="25.8.0" \
  --namespace gpu-operator \
  --set cdi.enabled=true

# 驗證 DRA 資源類別
kubectl get resourceclasses
kubectl get deviceclasses
```

## DRA 使用範例

### ResourceClaimTemplate

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-claim
spec:
  spec:
    devices:
      requests:
      - name: gpu
        deviceClassName: gpu.nvidia.com
```

### 使用 DRA 的 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: inference-pod
spec:
  containers:
  - name: inference
    image: nvcr.io/nvidia/pytorch:24.01-py3
    command: ["python", "-c", "import torch; print(torch.cuda.is_available())"]
    resources:
      claims:
      - name: gpu
  resourceClaims:
  - name: gpu
    resourceClaimTemplateName: gpu-claim
```

## GPU 時間切片 (Time Slicing)

```yaml
# ConfigMap for time-slicing
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  any: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        renameByDefault: false
        resources:
        - name: nvidia.com/gpu
          replicas: 4
```

## 監控 GPU

```bash
# 使用 dcgm-exporter 指標
kubectl port-forward -n gpu-operator svc/nvidia-dcgm-exporter 9400:9400

# 查看指標
curl localhost:9400/metrics | grep DCGM

# 或使用 nvtop (DGX Spark 專用)
nvtop
```

## 常見問題

### "error getting device memory: Not Supported"
這是 DGX Spark UMA 架構的已知問題，升級到 GPU Operator 25.10.0+ 可解決。

### Pod 無法調度到 GPU
```bash
# 檢查節點是否有 GPU 資源
kubectl describe node | grep "nvidia.com/gpu"

# 檢查 device plugin 狀態
kubectl logs -n gpu-operator -l app=nvidia-device-plugin-daemonset
```

### DRA ResourceClaim 卡在 Pending
```bash
# 檢查 DRA controller 日誌
kubectl logs -n gpu-operator -l app=nvidia-dra-driver-gpu
```
