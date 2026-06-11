FROM python:3.12-slim

# Cài đặt các thư viện cần thiết
RUN pip install --no-cache-dir flask prometheus-flask-exporter

# Thiết lập thư mục làm việc và sao chép mã nguồn
COPY app.py /app/app.py
WORKDIR /app

# Thiết lập biến môi trường cho Flask
ENV FLASK_APP=app.py
EXPOSE 8080

# Chạy ứng dụng trên cổng 8080
CMD ["flask", "run", "--host=0.0.0.0", "--port=8080"]
