# Giới thiệu

Bài lab tạo emptyDir kubernetes volume

# Mục tiêu bài lab

Bài lab giúp học viên nắm được mô hình hoạt động cơ bản của emptyDir volume

Sau bài lab, học viên cần nắm được các nội dung:
- Hiểu được cơ chế hoạt động cơ bản của emptyDir volume
- Có khả năng tạo được emptyDir volume nhằm tạo ra một phân vùng lưu trữ dữ liệu tạm thời, có khả năng chia sẻ giữa các container

# Chuẩn bị

Để hoàn thành bài lab, người học cần chuẩn bị:

- Cài đặt kubectl trên client
- Cluster Kubernetes đang hoạt động và người học có khả năng kết nối tới cluster này
- Có thể sử dụng các công cụ như minikube, docker-desktop kubernetes, KinD,... trên môi trường lab
- thực hiện cấu hình alias trên terminal bằng command "alias k=kubectl"

# Các bước thực hiện

### Bước 1: tạo deployment cho static website

Trên giao diện terminal, nhập command sau để tạo deployment template chạy ứng dụng web tĩnh

```bash
$ k create deploy static-web --image busybox --image nginx --dry-run=client -o yaml > templates/static-web.yaml
```

Kiểm tra nội dung template đã được generate và thêm một số cấu hình như sau:

```bash
$ cat templates/static-web.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: static-web
  name: static-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: static-web
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: static-web
    spec:
      containers:
      - image: busybox
        name: busybox
        resources: {}
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

### Bước 2: Thực hiện thay đổi templates/static-web.yaml và thêm các cấu hình volume

Dựa trên file template/static-web.yaml đã tạo, thực hiện các thay đổi như mẫu dưới đây:

> **Note**
>
> Học viên có thể lựa chọn trang web tĩnh theo ý mình trên trang web "https://www.free-css.com/"

```yaml
$ cat templates/static-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: static-web
  name: static-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: static-web
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: static-web
    spec:
      # thêm cấu hình volumes static-web
      volumes:
      - name: static-web
        emptyDir: {}
      # tạo initContainers và chuyển cấu hình của busybox tới đây
      initContainers:
      - image: busybox
        name: busybox
        resources: {}
        # thêm cấu hình mount volume /data/html trên container vào volume static-web
        volumeMounts:
        - mountPath: /data/html
          name: static-web
        # thêm cấu hình command nhằm thực hiện thao tác download file web tĩnh về thư mục /data/html
        command:
        - "ash"
        - "-c"
        - "wget https://www.free-css.com/assets/files/free-css-templates/download/page291/dozecafe.zip && unzip dozecafe.zip -d dozecafe && cp -r dozecafe/html/* /data/html/"
      containers:
      - image: nginx
        name: nginx
        resources: {}
        # thêm cấu hình mount volume /usr/share/nginx/html vào volume static-web
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: static-web
status: {}
```

Thực hiện tạo deployment static-web:

```bash
$ k create -f template/static-web.yaml
deployment.apps/static-web created
```

Kiểm tra trạng thái của deployment và xác nhận các pod trong deployment đã ở trạng thái READY 1/1:

```bash
$ k get deploy static-web
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
static-web   0/1     1            0           8s

$ k get deploy static-web
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
static-web   1/1     1            1           30s
```

### Bước 3: truy cập tới trang web tĩnh 

Trên terminal, học viên thực hiện thao tác tạo thiết lập port-forward cho phép truy cập tới deployment từ bên ngoài thông qua cổng 8080

```bash
$ k port-forward --address 0.0.0.0 deployment/static-web 8080:80
```

Sau đó, học viên thực hiện truy cập tới địa chỉ http://<client-ip-address>:8080 thông qua command `curl` hoặc qua giao diện web

> **Note**
>
> Với học viên sử dụng WSL, WSL2, thực hiện lấy thông tin địa chỉ IP của WSL trên file `/etc/resolv.conf`
> ```bash
> $ cat /etc/resolv.conf
> # This file was automatically generated by WSL. To stop automatic generation of this file, add the following entry to /etc/wsl.conf:
> # [network]
> # generateResolvConf = false
> nameserver 172.30.128.1
> ```
> 
> Sau đó thực hiện mở command-prom của windows với quyền admin và tạo port-forwarding với thông tin `connectaddress=<nameserver>`
> ```command
> netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=8080 connectaddress=172.30.128.1
> ```
> sau khi thực thi command đúng, việc mapping sẽ được thể hiện thông qua command trên command-prom:
> ```command
> netsh interface portproxy show all
>
> Listen on ipv4:             Connect to ipv4:
>
> Address         Port        Address         Port
> --------------- ----------  --------------- ----------
> 0.0.0.0         8080        172.30.128.1    8080
> ```
> Tham khảo tại: https://learn.microsoft.com/en-us/windows/wsl/networking

![image](/4-volume/1-emptyDir//images/Screenshot_42.png)

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