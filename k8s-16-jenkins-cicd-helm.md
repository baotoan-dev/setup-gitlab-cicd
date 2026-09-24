# ☸️ Bước 16 - Triển khai ứng dụng bằng Jenkins và Helm

## 1. 🎯 Mục đích

File này chuẩn hóa logic Jenkins + Helm để build/deploy application. Ví dụ pipeline/domain chi tiết hiện thiên về Develop; khi mở rộng Staging/Production phải chọn đúng kubeconfig, namespace, credential và domain của cluster đích.

Tài liệu này hướng dẫn triển khai ứng dụng bằng Jenkins và Helm.

---

## 2. ✅ Checklist nội dung

- [ ] Nguyên tắc thiết kế
- [ ] Rule App Type
- [ ] Rule Domain
- [ ] Rule TLS
- [ ] Rule Ingress
- [ ] Realtime Ingress
- [ ] Chuẩn bị Jenkins trước khi tạo Job
- [ ] Rule Replica
- [ ] Jenkins Job
- [ ] Source Helm
- [ ] Nội dung từng file Helm
- [ ] Cấu hình gRPC Service
- [ ] Cấu hình Media Service
- [ ] Luồng upload
- [ ] Kiểm tra sau khi deploy

---

## 3. 📐 Nguyên tắc thiết kế

```text
Jenkins = xử lý logic deploy
Helm = nhận values và render manifest
Kubernetes = chạy workload
```

Nguyên tắc chính:

```text
- Jenkins quyết định app type, domain, TLS secret, replicaCount, image, namespace.
- Helm không chứa business logic.
- Helm chỉ render Deployment, Service và Ingress.
- Frontend/Admin chỉ sở hữu route /.
- Backend sở hữu route REST `/api`.
- Mọi Backend mặc định có thêm Realtime Ingress riêng.
- Realtime Ingress expose hai prefix chuẩn: `/socket.io` cho NestJS Socket.IO và `/hub` cho .NET SignalR.
- Backend có Ingress riêng.
- gRPC có Ingress và values riêng.
- Ingress NGINX kết nối đến gRPC Pod bằng HTTP/2.
- gRPC Pod bắt buộc Kestrel listen HTTP/2 trên port 8080.
- Backend không còn phụ thuộc Frontend để expose API hoặc realtime endpoint.
- Realtime Ingress không rewrite path; request được giữ nguyên khi chuyển vào Backend.
- Realtime Ingress dùng timeout dài và cookie affinity để phù hợp với WebSocket, polling và fallback transport khi Backend chạy nhiều replica.
- Redis adapter/backplane do source ứng dụng xử lý việc truyền event giữa các Pod; Kubernetes Ingress chỉ xử lý routing và session affinity.
- Riêng `file-be` có thêm Ingress tài nguyên file và Ingress upload trực tiếp đến MinIO.
- IP MinIO chỉ dùng nội bộ, không trả về Frontend.
- Frontend upload qua domain `develop.file.sevago.local`.
- App có Ingress thì luôn sử dụng TLS.
- Không dùng biến INGRESS_ENABLED.
- Không dùng biến TLS_ENABLED.
- Không dùng values ingress.enabled.
- Không dùng values ingress.tls.enabled.
- Ingress được quyết định bằng app.type.
- TLS được quyết định bằng ingress.tls.secretName.
- Domain được Jenkins truyền vào thông qua ingress.host.
- replicaCount được Jenkins truyền vào khi deploy.
```

Luồng riêng của `file-be`:

```text
/api/*
  -> file-be
/v1/img/*, /v1/video/*, /v1/docs/*, /v1/files/*, /v1/assets/*
  -> file-be
/api/v1/internal-upload/*
  -> Ingress NGINX
  -> MinIO VPS:9000
```

---

## 4. 📐 Rule App Type

Jenkins xác định `APP_TYPE` từ tên repo:

```text
*-grpc    -> grpc
*-be      -> be
*-admin   -> admin
còn lại   -> fe
```

Ví dụ:

```text
shop-be         -> be
shop-admin      -> admin
shop-fe         -> fe
manufacturing-execution-grpc -> grpc
sso-fe          -> fe
sso-admin       -> admin
landing-page-fe -> fe
```

Ý nghĩa:

```text
app.type = fe
  -> render fe.ingress.yaml
  -> tạo route /
  -> luôn có TLS
app.type = admin
  -> render fe.ingress.yaml
  -> tạo route /
  -> luôn có TLS
app.type = be
  -> render be.ingress.yaml
  -> render realtime.ingress.yaml
  -> tạo route /api
  -> tạo route /socket.io và /hub
  -> luôn có TLS
app.type = grpc
  -> render grpc.ingress.yaml
  -> tạo route /
  -> Ingress kết nối backend bằng gRPC/HTTP2
  -> luôn có TLS
```

---

## 5. 🌐 Rule Domain

### 5.1. 🌐 Frontend Domain

```text
develop.${BASE_NAME}.sevago.local
```

Ví dụ repo `shop-fe`:

```text
develop.shop.sevago.local
```

### 5.2. 🌐 Admin Domain

```text
develop.admin.${BASE_NAME}.sevago.local
```

Ví dụ repo `shop-admin`:

```text
develop.admin.shop.sevago.local
```

### 5.3. 🌐 Backend Domain

Backend không có domain riêng.

Backend sẽ sử dụng: Domain Frontend, Hoặc Domain Admin.

Domain được chọn dựa trên cấu hình:

```text
BE_USE_FE_DOMAIN_BASE_NAMES
```

Nếu `BASE_NAME` nằm trong danh sách trên thì Backend sử dụng domain Frontend.

Ví dụ:

```text
office-be
  -> develop.office.sevago.local/api
```

Nếu `BASE_NAME` không nằm trong danh sách thì Backend sử dụng domain Admin.

Ví dụ:

```text
formula-price-be
  -> develop.admin.formula-price.sevago.local/api
```

### 5.4. 📌 Special Base Name

SSO sử dụng domain base là `account`.

```text
sso-fe develop
  -> develop.account.sevago.local
sso-admin develop
  -> develop.admin.account.sevago.local
sso-be develop
  -> develop.account.sevago.local/api
```

### 5.5. 🌐 gRPC Domain

gRPC sử dụng domain riêng:

```text
develop.${BASE_NAME}-grpc.sevago.local
```

Ví dụ repo `manufacturing-execution-grpc`:

```text
develop.manufacturing-execution-grpc.sevago.local
```

---

## 6. 🔐 Rule TLS

Mọi ứng dụng có Ingress đều sử dụng TLS.

TLS được Jenkins truyền thông qua:

```text
ingress.tls.secretName
```

Rule:

```text
Tên miền nội bộ -> local-tls
```

Ví dụ:

```text
develop.shop.sevago.local
  -> local-tls
  -> local-tls
shop.sevago.local
  -> local-tls
office.sevago.com.vn
  -> public-tls
```

---

## 7. 🌐 Rule Ingress

Helm sử dụng bốn template Ingress chính:

```text
templates/
├── fe.ingress.yaml
├── be.ingress.yaml
├── realtime.ingress.yaml
└── grpc.ingress.yaml
```

### 7.1. 🌐 fe.ingress.yaml

Áp dụng cho:

```text
- Frontend.
- Admin.
```

File này render route:

```text
/
```

Ví dụ:

```text
develop.office.sevago.local
  -> office-fe
```

```text
develop.admin.office.sevago.local
  -> office-admin
```

### 7.2. 🌐 be.ingress.yaml

Chỉ áp dụng cho:

```text
Backend
```

File này render route:

```text
/api
```

Ví dụ Backend sử dụng domain Frontend:

```text
develop.office.sevago.local/api
  -> office-be
```

Ví dụ Backend sử dụng domain Admin:

```text
develop.admin.formula-price.sevago.local/api
  -> formula-price-be
```

Backend sở hữu toàn bộ route `/api` và không còn phụ thuộc Frontend hoặc Admin để expose API.

### 7.3. 🌐 realtime.ingress.yaml

Chỉ áp dụng cho:

```text
Backend
```

Mọi Backend đều render Realtime Ingress mặc định, không cần khai báo thêm values và không cần Jenkins quyết định service nào có realtime.

File này render hai route chuẩn:

```text
/socket.io
  -> NestJS Socket.IO
/hub
  -> .NET SignalR
```

Cả hai route đều chuyển đến chính Kubernetes Service của Backend và không rewrite path.

```text
Client
  -> HTTPS / WSS
  -> Ingress NGINX
  -> Backend Service
  -> Backend Pod:8080
```

Realtime Ingress sử dụng timeout dài và cookie affinity để giữ kết nối realtime ổn định khi Backend chạy nhiều replica.

Service chưa implement một trong hai prefix vẫn được render route tương ứng; khi request đến endpoint không tồn tại thì Backend tự trả về 404.

### 7.4. 🌐 grpc.ingress.yaml

Chỉ áp dụng cho gRPC Service.

File này render route `/` và khai báo:

```yaml
nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
```

Annotation trên yêu cầu Ingress NGINX kết nối đến Service/Pod bằng gRPC trên HTTP/2.

```text
gRPC Client
  -> HTTPS/HTTP2
  -> Ingress NGINX
  -> GRPC/HTTP2 plaintext
  -> Kubernetes Service
  -> .NET gRPC Pod:8080
```

---

## 8. 🧰 Chuẩn bị Jenkins trước khi tạo Job

> **📍 Thực hiện tại:** VPS/Jenkins UI tại `172.17.79.21`.

### 8.1. 🔐 Tạo Harbor Credential

> **📍 Thực hiện tại:** VPS/Jenkins UI tại `172.17.79.21`.

Vào:

```text
Manage Jenkins → Credentials → System → Global Credentials → Add Credentials
```

Chọn:

```text
Kind: Username with password
```

Điền thông tin:

| Field       | Value                                 |
| ----------- | ------------------------------------- |
| Username    | `robot$sevago-develop+sevago-develop` |
| Password    | Harbor Robot Secret                   |
| ID          | `harbor-develop`                      |
| Description | `Harbor Develop Robot Account`        |

Không lưu username hoặc password Harbor trực tiếp trong Webhook hay Shell Script.

### 8.2. 🔐 Gắn Credential vào Freestyle Job

> **📍 Thực hiện tại:** VPS/Jenkins UI tại `172.17.79.21`.

Vào:

```text
Job → Configure → Environment
```

Bật:

```text
Use secret text(s) or file(s)
```

Thêm binding:

```text
Username and password (separated)
```

Điền:

```text
Credentials: harbor-develop
Username Variable: REGISTRY_USER
Password Variable: REGISTRY_SECRET
```

Jenkins sẽ tự inject hai biến này trước khi Shell Script chạy:

```text
REGISTRY_USER
REGISTRY_SECRET
```

Script chỉ cần sử dụng:

```bash
echo "${REGISTRY_SECRET}" | docker login "${REGISTRY_HOST}" -u "${REGISTRY_USER}" --password-stdin
```

### 8.3. 📦 Xóa thông tin Harbor khỏi Webhook

> **📍 Thực hiện tại:** VPS/Jenkins UI tại `172.17.79.21`.

Vào:

```text
Job → Configure → Triggers → Generic Webhook Trigger → Post content parameters
```

Xóa:

```text
REGISTRY_USER
REGISTRY_SECRET
```

Giữ lại các biến webhook phục vụ build:

```text
BRANCH
REPO
COMMIT
REPO_URL
COMMIT_AUTHOR
COMMIT_MESSAGE
```

> Ghi chú: Job trong tài liệu này là **Freestyle Job**, vì vậy không cần dùng `withCredentials(...)` trong script. `withCredentials(...)` chỉ dùng khi triển khai dưới dạng Jenkins Pipeline.

---

## 9. 📐 Rule Replica

Giá trị mặc định:

```text
replicaCount = 2
```

Riêng các module:

```text
webhook
cron-job
```

luôn deploy:

```text
replicaCount = 1
```

Jenkins truyền số lượng Replica vào Helm bằng:

```bash
--set replicaCount=<value>
```

Helm vẫn giữ `replicaCount` trong file values để làm giá trị mặc định khi không có override.

Không chỉnh trực tiếp `replicaCount` trong Helm khi triển khai thực tế nếu Jenkins đã quản lý giá trị này.

---

## 10. 🧰 Jenkins Job

> **📍 Thực hiện tại:** VPS/Jenkins UI tại `172.17.79.21`.

### 10.1. 🧰 10.1 Deploy k8s lỗi làm cho jenkins build lỗi luôn

Không thấy được lỗi tại sao deploy failed - Check config jenkins

### 10.2. 🧰 10.2 Deploy k8s lỗi không làm cho jenkins build lỗi

Thấy được lỗi tại sao deploy failed vì còn giữ pod lỗi

Thay lệnh `helm upgrade` bước 5 ở trên thành lệnh sau đây

Lệnh cũ

```bash
helm upgrade --install "${REPO}" app-helm \
  -n "${NAMESPACE}" -f "${VALUES_FILE}" --set "app.name=${REPO}" --set "app.type=${APP_TYPE}" \
  --set "ingress.host=${INGRESS_HOST}" --set "image.fullName=${IMAGE_FULL_NAME}" \
  --set "ingress.tls.secretName=${TLS_SECRET_NAME}" \
  --set "env.secretName=env-${COMMIT_SHORT}-${BUILD_NUMBER}" \
  "${HELM_EXTRA_ARGS[@]}" --atomic --wait --timeout 7m

```

Lệnh mới

```bash
if ! helm upgrade --install "${REPO}" app-helm \
  -n "${NAMESPACE}" -f "${VALUES_FILE}" --set "app.name=${REPO}" --set "app.type=${APP_TYPE}" \
  --set "ingress.host=${INGRESS_HOST}" --set "image.fullName=${IMAGE_FULL_NAME}" \
  --set "ingress.tls.secretName=${TLS_SECRET_NAME}" \
  --set "env.secretName=env-${COMMIT_SHORT}-${BUILD_NUMBER}" \
  "${HELM_EXTRA_ARGS[@]}" --wait --timeout 7m; then
  echo "ERROR: BUILD IMAGE OK BUT KUBERNETES DEPLOY FAILED"
  echo "FAILED PODS ARE KEPT FOR DEBUG"
  kubectl get pods -n "${NAMESPACE}" -l "app.kubernetes.io/instance=${REPO}" -o wide || true
  kubectl describe deployment "${REPO}" -n "${NAMESPACE}" || true
  kubectl get events -n "${NAMESPACE}" --sort-by='.metadata.creationTimestamp' | tail -50 || true
  exit 1
fi
```

---

## 11. ⛵ Source Helm check repo
