# ☸️ Bước 07 - Cài đặt Rancher

## 1. 🎯 Mục đích

File này cài Rancher stable lên cluster quản trị được chọn, cấu hình TLS/Ingress và kiểm tra truy cập. Domain/IP minh họa là Develop; không dùng chung secret/credential giữa các cluster.

| Thành phần | Giá trị                    |
| ---------- | -------------------------- |
| Rancher    | `v2.15.0`                  |
| Namespace  | `cattle-system`            |
| Domain     | `develop.k8s.sevago.local` |
| Ingress    | NGINX Ingress Controller   |
| Replicas   | `3`                        |

> Rancher `v2.15.0` là bản stable hỗ trợ Kubernetes `v1.36`. Luôn backup Rancher và đọc release note trước khi nâng cấp.

Certificate:

```text
/home/devops/certs/selfsigned.crt
/home/devops/certs/selfsigned.key
/home/devops/certs/root-ca.crt
```

---

## 2. ✅ Checklist nội dung

```text
[ ] cattle-system đã tạo
[ ] tls-rancher-ingress đã tạo
[ ] tls-ca đã tạo
[ ] Rancher deployed
[ ] Pod Running
[ ] Service hoạt động
[ ] Ingress hoạt động
[ ] HTTPS truy cập được
[ ] Đổi mật khẩu admin
[ ] Local cluster Active
```

---

## 3. 🔎 Kiểm tra môi trường

> **📍 Thực hiện tại:** Control-plane-1 của cluster cài Rancher hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Kiểm tra trạng thái các Node:

```bash
kubectl get nodes
```

Kết quả:

```text
STATUS
Ready
```

Kiểm tra NGINX Ingress Controller:

```bash
kubectl get pods -n ingress-nginx
```

Kết quả:

```text
Running
```

Kiểm tra Helm:

```bash
helm version
```

---

## 4. ⛵ Thêm Helm Repository

> **📍 Thực hiện tại:** Control-plane-1 của cluster cài Rancher hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Thêm Rancher Stable Repository:

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

Kiểm tra phiên bản:

```bash
helm search repo rancher-stable/rancher --versions | head
```

Kiểm tra điều kiện Kubernetes được khai báo trong Helm Chart:

```bash
helm show chart rancher-stable/rancher --version 2.15.0
```

Kết quả:

```yaml
version: 2.15.0
kubeVersion: < 1.37.0-0
```

Rancher `v2.15.0` hỗ trợ Kubernetes `v1.36`. Vẫn cần kiểm tra thực tế Pod, Service, Ingress và Local Cluster sau khi cài.

---

## 5. ➕ Tạo Namespace

> **📍 Thực hiện tại:** Control-plane-1 của cluster cài Rancher hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl create namespace cattle-system
```

Kiểm tra:

```bash
kubectl get ns cattle-system
```

Kết quả:

```text
STATUS
Active
```

---

## 6. 🔐 Tạo TLS Secret

> **📍 Thực hiện tại:** Control-plane-1 của cluster cài Rancher hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Tạo Secret chứa certificate và private key:

```bash
kubectl create secret tls tls-rancher-ingress --cert=/home/devops/certs/selfsigned.crt --key=/home/devops/certs/selfsigned.key -n cattle-system
```

Tạo Secret chứa Root CA:

```bash
kubectl create secret generic tls-ca --from-file=cacerts.pem=/home/devops/certs/root-ca.crt -n cattle-system
```

Kiểm tra:

```bash
kubectl get secret -n cattle-system
```

Kết quả:

```text
tls-ca
tls-rancher-ingress
```

---

## 7. ➕ Tạo file `rancher-values.yaml`

> **📍 Thực hiện tại:** Control-plane-1 của cluster cài Rancher hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Tạo file cấu hình:

```bash
nano rancher-values.yaml
```

Nội dung:

```yaml
hostname: develop.k8s.sevago.local

replicas: 3

privateCA: true

ingress:
  ingressClassName: nginx
  tls:
    source: secret
```

---

## 8. ⚙️ Cài đặt Rancher

> **📍 Thực hiện tại:** Control-plane-1 của cluster cài Rancher hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Không lưu bootstrap password trong Git hoặc trực tiếp trong `rancher-values.yaml`. Nhập password tạm tại terminal rồi cài Rancher:

```bash
read -rsp "Rancher bootstrap password: " RANCHER_BOOTSTRAP_PASSWORD; echo
helm upgrade --install rancher rancher-stable/rancher --namespace cattle-system --version 2.15.0 -f rancher-values.yaml --set-string bootstrapPassword="$RANCHER_BOOTSTRAP_PASSWORD" --wait --timeout 20m
unset RANCHER_BOOTSTRAP_PASSWORD
```

Kiểm tra Helm Release:

```bash
helm list -n cattle-system
```

Kết quả:

```text
NAME      NAMESPACE       STATUS
rancher   cattle-system   deployed
```

---

## 9. 🔎 Kiểm tra Pod

> **📍 Thực hiện tại:** Control-plane-1 của cluster cài Rancher hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl get pods -n cattle-system
```

Kết quả:

```text
NAME            READY   STATUS
rancher-*       1/1     Running
```

Kiểm tra quá trình rollout:

```bash
kubectl rollout status deployment/rancher -n cattle-system --timeout=20m
```

Kết quả:

```text
deployment "rancher" successfully rolled out
```

```bash
kubectl get deployment rancher -n cattle-system -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

Kết quả:

```text
rancher/rancher:v2.15.0
```

---

## 10. 🔎 Kiểm tra Service

> **📍 Thực hiện tại:** Control-plane-1 của cluster cài Rancher hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl get svc -n cattle-system
```

Kết quả:

```text
rancher
```

---

## 11. 🌐 Kiểm tra Ingress

> **📍 Thực hiện tại:** Control-plane-1 của cluster cài Rancher hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl get ingress -n cattle-system
```

Kết quả:

```text
HOSTS
develop.k8s.sevago.local
```

---

## 12. 📌 Truy cập Rancher

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt, có route/DNS tới Rancher.

Truy cập địa chỉ:

```text
https://develop.k8s.sevago.local
```

Đăng nhập lần đầu bằng user `admin` và **bootstrap password vừa nhập ở bước cài đặt**, sau đó đổi sang mật khẩu quản trị chính thức. Không ghi mật khẩu thật vào tài liệu hoặc repository.

---

## 13. 👤 Tạo người dùng

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt, có route/DNS tới Rancher.

Truy cập:

```text
Users & Authentication
```

Tạo tài khoản và phân quyền: Administrator, Cluster Owner, Project Owner, Project Member, Read Only.

> Không sử dụng tài khoản `admin` cho người dùng hằng ngày.

---

## 14. 🔎 Kiểm tra Cluster

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt, có route/DNS tới Rancher.

Truy cập:

```text
Cluster Management
```

Kiểm tra:

```text
local
Status: Active
```

---

## 15. 📌 Nâng cấp Rancher

> **📍 Thực hiện tại:** Control-plane-1 của cluster cài Rancher hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Cập nhật Helm Repository:

```bash
helm repo update
```

Nâng cấp Rancher:

```bash
helm upgrade rancher rancher-stable/rancher --namespace cattle-system --version <NEW_VERSION> -f rancher-values.yaml --wait --timeout 20m
```
