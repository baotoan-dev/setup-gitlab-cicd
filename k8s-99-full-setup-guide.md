# Quy trình Setup Kubernetes 5-Cluster (Sevago)

> Tổng hợp từ 22 file tài liệu `k8s-00` → `k8s-21`. Baseline version chốt tại 2026-09-03: **Kubernetes v1.36.x, Calico v3.32.1, MetalLB v0.16.1, Rancher v2.15.0**. Không dùng Kubernetes v1.37 vì Calico v3.32 chỉ được test chính thức với v1.34–1.36.
>
> ⚠️ **Cảnh báo lifecycle quan trọng nhất:** Ingress NGINX upstream đã **retired từ 3/2026**, không còn release/patch bảo mật. Toàn bộ bước Ingress NGINX trong tài liệu này (bước 6) là **legacy để duy trì kiến trúc hiện hữu** — không phải khuyến nghị cho cluster mới. Hướng migration dài hạn: Gateway API + controller còn support (Envoy Gateway, Kong, NGINX Gateway Fabric).

## Kiến trúc tổng thể

| Môi trường | Cluster ID | Vùng | Namespace app | Kubeconfig Jenkins |
|---|---|---|---|---|
| Develop | `sevago-develop` | Local | `sevago-develop` | `config-develop` |
| Staging | `sevago-staging-local` | Local | `sevago-staging` | `config-staging-local` |
| Staging | `sevago-staging-dmz` | DMZ | `sevago-staging` | `config-staging-dmz` |
| Production | `sevago-production-local` | Local | `sevago-production` | `config-production-local` |
| Production | `sevago-production-dmz` | DMZ | `sevago-production` | `config-production-dmz` |

Hạ tầng dùng chung 1 lần cho toàn hệ thống (không lặp theo cluster): **GitLab** (`172.17.79.10`), **Jenkins** (`172.17.79.21`), **Harbor** (`172.17.79.20`).

Mỗi bước dưới đây được **lặp lại độc lập cho từng cluster** (trừ khi ghi rõ dùng chung). Ví dụ IP dùng minh họa là của cluster Develop (`172.17.79.90–95`, VIP `172.17.79.159`).

---

## Bước 1 — Cài đặt môi trường chung (OS/runtime)

**File nguồn:** `k8s-02-install-common.md`
**Thực hiện trên:** cả 3 control-plane + 3 worker của cluster đích.

1. Mở firewall, chỉ trust subnet nội bộ K8s + public 22/80/443:
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
> ⚠️ Không public port `6443, 6444, 2379, 2380, 10250` hoặc NodePort ra Internet.

2. Đặt hostname đúng chuẩn (`sevago-dev-k8s-master-1/2/3`, `-node-1/2/3`):
```bash
hostnamectl --static
sudo hostnamectl set-hostname <hostname-cua-node>
nc -vz 172.17.79.159 6443
```

3. Tắt swap:
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
free -h        # kỳ vọng Swap: 0B
swapon --show  # kỳ vọng rỗng
```

4. Load kernel module:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

5. Sysctl network:
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

6. Kiểm tra cgroup v2: `stat -fc %T /sys/fs/cgroup` → phải là `cgroup2fs`.

7. Cài containerd, bật SystemdCgroup + config_path cho registry cert:
```bash
sudo apt update && sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo grep -q 'config_path = "/etc/containerd/certs.d"' /etc/containerd/config.toml || \
  sudo sed -i '/\[plugins."io.containerd.grpc.v1.cri".registry\]/a\      config_path = "/etc/containerd/certs.d"' /etc/containerd/config.toml
sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

8. Cài kubeadm/kubelet/kubectl (pin minor version, hold version):
```bash
export K8S_MINOR=v1.36
sudo apt install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

9. Tắt unattended-upgrades:
```bash
sudo systemctl stop unattended-upgrades || true
sudo systemctl disable unattended-upgrades || true
```

10. Cài công cụ debug: `netcat-openbsd jq bash-completion conntrack socat ipset ipvsadm`.

**Kiểm tra:** `systemctl is-active containerd` = active; `grep SystemdCgroup` = true; `apt-mark showhold` liệt kê đủ 3 package.

---

## Bước 2 — HA Control Plane + Calico

**File nguồn:** `k8s-03-control-plane-ha-calico.md`
**Thực hiện trên:** 3 control-plane (`.90`, `.91`, `.92`).

1. Cài HAProxy + Keepalived (cả 3 master): `sudo apt install -y haproxy keepalived netcat-openbsd`

2. Cấu hình HAProxy (`/etc/haproxy/haproxy.cfg`, giống nhau trên cả 3 master), health-check qua port 6444:
```
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
```
Kiểm tra: `sudo haproxy -c -f /etc/haproxy/haproxy.cfg` → `Configuration file is valid`.

3. Cấu hình Keepalived riêng từng master, VIP `172.17.79.159/24`, `virtual_router_id 79`:
   - Master-1 (`.90`): `state MASTER`, `priority 110`
   - Master-2 (`.91`): `state BACKUP`, `priority 100`
   - Master-3 (`.92`): `state BACKUP`, `priority 90`
   - Cùng dùng `track_script { chk_haproxy }` (script check `pidof haproxy`, `weight -20`)

4. Start service (cả 3 master):
```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl enable --now haproxy keepalived
sudo systemctl reload haproxy
sudo systemctl restart keepalived
```

5. Kiểm tra VIP chỉ có ở 1 master: `ip addr | grep 172.17.79.159`

6. Khởi tạo control plane đầu tiên (**chỉ master-1**):
```bash
sudo kubeadm init --control-plane-endpoint=172.17.79.159:6443 \
  --apiserver-advertise-address=172.17.79.90 --apiserver-bind-port=6444 \
  --pod-network-cidr=192.168.0.0/16 --upload-certs
```
Lưu lại 2 lệnh join in ra: 1 có `--control-plane` (cho master-2/3), 1 không (cho worker).

7. Cấu hình kubectl (master-1):
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
chmod 600 $HOME/.kube/config
```

8. Cài Calico **chỉ 1 lần** (master-1):
```bash
export CALICO_VERSION=v3.32.1
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/calico.yaml
```
> ⚠️ Không chạy lại lệnh này trên master-2/3.

9. Join master-2 và master-3:
```bash
sudo kubeadm join 172.17.79.159:6443 \
  --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane --certificate-key <CERTIFICATE_KEY> \
  --apiserver-advertise-address=172.17.79.91 --apiserver-bind-port=6444
```

10. Nếu certificate-key hết hạn:
```bash
sudo kubeadm init phase upload-certs --upload-certs
sudo kubeadm token create --print-join-command
```

**Kiểm tra:**
- `kubectl get nodes -o wide` → 3 master `Ready control-plane`
- `kubectl get pods -n kube-system -o wide` → calico-node, coredns, etcd, apiserver... đều `Running`
- `kubectl get --raw='/readyz?verbose'` → `readyz check passed`
- Etcd: `etcdctl ... endpoint health` → cả 3 `is healthy`
- Test failover: `sudo systemctl stop keepalived` trên master giữ VIP → VIP chuyển máy khác, API vẫn `ok`

---

## Bước 3 — Join Worker + kiểm tra

**File nguồn:** `k8s-04-join-worker-firewall-test.md`
**Thực hiện trên:** 3 worker (`.93`, `.94`, `.95`) + control-plane-1.

1. Kiểm tra API từ worker trước khi join:
```bash
curl -k https://172.17.79.159:6443
curl -k https://172.17.79.159:6443/healthz
```

2. Tạo join command (control-plane-1): `sudo kubeadm token create --print-join-command`

3. Join từng worker:
```bash
sudo kubeadm join 172.17.79.159:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

4. Xác nhận: `kubectl get nodes -o wide` → 3 node `Ready`

5. Kiểm tra kubelet (10250 — `401 Unauthorized` là bình thường):
```bash
sudo ss -lntp | grep 10250          # trên worker
curl -k -I https://172.17.79.93:10250/healthz   # từ master
```

6. Gán role worker (tùy chọn):
```bash
kubectl label node sevago-dev-k8s-node-1 node-role.kubernetes.io/worker=worker
```

7. Test schedule + NodePort (namespace tạm):
```bash
kubectl create namespace cluster-test
kubectl create deployment test-nginx --image=nginx:stable -n cluster-test
kubectl expose deployment test-nginx --port=80 --type=NodePort -n cluster-test
kubectl get svc test-nginx -n cluster-test
for ip in 172.17.79.93 172.17.79.94 172.17.79.95; do curl "http://${ip}:<NODE_PORT>"; done
```

8. Dọn dẹp: `kubectl delete namespace cluster-test`
> ⚠️ Không test bằng namespace thật rồi xóa (ví dụ `dev`).

**Kiểm tra:** node `Ready`, pod test `Running 1/1`, NodePort trả `Welcome to nginx!`.

---

## Bước 4 — MetalLB

**File nguồn:** `k8s-05-metallb.md`
**Thực hiện trên:** control-plane-1 / máy có kubeconfig.

> ⚠️ Pool IP phải cùng subnet Layer2 với node, không trùng IP node/VIP/gateway/DHCP. Một số VPS không hỗ trợ ARP cho IP phụ → cần BGP/routed IP thay thế.

1. Kiểm tra cluster: `kubectl get nodes -o wide`, `kubectl get pods -n kube-system`

2. Cài MetalLB (pin version):
```bash
export METALLB_VERSION=v0.16.1
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/${METALLB_VERSION}/config/manifests/metallb-native.yaml
kubectl get pods -n metallb-system -w
```

3. Tạo pool + advertisement:
```yaml
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
```
```bash
kubectl apply -f metallb-pool.yaml
```

4. Test Service LoadBalancer:
```bash
kubectl create namespace metallb-test
kubectl create deployment test-nginx --image=nginx:stable -n metallb-test
kubectl expose deployment test-nginx --port=80 --type=LoadBalancer -n metallb-test
kubectl get svc test-nginx -n metallb-test -w
curl http://172.17.79.160
```

5. `externalTrafficPolicy`: `Cluster` (mặc định, khuyến nghị cluster nhỏ) hoặc `Local` (giữ client IP, cần Ingress Pod đúng node).

6. Dọn dẹp: `kubectl delete namespace metallb-test`

**Kiểm tra:** speaker/controller `Running`; Service nhận External IP trong pool; curl trả nội dung nginx.

---

## Bước 5 — Ingress NGINX (legacy)

**File nguồn:** `k8s-06-ingress-nginx.md`
**Thực hiện trên:** control-plane-1 / máy có kubeconfig.

> ⚠️ Ingress NGINX upstream đã retired 3/2026. Chỉ dùng để duy trì kiến trúc hiện hữu, cần kế hoạch migration sang Gateway API.

1. Cài Helm nếu chưa có: `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`

2. Cài Ingress NGINX bằng Helm (LoadBalancer, timeout đã tinh chỉnh, tắt snippet annotation):
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.externalTrafficPolicy=Cluster \
  --set controller.config.allow-snippet-annotations="false" \
  --set controller.config.proxy-connect-timeout="5" \
  --set controller.config.proxy-read-timeout="60" \
  --set controller.config.proxy-send-timeout="60"
```
> Sau kiểm thử, pin `--version <CHART_VERSION>`.

3. Kiểm tra: `kubectl get pods -n ingress-nginx`, `kubectl get svc -n ingress-nginx` → EXTERNAL-IP trong pool MetalLB, port 80/443.

4. Test routing bằng app mẫu (namespace `ingress-test`):
```bash
kubectl create namespace ingress-test
kubectl create deployment test-nginx --image=nginx:stable -n ingress-test
kubectl expose deployment test-nginx --port=80 --name=test-nginx-svc -n ingress-test
```
Tạo Ingress test với host `test.dev.local`, path `/` → `test-nginx-svc:80`.

5. Map domain test + curl:
```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
sudo sh -c 'echo "172.17.79.160 test.dev.local" >> /etc/hosts'
curl http://test.dev.local
```

6. TLS: 3 cách — terminate ở reverse proxy ngoài / terminate tại Ingress bằng TLS Secret (xem Bước 10) / cert-manager.

7. Cleanup: `kubectl delete namespace ingress-test`

**Kiểm tra:** Pod Running, Service LoadBalancer có EXTERNAL-IP, curl domain test trả "Welcome to nginx".

---

## Bước 6 — Rancher

**File nguồn:** `k8s-07-rancher.md`
**Thực hiện trên:** control-plane-1 / máy có kubeconfig, UI qua browser.

1. Thêm Helm repo + kiểm tra version tương thích (Rancher v2.15.0 yêu cầu K8s `< 1.37.0-0`):
```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

2. Tạo namespace + TLS Secret:
```bash
kubectl create namespace cattle-system
kubectl create secret tls tls-rancher-ingress --cert=/home/devops/certs/selfsigned.crt --key=/home/devops/certs/selfsigned.key -n cattle-system
kubectl create secret generic tls-ca --from-file=cacerts.pem=/home/devops/certs/root-ca.crt -n cattle-system
```

3. `rancher-values.yaml`:
```yaml
hostname: develop.k8s.sevago.local
replicas: 3
privateCA: true
ingress:
  ingressClassName: nginx
  tls:
    source: secret
```

4. Cài Rancher (nhập bootstrap password tương tác, **không** lưu Git):
```bash
read -rsp "Rancher bootstrap password: " RANCHER_BOOTSTRAP_PASSWORD; echo
helm upgrade --install rancher rancher-stable/rancher --namespace cattle-system \
  --version 2.15.0 -f rancher-values.yaml \
  --set-string bootstrapPassword="$RANCHER_BOOTSTRAP_PASSWORD" --wait --timeout 20m
unset RANCHER_BOOTSTRAP_PASSWORD
```

5. Kiểm tra: `kubectl rollout status deployment/rancher -n cattle-system --timeout=20m`

6. Truy cập `https://develop.k8s.sevago.local` → login `admin` + bootstrap password → đổi mật khẩu ngay.

**Cảnh báo:** không lưu bootstrap/mật khẩu thật trong Git; không dùng `admin` cho việc hàng ngày; backup trước khi upgrade.

**Kiểm tra:** Pod `rancher-*` Running 1/1; image = `rancher/rancher:v2.15.0`; cluster `local` Status Active.

---

## Bước 7 — Rancher: tạo account & phân quyền

**File nguồn:** `k8s-08-rancher-account.md`
**Thực hiện:** hoàn toàn qua UI Rancher.

1. Tạo Local User: Username, Display Name, Password. Bật "Ask user to change password on next login".

2. Chọn **1** Global Permission:
   - **Administrator** — chỉ cấp cho DevOps Administrator
   - **Standard User** — DevOps/Tech Lead
   - **User-Base** — Developer/Tester/Support (chỉ đăng nhập)

3. Built-in Permissions — khuyến nghị **không chọn thêm** nếu không phải Administrator.

> Lưu ý: Global Permissions/Built-in chỉ quyết định quyền quản trị Rancher, không phải quyền thao tác resource K8s (xem Bước 8).

---

## Bước 8 — Rancher: Project & Namespace

**File nguồn:** `k8s-09-rancher-project-namespace.md`

1. Kiểm tra namespace hiện có: `kubectl get ns` — phân loại: hệ thống K8s (`kube-*`), hệ thống Rancher (`cattle-*, fleet-*, local`), hạ tầng (`ingress-nginx, metallb-system, cicd`), ứng dụng (`sevago-develop`).

2. Tạo Project qua UI (`Cluster Explorer → Projects/Namespaces → Create Project`), ví dụ Project **Sevago**.

3. Di chuyển namespace ứng dụng vào Project (`Move`).

4. Thêm member + Project Role: **Project Owner** (toàn quyền + quản lý member), **Project Member** (deploy/scale/log/exec, không sửa config), **Read Only** (chỉ xem).

> ⚠️ Local User không hỗ trợ phân quyền theo Group — phải add từng user thủ công. Nên chuyển sang LDAP/Keycloak khi có auth tập trung.

---

## Bước 9 — Harbor CA + containerd config

**File nguồn:** `k8s-10-harbor-registry-containerd.md`
**Thực hiện trên:** **TẤT CẢ** node (3 master + 3 worker) — thiếu ở node nào sẽ gây `ImagePullBackOff` khi Pod schedule tới đó.

```bash
# 1. DNS/hosts tạm
sudo grep -q 'registry.sevatech.local' /etc/hosts || echo '172.17.79.20 registry.sevatech.local' | sudo tee -a /etc/hosts
getent hosts registry.sevatech.local

# 2. Cài CA vào OS trust store
sudo cp /home/devops/root-ca.registry.crt /usr/local/share/ca-certificates/root-ca.registry.crt
sudo update-ca-certificates
curl -v https://registry.sevatech.local/v2/   # kỳ vọng SSL verify ok + 401 Unauthorized

# 3. Containerd certs.d
sudo mkdir -p /etc/containerd/certs.d/registry.sevatech.local
sudo cp /home/devops/root-ca.registry.crt /etc/containerd/certs.d/registry.sevatech.local/ca.crt
sudo tee /etc/containerd/certs.d/registry.sevatech.local/hosts.toml > /dev/null <<'EOF'
server = "https://registry.sevatech.local"
[host."https://registry.sevatech.local"]
  capabilities = ["pull", "resolve"]
  ca = "/etc/containerd/certs.d/registry.sevatech.local/ca.crt"
EOF
sudo systemctl restart containerd

# 4. Test pull
sudo ctr -n k8s.io images pull registry.sevatech.local/library/nginx:stable
```

**Kiểm tra:** `getent hosts` đúng IP; curl không lỗi SSL; `ctr pull` thành công không lỗi x509.

---

## Bước 10 — Private image pull secret (K8s side)

**File nguồn:** `k8s-11-pull-image-private-registry.md`
**Điều kiện:** Bước 9 đã hoàn tất trên mọi node.
**Thực hiện trên:** control-plane-1, đúng namespace môi trường.

```bash
# 1. Namespace
kubectl create namespace sevago-develop --dry-run=client -o yaml | kubectl apply -f -

# 2. Registry secret (dùng Harbor Robot Account, không hardcode)
read -rsp "Harbor Robot password: " HARBOR_ROBOT_PASSWORD; echo
kubectl create secret docker-registry registry-cred \
  --docker-server=registry.sevatech.local \
  --docker-username='robot$sevago-develop+sevago-develop' \
  --docker-password="$HARBOR_ROBOT_PASSWORD" -n sevago-develop \
  --dry-run=client -o yaml | kubectl apply -f -
unset HARBOR_ROBOT_PASSWORD

# 3. ServiceAccount
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: registry-pull-sa
  namespace: sevago-develop
imagePullSecrets:
  - name: registry-cred
EOF

# 4. Test pull bằng pod tạm (cleanup sau khi test, giữ lại secret/SA)
kubectl apply -f pull-test-pod.yaml
kubectl logs pull-test -n sevago-develop   # kỳ vọng: OK
kubectl delete pod pull-test -n sevago-develop --ignore-not-found
```

Khi deploy thật, thêm `serviceAccountName: registry-pull-sa` vào `spec.template.spec` của Deployment/Helm chart.

**Kiểm tra:** Pod test `Completed`/`Running`, log = `OK`, không `ImagePullBackOff`/x509/unauthorized.

---

## Bước 11 — CoreDNS nội bộ

**File nguồn:** `k8s-12-pod-coredns.md`
**Thực hiện trên:** control-plane-1.

```bash
# Backup
kubectl -n kube-system get configmap coredns -o yaml > coredns-backup.yaml

# Sửa ConfigMap
kubectl -n kube-system edit configmap coredns
```
Thêm block song song với `.:53`:
```text
sevago.local:53 {
    errors
    cache 30
    forward . 172.17.90.23
}
```
```bash
kubectl rollout restart deployment/coredns -n kube-system
kubectl rollout status deployment/coredns -n kube-system

# Test
kubectl run dns-test --image=busybox:1.36 --restart=Never -n default --command -- sleep 3600
kubectl exec dns-test -- nslookup develop.account.sevago.local
kubectl delete pod dns-test -n default --ignore-not-found
```

> Nguyên tắc: domain mới chỉ tạo record trên DNS công ty, không sửa CoreDNS mỗi lần.

**Kiểm tra:** `nslookup` trả đúng Name/Address.

---

## Bước 12 — Health check tổng thể (gate trước khi deploy)

**File nguồn:** `k8s-13-check-cluster-health.md`

```bash
kubectl cluster-info
kubectl get --raw='/readyz?verbose'
kubectl get nodes -o wide                 # 3 master + 3 worker Ready
kubectl get pods -A                       # không CrashLoopBackOff/ImagePullBackOff/Pending
curl -k https://172.17.79.159:6443/healthz
kubectl get pods -n metallb-system
kubectl get pods -n ingress-nginx; kubectl get svc -n ingress-nginx
kubectl rollout status deployment/rancher -n cattle-system --timeout=5m
kubectl get secret registry-cred -n sevago-develop
kubectl -n kube-system get configmap coredns -o yaml   # xác nhận block sevago.local
helm list -A
```

Đây là **checklist nghiệm thu** — chỉ tiếp tục sang CI/CD (Bước 13+) khi mọi mục pass.

---

## Bước 13 — Jenkins: kubectl/Helm + RBAC per-cluster

**File nguồn:** `k8s-14-jenkins-kubectl-helm.md`
**Thực hiện trên:** VPS Jenkins (`172.17.79.21`) + control-plane-1 cluster đích.

> Nguyên tắc: **1 cluster = 1 ServiceAccount = 1 kubeconfig riêng**, RBAC theo namespace, **không** dùng `admin.conf`/cluster-admin/`ClusterRoleBinding`.

1. Kiểm tra công cụ trong container Jenkins: `kubectl version --client`, `helm version`, `docker version`, `git --version`. Nếu thiếu, build lại image từ `Dockerfile.build.custom`.

2. Trust SSH host key GitLab (port 2222):
```bash
docker exec -u root jenkins sh -c '
mkdir -p "$JENKINS_HOME/.ssh"
ssh-keyscan -p 2222 gitlab.sevatech.local >> "$JENKINS_HOME/.ssh/known_hosts"
chown -R jenkins:jenkins "$JENKINS_HOME/.ssh"
chmod 700 "$JENKINS_HOME/.ssh"; chmod 600 "$JENKINS_HOME/.ssh/known_hosts"
'
```

3. Tạo namespace `cicd` + ServiceAccount (control-plane-1):
```bash
kubectl create namespace cicd --dry-run=client -o yaml | kubectl apply -f -
kubectl create serviceaccount jenkins-develop -n cicd --dry-run=client -o yaml | kubectl apply -f -
```

4. Tạo `ClusterRole jenkins-namespace-deployer` (quyền CRUD trên pods/services/deployments/ingresses... — **không** dùng ClusterRoleBinding).

5. `RoleBinding` theo từng namespace app:
```bash
for ns in sevago-develop sevaedu-develop sevaretail-develop; do
  cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-develop
  namespace: ${ns}
subjects:
- kind: ServiceAccount
  name: jenkins-develop
  namespace: cicd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-namespace-deployer
EOF
done
```

6. Tạo token lâu dài (Secret, **không** dùng `kubectl create token` vì có hạn):
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-develop-token
  namespace: cicd
  annotations:
    kubernetes.io/service-account.name: jenkins-develop
type: kubernetes.io/service-account-token
EOF
TOKEN=$(kubectl -n cicd get secret jenkins-develop-token -o jsonpath='{.data.token}' | base64 -d)
```

7. Tạo kubeconfig riêng, copy vào Jenkins:
```bash
scp /tmp/config-develop devops@172.17.79.21:/tmp/config-develop
sudo cp /tmp/config-develop jenkins-data/.kube/config-develop
sudo chown -R 1000:1000 jenkins-data/.kube
sudo chmod 600 jenkins-data/.kube/config-develop
docker compose -f build.yaml restart
```

**Kiểm tra:**
```bash
kubectl auth can-i patch deployments -n sevago-develop --as=system:serviceaccount:cicd:jenkins-develop   # yes
kubectl auth can-i get pods -n kube-system --as=system:serviceaccount:cicd:jenkins-develop   # no
kubectl auth can-i get nodes --as=system:serviceaccount:cicd:jenkins-develop   # no
```

---

## Bước 14 — TLS Secret

**File nguồn:** `k8s-15-kubernetes-tls-secret.md`

> TLS Secret là **namespace-scoped** — phải tạo riêng cho từng namespace/cluster dù dùng chung certificate. Không tham chiếu chéo Secret giữa Develop/Staging/Production.

Chuẩn hóa 2 tên Secret: `local-tls` (domain nội bộ `*.sevago.local`), `public-tls` (domain public `*.com.vn`).

1. Kiểm tra cert trước khi tạo Secret:
```bash
openssl x509 -in selfsigned.crt -noout -subject -issuer -dates
openssl x509 -in selfsigned.crt -noout -checkend 2592000   # còn hạn ít nhất 30 ngày
```

2. Đối chiếu cert khớp private key (SHA256 phải giống nhau):
```bash
openssl x509 -in selfsigned.crt -pubkey -noout | openssl pkey -pubin -outform DER | sha256sum
openssl pkey -in selfsigned.key -pubout -outform DER | sha256sum
```

3. Tạo Secret:
```bash
kubectl create secret tls local-tls --cert=selfsigned.crt --key=selfsigned.key \
  -n sevago-develop --dry-run=client -o yaml | kubectl apply -f -
```

**Kiểm tra:**
```bash
kubectl get secret -A | grep local-tls
curl -v https://develop.office.sevago.local/
openssl s_client -connect develop.office.sevago.local:443 -servername develop.office.sevago.local -showcerts
```

---

## Bước 15 — Jenkins CI/CD với Helm

**File nguồn:** `k8s-16-jenkins-cicd-helm.md`

Nguyên tắc thiết kế: **Jenkins quyết định business logic** (app type, domain, TLS secret, replicaCount, image, namespace), **Helm chỉ render manifest** thuần.

- App type theo tên repo: `*-grpc → grpc`, `*-be → be`, `*-admin → admin`, còn lại → `fe`.
- Domain: FE `develop.${BASE}.sevago.local`; Admin `develop.admin.${BASE}.sevago.local`; BE dùng domain FE/Admin `+ /api`; gRPC `develop.${BASE}-grpc.sevago.local`.
- TLS: domain nội bộ → `local-tls`; domain public → `public-tls`.
- 4 template ingress: `fe`, `be`, `realtime` (`/socket.io`, `/hub`), `grpc`.
- Replica: mặc định `2`; `webhook`/`cron-job` luôn `1`.

Tạo Harbor Credential trong Jenkins (`Manage Jenkins → Credentials`), Kind Username/password, ID `harbor-develop`.

Lệnh deploy chính (giữ pod lỗi để debug thay vì rollback im lặng):
```bash
helm upgrade --install "${REPO}" app-helm -n "${NAMESPACE}" -f "${VALUES_FILE}" \
  --set "app.type=${APP_TYPE}" --set "ingress.host=${INGRESS_HOST}" \
  --set "image.fullName=${IMAGE_FULL_NAME}" --set "ingress.tls.secretName=${TLS_SECRET_NAME}" \
  --wait --timeout 7m || {
    kubectl get pods -n "${NAMESPACE}" -l "app.kubernetes.io/instance=${REPO}" -o wide
    kubectl describe deployment "${REPO}" -n "${NAMESPACE}"
    exit 1
  }
```

> ⚠️ Không hardcode Harbor credential trong Webhook/Shell Script.

---

## Bước 16 — Jenkins Manual Job

**File nguồn:** `k8s-17-jenkins-cicd-manual-job.md`

Tạo Pipeline Job `sevago-develop-manual` (Active Choices Parameter) để build/deploy theo yêu cầu, không phụ thuộc webhook, branch cố định `develop`.

1. Cài plugin: Active Choices, Pipeline, Credentials Binding.
2. Credential `gitlab-token` (Secret text, scope `read_api`) + `harbor-develop` (Username/password).
3. Parameter `REPO_SELECT` dùng Groovy script gọi GitLab API (`/api/v4/projects?membership=true`) lấy danh sách repo trong `sevago/modules/`.
4. Pipeline tự resolve `COMMIT`, `COMMIT_AUTHOR`, `COMMIT_MESSAGE` bằng `git log`/`git rev-parse HEAD` (thay cho biến webhook cung cấp).
5. Dùng `--atomic --wait` + `kubectl rollout status` để pass/fail job.

---

## Bước 17 — API Gateway (tùy chọn)

**File nguồn:** `k8s-18-api-gateway-optional.md`

Chỉ thêm API Gateway (Kong/Envoy Gateway/APISIX) khi cần: auth tập trung, API key/quota, rate limit, request/response transform, analytics/developer portal — **không thêm chỉ để thay Ingress NGINX**.

3 mô hình:
- **A:** `Client → Ingress NGINX → API Gateway (internal) → Backend` — rollout dần, thêm 1 hop.
- **B:** `Client → MetalLB → API Gateway (LoadBalancer) → Backend` — Gateway là entrypoint trực tiếp, cần vận hành như thành phần critical.
- **C:** Gateway API (`GatewayClass/Gateway/HTTPRoute`) với Envoy Gateway/Kong/NGINX Gateway Fabric — hướng migration dài hạn khỏi Ingress NGINX.

---

## Bước 18 — Traffic flow (tài liệu tham chiếu)

**File nguồn:** `k8s-19-kubernetes-traffic-flow.md`

Luồng quản trị: `Admin/Jenkins/kubectl → API VIP (172.17.79.159:6443) → HAProxy+Keepalived → kube-apiserver`

Luồng ứng dụng: `Client → DNS → MetalLB External IP → node đang giữ IP → Service ingress-nginx (LoadBalancer) → Ingress Controller Pod → route theo Host/Path → Kubernetes Service (ClusterIP) → Pod`

Backend gọi Backend: **không qua** MetalLB/Ingress, gọi trực tiếp qua Service DNS (`http://office-be:8080`).

---

## Bước 19 — Worker inotify limit

**File nguồn:** `k8s-20-worker-inotify-limit-short.md`
**Thực hiện trên:** tất cả worker (có thể làm sớm, cùng lúc với Bước 1/3).

Tránh app dùng `FileSystemWatcher` (.NET, Node.js, Filebeat) bị `CrashLoopBackOff` do chạm giới hạn `fs.inotify.max_user_instances=128` mặc định.

```bash
sudo tee /etc/sysctl.d/99-kubernetes-inotify.conf <<EOF
fs.inotify.max_user_instances=1024
fs.inotify.max_user_watches=524288
fs.inotify.max_queued_events=32768
EOF
sudo sysctl --system
```

**Kiểm tra:** `sysctl fs.inotify.max_user_instances` = 1024; Pod không còn lỗi `configured user limit ... inotify instances`.

---

## Bước 20 — Beyla observability (bổ sung, không bắt buộc)

**File nguồn:** `k8s-21-beyla-application-observability.md`

Bước phụ trợ cho Grafana dashboard, dùng eBPF tự động thu HTTP telemetry, không sửa source code.

1. Kiểm tra prerequisite trên worker: kernel ≥ 5.8, BTF OK, tracefs OK.

2. Cài Beyla DaemonSet qua Helm (namespace `monitor`, pin version `1.16.10`):
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm upgrade --install beyla grafana/beyla --namespace monitor --version 1.16.10 \
  -f beyla-base.values.yaml -f beyla-develop.values.yaml --atomic --timeout 5m
```

3. Chỉ instrument `k8s_deployment_name: "*-be"` (loại trừ `*-fe`, `*-admin`, gRPC, background).

**Kiểm tra:** DaemonSet Ready đủ số worker; Prometheus Agent thấy target `health=up`; metric `http_server_request_duration_seconds_count` có dữ liệu trên Central Prometheus (`172.17.79.23`).

---

## Ghi nhớ khi lặp lại cho 5 cluster

1. **Không copy IP/hostname/pool giữa cluster** — mỗi cluster có network riêng.
2. **Không share Secret/TLS giữa cluster** dù dùng chung certificate — phải tạo Secret K8s riêng mỗi namespace/cluster.
3. **Jenkins: 1 cluster = 1 ServiceAccount = 1 kubeconfig riêng**, không dùng cluster-admin.
4. **Ingress NGINX là legacy** — với cluster mới, đánh giá Gateway API thay thế ngay từ Bước 5.
5. Thứ tự phụ thuộc lõi: **1→2→3→4→5→6-8→9-10→11→12 (gate)→13-16→17 (optional)→19 (có thể sớm hơn)→20 (sau khi có app thật)**.
