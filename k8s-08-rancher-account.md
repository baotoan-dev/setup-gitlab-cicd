# ☸️ Bước 08 - Tạo người dùng và phân quyền Rancher

## 1. 🎯 Mục đích

File này chuẩn hóa cách tạo local user và phân quyền Rancher để tránh dùng tài khoản admin hằng ngày.

Tài liệu này hướng dẫn tạo người dùng và phân quyền Rancher.

---

## 2. ✅ Checklist nội dung

- [ ] Credentials
- [ ] Global Permissions
- [ ] Built-in
- [ ] Khuyến nghị cấu hình

---

## 3. 🔐 Credentials

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Nhập thông tin tài khoản:

| Trường           | Mô tả                            |
| ---------------- | -------------------------------- |
| Username         | Tên đăng nhập                    |
| Display Name     | Tên hiển thị                     |
| Description      | Mô tả tài khoản (không bắt buộc) |
| New Password     | Mật khẩu                         |
| Confirm Password | Nhập lại mật khẩu                |

### 3.1. 📌 Yêu cầu đổi mật khẩu trong lần đăng nhập tiếp theo

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Tùy chọn:

```text
Ask user to change their password on next login
```

Yêu cầu người dùng đổi mật khẩu ở lần đăng nhập đầu tiên.

Khuyến nghị:

```text
Nên chọn
```

### 3.2. ➕ Tạo mật khẩu ngẫu nhiên

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Tùy chọn:

```text
Generate a random password
```

Rancher tự sinh mật khẩu ngẫu nhiên.

Khuyến nghị:

```text
Không chọn
```

---

## 4. 📌 Global Permissions

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Global Permissions quyết định quyền của tài khoản trên toàn bộ hệ thống Rancher.

> Chỉ nên chọn **một** quyền.

### 4.1. 📌 Administrator

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Có toàn quyền trên Rancher.

Có thể:

- Quản lý toàn bộ Cluster.
- Quản lý toàn bộ Project.
- Quản lý người dùng.
- Quản lý Authentication.
- Quản lý Settings.
- Quản lý Role.
- Quản lý Driver.
- Quản lý Feature Flags.

Khuyến nghị:

```text
Chỉ cấp cho DevOps Administrator.
```

### 4.2. 📌 Standard User

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Quyền người dùng thông thường.

Có thể:

- Đăng nhập Rancher.
- Truy cập các Cluster hoặc Project được cấp quyền.
- Tạo Cluster mới theo cấu hình mặc định của Rancher.

Không có quyền quản trị Rancher.

Khuyến nghị:

```text
Sử dụng cho DevOps, Tech Lead.
```

### 4.3. 📌 User-Base

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Chỉ có quyền đăng nhập.

Không có quyền quản trị.

Không được truy cập Cluster hoặc Project nếu chưa được cấp quyền.

Khuyến nghị:

```text
Sử dụng cho Developer, Tester, Support.
```

---

## 5. 📌 Built-in

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Built-in là các quyền quản trị bổ sung.

> Nếu đã chọn **Administrator** thì không cần chọn thêm bất kỳ quyền Built-in nào.

### 5.1. 📌 Configure Authentication

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Cho phép cấu hình phương thức đăng nhập.

Ví dụ: LDAP, OIDC, Active Directory, SAML.

Khuyến nghị:

```text
Không chọn
```

### 5.2. 📌 Create New Cluster Drivers

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Cho phép tạo Cluster Driver mới.

Khuyến nghị:

```text
Không chọn
```

### 5.3. 👤 Manage Roles

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Cho phép tạo, sửa và xóa Role Template.

Khuyến nghị:

```text
Không chọn
```

### 5.4. 📌 View Rancher Metrics

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Cho phép xem Metrics của Rancher thông qua API.

Khuyến nghị:

```text
Không chọn
```

### 5.5. 📌 Configure Feature Flags

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Cho phép bật hoặc tắt các tính năng thử nghiệm của Rancher.

Khuyến nghị:

```text
Không chọn
```

### 5.6. 📌 Create New Clusters

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Cho phép tạo Kubernetes Cluster mới.

Khuyến nghị:

```text
Không chọn
```

Nếu hạ tầng chỉ có DevOps triển khai Cluster thì không nên cấp quyền này.

### 5.7. 📌 Manage Settings

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Cho phép thay đổi cấu hình chung của Rancher.

Ví dụ: Server URL, Agent Settings, Catalog, Branding.

Khuyến nghị:

```text
Không chọn
```

### 5.8. 📌 Configure Node Drivers

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Cho phép cấu hình Node Driver.

Khuyến nghị:

```text
Không chọn
```

### 5.9. 📌 Manage Rancher Proxy

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Cho phép cấu hình Rancher Proxy.

Khuyến nghị:

```text
Không chọn
```

### 5.10. 📌 Manage Users

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

Cho phép: Tạo User, Xóa User, Đổi mật khẩu User, Khóa User.

Khuyến nghị:

```text
Không chọn
```

---

## 6. ⚙️ Khuyến nghị cấu hình

> **📍 Thực hiện tại:** Máy quản trị dùng trình duyệt đăng nhập Rancher.

| Đối tượng             | Global Permission | Built-in       |
| --------------------- | ----------------- | -------------- |
| Rancher Administrator | Administrator     | Không cần chọn |
| DevOps                | Standard User     | Không chọn     |
| Tech Lead             | Standard User     | Không chọn     |
| Developer             | User-Base         | Không chọn     |
| Tester                | User-Base         | Không chọn     |

Sau khi hoàn tất, nhấn:

```text
Create
```

Tài khoản sẽ được tạo thành công.

> **Lưu ý:** Global Permissions và Built-in chỉ quyết định quyền quản trị Rancher. Quyền thao tác với tài nguyên Kubernetes như Pod, Deployment, Service và Namespace sẽ được cấp ở bước phân quyền **Cluster** hoặc **Project**.
