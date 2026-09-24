# ☸️ Bước 10 - Cấu hình Harbor Registry cho Containerd

## 1. 🎯 Mục đích

File này cấu hình DNS, CA và containerd trên Kubernetes node để pull image private từ Harbor. Phải áp dụng trên mọi node của từng cluster.

File này cấu hình tất cả master và worker để pull image từ Harbor Registry nội bộ qua HTTPS/CA riêng.

Cần xử lý đủ 3 lớp:

```text
1. Node resolve được domain Harbor.
2. OS trust Root CA để curl/tool hệ thống không lỗi SSL.
3. containerd trust Root CA qua /etc/containerd/certs.d và hosts.toml.
```

Luồng pull image:

```text
Pod
  -> kubelet trên node
  -> containerd
  -> Harbor Registry HTTPS
  -> Pull image
```

Ví dụ:

```text
Harbor Domain: registry.sevatech.local
Harbor IP:     172.17.79.20
Root CA file:  /home/devops/root-ca.registry.crt
```

---

## 2. ✅ Checklist nội dung

```text
[ ] getent hosts registry.sevatech.local trả đúng IP
[ ] OS trust CA, curl /v2/ không lỗi SSL
[ ] /etc/containerd/config.toml có config_path
[ ] /etc/containerd/certs.d/registry.sevatech.local/ca.crt tồn tại
[ ] /etc/containerd/certs.d/registry.sevatech.local/hosts.toml tồn tại
[ ] containerd đã restart và active
[ ] ctr pull image không lỗi x509
```

---

## 3. 📌 Chạy trên đâu

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích; lặp lại trên từng node.

Chạy trên toàn bộ:

```text
3 master + 3 worker node
```

Vì Pod có thể schedule lên bất kỳ node nào, node nào thiếu CA/containerd config thì Pod trên node đó có thể bị `ImagePullBackOff` hoặc lỗi `x509`.

---

## 4. 🌐 DNS/hosts cho Harbor trên node

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích; lặp lại trên từng node.

Nếu chưa có DNS nội bộ, tạm thêm `/etc/hosts` trên tất cả master/worker:

```bash
sudo grep -q 'registry.sevatech.local' /etc/hosts || echo '172.17.79.20 registry.sevatech.local' | sudo tee -a /etc/hosts
```

Test:

```bash
getent hosts registry.sevatech.local
```

Kết quả đúng:

```text
172.17.79.20 registry.sevatech.local
```

> Về lâu dài nên dùng DNS nội bộ thay vì `/etc/hosts`.

---

## 5. ⚙️ Cài CA vào OS trust store

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích; lặp lại trên từng node.

Cần copy cert lên từng vps master và node

Giả định CA nằm ở:

```text
/home/devops/root-ca.registry.crt
```

Copy:

```bash
sudo cp /home/devops/root-ca.registry.crt /usr/local/share/ca-certificates/root-ca.registry.crt
sudo update-ca-certificates
```

Test bằng curl:

```bash
curl -v https://registry.sevatech.local/v2/
```

Kết quả đúng:

```text
SSL certificate verify ok
HTTP/1.1 401 Unauthorized
```

`401 Unauthorized` là đúng nếu registry private cần login.

---

## 6. 📦 Cấu hình containerd registry certs.d

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích; lặp lại trên từng node.

Tạo thư mục registry, Copy CA, Tạo `hosts.toml`, Restart containerd:

```bash
sudo mkdir -p /etc/containerd/certs.d/registry.sevatech.local
sudo cp /home/devops/root-ca.registry.crt /etc/containerd/certs.d/registry.sevatech.local/ca.crt
sudo tee /etc/containerd/certs.d/registry.sevatech.local/hosts.toml > /dev/null <<'EOF'
server = "https://registry.sevatech.local"
[host."https://registry.sevatech.local"]
  capabilities = ["pull", "resolve"]
  ca = "/etc/containerd/certs.d/registry.sevatech.local/ca.crt"
EOF
sudo systemctl restart containerd
sudo systemctl is-active containerd
```

Kiểm tra:

```bash
grep -n "config_path" /etc/containerd/config.toml
sudo ls -l /etc/containerd/certs.d/registry.sevatech.local/
sudo systemctl is-active containerd
```

Kỳ vọng:

```bash
config_path = '/etc/containerd/certs.d:/etc/docker/certs.d'
ca.crt
hosts.toml
active
```

---

## 7. 📦 Test containerd pull image

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích; lặp lại trên từng node.

Nếu image public/internal test có sẵn trong Harbor:

```bash
sudo ctr -n k8s.io images pull registry.sevatech.local/library/nginx:stable
```

Nếu registry yêu cầu login:

```bash
sudo ctr -n k8s.io images pull --user '<USERNAME>:<PASSWORD>' registry.sevatech.local/library/nginx:stable
```

Kết quả đúng:

```text
unpacking linux/amd64 sha256:...
done
```

---

## 8. 🛠️ Lỗi thường gặp

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích; lặp lại trên từng node.

### 8.1. 🌐 DNS lỗi

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích; lặp lại trên từng node.

```text
lookup registry.sevatech.local: no such host
```

Kiểm tra:

```bash
getent hosts registry.sevatech.local
cat /etc/hosts | grep registry.sevatech.local
```

### 8.2. 🔐 TLS/x509 lỗi

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích; lặp lại trên từng node.

```text
x509: certificate signed by unknown authority
```

Kiểm tra:

```bash
curl -v https://registry.sevatech.local/v2/
sudo ls -l /etc/containerd/certs.d/registry.sevatech.local/ca.crt
sudo cat /etc/containerd/certs.d/registry.sevatech.local/hosts.toml
grep -n 'config_path' /etc/containerd/config.toml
systemctl is-active containerd
```

### 8.3. 🛠️ Auth lỗi

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích; lặp lại trên từng node.

```text
401 Unauthorized
unauthorized
no basic auth credentials
```

Kiểm tra:

```text
Username/password hoặc robot account
Quyền pull project/repository
imagePullSecret trong đúng namespace
ServiceAccount đang dùng đúng secret
```
