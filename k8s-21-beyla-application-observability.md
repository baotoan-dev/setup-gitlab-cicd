# ☸️ Bước 21 - Triển khai Grafana Beyla Application Observability

## 1. 🎯 Mục đích và vị trí trong roadmap

Tài liệu này là **bước bổ sung phục vụ monitoring**, không phải bước bắt buộc của quá trình dựng Kubernetes ban đầu.

Thực hiện Bước 21 khi đang làm nhóm tài liệu Grafana `09.1.x`, cụ thể khi đến:

```text
09.1.1 - Grafana Kubernetes Detail
```

và cần hoàn thiện các panel:

```text
HTTP Request Rate by Pod
HTTP Latency P95 by Pod
HTTP Latency P99 by Pod
```

Lý do cần Bước 21:

```text
Monitoring Kubernetes hiện có
├── kube-state-metrics  -> trạng thái Kubernetes
├── Node Exporter       -> CPU/RAM/Disk node
├── cAdvisor            -> CPU/RAM/network container
└── Prometheus Agent    -> scrape + Remote Write

Nhưng chưa có application HTTP metrics
├── request rate theo Pod
├── HTTP status
└── HTTP latency theo Pod
```

Sau khi hoàn thành:

```text
K8s 21 - Beyla
      ↓
Central Prometheus có HTTP metrics theo Pod
      ↓
Quay lại 09.1.1
      ↓
Hoàn thiện Request Rate / P95 / P99
      ↓
Tiếp tục roadmap monitoring
```

> **Bước 21 không thay đổi cách dựng hoặc deploy Kubernetes/application.** Jenkins, Helm chart của application, Deployment, Service, Ingress NGINX, ConfigMap, Secret và CI/CD hiện tại giữ nguyên. Bước này chỉ bổ sung Beyla vào `layer-monitor` và chạy Beyla DaemonSet trong namespace `monitor`.

---

## 2. 📌 Mục tiêu

Triển khai Grafana Beyla để tự động thu thập application HTTP telemetry bằng eBPF mà không yêu cầu sửa source NestJS/.NET.

Mục tiêu cuối cùng:

```text
Pod A -> 200 req/s
Pod B -> 85 req/s
Pod C -> 310 req/s

Pod A -> p95 = 80 ms
Pod B -> p95 = 120 ms
Pod C -> p95 = 65 ms
```

Beyla phải cung cấp tối thiểu:

```text
Request count
Request rate
HTTP response status
HTTP server latency histogram
Kubernetes namespace
Kubernetes deployment
Kubernetes pod
Kubernetes container
Kubernetes node
```

Các Prometheus metrics chính:

```text
http_server_request_duration_seconds_count
http_server_request_duration_seconds_sum
http_server_request_duration_seconds_bucket
```

---

## 3. 📖 Các khái niệm cần hiểu

### 3.1. 📌 Grafana Beyla

> **📍 Thực hiện tại:** Monitor Server `172.17.79.23`.

Grafana Beyla là công cụ application auto-instrumentation dùng eBPF để quan sát HTTP/gRPC của process trên Linux mà không cần thêm Prometheus SDK vào source application.

```text
Application Pod
      ↓
eBPF quan sát request/response
      ↓
Beyla
      ↓
Prometheus metrics
```

Tên `Grafana Beyla` không có nghĩa là Grafana Server được cài vào Kubernetes.

Grafana Server của Sevago vẫn chạy trên Monitor Server:

```text
172.17.79.23
```

### 3.2. 📌 eBPF

eBPF cho phép quan sát process/kernel/network trên Linux mà không phải sửa business logic của application.

Trong Bước 21, eBPF giúp Beyla xác định:

```text
request bắt đầu lúc nào
request kết thúc lúc nào
HTTP method/status/route
process/container nào xử lý
Pod/Deployment nào chứa process đó
```

### 3.3. 📌 DaemonSet

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig của cluster đích; Develop: `172.17.79.90`.

Beyla được triển khai dạng Kubernetes DaemonSet.

```text
Worker node-1 -> 1 Beyla Pod
Worker node-2 -> 1 Beyla Pod
Worker node-3 -> 1 Beyla Pod
```

Mỗi Beyla Pod quan sát application process trên chính worker mà nó đang chạy. Vì vậy mỗi Kubernetes cluster phải có Beyla DaemonSet riêng.

### 3.4. 📌 RED metrics

Application Observability trong bước này tập trung vào RED metrics:

```text
R = Rate      -> request/giây
E = Errors    -> HTTP status/error
D = Duration  -> request mất bao lâu
```

### 3.5. 📌 req/s

`req/s` là số request trung bình mỗi giây trong một rolling window.

Ví dụ:

```promql
rate(http_server_request_duration_seconds_count[1m])
```

Nếu kết quả là `200`, hiểu là Pod xử lý trung bình khoảng `200 request/giây` trong 1 phút gần nhất.

### 3.6. 📌 Latency trong tài liệu này

Beyla metric:

```text
http_server_request_duration_seconds
```

là **server-side HTTP request duration** của application.

Nó trả lời:

```text
Pod A xử lý request nhanh hay chậm?
```

Nó không đại diện hoàn toàn cho end-to-end latency từ máy người dùng vì đường đi còn có thể gồm:

```text
Client network
Firewall / Reverse Proxy
Ingress
Service routing
```

### 3.7. 📌 Histogram

Beyla export HTTP duration dạng Prometheus Histogram:

```text
_count   -> tổng số request
_sum     -> tổng duration
_bucket  -> phân bố request theo latency bucket
```

`_bucket` dùng để tính percentile.

### 3.8. 📌 p50, p95, p99

Ví dụ:

```text
p95 = 120 ms
```

có nghĩa khoảng 95% request hoàn thành trong `120 ms` hoặc nhanh hơn; khoảng 5% còn lại chậm hơn mức đó.

```text
p50 -> latency trung vị
p95 -> latency của phần lớn request
p99 -> nhóm request chậm nhất
```

Dashboard Sevago ưu tiên `p95` và `p99`.

### 3.9. 📌 Prometheus Agent, Prometheus Server và Grafana

> **📍 Thực hiện tại:** Monitor Server `172.17.79.23`.

```text
Trong từng Kubernetes cluster
Prometheus Agent
  -> scrape metrics
  -> Remote Write

Monitor Server 172.17.79.23
Prometheus Server
  -> lưu/query metrics tập trung

Monitor Server 172.17.79.23
Grafana
  -> đọc Prometheus
  -> hiển thị dashboard
```

Beyla chỉ bổ sung nguồn application metrics, không thay thế các thành phần trên.

### 3.10. ⛵ Helm repository, Helm chart và values file

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig của cluster đích; Develop: `172.17.79.90`.

```text
grafana                       -> alias Helm repository trên master-1
grafana/beyla                 -> Helm chart Beyla
beyla-base.values.yaml        -> cấu hình chung do Sevago quản lý
beyla-develop.values.yaml     -> cấu hình Develop
beyla-staging.values.yaml     -> cấu hình Staging
beyla-production.values.yaml  -> cấu hình Production
```

Lệnh:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

chỉ thêm nguồn Helm chart của Grafana Labs vào Helm client. Lệnh này không cài Grafana Server vào Kubernetes.

---

## 4. 🧭 Phạm vi thay đổi

Bổ sung:

```text
namespace monitor
└── Beyla DaemonSet
```

Bổ sung trong Git:

```text
layer-monitor/kubernetes/beyla/
├── beyla-base.values.yaml
├── beyla-develop.values.yaml
├── beyla-staging.values.yaml
└── beyla-production.values.yaml
```

Giữ nguyên:

```text
Kubernetes control plane
Calico
MetalLB
Ingress NGINX
Application Deployment/Service
Application Helm chart
Jenkins pipeline
Harbor
Prometheus Agent
kube-state-metrics
Node Exporter
cAdvisor
Filebeat
Prometheus Server trên Monitor VPS
Grafana trên Monitor VPS
```

Không yêu cầu sửa:

```text
NestJS source
.NET source
application middleware
application /metrics endpoint
```

---

## 5. 🏗️ Kiến trúc sau khi có Beyla

```text
Client
  ↓
Ingress NGINX
  ↓
Service
  ↓
Application Pod
  │
  │ eBPF
  ↓
Beyla DaemonSet
  │
  │ :9090/metrics
  ↓
Prometheus Agent
  │
  │ Remote Write
  ↓
Prometheus Server 172.17.79.23
  ↓
Grafana 172.17.79.23
```

Prometheus Agent hiện tại đã có job `kubernetes-pods` scrape Pod có annotation:

```yaml
prometheus.io/scrape: "true"
prometheus.io/port: "..."
prometheus.io/path: "..."
```

Beyla chỉ cần expose `:9090/metrics` và gắn annotation tương ứng.

### 5.1. 🏗️ Kiến trúc 5 Kubernetes cluster

```text
Develop
└── 1 cluster

Staging
├── Local cluster
└── DMZ cluster

Production
├── Local cluster
└── DMZ cluster

TOTAL = 5 cluster
```

Mỗi cluster có:

```text
Beyla DaemonSet
      ↓
Prometheus Agent của chính cluster đó
      ↓ Remote Write
Central Prometheus 172.17.79.23
```

Beyla không hardcode `environment` và `cluster`. Hai label này do Prometheus Agent của từng cluster gắn bằng `external_labels`.

> Với 2 cluster Staging và 2 cluster Production, `cluster` phải unique theo từng physical cluster để Central Prometheus không trộn Local và DMZ.

---

## 6. 📍 Vị trí thực hiện

### 6.1. 📌 Develop Kubernetes

```text
MASTER
172.17.79.90  sevago-dev-k8s-master-1
172.17.79.91  sevago-dev-k8s-master-2
172.17.79.92  sevago-dev-k8s-master-3

WORKER
172.17.79.93  sevago-dev-k8s-node-1
172.17.79.94  sevago-dev-k8s-node-2
172.17.79.95  sevago-dev-k8s-node-3
```

### 6.2. 📌 Quy ước nơi chạy lệnh

```text
Local
  -> tạo/sửa values file
  -> Git commit/push

Worker .93/.94/.95
  -> kiểm tra kernel/BTF/eBPF prerequisites

Kubernetes master-1 .90
  -> Helm repository/version
  -> render/dry-run/deploy
  -> kiểm tra DaemonSet
  -> kiểm tra Prometheus Agent scrape

Monitor Server .23
  -> kiểm tra metrics Central Prometheus
  -> test req/s
  -> test p95/p99
  -> kiểm tra RAM Beyla bằng cAdvisor metrics
```

---

## 7. 🔎 Kiểm tra prerequisite trên worker

> **📍 Thực hiện tại:** Từng worker của Develop: `172.17.79.93`, `.94`, `.95`; khi rollout môi trường khác dùng worker của cluster đích.

Đăng nhập worker rồi chạy một block thay cho nhiều lệnh rời:

```bash
echo "===== HOST ====="
hostname

echo "===== KERNEL ====="
uname -r

echo "===== BTF ====="
test -r /sys/kernel/btf/vmlinux && echo "BTF: OK" || echo "BTF: MISSING"

echo "===== BPF FS ====="
mount | grep 'type bpf' || echo "BPF FS: NOT MOUNTED"

echo "===== TRACEFS ====="
test -d /sys/kernel/tracing && echo "TRACEFS: OK" || echo "TRACEFS: MISSING"
```

Kết quả đạt yêu cầu:

```text
Đúng hostname worker.
Kernel >= 5.8.
BTF: OK.
TRACEFS: OK.
```

`bpffs` nên được mount. Beyla application metrics vẫn có thể hoạt động khi một số pinned-map feature không khả dụng, nhưng BTF/kernel prerequisite phải đạt trước khi tiếp tục.

---

## 8. ⛵ Kiểm tra Kubernetes monitoring và Helm

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig của cluster đích; Develop: `172.17.79.90`.

Đăng nhập:

```bash
ssh devops@172.17.79.90
```

Chạy một block:

```bash
echo "===== CONTEXT ====="
kubectl config current-context

echo "===== NODES ====="
kubectl get nodes -o wide

echo "===== MONITOR NAMESPACE ====="
kubectl get namespace monitor

echo "===== PROMETHEUS AGENT ====="
kubectl -n monitor get deployment prometheus-agent
kubectl -n monitor get service prometheus-agent

echo "===== HELM ====="
helm version --short
```

Kết quả đạt yêu cầu:

```text
Đúng cluster Develop.
3 master Ready.
3 worker Ready.
namespace monitor = Active.
Prometheus Agent READY/AVAILABLE = 1.
Service prometheus-agent tồn tại.
Helm v3.x.x.
```

---

## 9. 🏷️ Chuẩn bị Helm repository và pin version Beyla

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig của cluster đích; Develop: `172.17.79.90`.

### 9.1. 📌 Thêm repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm repo list | grep '^grafana'
```

Kết quả mong đợi:

```text
grafana    https://grafana.github.io/helm-charts
```

> `grafana` chỉ là alias Helm repository chứa chart `grafana/beyla`. Grafana Server vẫn chạy trên Monitor VPS `172.17.79.23`.

### 9.2. 🏷️ Xác nhận version được pin

Version đã nghiệm thu trên Develop:

```bash
BEYLA_CHART_VERSION=1.16.10
```

Chạy một block:

```bash
echo "===== SEARCH CHART ====="
helm search repo grafana/beyla --version "${BEYLA_CHART_VERSION}"

echo "===== CHART METADATA ====="
helm show chart grafana/beyla --version "${BEYLA_CHART_VERSION}" |
  grep -E '^(name|version|appVersion):'

echo "===== DEFAULT VALUES ====="
helm show values grafana/beyla --version "${BEYLA_CHART_VERSION}" > /tmp/beyla-chart-default-values.yaml

test -s /tmp/beyla-chart-default-values.yaml && echo "Beyla chart values: OK"

grep -nE '^(privileged:|contextPropagation:|service:|podAnnotations:|config:|preset:|serviceMonitor:)' /tmp/beyla-chart-default-values.yaml
```

Kết quả đã nghiệm thu trên Develop:

```text
chart: grafana/beyla
chart version: 1.16.10
Beyla app version: 3.25.0
```

Mục này chỉ test **Helm client trên master-1 có nhìn thấy đúng chart/version trong Grafana Helm repository hay không**. Nó không test Kubernetes cluster, Monitor Server hay Grafana version.

Sau khi Develop đã nghiệm thu, dùng đúng version đã pin cho các cluster còn lại. Không dùng `latest`.

---

## 10. ➕ Tạo values trong Git

> **📍 Thực hiện tại:** Máy làm việc local có source `app-core`/repository cấu hình.

```bash
cd ~/app-core
mkdir -p layer-monitor/kubernetes/beyla
cd layer-monitor/kubernetes/beyla
```

Cấu trúc:

```text
layer-monitor/kubernetes/beyla/
├── beyla-base.values.yaml
├── beyla-develop.values.yaml
├── beyla-staging.values.yaml
└── beyla-production.values.yaml
```

### 10.1. 📌 `beyla-base.values.yaml`

```yaml
preset: application

privileged: true

contextPropagation:
  enabled: false

service:
  enabled: false

serviceMonitor:
  enabled: false

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"
  prometheus.io/path: "/metrics"

config:
  create: true
  data:
    attributes:
      kubernetes:
        enable: true

    prometheus_export:
      port: 9090
      path: /metrics

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 1Gi
```

Giải thích resource:

```text
memory request 256Mi
  -> scheduler dùng để tính placement.

memory limit 1Gi
  -> trần memory của Beyla Pod.
  -> không có nghĩa Beyla luôn sử dụng 1Gi RAM.
```

### 10.2. 📌 Chỉ instrument Backend `*-be`

Không instrument toàn namespace vì số process cần attach quá lớn và không cần thiết cho dashboard HTTP Backend.

Sevago dùng selector:

```text
k8s_deployment_name: "*-be"
```

Nó bao gồm các Backend HTTP như:

```text
author-management-be
customer-management-be
e-catalogue-be
file-be
office-be
sso-be
...
```

và loại khỏi scope hiện tại:

```text
*-admin
*-fe
manufacturing-plan-background
manufacturing-execution-grpc
```

`background` không cần HTTP req/s. gRPC sẽ bổ sung riêng nếu sau này cần gRPC observability.

### 10.3. 📌 `beyla-develop.values.yaml`

```yaml
config:
  data:
    discovery:
      instrument:
        - k8s_namespace: "sevago-develop"
          k8s_deployment_name: "*-be"
          exports:
            - metrics
```

### 10.4. 📌 `beyla-staging.values.yaml`

```yaml
config:
  data:
    discovery:
      instrument:
        - k8s_namespace: "sevago-staging"
          k8s_deployment_name: "*-be"
          exports:
            - metrics
```

### 10.5. 📌 `beyla-production.values.yaml`

```yaml
config:
  data:
    discovery:
      instrument:
        - k8s_namespace: "sevago-production"
          k8s_deployment_name: "*-be"
          exports:
            - metrics
```

Kiểm tra toàn bộ file bằng một block:

```bash
echo "===== FILES ====="
ls -1 *.values.yaml

echo "===== DEVELOP DISCOVERY ====="
grep -A8 'discovery:' beyla-develop.values.yaml

echo "===== BASE RESOURCES ====="
grep -A8 '^resources:' beyla-base.values.yaml
```

Kết quả mong đợi:

```text
4 values file tồn tại.
Develop có k8s_namespace=sevago-develop.
Develop có k8s_deployment_name=*-be.
Base có request memory=256Mi và limit memory=1Gi.
```

---

## 11. ⚙️ Commit cấu hình

> **📍 Thực hiện tại:** Máy làm việc local có source `app-core`/repository cấu hình.

```bash
cd ~/app-core

git status --short

git add layer-monitor/kubernetes/beyla/
git commit -m "feat: add beyla application observability"
git push
```

Kết quả mong đợi:

```text
4 file Beyla được commit/push thành công.
```

---

## 12. 🚀 Pull và validate trước khi deploy

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig của cluster đích; Develop: `172.17.79.90`.

```bash
cd ~/app-core
git pull

BASE_VALUES=~/app-core/layer-monitor/kubernetes/beyla/beyla-base.values.yaml
ENV_VALUES=~/app-core/layer-monitor/kubernetes/beyla/beyla-develop.values.yaml
BEYLA_CHART_VERSION=1.16.10

set -e

echo "===== VALUES ====="
test -f "${BASE_VALUES}"
test -f "${ENV_VALUES}"
echo "Develop Beyla values: OK"

echo "===== HELM TEMPLATE ====="
helm template beyla grafana/beyla --namespace monitor --version "${BEYLA_CHART_VERSION}" -f "${BASE_VALUES}" -f "${ENV_VALUES}" > /tmp/beyla-develop-rendered.yaml

grep -n '^kind: DaemonSet$' /tmp/beyla-develop-rendered.yaml
grep -n -A5 'prometheus.io/scrape' /tmp/beyla-develop-rendered.yaml
grep -n -A10 'k8s_deployment_name' /tmp/beyla-develop-rendered.yaml
grep -n 'privileged: true' /tmp/beyla-develop-rendered.yaml

echo "===== HELM DRY RUN ====="
helm upgrade --install beyla grafana/beyla --namespace monitor --version "${BEYLA_CHART_VERSION}" -f "${BASE_VALUES}" -f "${ENV_VALUES}" --dry-run > /tmp/beyla-develop-dry-run.txt

grep -m1 '^NAME:' /tmp/beyla-develop-dry-run.txt

echo "===== RESULT ====="
echo "Beyla pre-deploy validation: OK"
```

Kết quả đạt yêu cầu:

```text
Develop Beyla values: OK
Có DaemonSet
Có prometheus.io/scrape=true
Có selector *-be
Có privileged=true
NAME: beyla
Beyla pre-deploy validation: OK
```

---

## 13. 🚀 Deploy Beyla vào Develop

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig của cluster đích; Develop: `172.17.79.90`.

```bash
helm upgrade --install beyla grafana/beyla --namespace monitor --version "${BEYLA_CHART_VERSION}" -f "${BASE_VALUES}" -f "${ENV_VALUES}" --atomic --timeout 5m
```

Kết quả mong đợi:

```text
STATUS: deployed
```

---

## 14. 🔎 Kiểm tra DaemonSet, restart và log

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig của cluster đích; Develop: `172.17.79.90`.

Chạy một block:

```bash
set -e

echo "===== ROLLOUT ====="
kubectl -n monitor rollout status daemonset/beyla --timeout=300s

echo "===== DAEMONSET ====="
kubectl -n monitor get daemonset beyla

echo "===== POD STATUS ====="
kubectl -n monitor get pods \
  -l app.kubernetes.io/name=beyla \
  -o custom-columns='POD:.metadata.name,READY:.status.containerStatuses[0].ready,RESTARTS:.status.containerStatuses[0].restartCount,NODE:.spec.nodeName'

echo "===== FATAL/EBPF ERRORS ====="
for pod in $(kubectl -n monitor get pods -l app.kubernetes.io/name=beyla -o name); do
  echo "--- ${pod} ---"
  kubectl -n monitor logs "${pod}" --tail=300 |
    grep -Ei 'fatal|permission denied|btf|capabilit' || true
done

echo "===== INSTRUMENTED PROCESSES ====="
for pod in $(kubectl -n monitor get pods -l app.kubernetes.io/name=beyla -o name); do
  echo "--- ${pod} ---"
  kubectl -n monitor logs "${pod}" --tail=500 |
    grep 'instrumenting process' || true
done
```

Develop đạt khi:

```text
DaemonSet rollout thành công.
DESIRED=CURRENT=READY=AVAILABLE=3.
3 Beyla Pod READY=true.
Pod mới RESTARTS=0.
Không có fatal/BTF/capability error làm Beyla dừng.
Các process được instrument thuộc Backend scope.
```

Sau rollout nên quan sát thêm khoảng `5-10 phút` để chắc chắn Pod không restart trong giai đoạn discovery/attach process.

### 14.1. 📌 Nếu Beyla bị restart

Chỉ chạy phần này khi `RESTARTS > 0`.

```bash
echo "===== LAST TERMINATION ====="
for pod in $(kubectl -n monitor get pods -l app.kubernetes.io/name=beyla -o name); do
  echo "--- ${pod} ---"
  kubectl -n monitor get "${pod}" \
    -o jsonpath='READY={.status.containerStatuses[0].ready}{"\n"}RESTARTS={.status.containerStatuses[0].restartCount}{"\n"}LAST_REASON={.status.containerStatuses[0].lastState.terminated.reason}{"\n"}LAST_EXIT_CODE={.status.containerStatuses[0].lastState.terminated.exitCode}{"\n"}'
done
```

Nếu thấy:

```text
LAST_REASON=OOMKilled
LAST_EXIT_CODE=137
```

thì đó là container memory cgroup OOM. Không tăng RAM vô hạn; trước tiên xác nhận discovery vẫn chỉ là `*-be` và kiểm tra actual memory từ Central Prometheus ở Mục 17.

---

## 15. 🔎 Kiểm tra Prometheus Agent scrape Beyla

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig của cluster đích; Develop: `172.17.79.90`.

Prometheus Agent hiện tại đã có `kubernetes-pods`, không tạo scrape job mới.

Dùng một block tự mở/đóng port-forward:

```bash
set -e

kubectl -n monitor port-forward service/prometheus-agent 19090:9090 > /tmp/prometheus-agent-port-forward.log 2>&1 &
PF_PID=$!

cleanup() {
  kill "${PF_PID}" 2>/dev/null || true
}
trap cleanup EXIT

sleep 3

echo "===== BEYLA TARGETS ====="
curl -s http://127.0.0.1:19090/api/v1/targets |
jq -r '
  .data.activeTargets[]
  | select(
      .labels.job == "kubernetes-pods"
      and ((.labels.pod // "") | startswith("beyla-"))
    )
  | [
      .labels.pod,
      .health,
      .scrapeUrl
    ]
  | @tsv
'

cleanup
trap - EXIT
```

Kết quả Develop mong đợi:

```text
3 Beyla target.
health = up.
scrape URL = http://<BEYLA_POD_IP>:9090/metrics.
```

---

## 16. 🔎 Kiểm tra end-to-end trên Central Prometheus

> **📍 Thực hiện tại:** Monitor Server `172.17.79.23`.

```bash
ssh devops@172.17.79.23
```

Chạy một block để kiểm tra active deployment, Request Rate, histogram, P95 và P99:

```bash
PROMETHEUS_URL=http://127.0.0.1:14002

BASE_SELECTOR='environment="develop",cluster="sevago-develop",k8s_namespace_name="sevago-develop",k8s_deployment_name=~".*-be"'

echo "===== ACTIVE BACKEND DEPLOYMENTS ====="
curl -gsS -G "${PROMETHEUS_URL}/api/v1/query" \
  --data-urlencode \
  'query=count by (k8s_deployment_name) (http_server_request_duration_seconds_count{environment="develop",cluster="sevago-develop",k8s_namespace_name="sevago-develop",k8s_deployment_name=~".*-be"})' |
jq -r '
  .data.result[]
  | "\(.metric.k8s_deployment_name)\t\(.value[1])"
' |
sort

echo
echo "===== LATENCY HISTOGRAM SERIES ====="
curl -gsS -G "${PROMETHEUS_URL}/api/v1/query" \
  --data-urlencode \
  'query=count(http_server_request_duration_seconds_bucket{environment="develop",cluster="sevago-develop",k8s_namespace_name="sevago-develop",k8s_deployment_name=~".*-be"})' |
jq -r '.data.result[]?.value[1]'

echo
echo "===== REQUEST RATE BY POD ====="
curl -gsS -G "${PROMETHEUS_URL}/api/v1/query" \
  --data-urlencode \
  'query=sum by (k8s_deployment_name,k8s_pod_name) (rate(http_server_request_duration_seconds_count{environment="develop",cluster="sevago-develop",k8s_namespace_name="sevago-develop",k8s_deployment_name=~".*-be"}[1m]))' |
jq -r '
  .data.result[]
  | [
      .metric.k8s_deployment_name,
      .metric.k8s_pod_name,
      (.value[1] + " req/s")
    ]
  | @tsv
'

echo
echo "===== P95 LATENCY BY POD ====="
curl -gsS -G "${PROMETHEUS_URL}/api/v1/query" \
  --data-urlencode \
  'query=1000 * histogram_quantile(0.95, sum by (le,k8s_deployment_name,k8s_pod_name) (rate(http_server_request_duration_seconds_bucket{environment="develop",cluster="sevago-develop",k8s_namespace_name="sevago-develop",k8s_deployment_name=~".*-be"}[5m])))' |
jq -r '
  .data.result[]
  | [
      .metric.k8s_deployment_name,
      .metric.k8s_pod_name,
      (.value[1] + " ms")
    ]
  | @tsv
'

echo
echo "===== P99 LATENCY BY POD ====="
curl -gsS -G "${PROMETHEUS_URL}/api/v1/query" \
  --data-urlencode \
  'query=1000 * histogram_quantile(0.99, sum by (le,k8s_deployment_name,k8s_pod_name) (rate(http_server_request_duration_seconds_bucket{environment="develop",cluster="sevago-develop",k8s_namespace_name="sevago-develop",k8s_deployment_name=~".*-be"}[5m])))' |
jq -r '
  .data.result[]
  | [
      .metric.k8s_deployment_name,
      .metric.k8s_pod_name,
      (.value[1] + " ms")
    ]
  | @tsv
'
```

Kết quả đạt khi:

```text
Active Backend Deployment có dữ liệu.
Latency histogram count > 0.
Request Rate trả về k8s_deployment_name + k8s_pod_name + req/s.
P95 trả về latency theo Pod.
P99 trả về latency theo Pod.
```

Traffic Develop có thể thấp nên `req/s` nhỏ là bình thường. Các health endpoint cũng tạo HTTP telemetry.

### 16.1. 📌 Label phải dùng cho application

Metric Beyla thực tế có các label:

```text
k8s_container_name
k8s_deployment_name
k8s_namespace_name
k8s_node_name
k8s_pod_name
http_request_method
http_response_status_code
http_route
```

Trong Beyla scrape series:

```text
namespace="monitor"
pod="beyla-..."
```

là namespace/Pod của **Beyla exporter**, không phải application.

Khi lọc application phải dùng:

```text
k8s_namespace_name="sevago-develop"
k8s_deployment_name
k8s_pod_name
```

`service_name` hiện không được dùng làm định danh chính vì có thể được Beyla suy ra giống nhau giữa nhiều backend. Dashboard dùng `k8s_deployment_name` và `k8s_pod_name`.

---

## 17. 🔎 Kiểm tra RAM Beyla bằng Central Prometheus

> **📍 Thực hiện tại:** Monitor Server `172.17.79.23`.

Cluster Develop hiện không có Metrics API, vì vậy không dùng `kubectl top` làm bước nghiệm thu.

```bash
PROMETHEUS_URL=http://127.0.0.1:14002

echo "===== CURRENT BEYLA MEMORY ====="
curl -gsS -G "${PROMETHEUS_URL}/api/v1/query" \
  --data-urlencode \
  'query=max by (pod) (container_memory_working_set_bytes{environment="develop",cluster="sevago-develop",namespace="monitor",pod=~"beyla-.*",container="beyla"}) / 1024 / 1024' |
jq -r '
  .data.result[]
  | "\(.metric.pod)\t\(.value[1]) MiB"
'

echo
echo "===== PEAK BEYLA MEMORY 30M ====="
curl -gsS -G "${PROMETHEUS_URL}/api/v1/query" \
  --data-urlencode \
  'query=max by (pod) (max_over_time(container_memory_working_set_bytes{environment="develop",cluster="sevago-develop",namespace="monitor",pod=~"beyla-.*",container="beyla"}[30m])) / 1024 / 1024' |
jq -r '
  .data.result[]
  | "\(.metric.pod)\t\(.value[1]) MiB"
'
```

Dùng kết quả này để theo dõi consumption thực tế. `limit=1Gi` chỉ là trần, không phải RAM cố định bị chiếm bởi mỗi Beyla Pod.

Nếu sau một thời gian ổn định peak thấp đáng kể so với limit, có thể cân nhắc hạ limit sau khi test lại. Không giảm limit chỉ để tiết kiệm RAM nếu chưa có số liệu peak.

---

## 18. 📌 Query dùng cho Grafana `09.1.1`

> **📍 Thực hiện tại:** Monitor Server `172.17.79.23`.

Sau khi K8s 21 nghiệm thu thành công, quay lại `09.1.1 - Grafana Kubernetes Detail`.

### 18.1. 📌 HTTP Request Rate by Pod

```promql
sum by (k8s_deployment_name, k8s_pod_name) (
  rate(
    http_server_request_duration_seconds_count{
      environment="${environment}",
      cluster="${cluster}",
      k8s_namespace_name="sevago-${environment}",
      k8s_deployment_name=~".*-be"
    }[1m]
  )
)
```

Grafana:

```text
Visualization: Time series
Type: Range
Legend: {{k8s_deployment_name}} / {{k8s_pod_name}}
Unit: req/s
Min: 0
```

### 18.2. 📌 HTTP Latency P95 by Pod

```promql
histogram_quantile(
  0.95,
  sum by (le, k8s_deployment_name, k8s_pod_name) (
    rate(
      http_server_request_duration_seconds_bucket{
        environment="${environment}",
        cluster="${cluster}",
        k8s_namespace_name="sevago-${environment}",
        k8s_deployment_name=~".*-be"
      }[5m]
    )
  )
)
```

Grafana:

```text
Visualization: Time series
Type: Range
Legend: {{k8s_deployment_name}} / {{k8s_pod_name}}
Unit: seconds
Min: 0
```

### 18.3. 📌 HTTP Latency P99 by Pod

```promql
histogram_quantile(
  0.99,
  sum by (le, k8s_deployment_name, k8s_pod_name) (
    rate(
      http_server_request_duration_seconds_bucket{
        environment="${environment}",
        cluster="${cluster}",
        k8s_namespace_name="sevago-${environment}",
        k8s_deployment_name=~".*-be"
      }[5m]
    )
  )
)
```

---

## 19. ✅ Checklist nghiệm thu Develop

```text
[ ] 3 worker đáp ứng kernel/BTF/eBPF prerequisite
[ ] Grafana Helm repository có trên master-1
[ ] Pin grafana/beyla chart 1.16.10
[ ] Beyla app version 3.25.0
[ ] 4 values file đã commit
[ ] Discovery chỉ instrument sevago-develop + *-be
[ ] Helm template thành công
[ ] Helm dry-run thành công
[ ] Helm release beyla = deployed
[ ] DaemonSet Beyla Ready 3/3
[ ] Beyla Pod ổn định, không restart/OOM
[ ] Prometheus Agent thấy 3 Beyla target health=up
[ ] http_server_request_duration_seconds_count có dữ liệu
[ ] http_server_request_duration_seconds_bucket có dữ liệu
[ ] Metric có k8s_deployment_name
[ ] Metric có k8s_pod_name
[ ] Request Rate theo Pod có dữ liệu
[ ] P95 theo Pod có dữ liệu
[ ] P99 theo Pod có dữ liệu
[ ] RAM Beyla được theo dõi qua cAdvisor/Central Prometheus
```

Sau khi checklist đạt:

```text
K8s 21 Develop hoàn thành
→ quay lại 09.1.1
→ dựng Request Rate / P95 / P99
→ tiếp tục roadmap monitoring
```

---

## 20. 📌 Mở rộng sang Staging và Production

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig của cluster đích; Develop: `172.17.79.90`.

Chỉ thực hiện sau khi Develop đã nghiệm thu.

Ma trận values:

| Kubernetes cluster | Values                                                    |
| ------------------ | --------------------------------------------------------- |
| Develop            | `beyla-base.values.yaml` + `beyla-develop.values.yaml`    |
| Staging Local      | `beyla-base.values.yaml` + `beyla-staging.values.yaml`    |
| Staging DMZ        | `beyla-base.values.yaml` + `beyla-staging.values.yaml`    |
| Production Local   | `beyla-base.values.yaml` + `beyla-production.values.yaml` |
| Production DMZ     | `beyla-base.values.yaml` + `beyla-production.values.yaml` |

Mỗi cluster có một release `beyla` riêng trong namespace `monitor`.

### 20.1. 🚀 Quy trình deploy một cluster đích

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig của cluster đích; Develop: `172.17.79.90`.

Trước khi deploy phải xác nhận đúng cluster/context và đúng `external_labels` của Prometheus Agent.

Ví dụ Staging:

```bash
BASE_VALUES=~/app-core/layer-monitor/kubernetes/beyla/beyla-base.values.yaml
ENV_VALUES=~/app-core/layer-monitor/kubernetes/beyla/beyla-staging.values.yaml
BEYLA_CHART_VERSION=1.16.10

set -e

echo "===== CLUSTER ====="
kubectl config current-context
kubectl get nodes -o wide

echo "===== PROMETHEUS AGENT ====="
kubectl -n monitor get deployment prometheus-agent

echo "===== DRY RUN ====="
helm upgrade --install beyla grafana/beyla --namespace monitor --version "${BEYLA_CHART_VERSION}" -f "${BASE_VALUES}" -f "${ENV_VALUES}" --dry-run > /tmp/beyla-dry-run.txt

echo "===== DEPLOY ====="
helm upgrade --install beyla grafana/beyla --namespace monitor --version "${BEYLA_CHART_VERSION}" -f "${BASE_VALUES}" -f "${ENV_VALUES}" --atomic --timeout 5m

echo "===== VERIFY ====="
kubectl -n monitor rollout status daemonset/beyla --timeout=300s
kubectl -n monitor get daemonset beyla
kubectl -n monitor get pods -l app.kubernetes.io/name=beyla -o wide
```

Kết quả đạt yêu cầu:

```text
Đúng cluster đích.
Prometheus Agent tồn tại.
Helm deploy thành công.
DESIRED=CURRENT=READY=AVAILABLE.
Beyla Pod Running/Ready trên các worker application.
```

Không hardcode số Beyla Pod cho Staging/Production vì số worker giữa các cluster có thể khác nhau.

### 20.2. ✅ Nghiệm thu trên Central Prometheus

> **📍 Thực hiện tại:** Monitor Server `172.17.79.23`.

Query theo đúng `environment` và `cluster` thực tế của cluster đích:

```promql
sum by (cluster, k8s_deployment_name, k8s_pod_name) (
  rate(
    http_server_request_duration_seconds_count{
      environment="<environment>",
      cluster="<unique-cluster-label>",
      k8s_deployment_name=~".*-be"
    }[5m]
  )
)
```

Kết quả đạt:

```text
Chỉ Pod của cluster đang kiểm tra xuất hiện.
Không trộn dữ liệu giữa Local và DMZ.
```

---

## 21. ↩️ Rollback

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig của cluster đích; Develop: `172.17.79.90`.

```bash
helm uninstall beyla -n monitor

kubectl -n monitor get pods -l app.kubernetes.io/name=beyla
```

Kết quả mong đợi:

```text
release "beyla" uninstalled
No resources found
```

Rollback Beyla không thay đổi application Deployment/Service/Ingress và không yêu cầu rollback source application.

---

## 22. 📚 Tham chiếu kỹ thuật

- Grafana Beyla: https://grafana.com/docs/beyla/latest/
- Deploy Beyla bằng Helm: https://grafana.com/docs/beyla/latest/setup/kubernetes-helm/
- Beyla Kubernetes deployment: https://grafana.com/docs/beyla/latest/setup/kubernetes/
- Beyla exported metrics: https://grafana.com/docs/beyla/latest/metrics/
- Beyla service discovery: https://grafana.com/docs/beyla/latest/configure/service-discovery/
- Beyla Helm chart: https://github.com/grafana/beyla/tree/main/charts/beyla
