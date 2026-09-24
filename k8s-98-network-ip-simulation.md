# Mô phỏng chi tiết Server & IP — Cluster Develop (sevago-develop)

## 1. Bảng server thực tế

| # | Hostname | Vai trò | IP | OS | Ghi chú |
|---|---|---|---|---|---|
| 1 | `sevago-dev-k8s-master-1` | Control-plane 1 (HAProxy+Keepalived MASTER) | `172.17.79.90` | Ubuntu 24.04 | Chạy kubeadm init đầu tiên, giữ VIP lúc bình thường |
| 2 | `sevago-dev-k8s-master-2` | Control-plane 2 (HAProxy+Keepalived BACKUP prio 100) | `172.17.79.91` | Ubuntu 24.04 | Join bằng `--control-plane` |
| 3 | `sevago-dev-k8s-master-3` | Control-plane 3 (HAProxy+Keepalived BACKUP prio 90) | `172.17.79.92` | Ubuntu 24.04 | Join bằng `--control-plane` |
| 4 | `sevago-dev-k8s-node-1` | Worker 1 | `172.17.79.93` | Ubuntu 24.04 | Chạy Pod app |
| 5 | `sevago-dev-k8s-node-2` | Worker 2 | `172.17.79.94` | Ubuntu 24.04 | Chạy Pod app, có thể là node giữ MetalLB External IP |
| 6 | `sevago-dev-k8s-node-3` | Worker 3 | `172.17.79.95` | Ubuntu 24.04 | Chạy Pod app |
| — | **VIP API Server** | Virtual IP (Keepalived, không phải máy vật lý) | `172.17.79.159` | — | `kube-apiserver` endpoint dùng chung, luôn "nằm" trên 1 trong 3 master |
| — | **MetalLB Pool** | Dải IP ảo (không phải máy vật lý) | `172.17.79.160 – 172.17.79.169` | — | Gán cho Service `LoadBalancer`, do 1 node worker/master quảng bá ARP |
| — | Harbor Registry | Server ngoài cluster | `172.17.79.20` | — | `registry.sevatech.local` |
| — | GitLab | Server ngoài cluster | `172.17.79.10` | — | `gitlab.sevatech.local`, SSH port 2222 |
| — | Jenkins | Server ngoài cluster (VPS riêng, chạy Docker) | `172.17.79.21` | — | Container `jenkins`, có kubeconfig riêng cho 5 cluster |
| — | DNS công ty | Server ngoài cluster | ví dụ `172.17.90.23` | — | CoreDNS forward zone `sevago.local` về đây |
| — | Central Prometheus/Grafana | Server monitor | `172.17.79.23` | — | Nhận remote write từ Prometheus Agent mỗi cluster |

Rancher (quản lý cả 5 cluster) được cài **trên chính cluster Develop** ở namespace `cattle-system`, expose qua Ingress domain `develop.k8s.sevago.local`.

---

## 2. Sơ đồ mạng tổng thể

```
                     Subnet nội bộ K8s: 172.17.79.0/24
   ┌──────────────────────────────────────────────────────────────────┐
   │                                                                  │
   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
   │   │  master-1   │  │  master-2   │  │  master-3   │             │
   │   │ .90         │  │ .91         │  │ .92         │             │
   │   │ HAProxy     │  │ HAProxy     │  │ HAProxy     │             │
   │   │ Keepalived  │  │ Keepalived  │  │ Keepalived  │             │
   │   │ (MASTER)    │  │ (BACKUP)    │  │ (BACKUP)    │             │
   │   │ VIP .159 ●──┼──┼─────────────┼──┼─────────────┤ (VIP di chuyển
   │   └─────────────┘  └─────────────┘  └─────────────┘  khi failover)
   │                                                                  │
   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
   │   │  worker-1   │  │  worker-2   │  │  worker-3   │             │
   │   │ .93         │  │ .94 ◄─ giữ  │  │ .95         │             │
   │   │             │  │ MetalLB IP  │  │             │             │
   │   │             │  │ .160 (ARP)  │  │             │             │
   │   └─────────────┘  └─────────────┘  └─────────────┘             │
   │                                                                  │
   └──────────────────────────────────────────────────────────────────┘
              │                    │                    │
              ▼                    ▼                    ▼
     Harbor .20          GitLab .10           Jenkins .21
     (registry)          (SSH :2222)          (Docker container)
```

---

## 3. Mô phỏng luồng 1 — Admin dùng `kubectl` (luồng quản trị)

Kịch bản: bạn ở máy laptop, gõ `kubectl get nodes`, kubeconfig của bạn có `server: https://172.17.79.159:6443`.

```
Laptop bạn (ví dụ IP 172.17.79.50)
   │  kubectl get nodes
   │  gửi request tới https://172.17.79.159:6443
   ▼
VIP 172.17.79.159:6443   <-- Keepalived đang gán VIP này cho master-1 (172.17.79.90)
   │
   ▼
master-1 (172.17.79.90) — HAProxy đang lắng nghe port 6443
   │  HAProxy health-check backend: .90:6444 UP, .91:6444 UP, .92:6444 UP
   │  round-robin chọn 1 backend, ví dụ chọn .92:6444
   ▼
master-3 (172.17.79.92) — kube-apiserver (bind port 6444)
   │  xác thực client cert của bạn, xử lý request
   │  đọc/ghi etcd (etcd cũng chạy trên .90/.91/.92)
   ▼
Trả kết quả ngược lại theo đúng đường đã đi: master-3 → HAProxy trên master-1 → VIP → laptop bạn
```

Nếu `master-1` (đang giữ VIP) sập:
```
master-1 (.90) mất kết nối
   │
   ▼
Keepalived trên master-2 (.91, priority 100) phát hiện master-1 không còn gửi VRRP advertisement
   │
   ▼
master-2 tự gán VIP 172.17.79.159 vào interface của chính nó
   │
   ▼
Request tiếp theo tới 172.17.79.159:6443 giờ chạm vào HAProxy trên master-2
   │  HAProxy trên .91 vẫn có đủ danh sách backend .90/.91/.92 (nhưng .90 giờ DOWN)
   ▼
Route sang .91:6444 hoặc .92:6444 (còn sống) — kubectl của bạn không nhận ra gì đã đổi
```

---

## 4. Mô phỏng luồng 2 — Client gọi app qua domain (luồng ứng dụng)

Kịch bản: user mở browser gõ `https://develop.office.sevago.local`

```
Browser user (mạng công ty, ví dụ 172.17.90.100)
   │  DNS query "develop.office.sevago.local"
   ▼
DNS công ty (172.17.90.23) — có record trỏ develop.office.sevago.local -> 172.17.79.160
   │  trả về IP 172.17.79.160
   ▼
Browser kết nối HTTPS tới 172.17.79.160:443
   │
   ▼
172.17.79.160 là 1 IP trong MetalLB Pool (172.17.79.160-169)
   │  MetalLB đã gán/quảng bá ARP cho IP này TRÊN worker-2 (172.17.79.94)
   ▼
worker-2 (172.17.79.94) nhận packet tới .160:443
   │  kernel route vào Service "ingress-nginx-controller" (type LoadBalancer, ClusterIP nội bộ ví dụ 10.96.5.20)
   ▼
kube-proxy trên worker-2 forward tới 1 Pod ingress-nginx-controller
   (Pod này có thể thực tế đang chạy trên worker-1, worker-2, hoặc worker-3 — Service tự loadbalance)
   │  giả sử chọn Pod đang chạy trên worker-1 (172.17.79.93), Pod IP overlay ví dụ 192.168.35.10
   ▼
Ingress NGINX Controller Pod (192.168.35.10) đọc Ingress Rule:
   host: develop.office.sevago.local
     /      -> Service "office-fe-svc"  (ClusterIP 10.96.10.5)
     /api   -> Service "office-be-svc"  (ClusterIP 10.96.10.8)
   │  browser gọi "/" -> route tới office-fe-svc
   ▼
Service office-fe-svc (10.96.10.5) loadbalance tới 1 trong các Pod office-fe:
   - office-fe-7d9f8-abcde  chạy trên worker-2 (Pod IP 192.168.40.11)
   - office-fe-7d9f8-xyz12  chạy trên worker-3 (Pod IP 192.168.45.22)
   │  giả sử chọn Pod trên worker-3
   ▼
Pod office-fe (192.168.45.22) trên worker-3 (172.17.79.95) xử lý request, trả HTML về
   │
   ▼
Đường về: Pod -> Service -> Ingress Pod -> Service ingress-nginx -> worker-2 (giữ IP .160) -> DNS-resolved IP -> browser user
```

Khi frontend gọi tiếp API (`/api/orders`):
```
Browser -> https://develop.office.sevago.local/api/orders
   (đường đi giống hệt trên, nhưng Ingress route theo path /api)
   ▼
Ingress NGINX -> Service "office-be-svc" (10.96.10.8)
   ▼
Pod office-be-1 (192.168.40.30, trên worker-2)
   hoặc office-be-2 (192.168.45.31, trên worker-3) — Service loadbalance ngẫu nhiên
```

---

## 5. Mô phỏng luồng 3 — Backend gọi Backend (không qua Ingress/MetalLB)

Kịch bản: `office-be` cần gọi `sso-be` để verify token.

```
Pod office-be (192.168.40.30, trên worker-2)
   │  code gọi thẳng: http://sso-be:8080/verify
   │  (sso-be là 1 Kubernetes Service ClusterIP, ví dụ 10.96.20.15)
   ▼
CoreDNS trong cluster resolve "sso-be" -> ClusterIP 10.96.20.15 (không ra ngoài, không qua .160, không qua Ingress)
   ▼
kube-proxy route tới 1 Pod sso-be, ví dụ Pod sso-be-1 (192.168.35.40, trên worker-1)
   ▼
Trả kết quả trực tiếp về Pod office-be — toàn bộ nằm trong overlay network Calico (192.168.0.0/16), không đi qua 172.17.79.160
```

---

## 6. Mô phỏng luồng 4 — Pull image từ Harbor khi deploy

Kịch bản: Jenkins deploy app mới trên cluster Develop, kubelet trên worker-3 cần pull image.

```
Jenkins (172.17.79.21) chạy job:
   helm upgrade --install office-be ... --set image.fullName=registry.sevatech.local/sevago-develop/office-be:develop-abc123
   │  dùng kubeconfig "config-develop" -> server 172.17.79.159:6443 (VIP)
   ▼
kube-apiserver (qua VIP, như luồng 1) nhận Deployment mới, scheduler chọn worker-3 (172.17.79.95) để chạy Pod
   ▼
kubelet trên worker-3 thấy cần image "registry.sevatech.local/sevago-develop/office-be:develop-abc123"
   │  containerd trên worker-3 (đã có ca.crt + hosts.toml từ Bước 9 trong guide) resolve DNS
   ▼
DNS: registry.sevatech.local -> 172.17.79.20 (từ /etc/hosts hoặc DNS công ty)
   ▼
worker-3 (172.17.79.95) kết nối HTTPS tới Harbor (172.17.79.20:443)
   │  xác thực bằng imagePullSecret "registry-cred" (Harbor Robot Account) đã gắn vào ServiceAccount
   ▼
Harbor trả về image layers -> containerd lưu local trên worker-3 -> kubelet start container
```

---

## Tổng kết vai trò từng IP trong 1 bảng nhìn nhanh

| IP / Range | Là gì | Vai trò trong luồng traffic |
|---|---|---|
| `172.17.79.90-92` | 3 master vật lý | Chạy kube-apiserver, etcd, HAProxy, Keepalived |
| `172.17.79.93-95` | 3 worker vật lý | Chạy Pod app, Ingress Controller, có thể giữ MetalLB IP |
| `172.17.79.159` | VIP (ảo, do Keepalived) | Endpoint duy nhất cho API server, luôn "nhảy" sang master còn sống |
| `172.17.79.160-169` | MetalLB Pool (ảo, ARP trên 1 node) | External IP cho Service LoadBalancer (Ingress NGINX) |
| `10.96.x.x` | ClusterIP (nội bộ K8s, không route được từ ngoài) | Service DNS, dùng cho cả traffic từ Ingress vào Pod và Backend-to-Backend |
| `192.168.x.x` | Pod overlay IP (Calico) | IP thật của từng Pod, thay đổi mỗi khi Pod bị tạo lại |
| `172.17.79.10/20/21` | GitLab/Harbor/Jenkins | Hạ tầng CI/CD dùng chung ngoài cluster |

> Lưu ý: IP overlay Pod (`192.168.x.x`) và ClusterIP (`10.96.x.x`) trong các luồng trên là ví dụ minh họa hợp lý theo dải CIDR đã cấu hình (`--pod-network-cidr=192.168.0.0/16`), không phải giá trị cố định trong tài liệu gốc — trên cluster thật các IP này do Calico/kube-apiserver cấp động mỗi lần Pod được tạo.
