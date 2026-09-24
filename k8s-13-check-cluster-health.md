# ☸️ Bước 13 - Kiểm tra Health Kubernetes Cluster

## 1. 🎯 Mục đích

File này là checklist nghiệm thu cluster sau các bước nền tảng, trước khi cho Jenkins/Helm triển khai application. Chạy độc lập cho cả 5 cluster.

File này xác nhận toàn bộ Kubernetes cluster đã hoạt động ổn định và sẵn sàng deploy application.

Các thành phần cần kiểm tra:

```text
✓ Kubernetes HA Control Plane
✓ Calico
✓ CoreDNS
✓ MetalLB
✓ Ingress NGINX
✓ Rancher
✓ Harbor Registry
✓ ServiceAccount / imagePullSecret
✓ API VIP
```

Mục tiêu:

```text
Cluster Ready cho CI/CD và deploy ứng dụng.
```

---

## 2. ✅ Checklist nội dung

```text
[ ] API Server healthy
[ ] readyz passed
[ ] livez passed

[ ] 3 Master Ready
[ ] 3 Worker Ready

[ ] CoreDNS Running
[ ] Calico Running
[ ] etcd Running
[ ] kube-apiserver Running
[ ] kube-controller-manager Running
[ ] kube-scheduler Running
[ ] kube-proxy Running

[ ] API VIP 172.17.79.159 hoạt động

[ ] MetalLB Running
[ ] IP Pool hoạt động

[ ] Ingress NGINX Running
[ ] Ingress có External IP

[ ] Rancher Running
[ ] Rancher ingress/domain truy cập được từ mạng quản trị

[ ] Harbor Secret tồn tại
[ ] registry-pull-sa tồn tại

[ ] Helm hoạt động

[ ] CoreDNS đã cấu hình domain nội bộ
```

---

## 3. 🔎 Kiểm tra API Server

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl cluster-info
kubectl get --raw='/readyz?verbose'
kubectl get --raw='/livez?verbose'
```

Kết quả mong đợi:

```text
readyz check passed
livez check passed
```

---

## 4. 🔎 Kiểm tra Node

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl get nodes -o wide
```

Kết quả mong đợi:

```text
3 Master Ready
3 Worker Ready
```

Không có node:

```text
NotReady
Unknown
```

---

## 5. 🔎 Kiểm tra Pod hệ thống

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl get pods -A
```

Điều kiện đạt:

```text
STATUS = Running
STATUS = Completed (đối với Job)
```

Không có:

```text
CrashLoopBackOff
ImagePullBackOff
ErrImagePull
Pending
```

---

## 6. 🔎 Kiểm tra kube-system

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl get pods -n kube-system -o wide
```

Các thành phần bắt buộc:

```text
CoreDNS
Calico Node
Calico Controller
etcd
kube-apiserver
kube-controller-manager
kube-scheduler
kube-proxy
```

Tất cả phải ở trạng thái:

```text
Running
```

---

## 7. 🔎 Kiểm tra API VIP

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

VIP hiện tại:

```text
172.17.79.159
```

Kiểm tra:

```bash
curl -k https://172.17.79.159:6443/healthz
nc -zv 172.17.79.159 6443
```

Kết quả mong đợi:

```text
ok
succeeded
```

Điều này xác nhận API VIP và HAProxy đang hoạt động tại thời điểm kiểm tra. Để xác nhận khả năng HA thực tế, thực hiện bài kiểm tra Keepalived Failover tại Bước 03.

---

## 8. 🌐 Kiểm tra MetalLB

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

Kết quả mong đợi:

```text
controller Running
speaker Running trên tất cả node
```

Pool hiện tại:

```text
172.17.79.160 - 172.17.79.169
```

---

## 9. 🌐 Kiểm tra Ingress NGINX

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

> Nếu cluster đã migration khỏi ingress-nginx, thay mục này bằng health check của Gateway/controller đang vận hành và kiểm tra `Gateway`/`HTTPRoute` tương ứng.

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Kết quả mong đợi:

```text
ingress-nginx-controller Running
```

Service:

```text
TYPE         LoadBalancer
EXTERNAL-IP  172.17.79.160
```

---

## 10. 🔎 Kiểm tra Rancher

> **📍 Thực hiện tại:** Kubectl trên control-plane-1/máy có kubeconfig; truy cập UI từ máy quản trị.

```bash
kubectl get deployment,pods,svc,ingress -n cattle-system
kubectl rollout status deployment/rancher -n cattle-system --timeout=5m
```

Kết quả mong đợi:

```text
Deployment rancher rollout thành công
Rancher Pod Running/Ready
Ingress host: develop.k8s.sevago.local
```

Từ máy quản trị có route/DNS phù hợp, truy cập `https://develop.k8s.sevago.local` và đăng nhập bằng Rancher account được cấp quyền.

---

## 11. 🔐 Kiểm tra Harbor Pull Secret

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Namespace hiện tại:

```text
sevago-develop
```

Kiểm tra:

```bash
kubectl get secret registry-cred -n sevago-develop
kubectl get sa registry-pull-sa -n sevago-develop -o yaml
```

Kết quả mong đợi:

```text
registry-cred tồn tại

TYPE:
kubernetes.io/dockerconfigjson
```

ServiceAccount:

```text
imagePullSecrets:
  registry-cred
```

---

## 12. ⛵ Kiểm tra Helm

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

```bash
helm version
helm list -A
```

Kết quả mong đợi:

```text
Helm hoạt động bình thường
```

Có thể nhìn thấy:

```text
ingress-nginx
rancher
```

---

## 13. 🌐 Kiểm tra DNS nội bộ

> **📍 Thực hiện tại:** Control-plane-1 của cluster đích hoặc máy quản trị có kubeconfig. Develop: `172.17.79.90`.

Kiểm tra CoreDNS ConfigMap:

```bash
kubectl -n kube-system get configmap coredns -o yaml
```

Xác nhận có block:

```text
sevago.local:53 {
    errors
    cache 30
    forward . 172.17.90.23
}
```

Tạo Pod test:

```bash
kubectl run dns-health-test --image=busybox:1.36 --restart=Never --command -- sleep 300
```

Đợi Pod sẵn sàng:

```bash
kubectl wait --for=condition=Ready pod/dns-health-test --timeout=60s
```

Kiểm tra Kubernetes Service Discovery:

```bash
kubectl exec dns-health-test -- nslookup kubernetes.default.svc.cluster.local
```

Kiểm tra domain nội bộ:

```bash
kubectl exec dns-health-test -- nslookup develop.account.sevago.local
kubectl exec dns-health-test -- nslookup develop.office.sevago.local
kubectl exec dns-health-test -- nslookup develop.file.sevago.local
```

Kết quả mong đợi:

```text
Kubernetes Service resolve thành công
Các domain sevago.local resolve đúng IP từ DNS nội bộ
```

Dọn Pod test:

```bash
kubectl delete pod dns-health-test
```
