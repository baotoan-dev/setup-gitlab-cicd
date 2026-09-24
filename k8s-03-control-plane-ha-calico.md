# ☸️ Bước 03 - Khởi tạo Kubernetes HA Control Plane và cài Calico

## 1. 🎯 Mục đích

File này dựng HA Control Plane bằng HAProxy + Keepalived, bootstrap kubeadm và cài Calico. Thực hiện riêng trên từng cluster; các IP cụ thể trong file là của Develop.

Tài liệu này hướng dẫn khởi tạo Kubernetes HA Control Plane và cài Calico.

---

## 2. ✅ Checklist nội dung

- [ ] Mở firewall Kubernetes nội bộ
- [ ] Cài HAProxy + Keepalived
- [ ] Xác định private interface
- [ ] Cấu hình HAProxy
- [ ] Cấu hình Keepalived
- [ ] Start HAProxy + Keepalived
- [ ] Kiểm tra VIP
- [ ] Khởi tạo control plane đầu tiên
- [ ] Cấu hình kubectl
- [ ] Cài Calico
- [ ] Ghi chú firewall Calico
- [ ] Join master-2, master-3 vào control plane
- [ ] Nếu certificate key hết hạn
- [ ] Kiểm tra toàn cụm

---

## 3. ⚖️ Cài HAProxy + Keepalived

> **📍 Thực hiện tại:** Cả 3 control-plane của cluster đích. Develop: `172.17.79.90`, `.91`, `.92`.

```bash
sudo apt update
sudo apt install -y haproxy keepalived netcat-openbsd
```

Kiểm tra trên cả 3 master

```bash
haproxy -v
keepalived -v
nc -h
```

---

## 4. 📌 Xác định private interface

> **📍 Thực hiện tại:** Cả 3 control-plane của cluster đích. Develop: `172.17.79.90`, `.91`, `.92`.

```bash
ip -br addr
```

Kết quả đúng

Tìm interface đang giữ IP private:

```text
ens33  UP  172.17.79.90/24
```

Ví dụ trong file này dùng `ens33`. Nếu máy dùng interface khác thì sửa lại trong cấu hình Keepalived.

---

## 5. ⚖️ Cấu hình HAProxy

> **📍 Thực hiện tại:** Cả 3 control-plane của cluster đích. Develop: `172.17.79.90`, `.91`, `.92`.

```bash
sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon
    maxconn 4096
defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5s
    timeout client  50s
    timeout server  50s
frontend k8s_api_frontend
    bind *:6443
    default_backend k8s_api_backend
backend k8s_api_backend
    balance roundrobin
    option httpchk
    http-check connect ssl
    http-check send meth GET uri /readyz ver HTTP/1.1 hdr Host kubernetes
    http-check expect status 200
    server master-1 172.17.79.90:6444 check check-ssl verify none inter 2s fall 2 rise 2
    server master-2 172.17.79.91:6444 check check-ssl verify none inter 2s fall 2 rise 2
    server master-3 172.17.79.92:6444 check check-ssl verify none inter 2s fall 2 rise 2
EOF
```

Kiểm tra trên cả 3 master

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

Kết quả đúng

```text
Configuration file is valid
```

HAProxy kiểm tra `https://<master-ip>:6444/readyz` mỗi 2 giây. Backend fail 2 lần liên tiếp sẽ bị loại khỏi pool; pass 2 lần liên tiếp sẽ được đưa vào lại.

---

## 6. ⚖️ Cấu hình Keepalived

> **📍 Thực hiện tại:** Cả 3 control-plane của cluster đích. Develop: `172.17.79.90`, `.91`, `.92`.

### 6.1. 📌 Master-1: `172.17.79.90`

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích. Develop: `172.17.79.90`.

```bash
sudo tee /etc/keepalived/keepalived.conf > /dev/null <<'EOF'
vrrp_script chk_haproxy {
    script "pidof haproxy"
    interval 2
    weight -20
}
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 79
    priority 110
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass SevagoK8S
    }
    virtual_ipaddress {
        172.17.79.159/24
    }
    track_script {
        chk_haproxy
    }
}
EOF
```

### 6.2. 📌 Master-2: `172.17.79.91`

> **📍 Thực hiện tại:** Control-plane-2. Develop: `172.17.79.91`.

```bash
sudo tee /etc/keepalived/keepalived.conf > /dev/null <<'EOF'
vrrp_script chk_haproxy {
    script "pidof haproxy"
    interval 2
    weight -20
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 79
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass SevagoK8S
    }
    virtual_ipaddress {
        172.17.79.159/24
    }
    track_script {
        chk_haproxy
    }
}
EOF
```

### 6.3. 📌 Master-3: `172.17.79.92`

> **📍 Thực hiện tại:** Control-plane-3. Develop: `172.17.79.92`.

```bash
sudo tee /etc/keepalived/keepalived.conf > /dev/null <<'EOF'
vrrp_script chk_haproxy {
    script "pidof haproxy"
    interval 2
    weight -20
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 79
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass SevagoK8S
    }
    virtual_ipaddress {
        172.17.79.159/24
    }
    track_script {
        chk_haproxy
    }
}
EOF
```

---

## 7. ⚖️ Start HAProxy + Keepalived

> **📍 Thực hiện tại:** Cả 3 control-plane của cluster đích. Develop: `172.17.79.90`, `.91`, `.92`.

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl enable --now haproxy keepalived
sudo systemctl reload haproxy
sudo systemctl restart keepalived
```

Kiểm tra trên cả 3 master

```bash
systemctl is-active haproxy
systemctl is-active keepalived
```

Kết quả đúng

```text
active
active
```

---

## 8. 🔎 Kiểm tra VIP

> **📍 Thực hiện tại:** Cả 3 control-plane của cluster đích. Develop: `172.17.79.90`, `.91`, `.92`.

```bash
ip addr | grep 172.17.79.159 || true
```

Kết quả đúng

VIP chỉ xuất hiện trên 1 master, thường là master-1:

```text
inet 172.17.79.159/24 scope global secondary ens33
```

---

## 9. ➕ Khởi tạo control plane đầu tiên

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích. Develop: `172.17.79.90`.

```bash
sudo kubeadm init --control-plane-endpoint=172.17.79.159:6443 --apiserver-advertise-address=172.17.79.90 --apiserver-bind-port=6444 --pod-network-cidr=192.168.0.0/16 --upload-certs
```

Kết quả đúng

Cuối output có:

```text
Your Kubernetes control-plane has initialized successfully!
```

Lưu lại 2 lệnh:

```text
kubeadm join ... --control-plane --certificate-key ...
kubeadm join ...
```

Lệnh có `--control-plane` dùng cho master-2/master-3.

Lệnh không có `--control-plane` dùng cho worker.

---

## 10. ⚙️ Cấu hình kubectl

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích. Develop: `172.17.79.90`.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
chmod 600 $HOME/.kube/config
```

Kiểm tra trên master-1

```bash
kubectl cluster-info
kubectl get nodes -o wide
```

Kết quả đúng

Trước khi cài Calico, node có thể là `NotReady`:

```text
sevago-dev-k8s-master-1   NotReady
```

---

## 11. ⚙️ Cài Calico

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích. Develop: `172.17.79.90`.

Chỉ chạy 1 lần trên master-1

```bash
export CALICO_VERSION=v3.32.1
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/calico.yaml
```

> Không chạy lệnh này trên master-2/master-3. Calico là Kubernetes resource, apply 1 lần là đủ. Sau khi node mới join, Calico DaemonSet sẽ tự chạy trên node đó.

Kiểm tra

```bash
kubectl get pods -n kube-system -w
```

Kết quả đúng

```text
calico-node               Running
calico-kube-controllers   Running
coredns                   Running
```

Kiểm tra node:

```bash
kubectl get nodes -o wide
```

Kết quả đúng

```text
sevago-dev-k8s-master-1   Ready
```

---

## 12. 🧱 Kiểm tra Firewall Calico

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích. Develop: `172.17.79.90`.

Rule private subnet ở mục 2 đã cho phép traffic Calico giữa các Node.

Calico có thể sử dụng:

```text
IPIP: protocol 4
VXLAN: 4789/udp
BGP: 179/tcp
```

Kiểm tra mode Calico:

```bash
kubectl get ippool -o yaml
```

Kiểm tra Pod Calico:

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node -o wide
```

Kết quả mong đợi:

```text
calico-node Running trên toàn bộ Master và Worker Node
```

---

## 13. 🔗 Join master-2, master-3 vào control plane

> **📍 Thực hiện tại:** Control-plane-2 và control-plane-3 của cluster đích. Develop: `172.17.79.91` và `172.17.79.92`.

### 13.1. 📌 Chỉ chạy trên master-2: `172.17.79.91`

> **📍 Thực hiện tại:** Control-plane-2. Develop: `172.17.79.91`.

Dùng lệnh join control-plane đã lưu sau `kubeadm init`, thêm advertise address và bind port:

```bash
sudo kubeadm join 172.17.79.159:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERTIFICATE_KEY> --apiserver-advertise-address=172.17.79.91 --apiserver-bind-port=6444
```

### 13.2. 📌 Chỉ chạy trên master-3: `172.17.79.92`

> **📍 Thực hiện tại:** Control-plane-3. Develop: `172.17.79.92`.

```bash
sudo kubeadm join 172.17.79.159:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERTIFICATE_KEY> --apiserver-advertise-address=172.17.79.92 --apiserver-bind-port=6444
```

---

## 14. 📌 Nếu certificate key hết hạn

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích. Develop: `172.17.79.90`.

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

Lệnh này sinh `certificate-key` mới.

Tạo lại join command:

```bash
sudo kubeadm token create --print-join-command
```

Sau đó ghép thêm:

```text
--control-plane --certificate-key <CERTIFICATE_KEY> --apiserver-bind-port=6444
```

---

## 15. 🔎 Kiểm tra toàn cụm

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig; nếu thao tác HA thì kiểm tra cả 3 control-plane.

### 15.1. 🔎 Kiểm tra masters

> **📍 Thực hiện tại:** Cả 3 control-plane của cluster đích. Develop: `172.17.79.90`, `.91`, `.92`.

```bash
kubectl get nodes -o wide
```

Kết quả đúng

```text
sevago-dev-k8s-master-1   Ready   control-plane
sevago-dev-k8s-master-2   Ready   control-plane
sevago-dev-k8s-master-3   Ready   control-plane
```

### 15.2. 🔎 Kiểm tra kube-system pods

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích. Develop: `172.17.79.90`.

```bash
kubectl get pods -n kube-system -o wide
```

Kết quả đúng

Các pod chính phải `Running`:

```text
calico-node
calico-kube-controllers
coredns
etcd
kube-apiserver
kube-controller-manager
kube-proxy
kube-scheduler
```

### 15.3. 🔎 Kiểm tra API readiness

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích. Develop: `172.17.79.90`.

```bash
kubectl get --raw='/readyz?verbose'
```

Kết quả đúng

Cuối output có:

```text
readyz check passed
```

### 15.4. 🔎 Kiểm tra etcd

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig; nếu thao tác HA thì kiểm tra cả 3 control-plane.

```bash
kubectl -n kube-system get pods -l component=etcd -o wide
```

Kết quả mong đợi:

```text
NAME READY STATUS NODE
etcd-sevago-dev-k8s-master-1 1/1 Running sevago-dev-k8s-master-1
etcd-sevago-dev-k8s-master-2 1/1 Running sevago-dev-k8s-master-2
etcd-sevago-dev-k8s-master-3 1/1 Running sevago-dev-k8s-master-3
```

### 15.5. ⚖️ Kiểm tra HAProxy backend

> **📍 Thực hiện tại:** Control-plane-1 của Develop: `172.17.79.90`.

Kiểm tra readiness trực tiếp của từng kube-apiserver:

```bash
for IP in 172.17.79.90 172.17.79.91 172.17.79.92; do
  echo "===== ${IP}:6444 ====="
  curl -sk --connect-timeout 2 --max-time 3 "https://${IP}:6444/readyz" && echo || echo "FAIL"
done
```

Kết quả khi cả 3 master khỏe:

```text
===== 172.17.79.90:6444 =====
ok
===== 172.17.79.91:6444 =====
ok
===== 172.17.79.92:6444 =====
ok
```

Kiểm tra qua VIP nhiều lần để xác nhận HAProxy không route request vào kube-apiserver unhealthy:

```bash
for i in $(seq 1 20); do
  printf "%s  " "$(date '+%F %T')"
  curl -sk --connect-timeout 2 --max-time 3 https://172.17.79.159:6443/readyz && echo || echo "FAIL"
  sleep 1
done
```

Kết quả đúng:

```text
ok
ok
ok
...
```

Nếu một master bị lỗi `/readyz`, HAProxy sẽ đánh backend đó `DOWN` sau 2 lần check fail và chỉ route request vào các master còn healthy. VIP `172.17.79.159:6443` vẫn phải trả `ok` liên tục.

### 15.6. 🔎 Kiểm tra taint master

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích. Develop: `172.17.79.90`.

```bash
kubectl describe node sevago-dev-k8s-master-1 | grep -i taints
kubectl describe node sevago-dev-k8s-master-2 | grep -i taints
kubectl describe node sevago-dev-k8s-master-3 | grep -i taints
```

Kết quả đúng

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

Không xóa taint nếu sẽ có worker node riêng.

### 15.7. 🔎 Kiểm tra Etcd Endpoint

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích. Develop: `172.17.79.90`.

```bash
ETCD_POD="etcd-$(hostname)"
kubectl -n kube-system exec "$ETCD_POD" -- \
  etcdctl \
  --endpoints=https://172.17.79.90:2379,https://172.17.79.91:2379,https://172.17.79.92:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key endpoint health
```

Kết quả mong đợi:

```text
https://172.17.79.90:2379 is healthy
https://172.17.79.91:2379 is healthy
https://172.17.79.92:2379 is healthy
```

Kiểm tra trạng thái và leader:

```bash
kubectl -n kube-system exec "$ETCD_POD" -- \
  etcdctl \
  --endpoints=https://172.17.79.90:2379,https://172.17.79.91:2379,https://172.17.79.92:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key endpoint status --write-out=table
```

Kết quả mong đợi:

```text
Có đủ ba endpoint etcd.
Cả ba endpoint không có lỗi.
Một endpoint có IS LEADER = true.
Hai endpoint còn lại có IS LEADER = false.
RAFT TERM của ba member giống nhau
```

### 15.8. ⚖️ Kiểm tra Keepalived Failover

> **📍 Thực hiện tại:** Cả 3 control-plane của cluster đích. Develop: `172.17.79.90`, `.91`, `.92`.

Xác định Master đang giữ VIP:

```bash
ip addr | grep 172.17.79.159
```

Trên một Master khác, theo dõi API VIP:

```bash
watch -n 1 'nc -zv 172.17.79.159 6443'
```

Trên Master đang giữ VIP, tạm dừng Keepalived:

```bash
sudo systemctl stop keepalived
```

Kiểm tra VIP đã chuyển sang Master khác:

```bash
ip addr | grep 172.17.79.159
```

Kiểm tra Kubernetes API:

```bash
kubectl get --raw='/readyz'
```

Kết quả mong đợi:

```text
ok
```

Khởi động lại Keepalived:

```bash
sudo systemctl start keepalived
sudo systemctl is-active keepalived
```

Kết quả mong đợi:

```text
active
```
