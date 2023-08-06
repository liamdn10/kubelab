# Giới thiệu

Bài lab tạo ingress controller sử dụng nginx

> **Note**
> Trong trường hợp sử dụng KinD (Kubernetes in Docker), học viên lưu ý thiết lập port mapping cho docker container
> Học viên cũng có thể sử dụng file kind.conf được đính kèm trong thư mục

# Mục tiêu bài lab

Bài lab giúp học viên nắm được mô hình hoạt động của Ingress Controller thông qua nginx

Sau bài lab, học viên cần nắm được các nội dung:
- Hiểu được cơ chế hoạt động cơ bản của Ingress Controller
- Có khả năng tạo được Ingress Controller sử dụng Nginx
- Có khả năng tự mở rộng với các loại Ingress Controller khác

# Chuẩn bị

Để hoàn thành bài lab, người học cần chuẩn bị:

- Cài đặt kubectl trên client
- Cluster Kubernetes đang hoạt động và người học có khả năng kết nối tới cluster này
- Có thể sử dụng các công cụ như minikube, docker-desktop kubernetes, KinD,... trên môi trường lab
- thực hiện cấu hình alias trên terminal bằng command "alias k=kubectl"

# Các bước thực hiện

### Bước 1: 


# Clean up

Sau khi hoàn thành bài lab, học viên thực hiện xóa các tài nguyên

```bash
k delete -f template/
```
