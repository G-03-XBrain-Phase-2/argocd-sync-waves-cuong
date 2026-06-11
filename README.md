# Lab GitOps Nâng Cao: App-of-Apps, Sync Waves & CI/CD Pipeline

Dự án này là nơi lưu trữ và thực hành chuỗi các bài Lab nâng cao về GitOps sử dụng **Argo CD** (quản lý phân phối CD) phối hợp cùng **GitHub Actions** (tự động hóa kiểm tra CI).

---

## 📂 Cấu Trúc Thư Mục Dự Án

Dự án được cấu trúc theo mô hình **App-of-Apps** (Ứng dụng quản lý ứng dụng), chia nhỏ các ứng dụng độc lập vào các thư mục con riêng biệt để tăng tính mô-đun:

```text
lab-02-argocd-sync-waves/
├── .github/
│   └── workflows/
│       └── validate.yml       # [CI] Tự động validate manifest bằng Kubeconform trên PR
├── argocd/
│   ├── root.yaml              # [Root App] Quản lý tất cả ứng dụng con qua thư mục argocd/apps/
│   └── apps/
│       ├── sync-waves-app.yaml# [Child App] Ứng dụng 1: Database + Frontend Portfolio
│       └── counter-app.yaml   # [Child App] Ứng dụng 2: Visitor Counter (Flask + Redis)
├── manifests/
│   ├── sync-waves-app/        # Manifests ứng dụng 1 (Chạy ở namespace: database-frontend-lab)
│   │   ├── namespace.yaml     # Wave -10
│   │   ├── database-deployment.yaml # Wave -5 (Đã fix readinessProbe check 'pgrep sh')
│   │   ├── database-service.yaml    # Wave -5
│   │   ├── configmap.yaml     # Wave -3
│   │   ├── frontend-deployment.yaml # Wave 0
│   │   └── frontend-service.yaml    # Wave 0
│   └── counter-app/           # Manifests ứng dụng 2 (Chạy ở namespace: counter-lab)
│       ├── namespace.yaml     # Wave -10
│       ├── redis-deployment.yaml    # Wave -5
│       ├── redis-service.yaml # Wave -5
│       ├── configmap.yaml     # Wave -3 (Chứa mã nguồn Flask Python chạy động)
│       ├── counter-deployment.yaml  # Wave 0 (Chạy python:3.9-slim và cài requirements)
│       └── counter-service.yaml     # Wave 0
└── README.md                  # Hướng dẫn này
```

---

## 🛠️ Chi Tiết Các Bài Lab Đã Thực Hành

Dưới đây là tóm tắt các bài thực hành thiết yếu đã được triển khai và cấu hình thành công trong thư mục này:

### Lab 3: Kiểm tra tính năng Sync & Self-heal (Tự động phục hồi)
* **Ý nghĩa:** Đảm bảo Git luôn là nguồn sự thật duy nhất (Single Source of Truth).
* **Thực hành:** Khi cố tình sửa đổi thủ công tài nguyên trên Kubernetes (ví dụ: chạy lệnh `kubectl scale` tăng/giảm replicas của frontend), Argo CD phát hiện sự lệch cấu hình (drift) so với Git và ngay lập tức tự động đồng bộ (Self-heal) để kéo trạng thái cụm K8s về đúng cấu hình định nghĩa trên Git.

### Lab 4: Cơ chế Rollback chuẩn GitOps
* **Ý nghĩa:** Hiểu cách quay lui phiên bản an toàn trong GitOps.
* **Thực hành:** Thay vì sử dụng lệnh `kubectl rollout undo` (lệnh này sẽ bị tính năng Self-heal của Argo CD chặn lại và ghi đè ngược về phiên bản mới trên Git), chúng ta thực hiện Rollback chuẩn bằng cách tạo commit revert (`git revert HEAD`) và push lên Git. Argo CD sẽ tự động đồng bộ và rollback cụm K8s theo lịch sử Git.

### Lab 5: Mô hình App-of-Apps (1 Root quản lý nhiều con)
* **Ý nghĩa:** Tự động hóa hoàn toàn việc quản lý và phát hiện ứng dụng mới.
* **Thực hành:** Tạo một Root Application (`root.yaml`) giám sát thư mục `argocd/apps/` trên Git. Khi muốn thêm ứng dụng con mới (như `counter-app.yaml`), chỉ cần đẩy file định nghĩa vào thư mục này, Root App sẽ tự động phát hiện và triển khai mà không cần tạo tay trên giao diện Argo CD.

### Lab 6: Quản lý thứ tự triển khai với Sync Waves
* **Ý nghĩa:** Kiểm soát trình tự khởi chạy giữa các dịch vụ có tính phụ thuộc lẫn nhau.
* **Trình tự triển khai:** `Namespace (Wave -10) -> Database/Redis (Wave -5) -> ConfigMap (Wave -3) -> Web App/Frontend (Wave 0)`
* **Sửa lỗi Readiness Probe:** Khắc phục lỗi cấu hình kiểm tra của Database (đổi lệnh check từ `pgrep alpine` sang `pgrep sh`) để Kubelet báo cáo Pod Healthy, giúp Argo CD chuyển tiếp mượt mà sang Wave sau.

### Lab 7: Thiết lập CI plan-on-PR (Kiểm tra cú pháp trước khi Merge)
* **Ý nghĩa:** Tự động kiểm tra tính hợp lệ của manifest thông qua GitHub Actions và Kubeconform trên các Pull Request.
* **Cấu hình tối ưu:** Sử dụng cờ `-ignore-missing-schemas` để Kubeconform bỏ qua kiểm tra schema đối với các tài nguyên tùy biến của Argo CD (như `Application` trong thư mục `argocd/`), tránh báo lỗi đỏ giả, trong khi vẫn kiểm tra nghiêm ngặt toàn bộ manifest chuẩn ở thư mục `manifests/`.

---

## 💻 Hướng Dẫn Vận Hành & Lệnh Kiểm Tra

### 1. Áp dụng cấu hình GitOps (Nếu triển khai từ đầu)
Chỉ cần chạy lệnh tạo Root Application, Argo CD sẽ làm phần việc còn lại:
```bash
kubectl apply -f argocd/root.yaml
```

### 2. Kiểm tra danh sách Application của Argo CD
```bash
kubectl get applications -n argocd
```
*Kết quả mong muốn:*
```text
NAME             SYNC STATUS   HEALTH STATUS
root             Synced        Healthy
sync-waves-app   Synced        Healthy
counter-app      Synced        Healthy
```

### 3. Kiểm tra tài nguyên dưới K8s Cluster
Kiểm tra xem các pod và service của 2 ứng dụng đã chạy tốt và sẵn sàng chưa:

* **Ứng dụng Portfolio (sync-waves-app):**
  ```bash
  kubectl get all -n database-frontend-lab
  ```
* **Ứng dụng Đếm Lượt Truy Cập (counter-app):**
  ```bash
  kubectl get all -n counter-lab
  ```

### 4. Mở kết nối và truy cập ứng dụng trên Trình duyệt (Port-forward)
Vì chạy trên môi trường local (như Minikube/Kind), bạn cần forward cổng từ máy cá nhân vào Service của K8s để mở trình duyệt:

* **Mở trang Portfolio (Frontend):**
  ```bash
  kubectl port-forward svc/frontend-service -n database-frontend-lab 8080:80
  # Truy cập trên trình duyệt: http://localhost:8080
  ```
* **Mở trang Web Đếm Lượt Truy Cập (Visitor Counter):**
  ```bash
  kubectl port-forward svc/counter-service -n counter-lab 8081:5000
  # Truy cập trên trình duyệt: http://localhost:8081 (f5 liên tục để xem số lượng tăng lên)
  ```
