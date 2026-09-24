# Quy trình Setup Kubernetes 5-Cluster (Sevago)

> Tổng hợp từ 22 file tài liệu `k8s-00` → `k8s-21`. Baseline version chốt tại 2026-09-03: **Kubernetes v1.36.x, Calico v3.32.1, MetalLB v0.16.1, Rancher v2.15.0**.
>
> ⚠️ Ingress NGINX upstream đã **retired từ 3/2026**. Bước 5 trong tài liệu này là legacy, cần kế hoạch migration sang Gateway API.
>
> **Quy ước server (cluster Develop):** `master-1`=`172.17.79.90`, `master-2`=`172.17.79.91`, `master-3`=`172.17.79.92`, `worker-1`=`172.17.79.93`, `worker-2`=`172.17.79.94`, `worker-3`=`172.17.79.95`, `VIP API`=`172.17.79.159`, `Harbor`=`172.17.79.20`, `GitLab`=`172.17.79.10`, `Jenkins VPS`=`172.17.79.21`.

---

## Bước 1 — Cài đặt môi trường chung (OS/runtime)
**File nguồn:** `k8s-02-install-common.md` | **Thực hiện trên:** cả 6 node (master-1/2/3 + worker-1/2/3).

**[Server: cả 6 node] — Tác dụng: mở firewall, chỉ cho SSH/HTTP/HTTPS công khai + toàn bộ traffic nội bộ K8s.**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp; sudo ufw allow 80/tcp; sudo ufw allow 443/tcp
sudo ufw allow from 172.17.79.0/24
sudo ufw enable
```
> ⚠️ Không public port `6443,6444,2379,2380,10250` ra Internet.

**[Server: cả 6 node] — Tác dụng: đặt hostname đúng chuẩn.**
```bash
sudo hostnamectl set-hostname <hostname-cua-node>
```

**[Server: cả 6 node] — Tác dụng: tắt swap vĩnh viễn (kubelet yêu cầu).**
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

**[Server: cả 6 node] — Tác dụng: load kernel module `overlay`/`br_netfilter` cần cho container networking.**
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay; sudo modprobe br_netfilter
```

**[Server: cả 6 node] — Tác dụng: bật iptables nhìn thấy traffic bridge + IP forwarding.**
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

**[Server: cả 6 node] — Tác dụng: cài containerd, đồng bộ cgroup driver, bật đường dẫn cert cho registry riêng.**
```bash
sudo apt update && sudo apt install -y containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl enable --now containerd; sudo systemctl restart containerd
```

**[Server: cả 6 node] — Tác dụng: cài đúng version kubeadm/kubelet/kubectl và giữ nguyên version (không cho apt tự upgrade).**
```bash
export K8S_MINOR=v1.36
curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**Kiểm tra:** `systemctl is-active containerd` = active; `apt-mark showhold` đủ 3 package.

---

## Bước 2 — HA Control Plane + Calico
**File nguồn:** `k8s-03-control-plane-ha-calico.md` | **Thực hiện trên:** master-1, master-2, master-3.

**[Server: cả 3 master] — Tác dụng: cài HAProxy (load balancer API) + Keepalived (Virtual IP failover).**
```bash
sudo apt install -y haproxy keepalived
```

**[Server: cả 3 master] — Tác dụng: HAProxy lắng nghe VIP port 6443, phân phối tới kube-apiserver port 6444 của cả 3 master, tự loại backend chết.**
```
frontend k8s_api_frontend
    bind *:6443
    default_backend k8s_api_backend
backend k8s_api_backend
    balance roundrobin
    server master-1 172.17.79.90:6444 check
    server master-2 172.17.79.91:6444 check
    server master-3 172.17.79.92:6444 check
```

**[Server: cả 3 master] — Tác dụng: Keepalived bầu 1 máy giữ VIP `172.17.79.159` theo priority (master-1=110, master-2=100, master-3=90); máy giữ VIP tự động đổi khi máy hiện tại chết.**
```bash
sudo systemctl enable --now haproxy keepalived
```

**[Server: CHỈ master-1] — Tác dụng: khởi tạo cluster, khai báo endpoint chung là VIP (không phải IP riêng master-1).**
```bash
sudo kubeadm init --control-plane-endpoint=172.17.79.159:6443 \
  --apiserver-advertise-address=172.17.79.90 --apiserver-bind-port=6444 \
  --pod-network-cidr=192.168.0.0/16 --upload-certs
```

**[Server: master-1] — Tác dụng: cấp quyền kubectl cho user hiện tại.**
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**[Server: master-1 — CHỈ 1 LẦN] — Tác dụng: cài Calico CNI để Pod trên các node giao tiếp được với nhau. Đây là request gọi lên API server, không chạy lại trên master-2/3.**
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/calico.yaml
```

**[Server: master-2, master-3] — Tác dụng: join thành control-plane bổ sung, mỗi máy tự chạy kube-apiserver/etcd riêng (etcd cluster 3 node chịu lỗi 1 node).**
```bash
sudo kubeadm join 172.17.79.159:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane --certificate-key <KEY> --apiserver-advertise-address=<IP_MAY_NAY> --apiserver-bind-port=6444
```

**Kiểm tra:** `kubectl get nodes` → 3 master Ready; test failover bằng `sudo systemctl stop keepalived` trên máy giữ VIP → VIP tự chuyển máy khác.

---

## Bước 3 — Join Worker
**File nguồn:** `k8s-04-join-worker-firewall-test.md` | **Thực hiện trên:** worker-1/2/3 + master-1.

**[Server: master-1] — Tác dụng: sinh join command mới.**
```bash
sudo kubeadm token create --print-join-command
```

**[Server: worker-1, worker-2, worker-3] — Tác dụng: đăng ký worker vào cluster (không có `--control-plane` vì chỉ chạy Pod, không chạy control plane).**
```bash
sudo kubeadm join 172.17.79.159:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

**[Server: master-1] — Tác dụng: xác nhận worker Ready, test schedule Pod + NodePort.**
```bash
kubectl get nodes -o wide
kubectl create deployment test-nginx --image=nginx:stable -n cluster-test
kubectl expose deployment test-nginx --port=80 --type=NodePort -n cluster-test
```

**Kiểm tra:** node Ready; NodePort trả "Welcome to nginx!"; dọn `kubectl delete namespace cluster-test`.

---

## Bước 4 — MetalLB
**File nguồn:** `k8s-05-metallb.md` | **Thực hiện trên:** master-1.

**[Server: master-1] — Tác dụng: cài MetalLB — `controller` cấp IP, `speaker` (DaemonSet mọi node) quảng bá ARP.**
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml
```

**[Server: master-1] — Tác dụng: khai báo dải IP `172.17.79.160-169` MetalLB được cấp cho Service LoadBalancer, bật quảng bá Layer2.**
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata: {name: default-pool, namespace: metallb-system}
spec: {addresses: ["172.17.79.160-172.17.79.169"]}
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata: {name: default-l2, namespace: metallb-system}
spec: {ipAddressPools: ["default-pool"]}
```

**Kiểm tra:** tạo Service LoadBalancer test → nhận External IP trong pool → curl thành công → dọn namespace test.

---

## Bước 5 — Ingress NGINX (legacy)
**File nguồn:** `k8s-06-ingress-nginx.md` | **Thực hiện trên:** master-1.

**[Server: master-1] — Tác dụng: cài Ingress NGINX Controller, expose LoadBalancer nhận IP từ MetalLB, tắt snippet annotation (giảm rủi ro RCE), tinh chỉnh timeout.**
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.config.allow-snippet-annotations="false"
```

**Kiểm tra:** `kubectl get svc -n ingress-nginx` có EXTERNAL-IP; test route domain tạm qua `/etc/hosts` + curl; dọn namespace test.

---

## Bước 6 — Rancher
**File nguồn:** `k8s-07-rancher.md` | **Thực hiện trên:** master-1 + browser máy quản trị.

**[Server: master-1] — Tác dụng: tạo namespace + TLS Secret cho Rancher UI.**
```bash
kubectl create namespace cattle-system
kubectl create secret tls tls-rancher-ingress --cert=selfsigned.crt --key=selfsigned.key -n cattle-system
```

**[Server: master-1] — Tác dụng: cài Rancher server 3 replica, dùng biến môi trường tạm cho bootstrap password (không lưu Git).**
```bash
read -rsp "Rancher bootstrap password: " RANCHER_BOOTSTRAP_PASSWORD; echo
helm upgrade --install rancher rancher-stable/rancher --namespace cattle-system \
  --version 2.15.0 -f rancher-values.yaml --set-string bootstrapPassword="$RANCHER_BOOTSTRAP_PASSWORD"
unset RANCHER_BOOTSTRAP_PASSWORD
```

**[Máy quản trị, browser] — Tác dụng: đăng nhập lần đầu, đổi mật khẩu ngay.**
Truy cập `https://develop.k8s.sevago.local` → login `admin` + bootstrap password.

**Kiểm tra:** `kubectl rollout status deployment/rancher -n cattle-system`; cluster `local` Active.

---

## Bước 7 — Rancher: account & phân quyền
**File nguồn:** `k8s-08-rancher-account.md` | **Thực hiện:** UI Rancher (browser).

**[UI Rancher] — Tác dụng: tạo user local, ép đổi password lần đầu, gán đúng 1 Global Permission** (Administrator/Standard User/User-Base tùy vai trò).

---

## Bước 8 — Rancher: Project & Namespace
**File nguồn:** `k8s-09-rancher-project-namespace.md` | **Thực hiện:** master-1 (kiểm tra) + UI Rancher.

**[Server: master-1] — Tác dụng: rà soát namespace hiện có trước khi tổ chức Project.**
```bash
kubectl get ns
```
**[UI Rancher] — Tác dụng: tạo Project, di chuyển namespace app vào, thêm member theo role** (Project Owner/Member/Read Only).

---

## Bước 9 — Harbor CA + containerd config
**File nguồn:** `k8s-10-harbor-registry-containerd.md` | **Thực hiện trên:** TẤT CẢ 6 node.

**[Server: cả 6 node] — Tác dụng: trỏ domain Harbor về IP qua /etc/hosts, cài Root CA vào OS trust store.**
```bash
echo '172.17.79.20 registry.sevatech.local' | sudo tee -a /etc/hosts
sudo cp root-ca.registry.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

**[Server: cả 6 node] — Tác dụng: khai báo CA riêng cho containerd khi pull image từ domain Harbor (containerd không dùng chung OS trust store cho registry).**
```bash
sudo mkdir -p /etc/containerd/certs.d/registry.sevatech.local
sudo cp root-ca.registry.crt /etc/containerd/certs.d/registry.sevatech.local/ca.crt
sudo systemctl restart containerd
```

**Kiểm tra:** `sudo ctr -n k8s.io images pull registry.sevatech.local/library/nginx:stable` thành công không lỗi x509.

---

## Bước 10 — Private image pull secret
**File nguồn:** `k8s-11-pull-image-private-registry.md` | **Thực hiện trên:** master-1.

**[Server: master-1] — Tác dụng: tạo Secret chứa credential Harbor Robot Account (pull-only) để kubelet dùng khi pull image private.**
```bash
kubectl create secret docker-registry registry-cred \
  --docker-server=registry.sevatech.local \
  --docker-username='robot$sevago-develop+sevago-develop' \
  --docker-password="$HARBOR_ROBOT_PASSWORD" -n sevago-develop
```

**[Server: master-1] — Tác dụng: tạo ServiceAccount gắn sẵn imagePullSecrets để Deployment không cần khai báo lại.**
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata: {name: registry-pull-sa, namespace: sevago-develop}
imagePullSecrets: [{name: registry-cred}]
EOF
```

**Kiểm tra:** deploy Pod test dùng SA này, pull image thành công, log = OK.

---

## Bước 11 — CoreDNS nội bộ
**File nguồn:** `k8s-12-pod-coredns.md` | **Thực hiện trên:** master-1.

**[Server: master-1] — Tác dụng: thêm forward zone `sevago.local` sang DNS công ty, để Pod resolve domain nội bộ mà không cần sửa CoreDNS mỗi khi có domain mới.**
```bash
kubectl -n kube-system edit configmap coredns
# thêm block: sevago.local:53 { forward . 172.17.90.23 }
kubectl rollout restart deployment/coredns -n kube-system
```

**Kiểm tra:** Pod test `nslookup develop.account.sevago.local` trả đúng IP.

---

## Bước 12 — Health check tổng thể (gate)
**File nguồn:** `k8s-13-check-cluster-health.md` | **Thực hiện trên:** master-1.

**[Server: master-1] — Tác dụng: checklist xác nhận toàn bộ Bước 1-11 vẫn healthy, làm cổng kiểm soát trước khi cho CI/CD deploy.**
```bash
kubectl get --raw='/readyz?verbose'
kubectl get nodes -o wide
kubectl get pods -A
curl -k https://172.17.79.159:6443/healthz
kubectl get svc -n ingress-nginx
```

---

## Bước 13 — Jenkins: kubectl/Helm + RBAC per-cluster
**File nguồn:** `k8s-14-jenkins-kubectl-helm.md` | **Thực hiện trên:** Jenkins VPS (172.17.79.21) + master-1.

**[Server: Jenkins VPS, container jenkins] — Tác dụng: đảm bảo container có kubectl/Helm/git; nếu thiếu, build lại image từ Dockerfile tùy biến.**

**[Server: master-1] — Tác dụng: tạo ServiceAccount riêng cho cluster Develop trong namespace `cicd`.**
```bash
kubectl create namespace cicd
kubectl create serviceaccount jenkins-develop -n cicd
```

**[Server: master-1] — Tác dụng: gắn quyền CRUD (Pod/Service/Deployment/Ingress) vào ServiceAccount CHỈ trong namespace app cụ thể bằng RoleBinding (không dùng ClusterRoleBinding) — Jenkins không có quyền kube-system hay namespace app khác.**
```bash
kubectl apply -f rolebinding-jenkins-develop.yaml
```

**[Server: master-1] — Tác dụng: tạo token đăng nhập không hết hạn (Secret loại service-account-token, khác `kubectl create token` có hạn).**
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata: {name: jenkins-develop-token, namespace: cicd, annotations: {kubernetes.io/service-account.name: jenkins-develop}}
type: kubernetes.io/service-account-token
EOF
```

**[Server: master-1 → copy sang Jenkins VPS] — Tác dụng: đóng gói CA + token thành 1 kubeconfig riêng cho cluster Develop, copy vào Jenkins container.**
```bash
scp /tmp/config-develop devops@172.17.79.21:/tmp/config-develop
```

**Kiểm tra:** `kubectl auth can-i patch deployments -n sevago-develop --as=system:serviceaccount:cicd:jenkins-develop` → yes; `--as=... get nodes` → no.

---

## Bước 14 — TLS Secret
**File nguồn:** `k8s-15-kubernetes-tls-secret.md` | **Thực hiện trên:** máy quản trị (cert) + master-1.

**[Máy quản trị] — Tác dụng: kiểm tra cert còn hạn ≥30 ngày, đối chiếu SHA256 cert khớp private key trước khi tạo Secret.**
```bash
openssl x509 -in selfsigned.crt -noout -checkend 2592000
```

**[Server: master-1] — Tác dụng: tạo Secret TLS namespace-scoped (`local-tls`), Ingress sẽ tham chiếu tới Secret này để terminate HTTPS.**
```bash
kubectl create secret tls local-tls --cert=selfsigned.crt --key=selfsigned.key -n sevago-develop
```

> ⚠️ Secret không share được giữa namespace/cluster khác dù cùng certificate.

---

## Bước 15 — Jenkins CI/CD với Helm
**File nguồn:** `k8s-16-jenkins-cicd-helm.md` | **Thực hiện trên:** Jenkins VPS.

**[Jenkins UI] — Tác dụng: khai báo credential Harbor Robot Account để Job docker login mà không hardcode password.**

**[Server: Jenkins VPS, build agent] — Tác dụng: lệnh trung tâm deploy app bằng Helm; nếu fail giữ nguyên Pod lỗi để debug (không rollback ngầm).**
```bash
helm upgrade --install "${REPO}" app-helm -n "${NAMESPACE}" \
  --set "ingress.host=${INGRESS_HOST}" --set "image.fullName=${IMAGE_FULL_NAME}" \
  --wait --timeout 7m || { kubectl describe deployment "${REPO}" -n "${NAMESPACE}"; exit 1; }
```

---

## Bước 16 — Jenkins Manual Job
**File nguồn:** `k8s-17-jenkins-cicd-manual-job.md` | **Thực hiện trên:** Jenkins VPS.

**[Jenkins UI] — Tác dụng: cài plugin Active Choices/Pipeline/Credentials Binding, tạo Job `sevago-develop-manual` để build/deploy theo yêu cầu không cần webhook.**

**[Server: Jenkins VPS, Groovy script của parameter] — Tác dụng: gọi GitLab API lấy danh sách repo hiển thị dropdown cho người vận hành chọn.**

---

## Bước 17 — API Gateway (tùy chọn)
**File nguồn:** `k8s-18-api-gateway-optional.md` — không có lệnh, chỉ cân nhắc kiến trúc khi cần auth tập trung/rate limit/API key.

---

## Bước 18 — Traffic flow (tài liệu tham chiếu)
**File nguồn:** `k8s-19-kubernetes-traffic-flow.md` — không có lệnh, dùng tra cứu khi troubleshoot.

---

## Bước 19 — Worker inotify limit
**File nguồn:** `k8s-20-worker-inotify-limit-short.md` | **Thực hiện trên:** worker-1/2/3.

**[Server: worker-1, worker-2, worker-3] — Tác dụng: tăng giới hạn inotify kernel, tránh app dùng FileSystemWatcher (.NET/Node.js/Filebeat) bị CrashLoopBackOff.**
```bash
sudo tee /etc/sysctl.d/99-kubernetes-inotify.conf <<EOF
fs.inotify.max_user_instances=1024
fs.inotify.max_user_watches=524288
EOF
sudo sysctl --system
```

---

## Bước 20 — Beyla observability (bổ sung)
**File nguồn:** `k8s-21-beyla-application-observability.md` | **Thực hiện trên:** worker (kiểm tra) + master-1 (deploy) + Monitor Server 172.17.79.23 (xem kết quả).

**[Server: worker-1/2/3] — Tác dụng: kiểm tra kernel ≥5.8, có BTF/tracefs (yêu cầu bắt buộc để chạy eBPF).**

**[Server: master-1] — Tác dụng: cài Beyla DaemonSet (agent trên mỗi worker) qua Helm, chỉ instrument Deployment `*-be`.**
```bash
helm upgrade --install beyla grafana/beyla --namespace monitor --version 1.16.10 \
  -f beyla-base.values.yaml -f beyla-develop.values.yaml
```

**Kiểm tra:** DaemonSet Ready đủ worker; metric `http_server_request_duration_seconds_count` có dữ liệu trên Central Prometheus.

---

## Ghi nhớ khi lặp lại cho 5 cluster

1. Không copy IP/hostname/pool giữa cluster.
2. Không share Secret/TLS giữa cluster dù cùng certificate.
3. Jenkins: 1 cluster = 1 ServiceAccount = 1 kubeconfig riêng.
4. Ingress NGINX là legacy — cluster mới nên đánh giá Gateway API từ Bước 5.
5. Thứ tự lõi: 1→2→3→4→5→6-8→9-10→11→12(gate)→13-16→17(optional)→19→20.
