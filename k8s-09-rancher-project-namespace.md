# ☸️ Bước 09 - Quản lý Project, Namespace và thành viên trong Rancher

## 1. 🎯 Mục đích

File này chuẩn hóa Project, Namespace và membership trong Rancher để cách quản trị nhất quán giữa các môi trường.

Rancher bổ sung thêm một lớp quản lý là **Project**.

Quan hệ giữa các thành phần:

```text
Cluster
│
├── Project
│   ├── Namespace
│   ├── Namespace
│   └── Namespace
│
├── Project
│   ├── Namespace
│   └── Namespace
│
└── System Project
    ├── cattle-system
    ├── kube-system
    └── ...
```

Project được sử dụng để:

- Gom nhiều Namespace thành một nhóm.
- Phân quyền cho người dùng theo từng nhóm Namespace.
- Quản lý tài nguyên theo từng hệ thống.

---

## 2. ✅ Checklist nội dung

- [ ] Namespace hiện có
- [ ] Phân loại Namespace
- [ ] Tạo Project
- [ ] Di chuyển Namespace vào Project
- [ ] Thêm người dùng vào Project
- [ ] Ý nghĩa các Role
- [ ] Lưu ý khi sử dụng Local User
- [ ] Khuyến nghị
- [ ] Kiểm tra

---

## 3. 📌 Namespace hiện có

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig và máy quản trị Rancher tùy lệnh/UI.

Kiểm tra Namespace:

```bash
kubectl get ns
```

Ví dụ:

```text
cattle-capi-system
cattle-fleet-clusters-system
cattle-fleet-local-system
cattle-fleet-system
cattle-global-data
cattle-impersonation-system
cattle-local-user-passwords
cattle-system
cattle-turtles-system
cattle-ui-plugin-system

fleet-default
fleet-local

kube-system
kube-public
kube-node-lease

ingress-nginx
metallb-system

default

cicd

sevago-develop

local
m-c2ws7
p-jx92p
p-m2z9c
user-q5jlv
```

---

## 4. 📌 Phân loại Namespace

### 4.1. 📌 Namespace hệ thống Kubernetes

```text
default
kube-system
kube-public
kube-node-lease
```

> Không chỉnh sửa hoặc xóa.

### 4.2. 📌 Namespace hệ thống Rancher

```text
cattle-*
fleet-*
local
m-*
p-*
user-*
```

Các Namespace này được Rancher tạo để quản lý nội bộ.

> Không chỉnh sửa hoặc xóa.

### 4.3. 🏗️ Namespace hạ tầng

```text
ingress-nginx
metallb-system
cicd
```

Đây là các Namespace dành cho hạ tầng Kubernetes.

Chỉ DevOps quản lý.

### 4.4. 📌 Namespace ứng dụng

```text
sevago-develop
```

Đây là các Namespace triển khai ứng dụng.

Khuyến nghị đưa vào Project để dễ quản lý và phân quyền.

---

## 5. ➕ Tạo Project

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Truy cập:

```text
Cluster Explorer → Projects / Namespaces → Create Project
```

Tạo các Project:

| Project | Namespace        |
| ------- | ---------------- |
| Sevago  | `sevago-develop` |

Ví dụ:

```text
Cluster local
│
├── Project Sevago
└── System
```

> Không thêm Member khi tạo Project.

---

## 6. 📌 Di chuyển Namespace vào Project

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Sau khi tạo Project, truy cập:

```text
Cluster Explorer → Projects / Namespaces
```

Chọn Namespace cần di chuyển.

Nhấn:

```text
Move
```

Chọn Project tương ứng.

Ví dụ:

| Namespace        | Project |
| ---------------- | ------- |
| `sevago-develop` | Sevago  |

Sau khi hoàn thành:

```text
Cluster local
│
├── Project Sevago
│   └── sevago-develop
│
└── System
```

---

## 7. 👤 Thêm người dùng vào Project

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Sau khi Project đã được tạo, truy cập:

```text
Cluster Explorer → Projects / Namespaces → Chọn Project → ⋮ → Edit Config → Members
```

Nhấn:

```text
Add
```

Chọn: User, Project Role.

Ví dụ:

| User    | Project | Role           |
| ------- | ------- | -------------- |
| Anh Hải | Sevago  | Project Owner  |
| Anh Nam | Sevago  | Project Member |
| QA      | Sevago  | Read Only      |

Lưu cấu hình.

---

## 8. 👤 Ý nghĩa các Role

### 8.1. 📌 Project Owner

Có toàn quyền trong Project.

Có thể:

- Quản lý Namespace.
- Deploy ứng dụng.
- Restart Workload.
- Scale Workload.
- Xem Log.
- Exec Pod.
- Quản lý thành viên của Project.

### 8.2. 📌 Project Member

Có thể thao tác tài nguyên trong Project.

Ví dụ: Deploy, Restart, Scale, Xem Log, Exec Pod.

Không được thay đổi cấu hình Project hoặc quản lý thành viên.

### 8.3. 📌 Read Only

Chỉ có quyền xem.

Có thể: Xem Pod, Xem Deployment, Xem Service, Xem Log.

Không được: Deploy, Restart, Scale, Delete.

---

## 9. 📌 Lưu ý khi sử dụng Local User

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Hiện tại Rancher đang sử dụng **Local User**.

Khi đó, Rancher **không hỗ trợ phân quyền theo Group**.

Mỗi User phải được thêm thủ công vào từng Project.

Ví dụ:

```text
Project Sevago
├── Anh Hải
├── Anh Nam
└── QA
```

Nếu có thêm người dùng mới, cần vào đúng Project và thực hiện:

```text
Edit Config → Members → Add
```

để cấp quyền.

---

## 10. 💡 Khuyến nghị

Đối với hệ thống sử dụng Local User:

- Chỉ tạo Project khi cần tách quyền giữa các nhóm phát triển.
- Không nên tạo quá nhiều Project nếu số lượng người dùng lớn.

Khi triển khai hệ thống xác thực tập trung như: LDAP, Active Directory, Keycloak, Azure AD.

Nên chuyển sang phân quyền theo Group.

Ví dụ:

```text
Keycloak
│
├── Group DevOps
└── Group Sevago
```

Mapping trong Rancher:

```text
Group DevOps
    ↓
Cluster Owner

Group Sevago
    ↓
Project Member (Sevago)
```

Khi có nhân viên mới, chỉ cần thêm người dùng vào Group tương ứng trong hệ thống xác thực, Rancher sẽ tự áp dụng quyền.

---

## 11. 🔎 Kiểm tra

> **📍 Thực hiện tại:** Control-plane-1/máy có kubeconfig và máy quản trị Rancher tùy lệnh/UI.

Đăng nhập bằng từng tài khoản.

Kiểm tra:

```text
✓ Chỉ nhìn thấy Project được cấp quyền.

✓ Chỉ nhìn thấy Namespace của Project đó.

✓ Không nhìn thấy Project khác.

✓ Không truy cập được Namespace hệ thống.

✓ Thực hiện đúng các thao tác theo Role được cấp.
```
