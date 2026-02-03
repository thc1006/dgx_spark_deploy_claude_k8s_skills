# DGX Spark Kubernetes 部署指南

> 本文件為 Claude Code 專案指導文件，包含 DGX Spark 上部署 Kubernetes 的完整規劃與最佳實踐。

> You should rename this file's name from README.md into "CLAUDE.md"

> **最後更新**: 2026-02-03
> **硬體平台**: NVIDIA DGX Spark (GB10 Superchip)
> **目標**: 完整功能 Kubernetes + Gateway API + DRA 支援
> **作者** ：蔡秀吉（thc1006）
---

## 系統環境

| 項目 | 值 |
|------|-----|
| **硬體** | NVIDIA DGX Spark (GB10 Superchip) |
| **架構** | ARM64 (aarch64) - Cortex-X925/A725 |
| **OS** | Ubuntu 24.04.3 LTS |
| **Kernel** | 6.14.0-1015-nvidia (HWE Stack) |
| **GPU Driver** | 580.126.09 |
| **CUDA** | 13.0 |
| **統一記憶體** | 128GB UMA (CPU/GPU 共享) |

---

## 技術棧選型

### 核心組件

| 層級 | 選擇 | 版本 | 理由 |
|------|------|------|------|
| **Kubernetes** | Kubespray 安裝 | 1.34.x | DRA GA 支援，穩定版本 |
| **Container Runtime** | containerd | 2.0+ | K8s 1.36 強制要求 |
| **GPU 支援** | NVIDIA GPU Operator | 25.10.1+ | DGX Spark 官方支援 |
| **DRA Driver** | NVIDIA k8s-dra-driver-gpu | 25.8.0+ | 動態資源分配 |
| **Ingress** | Envoy Gateway | 1.6.3+ | Gateway API 實作，ARM64 支援 |
| **CNI** | Calico | 最新版 | ARM64 完整支援 |
| **Dashboard** | Headlamp | 最新版 | 官方推薦替代方案 |

### 為何不使用 Ingress-NGINX

```
IMPORTANT: Ingress-NGINX 將於 2026 年 3 月進入 kubernetes-retired

原因：
- 過去數年僅 1-2 位維護者，使用業餘時間維護
- 無法達到 Kubernetes 和使用者認為安全的標準
- InGate 替代方案開發也未能成功

影響：
- 2026/03 後不再發布新版本
- 不再修復 Bug 和安全漏洞
- 現有部署仍可運作，但風險逐漸增加

參考：https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/
```

### 為何不使用 Kubernetes Dashboard

```
IMPORTANT: Kubernetes Dashboard 已進入 kubernetes-retired

原因：
- 缺少活躍維護者和貢獻者
- Maintainer 沒有時間繼續維護

替代方案：
- Headlamp (官方推薦，SIG UI 子專案)
- K9s (CLI 工具)
- Lens / OpenLens (桌面應用)

參考：https://alexandre-vazquez.com/kubernetes-dashboard-alternatives-2026/
```

---

## 重要注意事項

### DGX Spark 統一記憶體 (UMA)

```
DGX Spark 使用統一記憶體架構 (UMA)：
- 128GB DRAM 由 CPU 和 GPU 共享
- nvidia-smi 顯示 "Memory-Usage: Not Supported" 是正常現象
- GPU Operator 25.10.0+ 已修復相關問題

參考：https://github.com/NVIDIA/gpu-operator/issues/1794
```

### DRA (Dynamic Resource Allocation) 狀態

```
Kubernetes 1.34: DRA Core APIs 達到 GA
Kubernetes 1.35: Feature Gate 鎖定，無法關閉
Kubernetes 1.36: 穩定運作

GA 功能：
- resource.k8s.io/v1 穩定 API
- Admin Access Labelling (Beta)
- Prioritized List (Beta)

Alpha 功能（謹慎使用）：
- Extended Resource Mapping
- Consumable Capacity
- Resource Health Status
```

### Kubernetes 1.36 重大變更

```
預計發布：2026/04/22

Breaking Changes：
1. containerd 1.x 終止支援 - 必須使用 containerd 2.0+
2. kube-proxy IPVS 模式移除 - 需遷移至 iptables/nftables
3. Docker Schema 1 映像不支援 - 需轉換為 OCI 格式

準備工作：
- 現在就使用 containerd 2.0
- 檢查所有容器映像是否為 OCI 格式
- 避免使用 IPVS 模式
```

---

## 部署架構

```
                        ┌─────────────────────────────────────────┐
                        │            External Traffic             │
                        └─────────────────┬───────────────────────┘
                                          │
                        ┌─────────────────▼───────────────────────┐
                        │         Envoy Gateway (Gateway API)     │
                        │         ┌─────────────────────────┐     │
                        │         │   GatewayClass: envoy   │     │
                        │         └─────────────────────────┘     │
                        └─────────────────┬───────────────────────┘
                                          │
              ┌───────────────────────────┼───────────────────────┐
              │                           │                       │
              ▼                           ▼                       ▼
    ┌─────────────────┐         ┌─────────────────┐     ┌─────────────────┐
    │   HTTPRoute     │         │   HTTPRoute     │     │   HTTPRoute     │
    │   /frontend     │         │   /api          │     │   /inference    │
    └────────┬────────┘         └────────┬────────┘     └────────┬────────┘
             │                           │                       │
             ▼                           ▼                       ▼
    ┌─────────────────┐         ┌─────────────────┐     ┌─────────────────┐
    │   Frontend      │         │   API Service   │     │   AI Inference  │
    │   Service       │         │   (Backend)     │     │   (GPU + DRA)   │
    └─────────────────┘         └─────────────────┘     └─────────────────┘
```

---

## 安裝步驟

### Phase 1: 準備工作

```bash
# 1. 安裝 Ansible 和依賴
sudo apt update && sudo apt install -y python3-pip python3-venv git

# 2. 建立虛擬環境
python3 -m venv ~/kubespray-venv
source ~/kubespray-venv/bin/activate

# 3. 下載 Kubespray
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
pip install -U -r requirements.txt

# 4. 複製範例配置
cp -rfp inventory/sample inventory/dgx-spark
```

### Phase 2: 配置 Kubespray

```yaml
# inventory/dgx-spark/group_vars/k8s_cluster/k8s-cluster.yml

# Kubernetes 版本
kube_version: v1.34.0

# Container Runtime - 使用 containerd 2.0
container_manager: containerd
containerd_version: 2.0.4

# 網路插件
kube_network_plugin: calico

# 啟用 DRA Feature Gate
kube_feature_gates:
  - DynamicResourceAllocation=true

# 啟用 Gateway API CRDs
gateway_api_enabled: true
```

### Phase 3: 部署 Kubernetes

```bash
# 部署叢集
ansible-playbook -i inventory/dgx-spark/inventory.ini \
  --become --become-user=root cluster.yml
```

### Phase 4: 安裝 GPU Operator

```bash
# 添加 NVIDIA Helm repo
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# 安裝 GPU Operator (DGX Spark 專用配置)
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace \
  --set driver.enabled=false \
  --set toolkit.enabled=true \
  --set devicePlugin.version=v0.17.4
```

### Phase 5: 安裝 Envoy Gateway

```bash
# 使用 Helm 安裝 Envoy Gateway
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.6.3 \
  -n envoy-gateway-system \
  --create-namespace

# 等待部署完成
kubectl wait --timeout=5m -n envoy-gateway-system \
  deployment/envoy-gateway --for=condition=Available

# 應用 Gateway 配置
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: default
spec:
  gatewayClassName: envoy
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
EOF
```

### Phase 6: 安裝 Headlamp (Dashboard 替代)

```bash
# 添加 Headlamp Helm repo
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm repo update

# 安裝 Headlamp
helm install headlamp headlamp/headlamp \
  --namespace kube-system \
  --set config.oidc.enabled=false
```

### Phase 7: 安裝 DRA Driver (可選)

```bash
# 安裝 NVIDIA DRA Driver
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm install nvidia-dra-driver-gpu nvidia/nvidia-dra-driver-gpu \
  --version="25.8.0" \
  --namespace gpu-operator
```

---

## Gateway API 使用範例

### 前端服務路由

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
  namespace: default
spec:
  parentRefs:
  - name: main-gateway
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-service
      port: 80
```

### API 後端路由（帶權重分流）

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
  namespace: default
spec:
  parentRefs:
  - name: main-gateway
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/v1
    backendRefs:
    - name: api-v1
      port: 8080
      weight: 90
    - name: api-v2
      port: 8080
      weight: 10
```

### AI 推理服務（使用 DRA）

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-claim-template
spec:
  spec:
    devices:
      requests:
      - name: gpu
        deviceClassName: gpu.nvidia.com
---
apiVersion: v1
kind: Pod
metadata:
  name: inference-pod
spec:
  containers:
  - name: inference
    image: nvcr.io/nvidia/pytorch:24.01-py3
    resources:
      claims:
      - name: gpu
  resourceClaims:
  - name: gpu
    resourceClaimTemplateName: gpu-claim-template
```

---

## 安全公告追蹤

### CVE-2026-1580, CVE-2026-24512, CVE-2026-24513, CVE-2026-24514

```
日期：2026-02-03
影響：ingress-nginx < v1.13.7 和 < v1.14.3
嚴重程度：HIGH (CVSS 8.8)

本專案使用 Envoy Gateway，不受此漏洞影響。

如果其他專案使用 ingress-nginx，請立即升級：
- v1.13.x → v1.13.7
- v1.14.x → v1.14.3
```

---

## 參考資源

### 官方文件
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway](https://gateway.envoyproxy.io/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html)
- [NVIDIA DRA Driver](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/dra-intro-install.html)
- [Kubespray](https://github.com/kubernetes-sigs/kubespray)
- [Headlamp](https://headlamp.dev/)

### 重要公告
- [Ingress NGINX Retirement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Kubernetes v1.34: DRA GA](https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/)
- [Kubernetes 1.36 Release](https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/README.md)

### 社群資源
- [DGX Spark Forums](https://forums.developer.nvidia.com/c/accelerated-computing/dgx-spark-gb10/719)
- [Kubernetes Slack](https://slack.k8s.io/)
- [CNCF Slack](https://slack.cncf.io/)

---

## Claude Code 使用提示

```
本專案使用 Claude Code 進行開發和維護。

常用指令：
- 部署 Kubernetes：參考上述 Phase 1-7 步驟
- 檢查 GPU 狀態：kubectl get pods -n gpu-operator
- 檢查 Gateway：kubectl get gateways,httproutes -A
- 查看 Dashboard：kubectl port-forward -n kube-system svc/headlamp 8080:80

注意事項：
- DGX Spark 使用 ARM64 架構，確保所有映像支援 aarch64
- 統一記憶體 (UMA) 導致 nvidia-smi 顯示異常是正常的
- 避免使用即將退役的 Ingress-NGINX 和 Kubernetes Dashboard
```
