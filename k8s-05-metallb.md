# ☸️ Bước 05 - Cài đặt MetalLB

## 1. 🎯 Mục đích

File này cài và nghiệm thu MetalLB cho cluster self-managed. Mỗi cluster phải có IP pool riêng theo inventory mạng; pool Develop chỉ là ví dụ cụ thể.

MetalLB dùng cho Kubernetes self-managed trên VPS/bare-metal/private cloud khi không có cloud load balancer.

Trong bộ tài liệu này, MetalLB chủ yếu cấp External IP cho Service `LoadBalancer` của Ingress NGINX.

```text
Client
  -> MetalLB External IP
  -> Service LoadBalancer
  -> Ingress NGINX Controller
  -> Service app
  -> Pod app
```

---

## 2. ✅ Checklist nội dung

```text
[ ] Node Ready
[ ] MetalLB controller Running
[ ] MetalLB speaker Running
[ ] IP pool không trùng IP đang dùng
[ ] IP pool reachable từ client/reverse proxy
[ ] IPAddressPool đã tạo
[ ] L2Advertisement đã tạo
[ ] Service LoadBalancer nhận External IP
[ ] Curl External IP thành công
[ ] Namespace test đã dọn
```

---

## 3. 🏗️ Điều kiện hạ tầng quan trọng

Với MetalLB Layer 2, IP pool phải là IP mà client/reverse proxy bên ngoài route tới được.

Ví dụ:

```text
Node subnet: 172.17.79.0/24
MetalLB pool: 172.17.79.160-172.17.79.169
```

Không được trùng:

- IP master.
- IP worker.
- API VIP.
- Gateway.
- DHCP pool.
- IP server khác.

> Trên một số VPS provider, MetalLB Layer 2 có thể không hoạt động nếu provider không cho ARP/NDP cho IP phụ. Khi đó cần routed IP, BGP, provider load balancer, hoặc public reverse proxy phía trước.

---

## 4. 📌 Mục tiêu

- Cài MetalLB.
- Tạo `IPAddressPool`.
- Tạo `L2Advertisement`.
- Test Service `LoadBalancer` nhận External IP.
- Curl được External IP.
- Dọn resource test.

---

## 5. ⚙️ Kiểm tra cluster trước khi cài

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

Tất cả node phải `Ready`, CoreDNS/Calico Running.

---

## 6. 🌐 Cài MetalLB

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Dùng version đã pin sau khi kiểm tra release note:

```bash
export METALLB_VERSION=v0.16.1
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/${METALLB_VERSION}/config/manifests/metallb-native.yaml
```

Test namespace:

```bash
kubectl get namespace metallb-system
```

Đợi Pod Running:

```bash
kubectl get pods -n metallb-system -w
```

Kết quả đúng:

```text
controller   Running
speaker      Running
```

Số lượng speaker thường tương ứng với số node mà DaemonSet schedule được.

---

## 7. ➕ Tạo IPAddressPool và L2Advertisement

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
cat <<EOF > metallb-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 172.17.79.160-172.17.79.169
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
EOF
kubectl apply -f metallb-pool.yaml
```

Test:

```bash
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

---

## 8. 🔎 Test Service LoadBalancer

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig để tạo resource; test truy cập từ máy client có route tới MetalLB pool.

Dùng namespace test riêng:

```bash
kubectl create namespace metallb-test
kubectl create deployment test-nginx --image=nginx:stable -n metallb-test
kubectl expose deployment test-nginx --port=80 --type=LoadBalancer -n metallb-test
kubectl get svc test-nginx -n metallb-test -w
```

Kết quả đúng:

```text
TYPE           LoadBalancer
EXTERNAL-IP    172.17.79.160 hoặc IP trong pool
```

Curl từ: master-1, master-2, master-3, worker.

- hoặc máy nội bộ có route tới subnet `172.17.79.0/24`

```bash
curl http://172.17.79.160
```

Thay bằng External IP thực tế từ `kubectl get svc`.

---

## 9. 🔀 externalTrafficPolicy

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig để tạo resource; test truy cập từ máy client có route tới MetalLB pool.

Với Ingress NGINX phía sau MetalLB, cluster nhỏ thường dùng:

```yaml
externalTrafficPolicy: Cluster
```

Ưu điểm:

- Traffic có thể route tới Pod dù Pod nằm trên node khác.
- Ít lỗi timeout khi Ingress Pod không nằm đúng node nhận traffic.

Đánh đổi:

- Có thể không giữ nguyên client IP.

Nếu cần giữ client IP:

```yaml
externalTrafficPolicy: Local
```

Khi dùng `Local`, phải đảm bảo Ingress Pod chạy trên các node có thể nhận traffic, thường bằng DaemonSet hoặc replica/affinity phù hợp.

---

## 10. 🌐 Troubleshooting MetalLB

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Nếu `EXTERNAL-IP` là `<pending>`:

```bash
kubectl get pods -n metallb-system
kubectl describe svc test-nginx -n metallb-test
kubectl get ipaddresspool -n metallb-system -o yaml
kubectl get l2advertisement -n metallb-system -o yaml
```

Nếu có External IP nhưng curl không được:

```text
Kiểm tra IP pool có cùng L2/reachable không
Kiểm tra firewall 80/443 hoặc port service
Kiểm tra ARP từ client/reverse proxy
Kiểm tra speaker Running trên node
Kiểm tra provider VPS có cho dùng IP phụ/ARP không
```

---

## 11. 🔎 Dọn resource test

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl delete namespace metallb-test
```
