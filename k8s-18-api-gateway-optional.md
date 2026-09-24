# ☸️ Bước 18 - Tùy chọn bổ sung API Gateway

## 1. 🎯 Mục đích

File này phân biệt ingress/gateway routing với API management và đưa ra phương án migration kiến trúc sau khi ingress-nginx retired.

Kiến trúc hiện hữu của bộ tài liệu đang có:

```text
MetalLB -> Ingress NGINX -> Service -> Pod
```

Mô hình này vẫn mô tả đúng hệ thống đang chạy, nhưng ingress-nginx đã retired và không nên là đích kiến trúc cho cluster mới.

API Gateway riêng **chưa bắt buộc** nếu:

- Chỉ cần route domain/path.
- Backend tự verify JWT bằng middleware/library chung.
- Chưa cần API key, quota, rate limit, developer portal, analytics.

---

## 2. ✅ Checklist nội dung

```text
[ ] Đã xác định rõ use case API Gateway
[ ] Đã chọn sản phẩm/implementation
[ ] Đã xác định gateway là public entrypoint hay internal sau Ingress
[ ] Đã có strategy TLS
[ ] Đã có strategy auth giữa gateway và backend
[ ] Đã có timeout/retry/circuit breaking policy
[ ] Đã có log/metrics/tracing
[ ] Đã có plan migration từng domain/path
[ ] Đã có rollback plan về đường traffic hiện hữu trong giai đoạn migration
```

---

## 3. 🌐 Khi nào nên thêm API Gateway?

Cân nhắc Kong Gateway, Envoy Gateway, Apache APISIX hoặc tương đương nếu cần:

```text
[ ] Auth plugin tập trung OIDC/OAuth2/JWT
[ ] API key/consumer/app credential
[ ] Rate limit/quota theo consumer/user/app
[ ] Request/response transform
[ ] API analytics tập trung
[ ] Developer portal
[ ] Versioning API theo policy
[ ] Multi-team API management
[ ] Plugin ecosystem
```

---

## 4. 🌐 Mô hình A: Ingress NGINX phía trước API Gateway

```text
Client
  -> Ingress NGINX
  -> API Gateway Service
  -> Backend Services
```

Ưu điểm:

- Giữ Ingress NGINX làm entrypoint đang có.
- API Gateway có thể chạy internal trong cluster.
- Dễ đưa vào dần theo từng domain/path.

Nhược điểm:

- Thêm một hop proxy.
- Cần cấu hình timeout/header/client IP kỹ hơn.

---

## 5. 🌐 Mô hình B: API Gateway nhận LoadBalancer trực tiếp

```text
Client
  -> MetalLB External IP
  -> API Gateway Service LoadBalancer
  -> Backend Services
```

Ưu điểm:

- API Gateway là entrypoint trực tiếp.
- Phù hợp nếu API Gateway thay Ingress cho API domain.

Nhược điểm:

- Cần vận hành Gateway như thành phần critical.
- Cần TLS, firewall, HA, observability riêng.

---

## 6. 🌐 Mô hình C: Gateway API

Gateway API dùng các resource mới hơn:

```text
GatewayClass
Gateway
HTTPRoute
```

Implementation có thể là:

```text
Envoy Gateway
Kong Gateway
NGINX Gateway Fabric
Traefik
```

Mô hình:

```text
Client
  -> Gateway
  -> HTTPRoute
  -> Service backend
```

Gateway API phù hợp nếu team muốn chuẩn hóa route/gateway hiện đại hơn Ingress.

---

## 7. 💡 Khuyến nghị cho hệ thống hiện tại

Tách hai quyết định độc lập:

```text
1. Migration ingress layer:
   Ingress NGINX (legacy) -> Gateway API/controller còn support

2. API management:
   Chỉ thêm API Gateway khi thật sự cần rate limit/quota/API key/plugin/analytics/developer portal.
```

Trong thời gian migration vẫn duy trì TLS, NetworkPolicy, request/limit/probe, logging/monitoring và Jenkins RBAC. Không thêm API Gateway chỉ để thay thế một cách máy móc cho Ingress NGINX; hãy chọn implementation dựa trên yêu cầu routing và API management thực tế.
