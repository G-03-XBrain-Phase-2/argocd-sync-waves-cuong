# Hướng dẫn thực hành W9 — Observability & Canary Delivery

Tài liệu này tóm tắt nội dung lý thuyết cốt lõi và các bước thực hiện 4 bài Lab thực hành cùng với thử thách (Challenge) từ slide gốc: [W9-chieu-obs-canary.html](file:///Users/enma/Downloads/Coding/Cloud_Engineer/Unitled/samples/gitops-cicd/lab-02-argocd-sync-waves/W9-chieu-obs-canary.html).

---

## 📌 Lý thuyết cốt lõi cần nhớ

1. **3 Trụ cột Observability:**
   * **Metrics (Chỉ số):** Trả lời câu hỏi *"Có vấn đề gì không?"* (Ví dụ: tỉ lệ lỗi, p95 latency). Thường nhẹ, lưu trữ lâu. Bài thực hành tập trung vào metrics thông qua Prometheus.
   * **Logs (Nhật ký):** Trả lời câu hỏi *"Lỗi cụ thể là gì?"*. Lưu vết chi tiết từng sự kiện riêng biệt nhưng rất nặng.
   * **Traces (Vết yêu cầu):** Trả lời câu hỏi *"Chậm/Lỗi ở đâu/bước nào?"*. Đo hành trình xuyên suốt các dịch vụ thông qua `trace_id`.

2. **Phân biệt SLA · SLO · SLI:**
   * **SLA (Service Level Agreement):** Cam kết chất lượng với khách hàng, đi kèm điều khoản phạt tài chính nếu vi phạm.
   * **SLO (Service Level Objective):** Mục tiêu chất lượng nội bộ (thường khắt khe hơn SLA).
   * **SLI (Service Level Indicator):** Số liệu thực tế đo đạc được để đối chiếu với SLO.

3. **Error Budget & Burn Rate:**
   * **Error Budget:** Ngân sách được phép lỗi (Ví dụ: SLO 99.5% $\rightarrow$ được phép lỗi 0.5%, tương đương ~216 phút/tháng).
   * **Burn Rate:** Tốc độ tiêu thụ ngân sách lỗi. Burn rate càng cao $\rightarrow$ ngân sách cạn càng nhanh $\rightarrow$ cảnh báo (Alert) càng phải gấp.
   * **Multi-window Burn Rate Alert:** Kỹ thuật cảnh báo thông minh sử dụng đồng thời 2 cửa sổ (ngắn - để bắt sự cố nghiêm trọng ngay lập tức; dài - để phát hiện rò rỉ lỗi âm ỉ).

---

## 🛠️ Chi tiết các bài Lab thực hành

### 1. Lab 1: Cài đặt hạ tầng qua GitOps (Prometheus + Argo Rollouts)
Cài đặt hạ tầng tự động thông qua cơ chế App-of-Apps của ArgoCD:
* Tạo file ứng dụng Helm cho Prometheus: `argocd/apps/kube-prometheus-stack.yaml`
* Tạo file ứng dụng Helm cho Argo Rollouts: `argocd/apps/argo-rollouts.yaml`
* Thực hiện `git push` để ArgoCD tự đồng bộ.
* **Kiểm tra kết quả:** Các pod ở namespace `monitoring` (Prometheus, Grafana) và namespace `argo-rollouts` phải ở trạng thái `Running`.
  > [!TIP]
  > Khởi động minikube với cấu hình đủ tài nguyên để gánh hệ thống giám sát: 
  > `minikube start --cpus=4 --memory=6g`

---

### 2. Lab 2: Viết ứng dụng API Flask có export `/metrics`
* Tạo thư mục `app/` chứa file `app.py` sử dụng thư viện `prometheus_flask-exporter` nhằm tự động phơi bày metrics ở đường dẫn `/metrics`.
* App sử dụng 2 biến môi trường:
  * `ERROR_RATE`: Tỉ lệ giả lập lỗi 500 khi nhận request.
  * `VERSION`: Đánh dấu version (`v1`, `v2`,...).
* Tạo `app/Dockerfile` để đóng gói ứng dụng.
* Build image và nạp trực tiếp vào cụm Minikube:
  ```bash
  docker build -t w9-api:1 app/
  minikube image load w9-api:1
  ```

---

### 3. Lab 3: Triển khai Manifest dạng Rollout & Xem Metric trên Prometheus
* Triển khai ứng dụng API ở namespace `demo` thông qua tài nguyên `Rollout` thay vì `Deployment` thông thường.
* Cấu hình chiến lược Canary trong `Rollout`:
  ```yaml
  strategy:
    canary:
      steps:
        - setWeight: 25
        - pause: {} # Chờ phê duyệt thủ công ở Lab 4
        - setWeight: 50
        - pause: { duration: 30s }
        - setWeight: 100
  ```
* Tạo `k8s-api/servicemonitor.yaml` để Prometheus bắt đầu thu thập metrics `/metrics` từ Service của API.
* Push mã nguồn lên Git và kiểm tra Prometheus targets xem trạng thái của target `api` đã `UP` chưa. Chạy script tạo traffic ảo để kiểm tra sự thay đổi của metric `flask_http_request_total`.

---

### 4. Lab 4: Chạy Canary duyệt thủ công (Manual Canary)
* Thay đổi `VERSION` từ `v1` thành `v2` trong file manifest `k8s-api/api.yaml` rồi thực hiện `git push`.
* Theo dõi tiến trình cập nhật:
  ```bash
  kubectl argo rollouts get rollout api -n demo --watch
  ```
* Hệ thống sẽ dừng lại ở mức 25% traffic cho bản `v2` do cấu hình `pause: {}`.
* Bạn tự ra quyết định dựa trên chất lượng phần mềm bằng cách chạy:
  * **Nếu ổn (Promote):** `kubectl argo rollouts promote api -n demo` để tăng tiếp traffic.
  * **Nếu lỗi (Abort):** `kubectl argo rollouts abort api -n demo` để rollback tức thì về bản cũ `v1`.

---

## 🏆 Bài tập lớn (Challenge: "Ship Smartly")

Yêu cầu thực hiện trong vòng **24 giờ** để hoàn thiện một pipeline release ứng dụng an toàn, tự phục hồi hoàn toàn thông qua GitOps và Observability:

1. **Vận hành qua Git:** Mọi thay đổi cấu hình phải được sync qua ArgoCD. Đảm bảo có thể thực hiện `git revert` và rollback thành công trong vòng **dưới 5 phút**.
2. **Đo lường & Cảnh báo:** Định nghĩa tối thiểu **1 SLO** cùng **1 Alert** kích hoạt khi chất lượng ứng dụng bị giảm sút. Alert này phải được cấu hình để gửi thông báo trực tiếp về **email cá nhân** của bạn.
3. **Canary tự động hóa (Auto-abort):**
   * Loại bỏ bước duyệt thủ công (`pause: {}`). Thay thế bằng việc gắn `AnalysisTemplate` vào quy trình Canary của Rollout.
   * `AnalysisTemplate` sẽ thực hiện truy vấn Prometheus định kỳ (ví dụ: mỗi 30 giây) để kiểm tra chất lượng của bản v2 (ví dụ: tỉ lệ request thành công $\ge 95\%$).
   * **Bản v2 tốt:** Hệ thống tự động nâng dần traffic lên 100% không cần can thiệp.
   * **Bản v2 bị lỗi:** (Ví dụ khi nâng cấp lên v2 với cấu hình giả lập lỗi `ERROR_RATE = 0.1`): `AnalysisTemplate` phát hiện tỉ lệ thành công không đạt tiêu chuẩn liên tiếp $\rightarrow$ Rollout tự động kích hoạt **abort** và rollback toàn bộ traffic về phiên bản cũ `v1` trong vòng **dưới 3 phút** mà không cần con người thao tác.
