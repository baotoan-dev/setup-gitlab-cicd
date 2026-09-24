# ☸️ Bước 14 - Cấu hình Jenkins deploy Kubernetes

## 1. 🎯 Mục đích

File này cấu hình Jenkins dùng kubectl/Helm với ServiceAccount, RBAC và kubeconfig **riêng cho từng một trong 5 cluster**, không dùng admin.conf hoặc cluster-admin lâu dài.

File này cấu hình Jenkins đang chạy bằng Docker để deploy application vào Kubernetes bằng:

```text
kubectl
Helm
ServiceAccount
RBAC
Kubeconfig
```

Jenkins đã được cài bằng Docker Compose ở tài liệu trước.

File này không cài Jenkins lại từ đầu, mà chỉ bổ sung khả năng deploy vào Kubernetes.

Mô hình hiện tại:

```text
1 Jenkins
1 Harbor

5 Kubernetes Cluster:
- sevago-develop
- sevago-staging-local
- sevago-staging-dmz
- sevago-production-local
- sevago-production-dmz
```

Trong mỗi cluster có nhiều namespace.

Ví dụ Develop Cluster:

```text
sevago-develop
sevaedu-develop
sevaretail-develop
```

Quy ước quyền Jenkins:

```text
1 cluster môi trường
=
1 ServiceAccount Jenkins
=
1 kubeconfig Jenkins
=
N namespace được cấp quyền deploy
```

Ví dụ Develop Cluster:

```text
ServiceAccount:
jenkins-develop

Kubeconfig:
config-develop

Deploy được:
sevago-develop
sevaedu-develop
sevaretail-develop
```

---

## 2. ✅ Checklist nội dung

```text
[ ] Jenkins container có Docker CLI
[ ] Jenkins container có Git
[ ] Jenkins container có kubectl
[ ] Jenkins container có Helm

[ ] Namespace cicd tồn tại
[ ] ServiceAccount cicd/jenkins-develop tồn tại

[ ] ClusterRole jenkins-namespace-deployer tồn tại

[ ] RoleBinding jenkins-develop tồn tại trong sevago-develop
[ ] RoleBinding jenkins-develop tồn tại trong sevaedu-develop
[ ] RoleBinding jenkins-develop tồn tại trong sevaretail-develop

[ ] RBAC chỉ giới hạn trong namespace được bind
[ ] Jenkins không có quyền kube-system
[ ] Jenkins không có quyền get nodes

[ ] kubeconfig config-develop tạo thành công
[ ] kubeconfig dùng ServiceAccount cicd/jenkins-develop
[ ] kubeconfig trỏ tới API VIP 172.17.79.159

[ ] kubeconfig đã copy vào jenkins-data/.kube/config-develop
[ ] Jenkins đọc được kubeconfig

[ ] Jenkins kubectl get pods OK trên các namespace được cấp quyền
[ ] Jenkins auth can-i OK
[ ] Jenkins helm list OK
[ ] Jenkins curl API VIP healthz OK

[ ] Jenkinsfile set đúng KUBECONFIG
[ ] Jenkinsfile set đúng NAMESPACE
```

---

## 3. 📌 Mục tiêu

Sau khi hoàn thành file này:

```text
Jenkins container có kubectl
Jenkins container có Helm
Jenkins gọi được Kubernetes API VIP
Jenkins deploy được vào các namespace được cấp quyền
Jenkins không dùng admin.conf lâu dài
Jenkins không có cluster-admin toàn cluster
```

Luồng CI/CD:

```text
Developer
  -> GitLab
  -> Jenkins Docker
  -> Build Docker image
  -> Push image lên Harbor
  -> Helm deploy vào Kubernetes API VIP
  -> Pod chạy trong namespace app
```

---

## 4. 🧰 Kiểm tra Jenkins có kubectl và Helm chưa

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` (và container `jenkins` khi lệnh yêu cầu).

Ví dụ container Jenkins tên là:

```text
jenkins
```

Vào Jenkins container:

```bash
docker exec -it jenkins bash
```

Kiểm tra:

```bash
kubectl version --client=true
helm version
docker version
git --version
```

Kết quả mong đợi:

```text
kubectl hoạt động
helm hoạt động
docker hoạt động
git hoạt động
```

Thoát container:

```bash
exit
```

Nếu gặp lỗi:

```text
kubectl: command not found
```

hoặc:

```text
helm: command not found
```

thì thực hiện mục 4 để build lại Jenkins image.

---

## 5. 📦 Bổ sung kubectl và Helm vào Jenkins image nếu chưa có

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` (và container `jenkins` khi lệnh yêu cầu).

Chỉ làm bước này nếu Jenkins image hiện tại chưa có `kubectl` hoặc `helm`.

### 5.1. 🧰 Sửa Dockerfile Jenkins

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` (và container `jenkins` khi lệnh yêu cầu).

Mở file:

```bash
nano ~/app-core/layer-cicd/Dockerfile.build.custom
```

Thay nội dung bằng:

```dockerfile
FROM jenkins/jenkins:lts

USER root

RUN apt-get update && \
    apt-get install -y \
        docker.io \
        git \
        openssh-client \
        curl \
        ca-certificates \
        apt-transport-https \
        gnupg \
        lsb-release && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

ARG K8S_MINOR=v1.36

RUN KUBECTL_VERSION=$(curl -L -s https://dl.k8s.io/release/stable-${K8S_MINOR#v}.txt) && \
    curl -fsSL -o /tmp/kubectl https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl && \
    chmod +x /tmp/kubectl && \
    mv /tmp/kubectl /usr/local/bin/kubectl

RUN curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

RUN usermod -aG docker jenkins

USER jenkins
```

> Cluster hiện tại đang dùng Kubernetes `v1.36.x`, nên dùng `ARG K8S_MINOR=v1.36`.

---

### 5.2. 📦 Build lại Jenkins image

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` (và container `jenkins` khi lệnh yêu cầu).

```bash
cd ~/app-core/layer-cicd
docker compose -f build.yaml down
docker compose -f build.yaml up -d --build
```

Kiểm tra container:

```bash
docker ps
```

---

### 5.3. 🧰 Xác nhận Jenkins đã có kubectl và Helm

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` (và container `jenkins` khi lệnh yêu cầu).

```bash
docker exec -it jenkins bash
```

Trong container:

```bash
kubectl version --client=true
helm version
docker version
git --version
```

Kết quả mong đợi:

```text
kubectl client version hiển thị được
helm version hiển thị được
docker version hiển thị được
git version hiển thị được
```

Thoát:

```bash
exit
```

Ghi chú:

```text
Chỉ cần build lại Jenkins image một lần.

Sau khi hoàn thành:
✓ Docker CLI
✓ Git
✓ SSH Client
✓ kubectl
✓ Helm

đã có trong Jenkins container.
```

### 5.4. ⚙️ Cấu hình GitLab SSH Host Key

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` (và container `jenkins` khi lệnh yêu cầu).

Pipeline clone source bằng SSH nên Jenkins phải trust SSH Host Key của GitLab.

```bash
docker exec -u root jenkins sh -c '
mkdir -p "$JENKINS_HOME/.ssh"
touch "$JENKINS_HOME/.ssh/known_hosts"
ssh-keyscan -p 2222 gitlab.sevatech.local >> "$JENKINS_HOME/.ssh/known_hosts"
sort -u "$JENKINS_HOME/.ssh/known_hosts" -o "$JENKINS_HOME/.ssh/known_hosts"
chown -R jenkins:jenkins "$JENKINS_HOME/.ssh"
chmod 700 "$JENKINS_HOME/.ssh"
chmod 600 "$JENKINS_HOME/.ssh/known_hosts"
'
```

Kiểm tra file:

```bash
docker exec jenkins sh -c '
ls -lah "$JENKINS_HOME/.ssh/known_hosts"
ssh-keygen -lf "$JENKINS_HOME/.ssh/known_hosts"
'
```

Kết quả mong đợi:

```text
known_hosts tồn tại
Hiển thị fingerprint của gitlab.sevatech.local
```

> Đối chiếu fingerprint với SSH Host Key do quản trị viên GitLab cung cấp trước khi sử dụng.

---

Sau bước này, hai file Pipeline mới được đổi:

StrictHostKeyChecking=no

thành:

StrictHostKeyChecking=yes

---

## 6. 👤 Quy hoạch account Jenkins

Cụm `sevago-develop` dùng một ServiceAccount Jenkins riêng:

| Cluster                 | Namespace ứng dụng  | ServiceAccount             | Kubeconfig                |
| ----------------------- | ------------------- | -------------------------- | ------------------------- |
| sevago-develop          | `sevago-develop`    | `jenkins-develop`          | `config-develop`          |
| sevago-staging-local    | `sevago-staging`    | `jenkins-staging-local`    | `config-staging-local`    |
| sevago-staging-dmz      | `sevago-staging`    | `jenkins-staging-dmz`      | `config-staging-dmz`      |
| sevago-production-local | `sevago-production` | `jenkins-production-local` | `config-production-local` |
| sevago-production-dmz   | `sevago-production` | `jenkins-production-dmz`   | `config-production-dmz`   |

Lý do đặt ServiceAccount trong namespace `cicd`:

```text
Tách account CI/CD khỏi namespace app
Dễ quản lý
Dễ audit
Không phải tạo ServiceAccount lặp lại trong từng namespace app
```

Jenkins chỉ deploy được vào namespace nào có `RoleBinding` trỏ tới ServiceAccount tương ứng.

---

## 7. ➕ Tạo namespace cicd

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

Ví dụ Develop Cluster:

```bash
kubectl create namespace cicd --dry-run=client -o yaml | kubectl apply -f -
```

Kiểm tra:

```bash
kubectl get namespace cicd
```

---

## 8. 👤 Tạo ServiceAccount Jenkins

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

Ví dụ cho Develop Cluster:

```bash
kubectl create serviceaccount jenkins-develop -n cicd --dry-run=client -o yaml | kubectl apply -f -
```

Kiểm tra:

```bash
kubectl get sa jenkins-develop -n cicd
```

---

## 9. 👤 Tạo ClusterRole dùng chung cho Jenkins deploy namespace

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

ClusterRole này định nghĩa Jenkins được thao tác các loại resource cần deploy app.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-namespace-deployer
rules:
- apiGroups: [""]
  resources:
    - pods
    - pods/log
    - services
    - endpoints
    - configmaps
    - secrets
    - serviceaccounts
  verbs:
    - get
    - list
    - watch
    - create
    - update
    - patch
    - delete
- apiGroups: ["apps"]
  resources:
    - deployments
    - replicasets
    - statefulsets
    - daemonsets
  verbs:
    - get
    - list
    - watch
    - create
    - update
    - patch
    - delete
- apiGroups: ["batch"]
  resources:
    - jobs
    - cronjobs
  verbs:
    - get
    - list
    - watch
    - create
    - update
    - patch
    - delete
- apiGroups: ["networking.k8s.io"]
  resources:
    - ingresses
  verbs:
    - get
    - list
    - watch
    - create
    - update
    - patch
    - delete
- apiGroups: ["discovery.k8s.io"]
  resources:
    - endpointslices
  verbs:
    - get
    - list
    - watch
    - create
    - update
    - patch
    - delete
EOF
```

Kiểm tra:

```bash
kubectl get clusterrole jenkins-namespace-deployer
```

> Không dùng `ClusterRoleBinding` để tránh Jenkins có quyền toàn cluster.

---

## 10. 🧰 Cấp quyền deploy cho Jenkins vào các namespace app

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

Ví dụ Develop Cluster có các namespace:

```text
sevago-develop
sevaedu-develop
sevaretail-develop
```

### 10.1. ➕ Tạo namespace app nếu chưa có

```bash
kubectl create namespace sevago-develop --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace sevaedu-develop --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace sevaretail-develop --dry-run=client -o yaml | kubectl apply -f -
```

---

### 10.2. 👤 Tạo RoleBinding cho từng namespace

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

RoleBinding sẽ gắn:

```text
ClusterRole:
jenkins-namespace-deployer

ServiceAccount:
cicd/jenkins-develop
```

vào từng namespace app.

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

Kiểm tra:

```bash
kubectl get rolebinding -A | grep jenkins-develop
```

Kết quả mong đợi:

```text
sevago-develop       jenkins-develop
sevaedu-develop      jenkins-develop
sevaretail-develop   jenkins-develop
```

---

## 11. 🔐 Kiểm tra quyền RBAC

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

Kiểm tra Jenkins có quyền trong namespace được cấp:

```bash
kubectl auth can-i get pods -n sevago-develop --as=system:serviceaccount:cicd:jenkins-develop
kubectl auth can-i patch deployments -n sevago-develop --as=system:serviceaccount:cicd:jenkins-develop
kubectl auth can-i create ingresses -n sevago-develop --as=system:serviceaccount:cicd:jenkins-develop
```

Kết quả mong đợi:

```text
yes
yes
yes
```

Kiểm tra không có quyền ở namespace hệ thống:

```bash
kubectl auth can-i get pods -n kube-system --as=system:serviceaccount:cicd:jenkins-develop
```

Kết quả mong đợi:

```text
no
```

Kiểm tra không có quyền cluster-wide:

```bash
kubectl auth can-i get nodes --as=system:serviceaccount:cicd:jenkins-develop
```

Kết quả mong đợi:

```text
no
```

---

## 12. 🧰 Tạo kubeconfig cho Jenkins

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

Ví dụ:

```text
sevago-dev-k8s-master-1
```

---

### 12.1. 🧰 Tạo token lâu dài cho Jenkins

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

Jenkins cần kubeconfig hoạt động lâu dài, vì vậy sử dụng ServiceAccount Token Secret.

Tạo Secret:

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
```

Đợi vài giây để Kubernetes sinh token:

```bash
kubectl -n cicd get secret jenkins-develop-token
```

Lấy token:

```bash
TOKEN=$(kubectl -n cicd get secret jenkins-develop-token -o jsonpath='{.data.token}' | base64 -d)
```

Kiểm tra:

```bash
echo $TOKEN
```

Kết quả mong đợi:

```text
eyJhbGciOi...
```

Ghi chú:

```text
Không sử dụng:

kubectl create token jenkins-develop

vì token tạo ra có thời hạn sử dụng.

Jenkins kubeconfig phải sử dụng token lấy từ:

Secret type:
kubernetes.io/service-account-token

để hoạt động ổn định lâu dài.
```

### 12.2. 📌 Lấy CA của cluster

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

```bash
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' > /tmp/k8s-ca.txt
```

Kiểm tra:

```bash
cat /tmp/k8s-ca.txt
```

Kết quả mong đợi:

```text
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS...
```

---

### 12.3. ➕ Tạo kubeconfig

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

```bash
cat > /tmp/config-develop <<EOF
apiVersion: v1
kind: Config
clusters:
- name: sevago-dev-k8s
  cluster:
    certificate-authority-data: $(cat /tmp/k8s-ca.txt)
    server: https://172.17.79.159:6443
contexts:
- name: jenkins-develop@sevago-dev-k8s
  context:
    cluster: sevago-dev-k8s
    user: jenkins-develop
    namespace: sevago-develop
current-context: jenkins-develop@sevago-dev-k8s
users:
- name: jenkins-develop
  user:
    token: ${TOKEN}
EOF
```

---

### 12.4. 🔎 Kiểm tra kubeconfig

```bash
kubectl --kubeconfig=/tmp/config-develop get pods -n sevago-develop
```

```bash
kubectl --kubeconfig=/tmp/config-develop auth can-i patch deployments -n sevago-develop
```

Kết quả mong đợi:

```text
No resources found
```

hoặc:

```text
Danh sách Pod
```

và:

```text
yes
```

---

### 12.5. 🧰 Copy kubeconfig sang VPS Jenkins

Ví dụ Jenkins VPS:

```text
build.sevatech.local
172.17.79.21
```

Copy file:

```bash
scp /tmp/config-develop devops@172.17.79.21:/tmp/config-develop
```

Kiểm tra trên Jenkins VPS:

```bash
ssh devops@172.17.79.21
ls -l /tmp/config-develop
```

Kết quả mong đợi:

```text
/tmp/config-develop tồn tại
```

---

## 13. 🧰 Đưa kubeconfig vào Jenkins

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` (và container `jenkins` khi lệnh yêu cầu).

Ví dụ:

```text
build.sevatech.local
172.17.79.21
```

---

### 13.1. ➕ Tạo thư mục kubeconfig, copy kubeconfig, phân quyền

```bash
cd ~/app-core
sudo mkdir -p jenkins-data/.kube
sudo cp /tmp/config-develop jenkins-data/.kube/config-develop
sudo chown -R 1000:1000 jenkins-data/.kube
sudo chmod 700 jenkins-data/.kube
sudo chmod 600 jenkins-data/.kube/config-develop
```

### 13.2. 🧰 Restart Jenkins

```bash
cd ~/app-core/layer-cicd
docker compose -f build.yaml restart
```

---

### 13.3. 🧰 Test trong Jenkins container

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` (và container `jenkins` khi lệnh yêu cầu).

Vào container:

```bash
docker exec -it jenkins bash
```

Thiết lập kubeconfig:

```bash
export KUBECONFIG=/var/jenkins_home/.kube/config-develop
```

Kiểm tra namespace được cấp quyền:

```bash
kubectl get pods -n sevago-develop
kubectl auth can-i patch deployments -n sevago-develop
helm version
kubectl version --client=true
```

Kiểm tra kết nối API:

```bash
curl -k https://172.17.79.159:6443/healthz
```

Kết quả mong đợi:

```text
No resources found
```

hoặc:

```text
Danh sách Pod
```

và:

```text
yes
```

và:

```text
ok
```

---

### 13.4. 🔎 Kiểm tra hoàn tất

Nếu các lệnh trên thành công:

```text
✓ Jenkins kết nối được Kubernetes API
✓ Jenkins dùng ServiceAccount jenkins-develop
✓ Jenkins có RBAC giới hạn namespace
✓ Jenkins sẵn sàng deploy bằng Helm
✓ Không dùng admin.conf
✓ Không cần cluster-admin
```

---

## 14. 🧰 Test trong Jenkins container

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` (và container `jenkins` khi lệnh yêu cầu).

```bash
docker exec -it jenkins bash
```

Trong container Jenkins:

```bash
export KUBECONFIG=/var/jenkins_home/.kube/config-develop
kubectl get pods -n sevago-develop
kubectl auth can-i patch deployments -n sevago-develop
helm list -n sevago-develop
curl -k https://172.17.79.159:6443/healthz
```

Kết quả mong đợi:

```text
kubectl truy cập được các namespace đã cấp quyền
auth can-i trả yes
helm list không lỗi
healthz trả ok
```

Thoát:

```bash
exit
```

---

## 15. 🧰 Cập nhật namespace mà Jenkins được phép deploy

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

Phần này dùng khi sau này có thêm hệ thống mới trong cùng cluster.

Ví dụ thêm namespace:

```text
sevacrm-develop
```

### 15.1. ➕ Tạo namespace mới

```bash
kubectl create namespace sevacrm-develop --dry-run=client -o yaml | kubectl apply -f -
```

### 15.2. 👤 Thêm RoleBinding cho Jenkins

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-develop
  namespace: sevacrm-develop
subjects:
- kind: ServiceAccount
  name: jenkins-develop
  namespace: cicd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-namespace-deployer
EOF
```

### 15.3. 🔎 Kiểm tra quyền

```bash
kubectl auth can-i get pods -n sevacrm-develop --as=system:serviceaccount:cicd:jenkins-develop
```

Kết quả mong đợi:

```text
yes
```

Sau khi thêm RoleBinding:

```text
Không cần tạo ServiceAccount mới
Không cần tạo kubeconfig mới
Không cần restart Jenkins
```

Jenkins dùng lại:

```text
config-develop
```

để deploy namespace mới.

---

## 16. 🚀 Gỡ quyền deploy khỏi namespace

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

Ví dụ không muốn Jenkins deploy vào:

```text

```

Xóa RoleBinding:

```bash

```

Kiểm tra:

```bash
kubectl auth can-i get pods --as=system:serviceaccount:cicd:jenkins-develop
```

Kết quả mong đợi:

```text
no
```

---

## 17. 🧰 Jenkinsfile mẫu

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` (và container `jenkins` khi lệnh yêu cầu).

Ví dụ deploy vào `sevago-develop`:

```groovy
pipeline {
  agent any

  environment {
    KUBECONFIG = '/var/jenkins_home/.kube/config-develop'
    NAMESPACE = 'sevago-develop'
    IMAGE_REPOSITORY = 'registry.sevatech.local/sevago-develop/office-fe'
    IMAGE_TAG = "${BUILD_NUMBER}"
  }

  stages {
    stage('Check Cluster') {
      steps {
        sh '''
          kubectl get pods -n $NAMESPACE
          helm list -n $NAMESPACE
        '''
      }
    }

    stage('Deploy') {
      steps {
        sh '''
          helm upgrade --install office-fe ./helm-chart \
            -n $NAMESPACE \
            --set image.repository=$IMAGE_REPOSITORY \
            --set image.tag=$IMAGE_TAG
        '''
      }
    }
  }
}
```

Pipeline deploy vào `sevago-develop` dùng kubeconfig sau:

```groovy
environment {
  KUBECONFIG = '/var/jenkins_home/.kube/config-develop'
  NAMESPACE = 'sevago-develop'
}
```

---

## 18. 🧰 Quy hoạch kubeconfig Jenkins

Jenkins lưu **một kubeconfig riêng cho từng cluster**; không dùng chung credential giữa Local/DMZ hoặc giữa các môi trường:

```text
/var/jenkins_home/.kube/config-develop
/var/jenkins_home/.kube/config-staging-local
/var/jenkins_home/.kube/config-staging-dmz
/var/jenkins_home/.kube/config-production-local
/var/jenkins_home/.kube/config-production-dmz
```

Danh tính Kubernetes tương ứng:

```text
Develop:
KUBECONFIG=/var/jenkins_home/.kube/config-develop

Staging:
KUBECONFIG=/var/jenkins_home/.kube/config-staging-dmz
KUBECONFIG=/var/jenkins_home/.kube/config-staging-local

Production:
KUBECONFIG=/var/jenkins_home/.kube/config-production-dmz
KUBECONFIG=/var/jenkins_home/.kube/config-production-local
```

Mỗi cluster có ServiceAccount riêng:

```text
Develop Cluster:
cicd/jenkins-develop

Staging Local Cluster:
cicd/jenkins-staging-local

Staging DMZ Cluster:
cicd/jenkins-staging-dmz

Production Local Cluster:
cicd/jenkins-production-local

Production DMZ Cluster:
cicd/jenkins-production-dmz
```

---

## 19. 🛠️ Troubleshooting

### 19.1. 🧰 Jenkins thiếu kubectl hoặc Helm

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` (và container `jenkins` khi lệnh yêu cầu).

Trong Jenkins container:

```bash
kubectl version --client=true
helm version
```

Nếu lỗi:

```text
command not found
```

thì quay lại mục 4 để build lại Jenkins image.

---

### 19.2. 🧰 Jenkins không kết nối được Kubernetes API

> **📍 Thực hiện tại:** VPS Jenkins `172.17.79.21` (và container `jenkins` khi lệnh yêu cầu).

Trong Jenkins container:

```bash
curl -k https://172.17.79.159:6443/healthz
```

Kiểm tra:

```text
Route từ Jenkins container tới API VIP
Firewall allow Jenkins host/container tới 6443
Kubeconfig server đúng API VIP
```

---

### 19.3. 🔐 Jenkins bị lỗi RBAC

> **📍 Thực hiện tại:** Control-plane-1/máy quản trị có kubeconfig của **cluster đích**; không chạy nhầm kubeconfig giữa 5 cluster.

Kiểm tra:

```bash
kubectl --kubeconfig=/var/jenkins_home/.kube/config-develop auth can-i get pods -n sevago-develop
kubectl --kubeconfig=/var/jenkins_home/.kube/config-develop auth can-i patch deployments -n sevago-develop
kubectl --kubeconfig=/var/jenkins_home/.kube/config-develop auth can-i create ingresses -n sevago-develop
```

Nếu Helm deploy lỗi `forbidden`, xem resource nào bị deny rồi bổ sung `ClusterRole` hoặc RoleBinding đúng mức.
