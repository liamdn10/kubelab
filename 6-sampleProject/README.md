# Giới thiệu

bài lab tổng hợp với thao tác build và deploy một ưng dụng Java Maven lên môi trường Kubernetes

project demo được lấy tại github: https://github.com/liamhubian/techmaster-obo-web

# Mục tiêu bài lab

# Chuẩn bị

Để hoàn thành bài lab, người học cần chuẩn bị:

- Cài đặt kubectl trên client
- Cluster Kubernetes đang hoạt động và người học có khả năng kết nối tới cluster này
- Có thể sử dụng các công cụ như minikube, docker-desktop kubernetes, KinD,... trên môi trường lab
- thực hiện cấu hình alias trên terminal bằng command "alias k=kubectl"

# Các bước thực hiện

### Bước 0: (optional) chuẩn bị Cluster

> **Note**
>
> - Học viên chuẩn bị môi trường Kubernetes cho bài lab
> - Với học viên sử dụng KinD, hãy tận dụng file kind.conf được chuẩn bị sẵn cho bài lab

### Bước 1: triển khai cơ sở dữ liệu MySQL

### Bước 2: cài đặt ứng dụng và đóng gói dưới dạng container

### Bước 3: triển khai ứng dụng trên môi trường Kubernetes

### Bước 4: truy cập tới ứng dụng

# Clean up

Sau khi hoàn thành bài lab, học viên thực hiện xóa các tài nguyên

```bash
k delete -f template/
```
> **Note**
>
> Học viên tạo portforward tới WSL2 thực hiện xóa proxy rule
> ```command
> netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0
> ```