---
name: ingress-migration
description: |
  Ingress 流量管理遷移技能。

  CRITICAL DEPRECATION NOTICE:
  - ingress-nginx 將於 2026-03 進入 kubernetes-retired
  - Kubernetes Dashboard 已退役

  此技能處理所有 Ingress/流量路由相關請求，
  自動使用 Gateway API 替代 ingress-nginx。

  觸發場景：
  - 任何 Ingress 配置請求
  - 網頁流量路由
  - TLS/HTTPS 設定
  - 負載均衡配置
  - 反向代理設定
triggers:
  - ingress
  - nginx ingress
  - ingress-nginx
  - ingress controller
  - load balancer
  - 負載均衡
  - traffic routing
  - 流量路由
  - reverse proxy
  - 反向代理
  - web traffic
  - 網頁流量
user_invocable: true
---

# Ingress 遷移指南

## 技術選型決策樹

```
使用者需求：配置 Ingress/流量路由
          │
          ▼
    ┌─────────────────────────────────────────┐
    │  Q: 使用者是否明確要求 ingress-nginx?   │
    └─────────────────────────────────────────┘
          │
    ┌─────┴─────┐
    │           │
   YES          NO
    │           │
    ▼           ▼
┌───────────┐  ┌─────────────────────────────────┐
│ 警告使用者 │  │ 直接使用 Gateway API            │
│ 即將退役   │  │ 實作：Envoy Gateway             │
│ 建議遷移   │  └─────────────────────────────────┘
└───────────┘
```

## 退役時間線

```
2025-11  官方宣布 Ingress-NGINX 退役計劃
2026-03  Ingress-NGINX 進入 kubernetes-retired
         ├── 不再發布新版本
         ├── 不再修復 Bug
         └── 不再修補安全漏洞

2026-04  Kubernetes 1.36 發布
         ├── containerd 1.x 終止支援
         └── kube-proxy IPVS 移除
```

## 等效功能對照表

| Ingress-NGINX | Gateway API (Envoy) |
|---------------|---------------------|
| `Ingress` | `HTTPRoute` |
| `IngressClass` | `GatewayClass` |
| `annotations` | `HTTPRoute filters` |
| `nginx.ingress.kubernetes.io/rewrite-target` | `URLRewrite filter` |
| `nginx.ingress.kubernetes.io/ssl-redirect` | `RequestRedirect filter` |
| `nginx.ingress.kubernetes.io/proxy-body-size` | `BackendTrafficPolicy` |

## 遷移範例

### 舊寫法 (ingress-nginx) - 請勿使用

```yaml
# ❌ DEPRECATED - DO NOT USE
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

### 新寫法 (Gateway API) - 請使用這個

```yaml
# ✅ RECOMMENDED - USE THIS
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
spec:
  gatewayClassName: envoy
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    hostname: example.com
    tls:
      mode: Terminate
      certificateRefs:
      - name: tls-secret
  - name: http
    port: 80
    protocol: HTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
  - name: main-gateway
  hostnames:
  - example.com
  rules:
  # HTTP to HTTPS redirect
  - matches:
    - path:
        type: PathPrefix
        value: /
    filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
spec:
  parentRefs:
  - name: main-gateway
    sectionName: https
  hostnames:
  - example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /app
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /
    backendRefs:
    - name: my-service
      port: 80
```

## 安裝 Envoy Gateway

```bash
# 確認未安裝 ingress-nginx
kubectl get pods -A | grep ingress-nginx

# 安裝 Envoy Gateway
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.6.3 \
  -n envoy-gateway-system \
  --create-namespace

# 驗證
kubectl get gatewayclass
kubectl get pods -n envoy-gateway-system
```

## 從 ingress-nginx 遷移步驟

1. **審計現有 Ingress 資源**
   ```bash
   kubectl get ingress -A -o yaml > ingress-backup.yaml
   ```

2. **安裝 Envoy Gateway** (與 ingress-nginx 並存)

3. **逐步遷移路由**
   - 為每個 Ingress 建立對應的 HTTPRoute
   - 測試新路由
   - 切換 DNS 或 LoadBalancer

4. **移除 ingress-nginx**
   ```bash
   helm uninstall ingress-nginx -n ingress-nginx
   kubectl delete namespace ingress-nginx
   ```

## 相關退役元件

### Kubernetes Dashboard → Headlamp

```bash
# 安裝 Headlamp (Dashboard 替代)
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm install headlamp headlamp/headlamp -n kube-system
```

### containerd 1.x → containerd 2.0+

```bash
# 檢查 containerd 版本
containerd --version

# K8s 1.36 需要 containerd 2.0+
```

## 參考資源

- [Ingress NGINX Retirement Announcement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway](https://gateway.envoyproxy.io/)
- [Migration Guide](https://www.pulumi.com/blog/ingress-nginx-to-gateway-api-kgateway/)
