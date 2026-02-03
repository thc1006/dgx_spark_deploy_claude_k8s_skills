---
name: frontend-deploy
description: |
  前端網頁伺服器部署技能。當使用者需要：
  - 部署前端應用 (React, Vue, Angular, 靜態網站)
  - 配置 Web 流量路由
  - 設定 HTTPS/TLS
  - 配置 CDN 或反向代理

  IMPORTANT: 此技能使用 Gateway API + Envoy Gateway。
  絕對不要使用 ingress-nginx，因為它將於 2026 年 3 月退役。
triggers:
  - frontend
  - web server
  - 前端部署
  - 網頁伺服器
  - react deploy
  - vue deploy
  - static site
  - 靜態網站
  - https
  - tls
  - reverse proxy
  - 反向代理
---

# 前端網頁伺服器部署

## CRITICAL: 技術選型

```
███████████████████████████████████████████████████████████████████
█                                                                 █
█  DO NOT USE: ingress-nginx                                      █
█  REASON: 將於 2026 年 3 月進入 kubernetes-retired               █
█                                                                 █
█  MUST USE: Gateway API + Envoy Gateway                          █
█  REASON: Kubernetes 官方標準，持續維護                           █
█                                                                 █
███████████████████████████████████████████████████████████████████
```

## 前置條件

確認 Envoy Gateway 已安裝：
```bash
kubectl get gatewayclass envoy
kubectl get gateway -A
```

如未安裝，執行：
```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.6.3 \
  -n envoy-gateway-system \
  --create-namespace
```

## 部署前端應用

### 步驟 1: 建立 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:alpine  # 或您的前端映像
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

### 步驟 2: 建立 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: default
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

### 步驟 3: 建立 Gateway (如尚未存在)

```yaml
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
```

### 步驟 4: 建立 HTTPRoute

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
  - "www.example.com"
  - "example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-service
      port: 80
```

## HTTPS 配置

### 建立 TLS 憑證

```bash
# 開發環境：自簽憑證
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=example.com"

kubectl create secret tls frontend-tls \
  --cert=tls.crt --key=tls.key

# 生產環境：使用 cert-manager
# kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml
```

### HTTPS Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: https-gateway
  namespace: default
spec:
  gatewayClassName: envoy
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    hostname: "*.example.com"
    tls:
      mode: Terminate
      certificateRefs:
      - name: frontend-tls
    allowedRoutes:
      namespaces:
        from: All
  - name: http-redirect
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
```

### HTTP 到 HTTPS 重導向

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-redirect
  namespace: default
spec:
  parentRefs:
  - name: https-gateway
    sectionName: http-redirect
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
```

## SPA (Single Page Application) 配置

React/Vue/Angular 等 SPA 需要特殊配置處理 client-side routing：

### Nginx ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-spa-config
data:
  default.conf: |
    server {
        listen 80;
        root /usr/share/nginx/html;
        index index.html;

        # SPA fallback
        location / {
            try_files $uri $uri/ /index.html;
        }

        # 靜態資源快取
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        # 禁止快取 index.html
        location = /index.html {
            expires -1;
            add_header Cache-Control "no-store, no-cache, must-revalidate";
        }
    }
```

### 使用 ConfigMap 的 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-spa
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend-spa
  template:
    metadata:
      labels:
        app: frontend-spa
    spec:
      containers:
      - name: frontend
        image: your-spa-image:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-spa-config
```

## 多環境部署

```yaml
# staging.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-staging
  namespace: staging
spec:
  parentRefs:
  - name: main-gateway
    namespace: default
  hostnames:
  - "staging.example.com"
  rules:
  - backendRefs:
    - name: frontend-service
      port: 80

---
# production.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-production
  namespace: production
spec:
  parentRefs:
  - name: main-gateway
    namespace: default
  hostnames:
  - "www.example.com"
  - "example.com"
  rules:
  - backendRefs:
    - name: frontend-service
      port: 80
```

## 驗證部署

```bash
# 取得 Gateway IP
GATEWAY_IP=$(kubectl get gateway main-gateway -o jsonpath='{.status.addresses[0].value}')

# 測試連線
curl -H "Host: www.example.com" http://$GATEWAY_IP/

# 檢查路由狀態
kubectl get httproute frontend-route -o yaml | grep -A10 status
```

## 故障排除

### 502 Bad Gateway
```bash
# 檢查後端 Pod 狀態
kubectl get pods -l app=frontend
kubectl logs -l app=frontend

# 檢查 Service endpoints
kubectl get endpoints frontend-service
```

### 路由不生效
```bash
# 確認 HTTPRoute 已被 Gateway 接受
kubectl describe httproute frontend-route

# 檢查 Envoy 配置
kubectl logs -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=main-gateway
```
