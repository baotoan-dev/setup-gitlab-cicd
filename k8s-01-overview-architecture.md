# ☸️ Bước 01 - Tổng quan kiến trúc Kubernetes self-managed

## 1. 🎯 Mục đích

Bộ tài liệu này là **baseline triển khai và vận hành Kubernetes self-managed cho toàn bộ 5 cluster** của hệ thống Sevago. Các bước 01–13 dùng cụm Develop làm ví dụ chi tiết có IP/hostname cụ thể; khi áp dụng cho Staging/Production phải thay bằng inventory của đúng cluster, không sao chép IP Develop sang môi trường khác.

| Môi trường | Cluster ID chuẩn          | Vùng          | Namespace ứng dụng chính |
| ---------- | ------------------------- | ------------- | ------------------------ |
| Develop    | `sevago-develop`          | Local/default | `sevago-develop`         |
| Staging    | `sevago-staging-local`    | Local         | `sevago-staging`         |
| Staging    | `sevago-staging-dmz`      | DMZ           | `sevago-staging`         |
| Production | `sevago-production-local` | Local         | `sevago-production`      |
| Production | `sevago-production-dmz`   | DMZ           | `sevago-production`      |

Mục tiêu chính:

- Chạy workload stateless trong Kubernetes và chuẩn hóa cách triển khai giữa 5 cluster.
- Giữ database, Redis, Kafka, Elasticsearch/OpenSearch, object/file storage và hạ tầng CI/CD ở ngoài Kubernetes theo kiến trúc hiện tại.
- Dùng HAProxy + Keepalived cho Kubernetes API VIP và MetalLB cho Service `LoadBalancer` trên hạ tầng on-prem/private cloud.
- Dùng Harbor Registry nội bộ và Jenkins/Helm cho luồng build/deploy.
- Dùng Rancher làm giao diện quản trị cluster nội bộ.
- Chuẩn hóa firewall, TLS, DNS, health check, RBAC và observability.

> **Lưu ý lifecycle 2026-09-03:** Ingress NGINX trong bộ tài liệu là thành phần **legacy để duy trì/tái lập kiến trúc đang chạy**. Upstream ingress-nginx đã dừng bảo trì từ tháng 3/2026; cluster mới nên ưu tiên Gateway API hoặc một ingress/gateway controller còn được hỗ trợ.

---

## 2. ✅ Checklist nội dung

```text
[ ] Kubernetes minor còn support
[ ] 3 master Ready
[ ] 3 worker Ready
[ ] API VIP hoạt động qua HAProxy + Keepalived
[ ] etcd health OK trên 3 master
[ ] Calico Running và Pod network OK
[ ] CoreDNS resolve Kubernetes service OK
[ ] DNS nội bộ cho Pod được xử lý bằng DNS thật hoặc CoreDNS customization
[ ] MetalLB cấp External IP cho Service LoadBalancer
[ ] Ingress NGINX hiện hữu nhận External IP và route HTTP/HTTPS OK
[ ] Có kế hoạch migration khỏi ingress-nginx đã retired
[ ] Chỉ public 80/443
[ ] Không expose app public bằng NodePort
[ ] Harbor CA/containerd OK trên tất cả node
[ ] imagePullSecret và ServiceAccount pull image OK
[ ] Jenkins dùng kubeconfig/RBAC phù hợp, không dùng admin lâu dài
[ ] Rancher chỉ truy cập qua mạng quản trị/VPN hoặc phạm vi nội bộ phù hợp
[ ] TLS Secret đúng namespace với Ingress
[ ] App có request/limit, probe, secret/configmap
[ ] NetworkPolicy được lên kế hoạch hoặc áp dụng dần
```

---

## 3. 🏷️ Version baseline đã kiểm chứng

Baseline dùng xuyên suốt bộ tài liệu tại ngày **2026-09-03**:

```text
Kubernetes: v1.36.x (patch hiện hành: v1.36.4)
Calico:     v3.32.1
MetalLB:    v0.16.1
Rancher:    v2.15.0 stable
```

Kubernetes `v1.37.0` đã phát hành, nhưng Calico `v3.32` hiện công bố test chính thức với Kubernetes `1.34`, `1.35`, `1.36`. Vì vậy bộ tài liệu giữ `v1.36.x` làm **validated baseline**, không coi đây là “latest version”.

Nguyên tắc:

- `kubeadm`, `kubelet`, `kubectl` nên cùng minor version; `kubectl` tuân thủ version-skew policy của Kubernetes.
- Khi cài mới, chọn patch mới nhất trong minor đã chốt sau khi test compatibility.
- Add-on phải pin version và kiểm tra compatibility trước khi rollout lên cả 5 cluster.
- Không pin một phiên bản alpha nếu đã có bản stable tương thích.
- Không dùng ingress-nginx làm lựa chọn mặc định cho cluster mới vì upstream đã retired.

Biến mẫu:

```bash
export K8S_MINOR=v1.36
export CALICO_VERSION=v3.32.1
export METALLB_VERSION=v0.16.1
export RANCHER_VERSION=2.15.0
```

Tham chiếu kiểm tra version/lifecycle:

```text
https://kubernetes.io/releases/1.36/
https://kubernetes.io/releases/1.37/
https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements
https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/
https://github.com/rancher/rancher/releases/tag/v2.15.0
```

---

## 4. 🏗️ Mô hình hạ tầng

Xem file sevago-system.xlsx để biết thêm chi tiết

### 4.1. 🏗️ Hạ tầng CI/CD

```text
GITLAB              172.17.79.10  gitlab.sevatech.local
JENKINS / BUILD     172.17.79.21  build.sevatech.local
HARBOR REGISTRY     172.17.79.20  registry.sevatech.local
```

### 4.2. 🏗️ Hạ tầng K8S develop

```text
MASTER / NODE
172.17.79.90  sevago-dev-k8s-master-1
172.17.79.91  sevago-dev-k8s-master-2
172.17.79.92  sevago-dev-k8s-master-3
172.17.79.93  sevago-dev-k8s-node-1
172.17.79.94  sevago-dev-k8s-node-2
172.17.79.95  sevago-dev-k8s-node-3

METALLB IP POOL 172.17.79.160-172.17.79.169
K8S API VIP 172.17.79.159  sevago-dev-k8s-api-vip
```

---

## 5. 🔀 Sơ đồ traffic tổng thể

```text
Admin / Jenkins / kubectl
  -> K8S API VIP 172.17.79.159:6443
  -> HAProxy + Keepalived trên 3 master
  -> kube-apiserver master-1/master-2/master-3

Internet / Client / Internal Client
  -> Firewall / Public Reverse Proxy nếu có
  -> MetalLB External IP 172.17.79.160-172.17.79.169
  -> Service LoadBalancer của Ingress NGINX
  -> Ingress NGINX Controller Pod
  -> Kubernetes Service
  -> Pod FE / Admin / Backend

Pod trong Kubernetes
  -> Database ngoài Kubernetes
  -> Redis ngoài Kubernetes
  -> Kafka ngoài Kubernetes
  -> Elasticsearch/OpenSearch ngoài Kubernetes
  -> File/Object Storage ngoài Kubernetes
  -> Harbor Registry khi pull image qua node/containerd
```

---

## 6. 📌 Thành phần chính

| Thành phần               | Cài ở đâu       | Chức năng                                                  |
| ------------------------ | --------------- | ---------------------------------------------------------- |
| Kubernetes control plane | 3 master        | API Server, scheduler, controller-manager, etcd            |
| Worker node              | 3 worker        | Chạy workload ứng dụng                                     |
| containerd               | Master + worker | Container runtime                                          |
| Calico                   | Trong cluster   | Pod networking + NetworkPolicy                             |
| HAProxy + Keepalived     | Trên 3 master   | VIP `172.17.79.159` cho Kubernetes API                     |
| MetalLB                  | Trong cluster   | Cấp External IP cho Service `LoadBalancer`                 |
| Ingress NGINX (legacy)   | Trong cluster   | HTTP/HTTPS entrypoint hiện hữu; cần kế hoạch migration     |
| Harbor Registry          | Ngoài cluster   | Lưu image private                                          |
| Jenkins                  | Ngoài cluster   | Build, push image, Helm deploy                             |
| Rancher                  | Trong cluster   | Giao diện quản trị nội bộ; không public trực tiếp Internet |

---

## 7. 🌐 Ingress NGINX, Gateway API và API Gateway

Trong bộ tài liệu này:

```text
Ingress NGINX = HTTP/HTTPS ingress controller
```

Ingress NGINX xử lý tốt:

- Domain-based routing.
- Path-based routing.
- TLS termination.
- HTTP load balancing.
- Timeout/body-size cơ bản.
- Một số annotation ingress-level.

Ingress NGINX **không thay thế hoàn toàn API Gateway** nếu hệ thống cần: API key management, OAuth/OIDC plugin tập trung, Rate limit/quota theo user/client/app.

- Request/response transformation nâng cao.
- API analytics/developer portal.
- Versioning API và policy tập trung.

Trạng thái kiến trúc hiện tại và hướng đi:

```text
Hiện hữu:
Client -> Ingress NGINX (legacy) -> Backend

Mục tiêu migration:
Client -> Gateway API implementation còn support -> Service backend

Nếu cần API management nâng cao:
Client -> API Gateway -> Service backend
```

Không nên thêm API Gateway chỉ để verify JWT nếu backend đã có middleware chuẩn và nhu cầu còn đơn giản.

---

## 8. 📌 Thành phần nên chạy trong Kubernetes

```text
frontend
admin
backend stateless
webhook stateless
business cronjob
worker stateless
BFF/public API
```

Lý do: Dễ deploy, rollback và scale, Có health check, Có rolling update, Dùng service discovery nội bộ, Dễ quản lý version image.

---

## 9. 📌 Thành phần nên để ngoài Kubernetes

```text
database
Redis
Kafka
Elasticsearch/OpenSearch / ELK
file/object storage
GitLab
Harbor Registry
Jenkins/build server
backup storage
monitoring
```

Lý do:

- Đây là các thành phần stateful hoặc critical infrastructure.
- Cần backup/restore/HA/storage rõ ràng.
- Nên đưa vào Kubernetes sau khi team đã vận hành cluster ổn định.

---

## 10. 🌐 Domain khuyến nghị

> Hệ thống hiện tại tiếp tục sử dụng `*.sevago.local`. Máy quản trị và máy người dùng phải sử dụng DNS nội bộ công ty hoặc resolver phù hợp. Nếu xây dựng lại DNS trong tương lai, nên cân nhắc chuyển sang subdomain của domain thật để tránh xung đột với mDNS `.local`.

Không nên expose mọi backend service thành public domain riêng.

Khuyến nghị public domain gọn:

```text
app.example.com       -> frontend
admin.example.com     -> admin
sso.example.com       -> SSO/Auth
api.example.com       -> public API hoặc BFF
```

Backend nội bộ dùng Kubernetes DNS:

```text
order-service.dev.svc.cluster.local
product-service.dev.svc.cluster.local
user-service.dev.svc.cluster.local
```

Service ngoài Kubernetes nên dùng DNS nội bộ thật:

```text
develop.redis.sevago.local
develop.kafka.sevago.local
registry.sevatech.local
```

> Không nên phụ thuộc lâu dài vào `/etc/hosts` khi số lượng domain tăng. Nếu chưa có DNS nội bộ, có thể dùng CoreDNS hosts plugin tạm thời trong cluster.

---

## 11. 🧱 Nguyên tắc firewall tổng quát

Chỉ public ra Internet khi có yêu cầu:

```text
80/tcp
443/tcp
```

Không public rộng:

```text
6443/tcp Kubernetes API
10250/tcp kubelet
2379-2380/tcp etcd
NodePort 30000-32767/tcp
Rancher UI quản trị trực tiếp từ Internet
```

Các port nội bộ cần thiết phải allow theo subnet cluster/private network:

```text
6443/tcp  API VIP cho node, admin, Jenkins
6444/tcp  kube-apiserver backend port giữa HAProxy và master nếu dùng mô hình 6444
2379-2380/tcp  etcd giữa các master
10250/tcp  kubelet từ control plane tới node
VRRP protocol 112 giữa các master nếu firewall chặn Keepalived
Calico traffic theo mode: IPIP/VXLAN/BGP
```

---

## 12. 🚀 Thứ tự triển khai

```text
1. Chuẩn bị master/worker và thông tin IP/domain.
2. Cài common package: containerd, kubeadm, kubelet, kubectl.
3. Cài HAProxy + Keepalived cho Kubernetes API VIP.
4. kubeadm init master-1 với control-plane-endpoint.
5. Cài Calico.
6. Join master-2/master-3 vào control plane.
7. Join worker node.
8. Cấu hình firewall nội bộ đầy đủ.
9. Cài MetalLB.
10. Với cluster hiện hữu: duy trì/tái lập Ingress NGINX trong thời gian migration; cluster mới chọn controller/Gateway API còn support.
11. Cấu hình DNS nội bộ cho node và Pod.
12. Cấu hình Harbor CA/containerd.
13. Tạo imagePullSecret/ServiceAccount.
14. Cài Rancher và giới hạn truy cập qua mạng quản trị/VPN/phạm vi nội bộ.
15. Cấu hình TLS Secret cho Ingress.
16. Cấu hình Jenkins kubectl/Helm với RBAC riêng.
17. Chạy health check tổng thể.
18. Deploy app thật bằng Helm/CI/CD.
```
