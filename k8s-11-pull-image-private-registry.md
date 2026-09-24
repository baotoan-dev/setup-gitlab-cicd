# ☸️ Bước 11 - Pull Image Private từ Harbor

## 1. 🎯 Mục đích

File này tạo imagePullSecret/ServiceAccount và kiểm tra Kubernetes pull image private từ Harbor. Resource phải được tạo độc lập trong namespace của từng cluster.

File này cấu hình Kubernetes pull image private từ Harbor bằng:

```text
docker-registry Secret
ServiceAccount
```

Namespace chuẩn của môi trường DEV:

```text
sevago-develop
```

Điều kiện bắt buộc:

```text
✓ Harbor hoạt động
✓ DNS registry.sevatech.local hoạt động
✓ CA đã trust trên tất cả node
✓ containerd đã cấu hình certs.d
✓ ctr pull image thành công trên tất cả node
```

---

## 2. ✅ Checklist nội dung

```text
[ ] DNS/CA/containerd Harbor OK trên tất cả node
[ ] Namespace sevago-develop tồn tại
[ ] registry-cred tồn tại đúng namespace
[ ] registry-cred type là kubernetes.io/dockerconfigjson
[ ] registry-pull-sa gắn registry-cred
[ ] ServiceAccount hiển thị imagePullSecrets registry-cred
[ ] Pod test pull image thành công
[ ] Pod test Running hoặc Completed
[ ] Không có ImagePullBackOff
[ ] Không có lỗi x509
[ ] Không có lỗi unauthorized
[ ] Deployment dùng serviceAccountName
[ ] Helm chart dùng serviceAccountName
[ ] Pod test đã được cleanup
```

---

## 3. 📌 Thông tin sử dụng

```text
Registry Domain : registry.sevatech.local

Namespace       : sevago-develop

Secret          : registry-cred

ServiceAccount  : registry-pull-sa

Image test      :
registry.sevatech.local/sevago-develop/sso-be:develop-23353d8
```

Khuyến nghị:

```text
Dùng Harbor Robot Account chỉ có quyền pull.
Không dùng tài khoản cá nhân.
Không hardcode username/password trong Deployment.
```

---

## 4. ➕ Tạo Namespace

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig; chọn đúng namespace môi trường.

```bash
kubectl create namespace sevago-develop --dry-run=client -o yaml | kubectl apply -f -
kubectl get namespace sevago-develop
```

---

## 5. 🔐 Tạo Registry Secret

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig; chọn đúng namespace môi trường.

Không ghi Harbor Robot password thật vào file hoặc shell script được commit. Nhập secret tại terminal và tạo Kubernetes Secret:

```bash
read -rsp "Harbor Robot password: " HARBOR_ROBOT_PASSWORD; echo
kubectl create secret docker-registry registry-cred --docker-server=registry.sevatech.local --docker-username='robot$sevago-develop+sevago-develop' --docker-password="$HARBOR_ROBOT_PASSWORD" -n sevago-develop --dry-run=client -o yaml | kubectl apply -f -
unset HARBOR_ROBOT_PASSWORD
```

Kiểm tra:

```bash
kubectl get secret registry-cred -n sevago-develop
kubectl describe secret registry-cred -n sevago-develop
```

Kết quả:

```text
TYPE kubernetes.io/dockerconfigjson
DATA 1
```

---

## 6. 👤 Tạo ServiceAccount

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig; chọn đúng namespace môi trường.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: registry-pull-sa
  namespace: sevago-develop
imagePullSecrets:
  - name: registry-cred
EOF
```

Kiểm tra:

```bash
kubectl get sa registry-pull-sa -n sevago-develop -o yaml
```

Hoặc:

```bash
kubectl describe sa registry-pull-sa -n sevago-develop
```

Kết quả mong đợi:

```text
Image pull secrets:
  registry-cred
```

---

## 7. 📦 Test Pull Image

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig; chọn đúng namespace môi trường.

Xóa Pod test cũ nếu tồn tại:

```bash
kubectl delete pod pull-test -n sevago-develop --ignore-not-found
```

Tạo Pod test:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pull-test
  namespace: sevago-develop
spec:
  serviceAccountName: registry-pull-sa
  restartPolicy: Never
  containers:
    - name: test
      image: registry.sevatech.local/sevago-develop/sso-be:develop-23353d8
      imagePullPolicy: Always
      command: ["sh","-c","echo OK && sleep 3"]
EOF
```

Theo dõi:

```bash
kubectl get pod pull-test -n sevago-develop -w
```

Kết quả đúng:

```text
Completed
```

hoặc:

```text
Running
```

Kiểm tra thêm:

```bash
kubectl describe pod pull-test -n sevago-develop
kubectl logs pull-test -n sevago-develop
```

Event đúng:

```text
Normal Pulled
Normal Created
Normal Started
```

Log đúng:

```text
OK
```

---

Sau khi Pod test pull image thành công, có thể chuyển sang bước cấu hình DNS nội bộ. Các mục tiếp theo trong file này dùng khi bắt đầu deploy ứng dụng thật.

---

## 8. 🚀 Áp Dụng Cho Deployment

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig; chọn đúng namespace môi trường.

Ví dụ:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sso-be
  namespace: sevago-develop

spec:
  replicas: 2

  selector:
    matchLabels:
      app: sso-be

  template:
    metadata:
      labels:
        app: sso-be

    spec:
      serviceAccountName: registry-pull-sa

      containers:
        - name: sso-be
          image: registry.sevatech.local/sevago-develop/sso-be:develop-23353d8
          imagePullPolicy: Always

          ports:
            - containerPort: 80
```

---

## 9. ⛵ Áp Dụng Cho Helm Chart

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig; chọn đúng namespace môi trường.

### 9.1. 📌 values.yaml

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig; chọn đúng namespace môi trường.

```yaml
namespace: sevago-develop

serviceAccount:
  name: registry-pull-sa

image:
  repository: registry.sevatech.local/sevago-develop/sso-be
  tag: develop-23353d8
```

### 9.2. 🚀 deployment.yaml

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig; chọn đúng namespace môi trường.

```yaml
spec:
  template:
    spec:
      serviceAccountName: { { .Values.serviceAccount.name } }

      containers:
        - name: { { .Chart.Name } }
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: Always
```

Deploy:

```bash
helm upgrade --install sso-be ./sso-be-chart -n sevago-develop --create-namespace
```

---

## 10. 🔎 Cleanup Pod Test

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig; chọn đúng namespace môi trường.

Sau khi test thành công, chỉ xóa Pod test.

Không xóa:

```text
registry-cred
registry-pull-sa
namespace
```

Lệnh:

```bash
kubectl delete pod pull-test -n sevago-develop --ignore-not-found
```
