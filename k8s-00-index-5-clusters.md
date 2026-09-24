# ☸️ Kubernetes - Index và quy ước 5 cluster

## 1. 🎯 Mục đích

> **📍 Thực hiện tại:** Không chạy lệnh — file điều hướng và quy ước chung.

File này là điểm bắt đầu của toàn bộ bộ tài liệu Kubernetes. Mục tiêu là ngăn nhầm cluster, nhầm namespace, nhầm kubeconfig hoặc sao chép IP của Develop sang Staging/Production.

---

## 2. 🏗️ Ma trận 5 cluster

> **📍 Thực hiện tại:** Không chạy lệnh — dùng để đối chiếu trước mọi thao tác.

| Môi trường | Cluster ID | Vùng | Namespace app chính | Kubeconfig Jenkins |
| --- | --- | --- | --- | --- |
| Develop | `sevago-develop` | Local/default | `sevago-develop` | `config-develop` |
| Staging | `sevago-staging-local` | Local | `sevago-staging` | `config-staging-local` |
| Staging | `sevago-staging-dmz` | DMZ | `sevago-staging` | `config-staging-dmz` |
| Production | `sevago-production-local` | Local | `sevago-production` | `config-production-local` |
| Production | `sevago-production-dmz` | DMZ | `sevago-production` | `config-production-dmz` |

> Archive hiện có IP/VIP/MetalLB pool chi tiết của Develop. Với Staging/Production phải lấy IP, hostname, API VIP, subnet, MetalLB pool và DNS từ inventory hạ tầng tương ứng; **không suy đoán hoặc tái sử dụng IP Develop**.

---

## 3. 🧭 Quy ước nơi thực hiện

> **📍 Thực hiện tại:** Không chạy lệnh — đọc trước khi dùng các file bước 01–21.

Mỗi mục trong bộ tài liệu đều có dòng `📍 Thực hiện tại`. Quy ước chung:

```text
Control-plane-1 / máy có kubeconfig
  -> kubectl, Helm, tạo resource, health check cluster

Cả 3 control-plane
  -> HAProxy, Keepalived, VIP, các bước HA bắt buộc trên từng master

Tất cả worker
  -> sysctl/kernel/runtime hoặc kiểm tra node-level

VPS Jenkins 172.17.79.21
  -> Jenkins image, Jenkins credential/job, kubeconfig đặt trong Jenkins

Monitor Server 172.17.79.23
  -> Central Prometheus/Grafana query trong tài liệu observability

Máy quản trị/client
  -> Browser Rancher, curl/openssl/domain test khi yêu cầu đường mạng người dùng
```

---

## 4. 🏷️ Baseline công nghệ

> **📍 Thực hiện tại:** Không chạy lệnh — dùng để pin/đối chiếu version.

Baseline đã kiểm chứng tại `2026-09-03`:

```text
Kubernetes v1.36.x (patch hiện hành v1.36.4)
Calico v3.32.1
MetalLB v0.16.1
Rancher v2.15.0 stable
```

Kubernetes `v1.37.0` đã tồn tại, nhưng Calico v3.32 công bố test chính thức đến Kubernetes 1.36 nên bộ docs giữ v1.36 làm baseline. Ingress NGINX đã retired từ tháng 3/2026; chỉ duy trì cho kiến trúc hiện hữu và phải có kế hoạch migration.

---

## 5. 📚 Thứ tự tài liệu

> **📍 Thực hiện tại:** Không chạy lệnh — dùng làm roadmap.

```text
01  Tổng quan kiến trúc
02  Common OS/runtime/Kubernetes packages
03  HA Control Plane + Calico
04  Join worker + firewall/test
05  MetalLB
06  Ingress NGINX legacy + migration note
07  Rancher
08  Rancher account/RBAC UI
09  Rancher Project/Namespace
10  Harbor CA + containerd
11  Private image pull
12  CoreDNS nội bộ
13  Cluster health check
14  Jenkins kubectl/Helm + 5 kubeconfig
15  TLS Secret
16  Jenkins CI/CD + Helm
17  Jenkins Manual Job Develop
18  API Gateway/Gateway API options
19  Traffic flow
20  Worker inotify limit
21  Beyla application observability
```

---

## 6. 🔐 Quy tắc an toàn khi lưu trữ tài liệu

> **📍 Thực hiện tại:** Không chạy lệnh — áp dụng cho Git/File Server nơi lưu bộ docs.

Không lưu password, token, private key hoặc Harbor/Rancher secret thật trong Markdown. Tài liệu chỉ dùng placeholder hoặc nhập secret tương tác tại terminal. Trước khi commit, quét nhanh các file để phát hiện credential vô tình bị paste.

---

## 7. 🌐 Lifecycle Ingress NGINX

> **📍 Thực hiện tại:** Không chạy lệnh — quyết định kiến trúc.

`ingress-nginx` upstream đã nghỉ bảo trì. Sơ đồ/lệnh Ingress NGINX trong các bước 01, 06, 16, 18, 19 mô tả **đường traffic hiện hữu** chứ không phải lựa chọn khuyến nghị cho cluster mới. Hướng migration nên đánh giá Gateway API và controller còn được support, sau đó chuyển từng domain/path với rollback plan rõ ràng.

---

## 8. 📚 Tham chiếu kỹ thuật

> **📍 Thực hiện tại:** Không chạy lệnh — nguồn kiểm chứng version/lifecycle.

```text
https://kubernetes.io/releases/1.36/
https://kubernetes.io/releases/1.37/
https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements
https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/
https://github.com/rancher/rancher/releases/tag/v2.15.0
```
