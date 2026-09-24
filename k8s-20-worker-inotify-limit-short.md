# ☸️ Bước 20 - Cấu hình Inotify Limit cho Kubernetes Worker Node

## 1. 🎯 Mục đích

File này chuẩn hóa `inotify` limit trên worker node để tránh ứng dụng dùng nhiều `FileSystemWatcher` bị `CrashLoopBackOff`, đồng thời quy định cách kiểm tra Pod sau khi thay đổi. Áp dụng cho worker của **từng cluster trong 5 cluster**; không chỉ Develop.

---

## 2. 📌 Thực hiện ở đâu

> **📍 Thực hiện tại:** Tất cả worker node của cluster đích; Develop: `172.17.79.93`, `.94`, `.95`.

Thực hiện trên **tất cả Kubernetes worker node**.

Ví dụ Develop:

```text
sevago-dev-k8s-node-1
sevago-dev-k8s-node-2
sevago-dev-k8s-node-3
```

Không cần thực hiện trên control-plane/master nếu master không chạy application workload.

---

## 3. ℹ️ Nguyên nhân

Một số application/container như `.NET FileSystemWatcher`, ASP.NET Core config reload, Node.js, Filebeat... sử dụng Linux `inotify`.

Khi một worker node chạy nhiều Pod, giới hạn mặc định có thể bị chạm:

```text
fs.inotify.max_user_instances = 128
```

Lỗi thường gặp:

```text
System.IO.IOException:
The configured user limit (128) on the number of inotify instances has been reached
```

Hậu quả:

```text
Application không startup được
Exit Code 1
CrashLoopBackOff
Pod restart liên tục
Startup Probe connection refused
```

---

## 4. 🔎 Kiểm tra

> **📍 Thực hiện tại:** Tất cả worker node của cluster đích; Develop: `172.17.79.93`, `.94`, `.95`.

```bash
sysctl fs.inotify.max_user_instances
sysctl fs.inotify.max_user_watches
sysctl fs.inotify.max_queued_events
```

Ví dụ trước khi update:

```text
fs.inotify.max_user_instances = 128
fs.inotify.max_user_watches = 60946
fs.inotify.max_queued_events = 16384
```

Có thể kiểm tra nhanh số `inotify` đang được sử dụng:

```bash
sudo find /proc/[0-9]*/fd -lname 'anon_inode:inotify' 2>/dev/null | wc -l
```

Nếu giá trị thực tế tiến gần `128` thì node có nguy cơ chạm giới hạn.

---

## 5. ⚙️ Update

> **📍 Thực hiện tại:** Tất cả worker node của cluster đích; Develop: `172.17.79.93`, `.94`, `.95`.

Tạo file:

```bash
sudo nano /etc/sysctl.d/99-kubernetes-inotify.conf
```

Nội dung:

```text
# Kubernetes/container workloads có thể tạo nhiều FileSystemWatcher.
fs.inotify.max_user_instances=1024

# Tăng số lượng file/directory tối đa có thể theo dõi.
fs.inotify.max_user_watches=524288

# Tăng queue cho các inotify event đang chờ xử lý.
fs.inotify.max_queued_events=32768
```

Apply:

```bash
sudo sysctl --system
```

Hoặc:

```bash
sudo sysctl -p /etc/sysctl.d/99-kubernetes-inotify.conf
```

Không cần reboot node.

> Cần cấu hình đồng bộ trên tất cả worker node vì Pod có thể được Kubernetes schedule sang bất kỳ worker nào.

---

## 6. 🔎 Test kết quả

### 6.1. ⚙️ Kiểm tra cấu hình trên worker

> **📍 Thực hiện tại:** Tất cả worker node của cluster đích; Develop: `172.17.79.93`, `.94`, `.95`.

```bash
sysctl fs.inotify.max_user_instances
sysctl fs.inotify.max_user_watches
sysctl fs.inotify.max_queued_events
```

Kết quả mong đợi:

```text
fs.inotify.max_user_instances = 1024
fs.inotify.max_user_watches = 524288
fs.inotify.max_queued_events = 32768
```

### 6.2. 🔎 Kiểm tra Pod

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig của cluster đích.

Nếu Pod trước đó đang `CrashLoopBackOff`, xóa Pod để Deployment tạo lại:

```bash
kubectl delete pod <pod-name> -n <namespace>
```

Theo dõi:

```bash
kubectl get pods -n <namespace> -w
```

Kiểm tra log:

```bash
kubectl logs <pod-name> -n <namespace> --tail=200
```

Không được còn lỗi:

```text
The configured user limit (...) on the number of inotify instances has been reached
```

Kiểm tra restart:

```bash
kubectl get pod <pod-name> \
  -n <namespace> \
  -o jsonpath='{range .status.containerStatuses[*]}Container={.name}{"\n"}RestartCount={.restartCount}{"\n"}LastReason={.lastState.terminated.reason}{"\n"}ExitCode={.lastState.terminated.exitCode}{"\n"}{end}'
```

Kết quả mong đợi:

```text
Pod Running/Ready
Restart Count không tiếp tục tăng
Không còn CrashLoopBackOff do inotify
```

> `inotify` và `OOMKilled` là hai vấn đề khác nhau. Nếu Pod vẫn bị `OOMKilled`, cần kiểm tra riêng memory limit và memory usage của application.
