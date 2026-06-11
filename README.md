# BÁO CÁO MINH CHỨNG THỰC HÀNH (EVIDENCE REPORT) — TUẦN 9

* **Học viên:** Trần Mạnh Cường
* **Repository:** `https://github.com/G-03-XBrain-Phase-2/argocd-sync-waves-cuong`
* **Môi trường thực hành:** Minikube (Docker driver)

---

## 📋 PHẦN 1: GITOPS, APP-OF-APPS & SYNC WAVES (BUỔI SÁNG)

### 1. Lab 3: Kiểm tra tính năng Sync & Self-heal (Tự động phục hồi)

* **Nội dung:** Thực hiện kiểm tra tính năng tự phục hồi và chống lệch cấu hình bằng cách:
  1. Chạy lệnh xóa Pod thủ công trên cụm Kubernetes để kiểm tra cơ chế tự phục hồi của ReplicaSet.
  2. Quan sát ArgoCD và cụm Kubernetes tự động khắc phục, thay thế Pod bị xóa bằng một Pod mới gần như ngay lập tức.
  3. Kiểm tra nhật ký sự kiện (Events) trên ArgoCD để xác nhận hoạt động tự phục hồi.

* **📸 MINH CHỨNG HÌNH ẢNH:**
  
  **Ảnh 1: Lệnh xóa Pod thủ công trên cluster**
  ![Lệnh xóa Pod](public/kubectldelete.png)
  *(Mô tả: Sử dụng lệnh `kubectl delete pod` để xóa Pod thuộc ứng dụng `sync-waves-app`).*

  **Ảnh 2: Pod mới được tự động tạo lập ngay lập tức**
  ![Pod tự động tạo lại](public/selfheal.png)
  *(Mô tả: Trong khi Pod cũ `database-589bc78d8d-gdcqj` đang trong trạng thái tắt (`Terminating`), cụm đã tự động khởi tạo Pod mới thay thế là `database-589bc78d8d-chht4` nhờ cơ chế self-heal).*

  **Ảnh 3: Nhật ký sự kiện ghi nhận đồng bộ thành công**
  ![Events ArgoCD](public/eventself.png)
  *(Mô tả: Giao diện ArgoCD Events ghi nhận quá trình đồng bộ và tự động kiểm tra trạng thái tài nguyên hoạt động trơn tru).*

---

### 2. Lab 4: Cơ chế Rollback chuẩn GitOps

* **Nội dung:** Thực hiện revert commit trên Git (`git revert HEAD`) và push lên repo. Quan sát ArgoCD đồng bộ và tự động rollback ứng dụng về phiên bản cũ an toàn.
* **📸 MINH CHỨNG HÌNH ẢNH:**
  ![Git Rollback History](path/to/anh_rollback_git.png)
  *\*Chú thích: Anh chụp lại màn hình lịch sử commit trên GitHub hoặc Git Log trên terminal hiển thị commit revert, cùng với giao diện ArgoCD đã Sync về đúng commit ID đó.*

---

### 3. Lab 5: Mô hình App-of-Apps

* **Nội dung:** Root Application (`root.yaml`) giám sát thư mục `argocd/apps/`. Khi thêm ứng dụng con mới (`counter-app.yaml`), Root App tự động phát hiện và sinh ra app con trên cụm.
* **📸 MINH CHỨNG HÌNH ẢNH:**
  ![App-of-Apps Tree](path/to/anh_app_of_apps.png)
  *\*Chú thích: Anh chụp màn hình giao diện chính của ArgoCD hiển thị cây ứng dụng gồm Root Application kết nối tới các ứng dụng con (`sync-waves-app`, `counter-app`,...).*

---

### 4. Lab 6: Quản lý thứ tự triển khai với Sync Waves

* **Nội dung:** Kiểm soát trình tự chạy resource: `Namespace (Wave -10) -> Database/Redis (Wave -5) -> ConfigMap (Wave -3) -> Web App/Frontend (Wave 0)`. Đã sửa lỗi readinessProbe cho DB (`pgrep sh`) để qua wave thành công.
* **📸 MINH CHỨNG HÌNH ẢNH:**
  ![Sync Waves Progress](path/to/anh_sync_waves.png)
  *\*Chú thích: Anh chụp lại giao diện ArgoCD trong quá trình đồng bộ, hiển thị rõ tiến trình các resource được tạo lần lượt theo các lớp (Wave) khác nhau.*

---

### 5. Lab 7: Thiết lập CI plan-on-PR

* **Nội dung:** Tự động validate file manifest Kubernetes và ứng dụng ArgoCD bằng Kubeconform tích hợp trong GitHub Actions khi có Pull Request.
* **📸 MINH CHỨNG HÌNH ẢNH:**
  ![GitHub Actions Validation](path/to/anh_ci_pr.png)
  *\*Chú thích: Anh chụp màn hình kết quả chạy thành công (tích xanh) của workflow `GitOps PR Validation` trên giao diện Pull Request ở GitHub.*

---

## 📊 PHẦN 2: OBSERVABILITY & CANARY DELIVERY (BUỔI CHIỀU)

### 1. Lab 1: Cài đặt hạ tầng qua GitOps (Prometheus + Argo Rollouts)

* **Nội dung:** Cài đặt bộ giám sát và điều phối Canary thông qua khai báo Git. Sửa lỗi xung đột cấu hình Default Datasource của Loki để chạy thành công.
* **📸 MINH CHỨNG HÌNH ẢNH:**
  ![Infrastructure Apps Synced](path/to/anh_infra_synced.png)
  *\*Chú thích: Anh chụp giao diện ArgoCD hiển thị hai ứng dụng `kube-prometheus-stack` và `argo-rollouts` đều ở trạng thái xanh lá (`Synced` và `Healthy`).*

  ![Infrastructure Pods Running](path/to/anh_infra_pods.png)
  *\*Chú thích: Anh chụp lại kết quả chạy lệnh `kubectl get pods -n monitoring` và `kubectl get pods -n argo-rollouts` hiển thị toàn bộ Pods của Prometheus, Grafana, Loki và Rollout controller đều ở trạng thái `Running`.*

---

### 2. Lab 2: Viết ứng dụng API Flask có export `/metrics`

* **Nội dung:** Viết code tích hợp `prometheus-flask-exporter` hỗ trợ route `/metrics` và build Docker Image `w9-api:1` nạp vào Minikube.
* **📸 MINH CHỨNG HÌNH ẢNH:**
  ![Loaded Docker Image](path/to/anh_image_load.png)
  *\*Chú thích: Anh chụp lại terminal sau khi chạy lệnh `minikube image ls -p w9 | grep w9-api` cho thấy image `w9-api:1` đã có sẵn trong cụm.*

---

### 3. Lab 3: Triển khai Manifest API dạng Rollout & Xem Metric

* **Nội dung:** Tạo traffic giả lập gửi liên tục vào API. Kiểm tra targets và biểu đồ tăng trưởng request trên Prometheus.
* **📸 MINH CHỨNG HÌNH ẢNH:**
  ![Prometheus Target UP](path/to/anh_prometheus_target.png)
  *\*Chú thích: Anh chụp giao diện Prometheus (`http://localhost:9090`) phần Status -> Targets cho thấy target `demo/api` đang hiển thị trạng thái `UP`.*

  ![Prometheus Metrics Query](path/to/anh_metrics_query.png)
  *\*Chú thích: Anh chụp giao diện Prometheus khi chạy câu truy vấn `flask_http_request_total{namespace="demo"}` hiển thị bảng/biểu đồ các chỉ số request tăng dần liên tục.*

---

### 4. Lab 4: Chạy Canary duyệt thủ công (Manual Canary)

* **Nội dung:** Đẩy phiên bản `v2` của API. Quan sát tiến trình tạm dừng ở mức 25% traffic và tiến hành dùng CLI promote lên 100% thủ công.
* **📸 MINH CHỨNG HÌNH ẢNH:**
  ![Canary Paused 25%](path/to/anh_canary_paused.png)
  *\*Chú thích: Anh chụp màn hình terminal lệnh `kubectl argo rollouts get rollout api -n demo --watch` hiển thị trạng thái `Paused` ở mức 25% traffic.*

  ![Canary Promoted 100%](path/to/anh_canary_promoted.png)
  *\*Chú thích: Anh chụp màn hình terminal sau khi chạy lệnh `promote` cho thấy Rollout đã chuyển đổi hoàn toàn và chạy đủ 100% ở bản mới `v2`.*

---

## 🏆 PHẦN 3: THỬ THÁCH LỚN (CHALLENGE: "SHIP SMARTLY")

Phần này ghi nhận kết quả hoàn thiện của hệ thống tự động hoá hoàn toàn quá trình Canary và đo lường cảnh báo lỗi (SLO Alert).

### 1. Trạng thái GitOps đồng bộ (No Drift)

* **Nội dung:** Mọi cấu hình (bao gồm `AnalysisTemplate` và SLO Rules) đều được quản lý qua Git và đồng bộ qua ArgoCD.
* **📸 MINH CHỨNG HÌNH ẢNH:**
  ![GitOps Clean Sync](path/to/anh_challenge_synced.png)
  *\*Chú thích: Anh chụp giao diện ứng dụng `api` trên ArgoCD hiển thị trạng thái Synced xanh sạch.*

---

### 2. Minh chứng Canary tự động Abort & Rollback (Quan trọng nhất)

* **Nội dung:** Nâng phiên bản mới và cấu hình `ERROR_RATE = 0.1` (tiêm lỗi). Quan sát `AnalysisTemplate` truy vấn Prometheus phát hiện chất lượng lỗi $\rightarrow$ tự động kích hoạt hủy bỏ (Abort) và rollback về bản cũ trong vòng dưới 3 phút mà không cần người bấm phím.
* **📸 MINH CHỨNG HÌNH ẢNH HOẶC CLIP:**
  ![Auto-abort CLI Watch](path/to/anh_auto_abort.png)
  *\*Chú thích: Anh chụp lại màn hình terminal CLI watch hiển thị trạng thái `Degraded` / `Aborted` kèm theo lịch sử các lần đo đạc của `AnalysisRun` bị thất bại và tự động chuyển hướng traffic về lại phiên bản cũ.*

---

### 3. Minh chứng Cảnh báo (Alert) gửi về Email cá nhân

* **Nội dung:** Kích hoạt cảnh báo đỏ dựa trên SLO và kiểm tra hòm thư cá nhân.
* **📸 MINH CHỨNG HÌNH ẢNH:**
  ![Personal Email Alert](path/to/anh_email_alert.png)
  *\*Chú thích: Anh chụp lại giao diện **Hộp thư email cá nhân** hiển thị bức thư thông báo cảnh báo lỗi từ Alertmanager của hệ thống.*

---

### 📝 4. Giải thích chi tiết cấu hình đo lường & ngưỡng (SLO & Alert)

#### a) Ngưỡng đo lường tự động của Canary (AnalysisTemplate)

* **Câu truy vấn PromQL dùng trong AnalysisTemplate:**

  ```promql
  # Điền câu query tính success rate anh dùng ở đây. Ví dụ:
  sum(rate(flask_http_request_duration_seconds_count{status!~"5..", pod=~"api-.*"}[2m])) / sum(rate(flask_http_request_duration_seconds_count{pod=~"api-.*"}[2m]))
  ```

* **Giải thích logic & Ngưỡng:**
  * *Ngưỡng đạt:* Tỉ lệ thành công (Success Rate) phải $\ge 95\%$ (`result >= 0.95`).
  * *Cơ chế đánh giá:* Đo đạc định kỳ mỗi `30s` một lần. Nếu kết quả không đạt quá `3` lần (`failureLimit: 3`), hệ thống sẽ lập tức abort và rollback.

#### b) Cấu hình SLO & Alert cảnh báo

* **Chỉ số đo lường (SLI):** ... *(ví dụ: Tỷ lệ request HTTP thành công trên tổng số request trong vòng 5 phút)*
* **Mục tiêu đặt ra (SLO):** ... *(ví dụ: 99.0% request phải thành công)*
* **Cú pháp quy tắc Alert (PrometheusRule):**

  ```yaml
  # Anh có thể dán đoạn cấu hình Alert Rule ở đây
  ```

* **Giải thích hoạt động:** Khi lỗi được inject làm chất lượng tụt dưới ngưỡng SLO $\rightarrow$ Alert rule chuyển sang trạng thái `Firing` $\rightarrow$ Alertmanager tiếp nhận và gửi mail về hòm thư cấu hình sẵn.
