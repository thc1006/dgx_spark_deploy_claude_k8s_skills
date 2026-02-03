---
name: k8s-deploy
description: |
  Kubernetes 叢集部署與管理技能。使用此技能來：
  - 部署新的 Kubernetes 叢集 (使用 Kubespray)
  - 升級現有叢集版本
  - 添加或移除節點
  - 配置 containerd、CNI、GPU Operator
  適用於 DGX Spark ARM64 環境。
triggers:
  - deploy kubernetes
  - install k8s
  - 部署 kubernetes
  - 安裝 k8s
  - kubespray
  - 叢集部署
---

# Kubernetes 叢集部署技能

## 環境需求

- **平台**: DGX Spark (ARM64/aarch64)
- **OS**: Ubuntu 24.04 LTS
- **Container Runtime**: containerd 2.0+
- **目標版本**: Kubernetes 1.34.x

## 部署前檢查清單

```bash
# 1. 檢查系統架構
uname -m  # 應顯示 aarch64

# 2. 檢查 Python 環境
source ~/kubespray-venv/bin/activate
python --version

# 3. 檢查 Ansible
ansible --version

# 4. 檢查 SSH 連線 (多節點時)
ssh-keygen -t ed25519 -C "kubespray"
```

## 單節點部署 (DGX Spark)

```bash
cd ~/kubespray

# 複製配置範本
cp -rfp inventory/sample inventory/dgx-spark

# 編輯 inventory
cat > inventory/dgx-spark/inventory.ini << 'EOF'
[all]
dgx-spark ansible_host=127.0.0.1 ansible_connection=local

[kube_control_plane]
dgx-spark

[etcd]
dgx-spark

[kube_node]
dgx-spark

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
EOF

# 配置叢集參數
cat > inventory/dgx-spark/group_vars/k8s_cluster/k8s-cluster.yml << 'EOF'
kube_version: v1.34.0
container_manager: containerd
containerd_version: 2.0.4
kube_network_plugin: calico
kube_proxy_mode: iptables
enable_nodelocaldns: true

# DRA Feature Gate
kube_feature_gates:
  - DynamicResourceAllocation=true

# Gateway API
gateway_api_enabled: true
EOF

# 執行部署
ansible-playbook -i inventory/dgx-spark/inventory.ini \
  --become --become-user=root cluster.yml
```

## 多節點部署

```bash
# 編輯 inventory/dgx-spark/inventory.ini
# 添加所有節點的 IP 和角色

# 確保 SSH 金鑰已複製到所有節點
ssh-copy-id user@node-ip

# 執行部署
ansible-playbook -i inventory/dgx-spark/inventory.ini \
  --become --become-user=root cluster.yml
```

## 部署後驗證

```bash
# 配置 kubectl
mkdir -p ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# 驗證叢集
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

## 常見問題

### containerd 版本不符
```bash
# 確認 containerd 版本
containerd --version

# 如需升級，編輯 group_vars
containerd_version: 2.0.4
```

### ARM64 映像問題
```bash
# 確認所有 Pod 使用多架構映像
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u
```

## 注意事項

- 單節點叢集不具高可用性，適用於開發/測試
- DGX Spark 使用 UMA，nvidia-smi 顯示異常是正常的
- 避免使用 IPVS 模式 (K8s 1.36 將移除)
