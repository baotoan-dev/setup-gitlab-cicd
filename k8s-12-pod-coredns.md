# ☸️ Bước 12 - Cấu hình DNS nội bộ cho Pod

## 1. 🎯 Mục đích

File này cấu hình CoreDNS để Pod resolve domain nội bộ của công ty và kiểm tra service discovery. Thay đổi được thực hiện độc lập trên từng cluster.

Cho phép toàn bộ Pod trong Kubernetes resolve được các domain nội bộ của Sevago thông qua DNS nội bộ công ty.

Mô hình mong muốn:

```text
Pod
  |
CoreDNS
  |
DNS nội bộ công ty
  |
*.sevago.local
```

Ví dụ:

```text
develop.account.sevago.local
develop.office.sevago.local
develop.admin.office.sevago.local
develop.kafka.sevago.local
develop.redis.sevago.local
develop.file.sevago.local
develop.elk.sevago.local
```

Khi DNS nội bộ công ty có thêm domain mới thì Pod trong Kubernetes tự động resolve được mà không cần sửa CoreDNS.

> Kubernetes Pod resolve `sevago.local` thông qua CoreDNS nên không phụ thuộc mDNS. Máy người dùng truy cập domain này vẫn phải sử dụng DNS nội bộ công ty hoặc resolver phù hợp.

---

## 2. ✅ Checklist nội dung

```text
[ ] DNS công ty quản lý zone sevago.local
[ ] CoreDNS forward sevago.local về DNS công ty
[ ] CoreDNS rollout thành công
[ ] Pod resolve được domain nội bộ
[ ] Domain mới chỉ tạo trên DNS công ty
```

---

## 3. ⚙️ Cấu hình Nano làm editor mặc định cho Kubernetes

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
echo 'export EDITOR=nano' >> ~/.bashrc
echo 'export KUBE_EDITOR=nano' >> ~/.bashrc
source ~/.bashrc
```

Kiểm tra:

```bash
echo $EDITOR
echo $KUBE_EDITOR
```

Kết quả:

```text
nano
nano
```

---

## 4. 🏗️ Kiến trúc DNS

DNS nội bộ công ty quản lý toàn bộ zone:

```text
sevago.local
```

Ví dụ:

```text
develop.account.sevago.local
develop.office.sevago.local
develop.admin.office.sevago.local
develop.kafka.sevago.local
develop.redis.sevago.local
develop.file.sevago.local
develop.elk.sevago.local
```

Việc tạo mới domain được thực hiện trên DNS nội bộ công ty.

Không thực hiện trong Kubernetes.

---

Pod không truy vấn trực tiếp DNS công ty.

Pod sử dụng CoreDNS:

```text
Pod
  |
CoreDNS
```

CoreDNS có nhiệm vụ:

```text
cluster.local
↓
Kubernetes Service Discovery

sevago.local
↓
Forward sang DNS nội bộ công ty

Internet
↓
Forward sang DNS hệ điều hành
```

---

## 5. 📐 Nguyên tắc thiết kế

Không quản lý domain nội bộ bằng block:

```text
hosts {
  172.17.x.x domain.sevago.local
}
```

trừ trường hợp tạm thời hoặc môi trường lab.

Domain nội bộ phải được quản lý tập trung trên DNS công ty.

Khi có domain mới:

```text
develop.rabbitmq.sevago.local
develop.minio.sevago.local
develop.smtp.sevago.local
```

chỉ cần tạo record trên DNS công ty.

Không cần:

```text
Sửa CoreDNS
Restart CoreDNS
Sửa ConfigMap
```

---

## 6. 🌐 Backup CoreDNS

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl -n kube-system get configmap coredns -o yaml > coredns-backup.yaml
```

---

## 7. 🌐 Cấu hình CoreDNS

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Mở ConfigMap:

```bash
kubectl -n kube-system edit configmap coredns
```

Thêm block:

```text
sevago.local:53 {
    errors
    cache 30
    forward . 172.17.90.23
}
```

Trong đó:

```text
172.17.90.23
```

là DNS nội bộ công ty.

Thêm song song với block .:53 như bên dưới:

```text
sevago.local:53 {
    errors
    cache 30
    forward . 172.17.90.23
}
.:53 {
   ...
}
```

---

## 8. 🌐 Restart CoreDNS

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl rollout restart deployment/coredns -n kube-system
kubectl rollout status deployment/coredns -n kube-system
```

Kiểm tra:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

---

## 9. 🌐 Test DNS từ Pod

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Tạo Pod test:

```bash
kubectl run dns-test --image=busybox:1.36 --restart=Never -n default --command -- sleep 3600
```

Test:

```bash
kubectl exec dns-test -- nslookup develop.account.sevago.local
kubectl exec dns-test -- nslookup develop.office.sevago.local
kubectl exec dns-test -- nslookup develop.admin.office.sevago.local
```

Kết quả mong đợi:

```text
Name: develop.account.sevago.local
Address: x.x.x.x

Name: develop.office.sevago.local
Address: x.x.x.x

Name: develop.admin.office.sevago.local
Address: x.x.x.x
```

---

## 10. 🛠️ Troubleshooting

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

### 10.1. 📌 Local resolve được nhưng Pod không resolve được

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Kiểm tra:

```bash
kubectl exec dns-test -- nslookup develop.account.sevago.local
```

Kiểm tra CoreDNS:

```bash
kubectl get configmap coredns -n kube-system -o yaml
kubectl logs -n kube-system deploy/coredns
```

Kiểm tra DNS công ty:

```bash
nslookup develop.account.sevago.local 172.17.90.23
```

---

### 10.2. 🌐 Xoá dns-test

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Sau khi hoàn thành kiểm tra DNS:

```bash
kubectl delete pod dns-test -n default --ignore-not-found
```

Kiểm tra:

```bash
kubectl get pod dns-test -n default
```

Kết quả mong đợi:

```text
Error from server (NotFound): pods "dns-test" not found
```

### 10.3. 🛠️ Pod resolve được nhưng HTTPS lỗi

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Ví dụ:

```text
certificate verify failed
```

Nguyên nhân:

```text
Pod chưa trust Root CA nội bộ
```

Đây là lỗi TLS, không phải lỗi DNS.
