# ☸️ Bước 04 - Join Worker Node, cấu hình Firewall và kiểm tra

## 1. 🎯 Mục đích

File này join worker, kiểm tra firewall/kubelet/networking và chạy workload test sau khi Control Plane đã sẵn sàng. Lặp lại độc lập trên từng cluster đích.

Tài liệu này hướng dẫn:

- Join 03 worker node vào Kubernetes HA cluster đã có sẵn 03 master.
- Cấu hình firewall phục vụ Kubernetes và Calico.
- Kiểm tra kết nối tới Kubernetes API VIP.
- Kiểm tra kubelet (`10250`).
- Xác nhận Calico networking hoạt động.
- Triển khai workload test trên worker node.
- Kiểm tra `kubectl logs` và `kubectl exec`.
- Kiểm tra NodePort nội bộ.
- Dọn dẹp tài nguyên test sau khi hoàn tất.

---

## 2. ✅ Checklist nội dung

```text
[ ] Worker truy cập được API VIP 172.17.79.159:6443
[ ] Join cluster thành công
[ ] Tất cả node ở trạng thái Ready
[ ] Private subnet Kubernetes được trust, kubelet không public ra Internet
[ ] Calico node-to-node traffic không bị firewall chặn
[ ] Pod test được schedule lên worker node
[ ] Pod trạng thái Running
[ ] NodePort nội bộ truy cập thành công
[ ] kubectl logs hoạt động bình thường
[ ] kubectl exec hoạt động bình thường
[ ] Namespace cluster-test đã được xóa
```

---

## 3. 🔎 Kiểm tra Kubernetes API từ worker node

> **📍 Thực hiện tại:** Từng worker của cluster đích. Develop: `172.17.79.93`, `.94`, `.95`.

### 3.1. 🔎 Kiểm tra API Root

> **📍 Thực hiện tại:** Từng worker của cluster đích. Develop: `172.17.79.93`, `.94`, `.95`.

```bash
curl -k https://172.17.79.159:6443
```

Kết quả mong muốn:

```text
forbidden: User "system:anonymous" cannot get path "/"
```

---

### 3.2. 🩺 Kiểm tra Health Endpoint

> **📍 Thực hiện tại:** Từng worker của cluster đích. Develop: `172.17.79.93`, `.94`, `.95`.

```bash
curl -k https://172.17.79.159:6443/healthz
```

Kết quả mong muốn:

```text
ok
```

---

## 4. ➕ Tạo join command

> **📍 Thực hiện tại:** Control-plane-1 hoặc máy quản trị có kubeconfig của cluster đích.

```bash
sudo kubeadm token create --print-join-command
```

Ví dụ:

```bash
kubeadm join 172.17.79.159:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

Lưu lại toàn bộ lệnh.

---

## 5. 🔗 Join worker node vào cluster

> **📍 Thực hiện tại:** Từng worker của cluster đích. Develop: `172.17.79.93`, `.94`, `.95`.

```bash
sudo kubeadm join 172.17.79.159:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

Kết quả mong muốn:

```text
This node has joined the cluster
```

---

### 5.1. 📌 Xác nhận worker đã tham gia cluster

> **📍 Thực hiện tại:** Control-plane-1 hoặc máy quản trị có kubeconfig của cluster đích.

```bash
kubectl get nodes -o wide
```

Kết quả mong muốn:

```text
STATUS = Ready
```

Ví dụ:

```text
sevago-dev-k8s-node-1   Ready
sevago-dev-k8s-node-2   Ready
sevago-dev-k8s-node-3   Ready
```

---

## 6. 🔎 Kiểm tra kubelet (10250)

> **📍 Thực hiện tại:** Control-plane-1 hoặc máy quản trị có kubeconfig của cluster đích.

### 6.1. 🔎 Kiểm tra kubelet đang listen

> **📍 Thực hiện tại:** Từng worker của cluster đích. Develop: `172.17.79.93`, `.94`, `.95`.

```bash
sudo ss -lntp | grep 10250
```

Kết quả mong muốn:

```text
LISTEN 0 4096 *:10250 *:*
```

---

### 6.2. 🔎 Kiểm tra kết nối từ master

> **📍 Thực hiện tại:** Một control-plane của cluster đích; Develop: `172.17.79.90`.

```bash
curl -k https://172.17.79.93:10250/healthz
curl -k https://172.17.79.94:10250/healthz
curl -k https://172.17.79.95:10250/healthz
```

Hoặc:

```bash
curl -k -I https://172.17.79.93:10250/healthz
curl -k -I https://172.17.79.94:10250/healthz
curl -k -I https://172.17.79.95:10250/healthz
```

Kết quả mong muốn:

```text
401 Unauthorized
```

hoặc

```text
Unauthorized
```

Điều này xác nhận: Firewall không chặn traffic, Kubelet đang hoạt động, Authentication của kubelet được bật.

> ⚠️ Trên Kubernetes hiện đại, `401 Unauthorized` là kết quả bình thường.

---

### 6.3. 🔎 Kiểm tra end-to-end qua Kubernetes API

> **📍 Thực hiện tại:** Control-plane-1 hoặc máy quản trị có kubeconfig của cluster đích.

```bash
kubectl get nodes
kubectl get pods -A
```

Kiểm tra thực tế:

```bash
kubectl logs -n kube-system <POD_NAME>
kubectl exec -n kube-system <POD_NAME> -- hostname
```

Kết quả mong muốn: `kubectl logs` hoạt động, `kubectl exec` hoạt động, Không timeout tới kubelet.

Điều này xác nhận:

```text
kube-apiserver <-> kubelet
```

đang hoạt động bình thường.

---

## 7. 👤 Gán role cho worker node (tùy chọn)

> **📍 Thực hiện tại:** Control-plane-1 hoặc máy quản trị có kubeconfig của cluster đích.

```bash
kubectl label node sevago-dev-k8s-node-1 node-role.kubernetes.io/worker=worker
kubectl label node sevago-dev-k8s-node-2 node-role.kubernetes.io/worker=worker
kubectl label node sevago-dev-k8s-node-3 node-role.kubernetes.io/worker=worker
```

Kiểm tra:

```bash
kubectl get nodes
```

---

## 8. 🔎 Kiểm tra scheduling pod trên worker

> **📍 Thực hiện tại:** Control-plane-1 hoặc máy quản trị có kubeconfig của cluster đích.

Tạo namespace test:

```bash
kubectl create namespace cluster-test
```

Tạo deployment:

```bash
kubectl create deployment test-nginx --image=nginx:stable -n cluster-test
```

Kiểm tra:

```bash
kubectl get pods -n cluster-test -o wide
```

Kết quả mong muốn:

```text
STATUS Running
READY 1/1
NODE là một worker node
```

> 📝 Nếu môi trường không có Internet, sử dụng image đã mirror vào Harbor.

Ví dụ:

```text
registry.sevatech.local/library/nginx:stable
```

---

## 9. 🔎 Kiểm tra NodePort nội bộ

> **📍 Thực hiện tại:** Control-plane-1 hoặc máy quản trị có kubeconfig của cluster đích.

Tạo Service:

```bash
kubectl expose deployment test-nginx --port=80 --type=NodePort -n cluster-test
```

Kiểm tra:

```bash
kubectl get svc test-nginx -n cluster-test
```

Lấy giá trị `NODE_PORT`.

---

### 9.1. 🔎 Kiểm tra truy cập

> **📍 Thực hiện tại:** Control-plane-1 hoặc máy quản trị có kubeconfig của cluster đích.

```bash
for ip in 172.17.79.93 172.17.79.94 172.17.79.95; do curl "http://${ip}:<NODE_PORT>"; done
```

Kết quả mong muốn:

```html
Welcome to nginx!
```

> ⚠️ NodePort chỉ dùng cho mục đích kiểm thử nội bộ, không sử dụng để publish service ứng dụng.

---

## 10. 🔎 Kiểm tra logs và exec

> **📍 Thực hiện tại:** Control-plane-1 hoặc máy quản trị có kubeconfig của cluster đích.

```bash
kubectl logs deploy/test-nginx -n cluster-test
kubectl exec -n cluster-test deploy/test-nginx -- nginx -v
```

Kết quả mong muốn: Hiển thị log nginx, Hiển thị phiên bản nginx, Không timeout tới kubelet.

---

## 11. 🔎 Dọn dẹp môi trường test

> **📍 Thực hiện tại:** Control-plane-1 hoặc máy quản trị có kubeconfig của cluster đích.

```bash
kubectl delete namespace cluster-test
```

> ⚠️ Không sử dụng namespace thật (ví dụ `dev`) để test rồi xóa.
