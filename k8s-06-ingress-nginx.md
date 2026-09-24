# ☸️ Bước 06 - Ingress NGINX legacy và kế hoạch migration

## 1. 🎯 Mục đích

File này ghi lại cách **duy trì hoặc tái lập Ingress NGINX của kiến trúc hiện hữu** để bảo đảm 5 cluster có cùng cách route trong giai đoạn chuyển đổi. Thực hiện riêng trên từng cluster với kubeconfig/MetalLB pool/domain của cluster đó. Không dùng file này như khuyến nghị chọn ingress controller mới.

> **Cảnh báo lifecycle 2026-09-03:** upstream ingress-nginx đã retired từ tháng 3/2026 và không còn release, bug fix hoặc security fix. Cluster hiện hữu vẫn có thể chạy, nhưng cần lập kế hoạch migration sang Gateway API hoặc controller khác còn support.

Ingress NGINX hiện là HTTP/HTTPS entrypoint cho cluster.

```text
Client
  -> Firewall / Public Reverse Proxy nếu có
  -> MetalLB External IP:80/443
  -> Service LoadBalancer ingress-nginx-controller
  -> Ingress NGINX Controller Pod
  -> Kubernetes Service
  -> Pod
```

> Ingress NGINX là ingress controller, không phải API Gateway đầy đủ. Nếu cần API Gateway, xem mục 14 trong file này.

---

## 2. ✅ Checklist nội dung

```text
[ ] Helm đã cài
[ ] ingress-nginx namespace Active
[ ] Ingress Controller Running
[ ] Service ingress-nginx-controller LoadBalancer
[ ] External IP từ MetalLB OK
[ ] externalTrafficPolicy = Cluster hoặc Local theo thiết kế
[ ] Firewall chỉ public 80/443
[ ] App test Running
[ ] Service test có endpoint
[ ] Ingress route đúng domain/path
[ ] Curl HTTP OK
[ ] TLS plan đã rõ
[ ] ConfigMap timeout/snippet an toàn
[ ] Namespace test đã dọn
[ ] Đã xác định có cần API Gateway riêng hay chưa
[ ] Đã có owner và kế hoạch migration khỏi ingress-nginx
```

---

## 3. 📌 Mục tiêu

- Cài Ingress NGINX bằng Helm để dễ pin/upgrade/rollback.
- Service ingress-nginx-controller nhận External IP từ MetalLB.
- Chỉ public 80/443.
- Test route domain/path.
- Cấu hình timeout/body-size/snippet annotation an toàn.
- Tách namespace test riêng, không dùng `dev` để tránh xóa nhầm.

---

## 4. ⛵ Cài Helm nếu chưa có

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## 5. 🌐 Cài Ingress NGINX bằng Helm

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Cài/upgrade, chạy trên master-1 hoặc máy đã có kubeconfig.:

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.externalTrafficPolicy=Cluster \
  --set controller.config.allow-snippet-annotations="false" \
  --set controller.config.proxy-connect-timeout="5" \
  --set controller.config.proxy-read-timeout="60" \
  --set controller.config.proxy-send-timeout="60"
```

> Sau khi kiểm thử, pin chart version bằng `--version <CHART_VERSION>` để bảo đảm khả năng tái lập.

---

## 6. 🌐 Kiểm tra Ingress NGINX

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Kết quả đúng:

```text
ingress-nginx-controller Pod Running
Service ingress-nginx-controller TYPE LoadBalancer
EXTERNAL-IP nằm trong pool MetalLB
PORT(S) có 80 và 443
```

Nếu External IP pending, quay lại kiểm tra MetalLB.

---

## 7. 🔎 Test routing bằng app mẫu

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Tạo namespace test riêng:

```bash
kubectl create namespace ingress-test
```

Deploy app:

```bash
kubectl create deployment test-nginx --image=nginx:stable -n ingress-test
kubectl expose deployment test-nginx --port=80 --name=test-nginx-svc -n ingress-test
```

Tạo Ingress:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-nginx-ingress
  namespace: ingress-test
spec:
  ingressClassName: nginx
  rules:
  - host: test.dev.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-nginx-svc
            port:
              number: 80
EOF
```

Kiểm tra:

```bash
kubectl get ingress -n ingress-test
kubectl describe ingress test-nginx-ingress -n ingress-test
```

---

## 8. 🌐 Map domain test và curl

> **📍 Thực hiện tại:** Lấy thông tin bằng kubeconfig trên control-plane-1; sửa hosts/curl trên máy client test.

Lấy External IP:

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

Ví dụ External IP là `172.17.79.160`, trên máy client/test:

```bash
sudo sh -c 'echo "172.17.79.160 test.dev.local" >> /etc/hosts'
getent hosts test.dev.local
curl http://test.dev.local
```

Kết quả đúng là trang Welcome to nginx.

---

## 9. 🌐 Ví dụ Ingress thực tế cho API

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
  namespace: develop
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 8080
```

Khuyến nghị domain:

```text
app.example.com       -> frontend
admin.example.com     -> admin
sso.example.com       -> SSO
api.example.com       -> public API / BFF
```

Không expose mỗi backend nội bộ thành một public domain nếu không cần.

---

## 10. 🔐 TLS

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

TLS có thể xử lý theo một trong ba cách:

```text
1. Terminate TLS ở public reverse proxy bên ngoài, rồi forward HTTP nội bộ tới Ingress.
2. Terminate TLS tại Ingress NGINX bằng Kubernetes TLS Secret.
3. Dùng cert-manager cho public domain hoặc CA nội bộ.
```

Nếu dùng TLS Secret thủ công, xem file `k8s-15-kubernetes-tls-secret.md`.

---

## 11. 🔎 ConfigMap kiểm tra lại

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl get configmap ingress-nginx-controller -n ingress-nginx -o yaml
```

Cần có:

```yaml
allow-snippet-annotations: "false"
proxy-connect-timeout: "5"
proxy-read-timeout: "60"
proxy-send-timeout: "60"
```

Không bật `allow-snippet-annotations` nếu không thật sự cần, vì annotation snippet có thể tăng rủi ro bảo mật.

---

## 12. 🔎 Cleanup test

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl delete namespace ingress-test
```

> Chỉ xóa namespace test được tạo trong tài liệu; không xóa `sevago-develop`.

---

## 13. 🛠️ Troubleshooting

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Nếu curl domain không được:

```text
Kiểm tra DNS hoặc /etc/hosts trên client
Kiểm tra External IP của ingress-nginx-controller
Kiểm tra MetalLB speaker/controller
Kiểm tra firewall 80/443
Kiểm tra Ingress Resource đúng host/path
Kiểm tra Service backend có endpoint không
Kiểm tra Pod app Running không
```

Lệnh kiểm tra endpoint:

```bash
kubectl get svc,endpoints -n ingress-test
kubectl get ingress -n ingress-test -o yaml
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

---

## 14. 🌐 Khi nào cần API Gateway riêng?

Hiện tại tài liệu dùng:

```text
Client -> Ingress NGINX -> Backend Service
```

Cấu hình này đủ nếu cần: Domain/path routing, TLS termination, Reverse proxy HTTP/HTTPS, Backend tự verify token.

Cân nhắc thêm API Gateway như Kong/Envoy Gateway/APISIX khi cần:

- Auth plugin tập trung.
- API key / consumer / app credential.
- Rate limit/quota theo user/client.
- Request/response transform.
- API analytics/developer portal.
- Policy API tập trung cho nhiều team.

Mô hình bổ sung gợi ý:

```text
Client
  -> Ingress NGINX hoặc LoadBalancer
  -> API Gateway
  -> Backend Service
```

Hướng migration ưu tiên là dùng Gateway API với implementation còn được support:

```text
GatewayClass / Gateway / HTTPRoute
  -> Envoy Gateway hoặc Kong Gateway
  -> Service backend
```

Không thêm API Gateway chỉ để verify JWT nếu backend đã có middleware chung, vì sẽ tăng độ phức tạp vận hành.
