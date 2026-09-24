# ☸️ Bước 02 - Cài đặt môi trường chung cho Kubernetes

## 1. 🎯 Mục đích

File này chuẩn hóa OS, runtime và Kubernetes package trên toàn bộ control-plane/worker của một cluster trước khi bootstrap. Cùng quy trình được dùng cho cả 5 cluster; IP/hostname minh họa là Develop.

File này chuẩn bị môi trường chung trên toàn bộ máy trong cluster:

```text
3 master + 3 worker node
```

Baseline khuyến nghị:

```text
Ubuntu 24.04 LTS
Kubernetes v1.36.x
containerd
cgroup v2
Calico CNI
```

---

## 2. ✅ Checklist nội dung

```text
[ ] Hostname đúng trên từng node
[ ] /etc/hosts có API VIP và node hostname
[ ] Swap đã tắt
[ ] overlay và br_netfilter đã load
[ ] sysctl network OK
[ ] cgroup v2 đang bật
[ ] containerd active running
[ ] SystemdCgroup = true
[ ] containerd registry config_path đã bật
[ ] kubeadm/kubelet/kubectl đã cài
[ ] Kubernetes package đã hold
[ ] unattended-upgrades đã tắt hoặc kiểm soát
[ ] netcat/jq/debug tools đã có
```

---

## 3. 📌 Mục tiêu

Sau bước này, mỗi máy cần đạt:

- Swap đã tắt.
- Allow VLAN toàn bộ k8s
- Kernel module cho container networking đã bật.
- Sysctl network đã cấu hình.
- cgroup v2 hoạt động.
- `containerd` đã cài và dùng `SystemdCgroup = true`.
- `containerd` đã bật `config_path` để dùng `/etc/containerd/certs.d` cho registry về sau.
- `kubeadm`, `kubelet`, `kubectl` đã cài cùng minor version.
- Kubernetes package đã hold version.
- Auto upgrade đã tắt hoặc được kiểm soát.

---

## 4. 🧱 Mở Firewall Kubernetes nội bộ

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích. Develop: `172.17.79.90–172.17.79.95`.

Hệ thống hiện tại trust private subnet dành riêng cho Kubernetes:

```text
172.17.79.0/24
```

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw allow from 172.17.79.0/24 comment 'K8s'
sudo ufw enable
sudo ufw status numbered
```

Rule này cho phép traffic nội bộ cần thiết: Kubernetes API, HAProxy backend, etcd, kubelet, VRRP và Calico.

### 4.1. 🔎 Kiểm tra

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích. Develop: `172.17.79.90–172.17.79.95`.

```bash
sudo ufw status numbered
```

Kết quả mong đợi:

```text
ALLOW IN    172.17.79.0/24
```

> Chỉ trust private subnet Kubernetes. Không public các port `6443`, `6444`, `2379`, `2380`, `10250` hoặc NodePort ra Internet.

---

## 5. 🔎 Kiểm tra hostname trên từng node

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích. Develop: `172.17.79.90–172.17.79.95`.

Kiểm tra hostname:

```bash
hostnamectl --static
```

Kết quả phải tương ứng với từng máy (tức trùng tên):

```text
sevago-dev-k8s-master-1
sevago-dev-k8s-master-2
sevago-dev-k8s-master-3
sevago-dev-k8s-node-1
sevago-dev-k8s-node-2
sevago-dev-k8s-node-3
```

Chỉ đặt lại hostname nếu kết quả chưa đúng:

```bash
sudo hostnamectl set-hostname <hostname-cua-node>
```

Ví dụ trên master-1:

```bash
sudo hostnamectl set-hostname sevago-dev-k8s-master-1
```

Kiểm tra API VIP:

```bash
nc -vz 172.17.79.159 6443
```

> Cluster sử dụng trực tiếp `172.17.79.159:6443` làm
> `controlPlaneEndpoint`, vì vậy không cần khai báo API VIP hoặc hostname
> của các node trong `/etc/hosts`.

---

## 6. 📌 Tắt swap

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích. Develop: `172.17.79.90–172.17.79.95`.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Test:

```bash
free -h
swapon --show
```

Kết quả đúng:

```text
Swap: 0B
```

và `swapon --show` không có dòng nào.

---

## 7. 📌 Load kernel modules

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích. Develop: `172.17.79.90–172.17.79.95`.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

Test:

```bash
lsmod | grep -E 'overlay|br_netfilter'
```

---

## 8. 🌐 Sysctl cho Kubernetes network

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích. Develop: `172.17.79.90–172.17.79.95`.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

Test:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

Kết quả đúng:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

---

## 9. 🔎 Kiểm tra cgroup v2

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích. Develop: `172.17.79.90–172.17.79.95`.

```bash
stat -fc %T /sys/fs/cgroup
```

Kết quả đúng:

```text
cgroup2fs
```

---

## 10. ⚙️ Cài containerd

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích. Develop: `172.17.79.90–172.17.79.95`.

```bash
sudo apt update
sudo apt install -y containerd
```

Tạo config mặc định:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

Bật systemd cgroup:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Bật `config_path` cho registry certs/mirror:

```bash
sudo grep -q 'config_path = "/etc/containerd/certs.d"' /etc/containerd/config.toml || sudo sed -i '/\[plugins."io.containerd.grpc.v1.cri".registry\]/a\      config_path = "/etc/containerd/certs.d"' /etc/containerd/config.toml
```

Restart:

```bash
sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

Test:

```bash
systemctl is-active containerd
grep SystemdCgroup /etc/containerd/config.toml
grep -n 'config_path' /etc/containerd/config.toml
```

Kết quả đúng:

```text
active
SystemdCgroup = true
config_path = "/etc/containerd/certs.d"
```

---

## 11. ⚙️ Cài kubeadm, kubelet, kubectl

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích. Develop: `172.17.79.90–172.17.79.95`.

Khai báo minor version:

```bash
export K8S_MINOR=v1.36
```

Cài package nền:

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
```

Add Kubernetes key:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add repo:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Cài:

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Test:

```bash
kubeadm version
kubectl version --client=true
apt-mark showhold | grep -E 'kubeadm|kubelet|kubectl'
```

---

## 12. 📌 Tắt unattended-upgrades hoặc kiểm soát upgrade

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích. Develop: `172.17.79.90–172.17.79.95`.

Với cụm `sevago-develop`, không nên để OS tự upgrade kubelet/container runtime ngoài kế hoạch.

```bash
sudo systemctl stop unattended-upgrades || true
sudo systemctl disable unattended-upgrades || true
```

Test:

```bash
systemctl is-enabled unattended-upgrades || true
systemctl is-active unattended-upgrades || true
```

Kết quả mong muốn:

```text
disabled
inactive
```

> Nếu tổ chức bắt buộc dùng unattended-upgrades, cần exclude Kubernetes/containerd package và có lịch bảo trì riêng.

---

## 13. ⚙️ Cài công cụ debug cơ bản

> **📍 Thực hiện tại:** Tất cả control-plane và worker của cluster đích. Develop: `172.17.79.90–172.17.79.95`.

```bash
sudo apt update
sudo apt install -y netcat-openbsd jq bash-completion conntrack socat ipset ipvsadm
```

Bật completion nếu cần:

```bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
```
