# SỰ CỐ ALERTMANAGER KHÔNG GỬI ĐƯỢC EMAIL CẢNH BÁO & CÁCH KHẮC PHỤC

Tài liệu này ghi lại chi tiết các lỗi dẫn đến việc Alertmanager không thể gửi email cảnh báo về hộp thư cá nhân và quy trình khắc phục sự cố.

---

## 1. Các lỗi đã gặp phải (Root Causes)

Trong quá trình vận hành luồng Canary tự động và tích hợp cảnh báo SLO, hệ thống đã gặp hai vấn đề lớn liên quan đến cấu hình của Alertmanager:

### Lỗi 1: Validate cấu hình thất bại trên Prometheus Operator (`undefined receiver "null"`)
* **Nguyên nhân:** Helm Chart của `kube-prometheus-stack` sinh ra các route định tuyến mặc định (như định tuyến cảnh báo `Watchdog` tới một receiver có tên `"null"` để ẩn các cảnh báo hệ thống không cần thiết). Khi ghi đè cấu hình trong file `argocd/apps/kube-prometheus-stack.yaml` tại mục `receivers`, chúng ta chỉ khai báo duy nhất receiver có tên `'email'`. 
* **Hậu quả:** Bộ điều phối Prometheus Operator khi kiểm tra tính hợp lệ của cấu hình đã phát hiện ra route `Watchdog` trỏ đến receiver `"null"` không tồn tại trong danh sách định nghĩa. Nó báo lỗi:
  ```text
  sync "monitoring/kube-prometheus-stack-alertmanager" failed: provision alertmanager configuration: failed to initialize from secret: undefined receiver "null" used in route
  ```
  Do đó, Prometheus Operator từ chối áp dụng và nạp cấu hình mới. Các thông số cấu hình SMTP và email cá nhân đã bị bỏ qua, dẫn đến việc Alertmanager tiếp tục chạy cấu hình cũ không có SMTP.

### Lỗi 2: Sai lệch địa chỉ gửi thư (`smtp_from` không hợp lệ)
* **Nguyên nhân:** Địa chỉ gửi thư cấu hình ban đầu là `smtp_from: 'alertmanager@example.com'`, trong khi tài khoản xác thực gửi đi qua máy chủ Gmail SMTP của Google là `asmrcreator312@gmail.com`.
* **Hậu quả:** Khi cố gắng gửi email từ tên miền không được xác thực (`example.com`) thông qua máy chủ SMTP của Google với tài khoản Gmail cá nhân, hệ thống lọc thư rác của Google sẽ đánh giá đây là giả mạo (spoofing) và từ chối chuyển tiếp thư hoặc lọc thẳng thư vào hòm thư Spam (thư rác).

---

## 2. Các bước xử lý cụ thể (Solutions)

### Bước 1: Khai báo bổ sung receiver `null` và khớp địa chỉ SMTP gửi đi
Chỉnh sửa file cấu hình GitOps [kube-prometheus-stack.yaml](file:///Users/enma/Downloads/Coding/Cloud_Engineer/Unitled/samples/gitops-cicd/lab-02-argocd-sync-waves/argocd/apps/kube-prometheus-stack.yaml):
1. Thêm receiver có tên `"null"` vào cuối danh sách `receivers`.
2. Đổi giá trị `smtp_from` thành `asmrcreator312@gmail.com` trùng khớp với `smtp_auth_username`.

*Chi tiết cấu hình sau khi sửa đổi:*
```yaml
alertmanager:
  config:
    global:
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'asmrcreator312@gmail.com'
      smtp_auth_username: 'asmrcreator312@gmail.com'
      smtp_auth_password: 'YOUR_APP_PASSWORD'
      smtp_require_tls: true
    route:
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'email'
    receivers:
    - name: 'email'
      email_configs:
      - to: 'asmrcreator312@gmail.com'
        send_resolved: true
    - name: 'null'
```

### Bước 2: Commit thay đổi lên Git & Sync lại ArgoCD
Thực hiện commit cấu hình và push lên nhánh chính của GitHub để ArgoCD quét và nạp trạng thái mong muốn mới:
```bash
git add argocd/apps/kube-prometheus-stack.yaml
git commit -m "fix: Add null receiver and fix smtp_from in Alertmanager config"
git push origin main
```
Sau đó, tiến hành Hard Refresh trên ArgoCD để ép buộc các ứng dụng `root` và `kube-prometheus-stack` đồng bộ hóa cấu hình tức thì từ Git:
```bash
kubectl patch app root -n argocd -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}' --type=merge
kubectl patch app kube-prometheus-stack -n argocd -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}' --type=merge
```

### Bước 3: Khởi động lại Alertmanager StatefulSet để áp dụng cấu hình ngay lập tức
Do Kubelet có độ trễ nhất định khi cập nhật các Secret được mount dạng volume vào Pod (thường mất từ 1-2 phút), thực hiện restart StatefulSet của Alertmanager để ép nó nạp lại Secret mới nhất lập tức:
```bash
kubectl rollout restart statefulset/alertmanager-kube-prometheus-stack-alertmanager -n monitoring
```

---

## 3. Kết quả xác minh
* Lệnh kiểm tra cấu hình thực tế trong container Alertmanager cho thấy cấu hình SMTP và email gửi đi/đến cùng danh sách receiver đã được nạp thành công:
  `kubectl exec -n monitoring alertmanager-kube-prometheus-stack-alertmanager-0 -c alertmanager -- cat /etc/alertmanager/config_out/alertmanager.env.yaml`
* Khi kích hoạt lại Canary có chứa lỗi (`ERROR_RATE: "1"`), Prometheus phát hiện tỉ lệ thành công giảm sâu dưới 95% và kích hoạt cảnh báo `APISuccessRateLow`.
* Cảnh báo được gửi thành công sang Alertmanager, Alertmanager định tuyến và gửi email thông báo từ hòm thư `asmrcreator312@gmail.com` về hòm thư cá nhân của học viên thành công (với trạng thái gửi thành công hiển thị qua metric `alertmanager_notifications_total{integration="email"}`).
