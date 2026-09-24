# ☸️ Bước 15 - Quản lý TLS Secret Kubernetes

## 1. 🎯 Mục đích

File này chuẩn hóa nguồn certificate, tên TLS Secret và cách tạo/kiểm tra Secret trên 5 cluster. TLS Secret là namespace-scoped và không được hiểu là dùng chung giữa hai cluster.

Chuẩn hóa TLS Secret sử dụng trong Kubernetes.

Tên Secret chuẩn:

```text
local-tls
public-tls
```

Ý nghĩa:

```text
local-tls → dùng cho domain nội bộ (*.sevago.local)
public-tls → dùng cho domain public (*.com.vn)
```

---

## 2. ✅ Checklist nội dung

```text
[ ] local-tls tồn tại trong mọi namespace có domain nội bộ
[ ] public-tls tồn tại trong namespace dùng domain public
[ ] Jenkins truyền đúng TLS secret
[ ] Ingress và TLS Secret cùng namespace
[ ] HTTPS hoạt động bình thường
[ ] Cert đúng SAN/domain
[ ] DNS trỏ đúng Ingress
```

---

## 3. 🔐 Quy hoạch TLS Secret cho 5 cluster

| Cluster ID                | Namespace ứng dụng chính | Secret nội bộ |
| ------------------------- | ------------------------ | ------------- |
| `sevago-develop`          | `sevago-develop`         | `local-tls`   |
| `sevago-staging-local`    | `sevago-staging`         | `local-tls`   |
| `sevago-staging-dmz`      | `sevago-staging`         | `local-tls`   |
| `sevago-production-local` | `sevago-production`      | `local-tls`   |
| `sevago-production-dmz`   | `sevago-production`      | `local-tls`   |

`local-tls` là namespace-scoped và **phải tồn tại độc lập trong từng cluster/namespace**. Việc hai cluster cùng có namespace `sevago-staging` hoặc `sevago-production` không có nghĩa chúng dùng chung Secret.

---

## 4. 📐 Nguyên tắc

TLS Secret là resource:

```text
namespace-scoped
```

Do đó:

```text
1 Secret không thể dùng chung giữa nhiều namespace
```

Ví dụ, Secret ở Develop không thể được tham chiếu trực tiếp từ namespace/cluster khác:

```text
❌ sevago-develop/local-tls -> sevago-staging hoặc sevago-production
```

Phải tạo `local-tls` độc lập trong đúng namespace của từng cluster đích.

Tuy nhiên:

```text
✓ cùng một certificate
✓ cùng một private key
```

có thể được sử dụng để tạo nhiều Secret ở nhiều namespace khác nhau.

---

## 5. 📌 Cert Nguồn

> **📍 Thực hiện tại:** VPS/máy quản trị đang lưu certificate nguồn và có kubeconfig của cluster đích; Develop thường dùng `172.17.79.90`.

### 5.1. 🌐 Local Domain

```text
/home/devops/certs/selfsigned.crt
/home/devops/certs/selfsigned.key
```

Dùng tạo:

```text
local-tls
```

### 5.2. 🔎 Kiểm tra Certificate nguồn

> **📍 Thực hiện tại:** VPS/máy quản trị đang lưu certificate nguồn và có kubeconfig của cluster đích; Develop thường dùng `172.17.79.90`.

Kiểm tra certificate nội bộ:

```bash
openssl x509 -in /home/devops/certs/selfsigned.crt -noout -subject -issuer -dates
openssl x509 -in /home/devops/certs/selfsigned.crt -noout -checkend 2592000
```

Kiểm tra certificate public:

```bash
openssl x509 -in /home/devops/certs/public/fullchain.pem -noout -subject -issuer -dates
openssl x509 -in /home/devops/certs/public/fullchain.pem -noout -checkend 2592000
```

Trong đó:

```text
2592000 giây = 30 ngày
```

Kết quả mong đợi:

```text
Certificate will not expire
```

Kiểm tra certificate nội bộ có khớp private key:

```bash
openssl x509 -in /home/devops/certs/selfsigned.crt -pubkey -noout | openssl pkey -pubin -outform DER | sha256sum
openssl pkey -in /home/devops/certs/selfsigned.key -pubout -outform DER | sha256sum
```

Kiểm tra certificate public có khớp private key:

```bash
openssl x509 -in /home/devops/certs/public/fullchain.pem -pubkey -noout | openssl pkey -pubin -outform DER | sha256sum
openssl pkey -in /home/devops/certs/public/privkey.pem -pubout -outform DER | sha256sum
```

Kết quả mong đợi:

```text
Hai mã SHA256 của certificate và private key giống nhau.
```

Nếu khác nhau, không tạo hoặc cập nhật Kubernetes TLS Secret.

---

## 6. 🔐 Tạo hoặc cập nhật local-tls trên cluster đích

> **📍 Thực hiện tại:** VPS/máy quản trị đang lưu certificate nguồn và có kubeconfig của cluster đích; Develop thường dùng `172.17.79.90`.

Thêm domain vào `[alt_names]` trong `san.cnf`. Gen lại `selfsigned.key` + `selfsigned.csr` - hoặc có thể chỉ gen lại `selfsigned.csr`

```bash
openssl req -new -key selfsigned.key -out selfsigned.csr -config san.cnf
```

Ký lại bằng `root-ca.key` → `selfsigned.crt` mới

```bash
openssl x509 -req -in selfsigned.csr -CA root-ca.crt -CAkey root-ca.key -CAcreateserial -out selfsigned.crt -days 825 -sha256 -extfile san.cnf -extensions req_ext
```

Chọn đúng kubeconfig của cluster đích và namespace tương ứng. Ví dụ Develop:

```bash
kubectl create secret tls local-tls --cert=/home/devops/certs/selfsigned.crt --key=/home/devops/certs/selfsigned.key -n sevago-develop --dry-run=client -o yaml | kubectl apply -f -
```

Với Staging/Production, đổi `KUBECONFIG` sang cluster Local/DMZ tương ứng và đặt `NAMESPACE=sevago-staging` hoặc `NAMESPACE=sevago-production`. Chỉ dùng cùng certificate nếu SAN của certificate bao phủ domain của môi trường đích.

Kiểm tra:

```bash
openssl x509 -in ~/certs/selfsigned.crt -noout -text | grep -A2 "Subject Alternative Name"
kubectl get secret -A | grep local-tls
```

Kết quả mong đợi:

```text
// Có chứa domain mới thêm và bên dưới
sevago-develop   local-tls   kubernetes.io/tls
```

---

## 7. 🧰 Jenkins Rule

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` hoặc Jenkinsfile đang quản lý CI/CD.

Jenkins luôn truyền TLS Secret nội bộ:

```bash
TLS_SECRET_NAME="local-tls"
```

---

## 8. ⛵ Helm

### 8.1. 📌 values.yaml

```yaml
ingress:
  tls:
    secretName: ""
```

### 8.2. 🧰 Jenkins

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` hoặc Jenkinsfile đang quản lý CI/CD.

```bash
--set ingress.tls.secretName="${TLS_SECRET_NAME}"
```

---

## 9. 🌐 Kiểm Tra Ingress

> **📍 Thực hiện tại:** Kubectl trên máy có kubeconfig; curl/openssl trên máy client có route/DNS tới domain cần kiểm tra.

Ví dụ:

```bash
kubectl get ingress office-fe -n sevago-develop -o yaml
```

Kỳ vọng:

```yaml
tls:
  - hosts:
      - develop.office.sevago.local
    secretName: local-tls
```

---

## 10. 🔎 Test HTTPS

> **📍 Thực hiện tại:** Kubectl trên máy có kubeconfig; curl/openssl trên máy client có route/DNS tới domain cần kiểm tra.

Nếu dùng cert nội bộ và client chưa trust Root CA:

```bash
curl -vk https://develop.office.sevago.local/
```

Nếu client đã trust Root CA:

```bash
curl -v https://develop.office.sevago.local/
```

Public domain:

```bash
curl -v https://office.sevago.com.vn/
```

---

## 11. 🔎 Test Bằng OpenSSL

> **📍 Thực hiện tại:** Kubectl trên máy có kubeconfig; curl/openssl trên máy client có route/DNS tới domain cần kiểm tra.

```bash
openssl s_client -connect develop.office.sevago.local:443 -servername develop.office.sevago.local -showcerts
```

Public:

```bash
openssl s_client -connect office.sevago.com.vn:443 -servername office.sevago.com.vn -showcerts
```
