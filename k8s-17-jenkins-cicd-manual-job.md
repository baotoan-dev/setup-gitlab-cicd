# ☸️ Bước 17 - Tạo Jenkins Manual Job

## 1. 🎯 Mục đích

File này tạo Jenkins Manual Job cho **Develop** để build/deploy theo yêu cầu, tái sử dụng logic CI/CD hiện hữu nhưng không phụ thuộc webhook.

Tạo một Jenkins Pipeline Job để build và deploy thủ công lên môi trường **Develop**.

Job này kế thừa toàn bộ logic build/deploy của Job webhook tự động, nhưng bổ sung các bước cần thiết để tự lấy những biến mà webhook trước đây cung cấp.

Đặc điểm: Tên Job: `sevago-develop-manual`, Chỉ chọn Repository cần build, Branch luôn cố định là `develop`.

- Danh sách Repository được tải từ GitLab API
- Tự tìm `REPO_PATH` và tạo `REPO_URL`
- Tự lấy `COMMIT`, `COMMIT_SHORT`, `COMMIT_AUTHOR` và `COMMIT_MESSAGE`
- Build Docker Image
- Push Image lên Harbor
- Deploy bằng Helm lên Kubernetes
- Không chạy các bước kiểm tra Pod, Service, Ingress hoặc Rollout sau deploy
- Không hardcode GitLab Token hoặc Harbor Credential

---

## 2. ✅ Checklist nội dung

- [ ] Luồng hoạt động
- [ ] Plugin cần cài
- [ ] Tạo GitLab API Credential
- [ ] Tạo Harbor Credential
- [ ] Tạo Job `sevago-develop-manual`
- [ ] Tạo Parameter `REPO_SELECT`
- [ ] Cấu hình Groovy Script tải Repository từ GitLab
- [ ] Cấu hình Pipeline
- [ ] Credential Pipeline sử dụng
- [ ] Cách chạy Job
- [ ] Điểm khác nhau giữa Job auto và Job manual

---

## 3. 🔀 Luồng hoạt động

Job webhook tự động nhận sẵn:

```text
BRANCH
REPO
REPO_URL
COMMIT
COMMIT_AUTHOR
COMMIT_MESSAGE
```

Job manual chỉ nhận:

```text
REPO_SELECT
```

Vì vậy Job manual phải tự bổ sung:

```text
REPO_SELECT → BRANCH=develop → Gọi GitLab API tìm REPO_PATH → Tạo REPO_URL → Clone source → Lấy COMMIT → Tính COMMIT_SHORT → Lấy COMMIT_AUTHOR → Lấy COMMIT_MESSAGE → Chạy lại toàn bộ logic build/deploy cũ
```

Phần build/deploy chính vẫn giữ nguyên:

```text
Clone source → Clone app-core → Clone app-helm → Xác định FE/BE/Admin → Tính domain, TLS và replicaCount → Build Docker Image → Push Harbor → Update Kubernetes Secret → Helm upgrade --install → Cleanup
```

---

## 4. ⚙️ Plugin cần cài

> **📍 Thực hiện tại:** Jenkins UI/VPS `172.17.79.21`.

Vào:

```text
Manage Jenkins → Plugins
```

Cài:

```text
Active Choices
Pipeline
Credentials Binding
```

Khởi động lại Jenkins nếu được yêu cầu.

> `SSH Agent` không bắt buộc vì Job hiện tại clone source bằng SSH key có sẵn tại `/var/jenkins_home/.ssh/`.

---

## 5. 🔐 Tạo GitLab API Credential

> **📍 Thực hiện tại:** Jenkins UI/VPS `172.17.79.21`.

Vào:

```text
Manage Jenkins → Credentials → System → Global Credentials → Add Credentials
```

Chọn:

```text
Kind: Secret text
```

Điền:

| Field       | Value                        |
| ----------- | ---------------------------- |
| Secret      | GitLab Personal Access Token |
| ID          | `gitlab-token`               |
| Description | GitLab API Token             |

Token chỉ cần quyền:

```text
read_api
```

Credential này được dùng cho:

- Active Choices gọi GitLab API để tải danh sách Repository
- Pipeline gọi GitLab API để tìm chính xác `path_with_namespace`

---

## 6. 🔐 Tạo Harbor Credential

> **📍 Thực hiện tại:** Jenkins UI/VPS `172.17.79.21`.

Vào:

```text
Manage Jenkins → Credentials → System → Global Credentials → Add Credentials
```

Chọn:

```text
Kind: Username with password
```

Điền:

| Field       | Value                                 |
| ----------- | ------------------------------------- |
| Username    | `robot$sevago-develop+sevago-develop` |
| Password    | Harbor Robot Secret                   |
| ID          | `harbor-develop`                      |
| Description | Harbor Develop Robot                  |

Pipeline sẽ tự inject:

```text
REGISTRY_USER
REGISTRY_SECRET
```

thông qua:

```groovy
usernamePassword(
    credentialsId: 'harbor-develop',
    usernameVariable: 'REGISTRY_USER',
    passwordVariable: 'REGISTRY_SECRET'
)
```

Không ghi trực tiếp Harbor username/password trong Pipeline.

---

## 7. 🧰 Tạo Job `sevago-develop-manual`

> **📍 Thực hiện tại:** Jenkins UI/VPS `172.17.79.21`.

Vào:

```text
New Item
```

Nhập tên:

```text
sevago-develop-manual
```

Chọn loại:

```text
Pipeline
```

Bấm:

```text
OK
```

Trong phần cấu hình Job, bật:

```text
This project is parameterized
```

---

## 8. ➕ Tạo Parameter `REPO_SELECT`

> **📍 Thực hiện tại:** Jenkins UI/VPS `172.17.79.21`.

Trong phần Parameters, chọn:

```text
Add Parameter → Active Choices Parameter
```

Cấu hình:

| Field         | Value                                  |
| ------------- | -------------------------------------- |
| Name          | `REPO_SELECT`                          |
| Default Value | Để trống                               |
| Description   | Tên Repository trong `sevago/modules/` |

Ví dụ:

```text
account-fe
account-be
office-fe
office-be
```

Người chạy Job nhập tên Repository cần build.

---

## 9. ⚙️ Cấu hình Groovy Script tải Repository từ GitLab

> **📍 Thực hiện tại:** Jenkins UI/VPS `172.17.79.21`.

Trong `REPO_SELECT`, bật:

```text
Groovy Sandbox
```

Dán Groovy Script sau:

```groovy
import com.cloudbees.plugins.credentials.CredentialsProvider
import org.jenkinsci.plugins.plaincredentials.StringCredentials
import jenkins.model.Jenkins
import groovy.json.JsonSlurper

try {
    String credentialId = 'gitlab-token'
    String gitlabApi = 'https://gitlab.sevatech.local/api/v4'

    def credential = CredentialsProvider.lookupCredentialsInItemGroup(
        StringCredentials.class, Jenkins.get(), null, null
    ).find { it.id == credentialId }

    if (!credential) return ["ERROR: Không tìm thấy Secret text có ID ${credentialId}"]

    String token = credential.secret.plainText
    List<String> repos = []
    int page = 1

    while (true) {
        String apiUrl = "${gitlabApi}/projects?membership=true&simple=true&per_page=100&page=${page}"
        def process = ['curl', '-sk', '--header', "PRIVATE-TOKEN: ${token}", apiUrl].execute()
        String output = process.text
        int exitCode = process.waitFor()

        if (exitCode != 0) return ["ERROR: Không gọi được GitLab API, exitCode=${exitCode}"]

        List projects = new JsonSlurper().parseText(output) as List
        if (!projects) break

        projects.each { project ->
            String repoName = project.path?.toString()
            String repoPath = project.path_with_namespace?.toString()
            if (repoName && repoPath?.startsWith('sevago/modules/')) repos.add(repoName)
        }

        if (projects.size() < 100) break
        page++
    }

    repos = repos.unique().sort()
    return repos ?: ['ERROR: Không tìm thấy repository trong sevago/modules/']
} catch (Exception e) {
    return ["ERROR: ${e.class.simpleName}: ${e.message}"]
}
```

### 9.1. 📌 Fallback Script

Dán:

```groovy
return ['ERROR: Không thể tải danh sách repository từ GitLab']
```

Sau khi lưu, dropdown có dạng:

```text
account-fe
account-be
customer-fe
customer-be
warehouse-fe
warehouse-be
...
```

Nếu Jenkins yêu cầu Script Approval, vào:

```text
Manage Jenkins → In-process Script Approval
```

Approve các signature đang chờ rồi tải lại trang:

```text
Build with Parameters
```

---

## 10. 🧰 Cấu hình Pipeline

> **📍 Thực hiện tại:** Jenkins UI/VPS `172.17.79.21`.

Trong Job, kéo xuống phần:

```text
Pipeline
```

Chọn:

```text
Definition: Pipeline script
```

Xem job ở jenkins

---

## 11. 🔐 Credential Pipeline sử dụng

> **📍 Thực hiện tại:** Jenkins UI/VPS `172.17.79.21`.

Pipeline sử dụng đúng hai Credential:

| Credential ID    | Kind                   | Công dụng                                 |
| ---------------- | ---------------------- | ----------------------------------------- |
| `gitlab-token`   | Secret text            | Tải danh sách repo và tìm repository path |
| `harbor-develop` | Username with password | Đăng nhập Harbor                          |

---

## 12. 🧰 Cách chạy Job

Vào:

```text
sevago-develop-manual → Build with Parameters
```

Chọn Repository tại:

```text
REPO_SELECT
```

Sau đó bấm:

```text
Build
```

Pipeline sẽ thực hiện:

```text
1. Cố định branch develop
2. Tìm repository path từ GitLab API
3. Tạo REPO_URL
4. Clone repository branch develop
5. Lấy COMMIT
6. Tính COMMIT_SHORT
7. Lấy COMMIT_AUTHOR và COMMIT_MESSAGE
8. Clone app-core
9. Clone app-helm
10. Xác định FE, BE hoặc Admin
11. Xác định domain, TLS và replicaCount
12. Build Docker image
13. Push image lên Harbor
14. Cập nhật Kubernetes Secret env
15. Chạy Helm upgrade --install
16. Xóa Docker image local và thư mục tạm
```

Pipeline sử dụng Helm với --atomic, --wait và kiểm tra Deployment bằng kubectl rollout status. Job chỉ thành công khi Helm deploy và Kubernetes rollout hoàn tất.

```bash
kubectl get pods
kubectl get svc
kubectl get ingress
kubectl rollout status
```

---

## 13. 🧰 Điểm khác nhau giữa Job auto và Job manual

| Biến             | Job auto           | Job manual                    |
| ---------------- | ------------------ | ----------------------------- |
| `BRANCH`         | Webhook truyền vào | Cố định `develop`             |
| `REPO`           | Webhook truyền vào | Lấy từ `REPO_SELECT`          |
| `REPO_URL`       | Webhook truyền vào | Tạo từ GitLab API             |
| `COMMIT`         | Webhook truyền vào | Lấy bằng `git rev-parse HEAD` |
| `COMMIT_SHORT`   | Tính ngay đầu Job  | Tính sau khi clone            |
| `COMMIT_AUTHOR`  | Webhook truyền vào | Lấy bằng `git log`            |
| `COMMIT_MESSAGE` | Webhook truyền vào | Lấy bằng `git log`            |
| Build/deploy     | Logic hiện tại     | Giữ nguyên logic hiện tại     |
