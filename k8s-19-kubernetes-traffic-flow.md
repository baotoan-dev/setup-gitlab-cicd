# ☸️ Bước 19 - Luồng Traffic Kubernetes

## 1. 🎯 Mục đích

File này giải thích đường đi của traffic quản trị, traffic ứng dụng và service-to-service để vận hành/troubleshoot nhất quán trên 5 cluster.

Tài liệu này mô tả luồng quản trị và luồng ứng dụng của kiến trúc hiện hữu trên 5 cluster. Các IP/domain minh họa cụ thể là Develop. Ingress NGINX trong sơ đồ là thành phần legacy đang vận hành; đích migration là Gateway API/controller còn support.

---

## 2. ✅ Checklist nội dung

- [ ] Luồng quản trị Kubernetes
- [ ] Luồng truy cập ứng dụng
- [ ] Vai trò của MetalLB
- [ ] Node nhận traffic
- [ ] Vai trò của Ingress NGINX
- [ ] Kubernetes Service
- [ ] Backend gọi Backend
- [ ] Luồng tổng thể

---

## 3. 🔀 Luồng quản trị Kubernetes

Luồng này chỉ phục vụ quản trị cluster (kubectl, Jenkins, Rancher...), **không phải luồng truy cập ứng dụng**.

```text
Admin / Jenkins / kubectl
          │
          ▼
Kubernetes API VIP (172.17.79.159:6443)
          │
          ▼
HAProxy + Keepalived
          │
          ▼
kube-apiserver
(master-1 / master-2 / master-3)
          │
          ▼
Scheduler / Controller Manager / etcd
```

---

## 4. 🔀 Luồng truy cập ứng dụng

Người dùng truy cập ứng dụng thông qua domain.

Ví dụ:

```text
https://develop.office.sevago.local
```

DNS trả về:

```text
172.17.79.160
```

Đây là External IP của Service `LoadBalancer` do MetalLB cấp.

Luồng xử lý:

```text
Client
    │
    ▼
DNS
    │
    ▼
172.17.79.160
(MetalLB External IP)
    │
    ▼
Node đang sở hữu IP này
(master hoặc worker tùy thời điểm)
    │
    ▼
Service ingress-nginx
(Type: LoadBalancer)
    │
    ▼
Ingress NGINX Controller Pod
    │
    ▼
Ingress Rule
(Host + Path)
    │
    ▼
Kubernetes Service (ClusterIP)
    │
    ▼
Application Pod
```

---

## 5. 🌐 Vai trò của MetalLB

MetalLB chỉ có hai nhiệm vụ:

- Cấp External IP cho Service `LoadBalancer`.
- Quảng bá IP đó ra mạng (Layer2 hoặc BGP).

Ví dụ:

```text
MetalLB Pool

172.17.79.160
...
172.17.79.169
```

Ingress NGINX được cấp:

```text
172.17.79.160
```

Client chỉ truy cập IP này.

MetalLB **không thực hiện**: Reverse Proxy, HTTP Routing, TLS Termination, Load balancing giữa Pod.

Các nhiệm vụ trên do Ingress NGINX và Kubernetes đảm nhiệm.

---

## 6. 🔀 Node nhận traffic

Trong Layer2 mode, MetalLB sẽ chọn một node để quảng bá External IP.

Ví dụ:

```text
master-1
master-2
master-3

node-1
node-2  ← đang sở hữu IP
node-3
```

Request sẽ đi vào:

```text
Client
    │
    ▼
172.17.79.160
    │
    ▼
node-2
```

Nếu node-2 gặp sự cố:

```text
Client
    │
    ▼
172.17.79.160
    │
    ▼
node-3
```

MetalLB sẽ tự động chuyển quyền sở hữu IP.

---

## 7. 🌐 Vai trò của Ingress NGINX

Ingress Controller đọc Host và Path để quyết định Service đích.

Ví dụ:

```text
develop.office.sevago.local
        │
        ├── /
        │      ▼
        │   office-fe Service
        │
        └── /api
               ▼
          office-be Service
```

Hay:

```text
develop.account.sevago.local
        │
        ├── /
        │      ▼
        │   sso-fe Service
        │
        └── /api
               ▼
           sso-be Service
```

---

## 8. 📌 Kubernetes Service

Sau khi Ingress chọn được Service, Kubernetes sẽ tự cân bằng tải giữa các Pod.

Ví dụ:

```text
office-be Service
        │
        ├────────────┐
        │            │
        ▼            ▼
office-be-1     office-be-2
        │            │
        └──────┬─────┘
               ▼
         office-be-3
```

Kubernetes sẽ tự chọn Pod phù hợp.

---

## 9. 📌 Backend gọi Backend

Backend trong cluster không cần đi qua MetalLB hoặc Ingress.

Luồng chuẩn:

```text
sso-be Pod
      │
      ▼
office-be Service
      │
      ▼
office-be Pod
```

Ví dụ:

```text
http://office-be
```

hoặc

```text
http://office-be:8080
```

---

## 10. 🔀 Luồng tổng thể

```text
                          Client
                             │
                             ▼
                 develop.office.sevago.local
                             │
                             ▼
                            DNS
                             │
                             ▼
                172.17.79.160 (MetalLB)
                             │
                             ▼
             Node đang quảng bá External IP
                 (master hoặc worker)
                             │
                             ▼
     Service ingress-nginx (Type: LoadBalancer)
                             │
                             ▼
           Ingress NGINX Controller Pod
                             │
             ┌───────────────┴───────────────┐
             ▼                               ▼
      Host=/                          Host=/api
             │                               │
             ▼                               ▼
     office-fe Service               office-be Service
             │                               │
             ▼                               ▼
     Kubernetes Service Load Balancing
             │                               │
             ▼                               ▼
        office-fe Pods                  office-be Pods
```
