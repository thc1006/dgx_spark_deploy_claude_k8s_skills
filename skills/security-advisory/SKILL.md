---
name: security-advisory
description: |
  Kubernetes 安全公告追蹤技能。使用此技能來：
  - 檢查當前環境的 CVE 影響
  - 獲取安全修補建議
  - 追蹤已知漏洞
  - 執行安全掃描
triggers:
  - cve
  - security advisory
  - 安全漏洞
  - 安全公告
  - vulnerability
---

# Kubernetes 安全公告追蹤

## 當前已知漏洞 (2026-02)

### CVE-2026-1580, CVE-2026-24512, CVE-2026-24513, CVE-2026-24514

```
日期：2026-02-03
元件：ingress-nginx
嚴重程度：HIGH (CVSS 8.8)

受影響版本：
- ingress-nginx < v1.13.7
- ingress-nginx < v1.14.3

修復版本：
- v1.13.7
- v1.14.3

本專案狀態：不受影響 (使用 Envoy Gateway)
```

## 檢查環境是否受影響

### Ingress-NGINX 檢查

```bash
# 檢查是否安裝 ingress-nginx
kubectl get pods --all-namespaces \
  --selector app.kubernetes.io/name=ingress-nginx

# 如有安裝，檢查版本
kubectl get deployment -n ingress-nginx \
  ingress-nginx-controller \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### 容器映像掃描

```bash
# 使用 trivy 掃描
trivy image <image-name>

# 掃描整個叢集
trivy k8s --report summary cluster
```

### RBAC 檢查

```bash
# 檢查過度授權的 ClusterRoleBinding
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'

# 檢查 default ServiceAccount 權限
kubectl auth can-i --list --as=system:serviceaccount:default:default
```

## 安全最佳實踐

### Pod Security Standards

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### 限制特權容器

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
```

## 訂閱安全公告

### 郵件列表

- kubernetes-security-announce@googlegroups.com
- dev@kubernetes.io

### RSS/Feeds

- https://kubernetes.io/feed.xml
- https://github.com/kubernetes/kubernetes/releases.atom

### 追蹤資源

- [Kubernetes Security](https://kubernetes.io/docs/concepts/security/)
- [CVE Database](https://www.cve.org/)
- [NIST NVD](https://nvd.nist.gov/)

## 緊急回應流程

1. **評估影響範圍**
   ```bash
   kubectl get pods -A -o json | jq '.items[].spec.containers[].image' | sort -u
   ```

2. **隔離受影響工作負載**
   ```bash
   kubectl cordon <affected-node>
   kubectl drain <affected-node> --ignore-daemonsets
   ```

3. **套用修補**
   ```bash
   helm upgrade <release> <chart> --set image.tag=<patched-version>
   ```

4. **驗證修補**
   ```bash
   kubectl rollout status deployment/<name>
   trivy image <new-image>
   ```

5. **恢復服務**
   ```bash
   kubectl uncordon <node>
   ```

## 即將退役的元件

| 元件 | 退役日期 | 替代方案 |
|------|----------|----------|
| Ingress-NGINX | 2026-03 | Gateway API |
| Kubernetes Dashboard | 已退役 | Headlamp |
| containerd 1.x | K8s 1.36 | containerd 2.0+ |
| kube-proxy IPVS | K8s 1.36 | iptables/nftables |
