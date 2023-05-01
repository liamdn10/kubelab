# Giới thiệu

Bài lab tạo service clusterIp đơn giản

# Mục tiêu bài lab

Bài lab giúp học viên nắm được mô hình hoạt động của clusterIp service

Sau bài lab, học viên cần nắm được các nội dung:
- Hiểu được cơ chế hoạt động cơ bản của clusterIp service
- Có khả năng tạo được clusterIP service phục vụ truy cập tới ứng dụng

# Chuẩn bị

Để hoàn thành bài lab, người học cần chuẩn bị:

- Cài đặt kubectl trên client
- Cluster Kubernetes đang hoạt động và người học có khả năng kết nối tới cluster này
- Có thể sử dụng các công cụ như minikube, docker-desktop kubernetes, KinD,... trên môi trường lab
- thực hiện cấu hình alias trên terminal bằng command "alias k=kubectl"

# Các bước thực hiện

### Bước 1: tạo deployment với image whoami

Trên giao diện terminal, nhập command sau để tạo deployment template với image whoami:

```bash
$ k create deployment whoami --image containous/whoami --dry-run=client -o yaml > template/whoami-deploy.yaml
```

Kiểm tra nội dung template đã được generate:

```bash
$ cat template/whoami-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: whoami
  name: whoami
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: whoami
    spec:
      containers:
      - image: containous/whoami
        name: whoami
        resources: {}
status: {}
```

Thực hiện tạo deployment whoami:

```bash
$ k create -f template/whoami-deploy.yaml
deployment.apps/whoami created
```

Kiểm tra trạng thái của deployment và xác nhận các pod trong deployment đã ở trạng thái READY 1/1:

```bash
$ k get deployment whoami
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
whoami   1/1     1            1           18s
```

Học viên cũng có thể sử dụng command tắt để kiểm tra:

```bash
$ k get deploy whoami
```

### Bước 2: tạo service whoami

Trên giao diện terminal, nhập command sau để tạo clusterIp service whoami::

```bash
k create service clusterip whoami --tcp=80:80 --dry-run=client -o yaml > template/whoami-svc.yaml
```

Kiểm tra service template đã được tạo:

> **Note**
> Học viên lưu ý kiểm tra phần spec.selector của service phải đúng với ít nhất một label được khai báo trong phần spec.selector.mathLabels của deployment.
> Nếu thông tin trên không đúng, hãy sửa lại

```bash
$ cat template/whoami-svc.yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: whoami
  name: whoami
spec:
  ports:
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: whoami
  type: ClusterIP
status:
  loadBalancer: {}
```

Tạo service:

```bash
$ k create -f template/whoami-svc.yaml
service/whoami created
```

Kiểm tra trạng thái của service sau khi tạo:

```bash
$ k get service whoami
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
whoami       ClusterIP   10.96.237.43   <none>        80/TCP    50s
```

> Học viên có thể sử dụng command tắt để kiểm tra:

```bash
k get svc whoami
```

### Bước 3: Kiểm tra các thông tin trên service

Thực hiện command sau nhằm lấy ra thông tin mô tả của service whoami

```bash
$ k describe service whoami
Name:              whoami
Namespace:         default
Labels:            app=whoami
Annotations:       <none>
Selector:          app=whoami
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.237.43
IPs:               10.96.237.43
Port:              80-80  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.4:80
Session Affinity:  None
Events:            <none>
```

> Lưu lại thông tin trường "Endpoints" nhằm đối chiếu với địa chỉ ip của các pod nằm trong deployment whoami.

```bash
$ k get pods --selector app=whoami -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP           NODE                       NOMINATED NODE   READINESS GATES
whoami-85dd4d7ccb-vt6bn   1/1     Running   0          21m   10.244.1.4   create-simple-pod-worker   <none>           <none>
```

Học viên thực hiện đối chiếu danh sách địa chỉ IP của service và địa chỉ IP của các pod nằm trong deployment whoami

Sau đó, học viên tiếp tục kiểm tra thông tin "endpoint" với command sau và cho đánh giá

```bash
k describe endpoints whoami
```
### Bước 4: thực hiện truy cập tới ứng dụng whoami thông qua service


> **Note**
> Học viên thực hiện kiểm tra truy cập tới service whoami dựa trên một trong hai cách sau:
> - 4-1: sử dụng port-forwarding tới service whoami
> - 4-2: tạo pod ubuntu và truy cập tới service whoami


### Bước 4-1: sử dụng kube-proxy

Thực hiện tạo kube-proxy cho phép truy cập tới service

```bash
$ k port-forward service/whoami 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```

Thực hiện thao tác curl tới port 8080 và kiểm tra kết quả trả về, đối chiếu thông tin Hostname và IP với thông tin của các Pod nằm trong deployment whoami và đánh giá.

```bash
$ curl localhost:8080
Hostname: whoami-85dd4d7ccb-vt6bn
IP: 127.0.0.1
IP: ::1
IP: 10.244.1.4
IP: fe80::70ab:a7ff:fefa:6b57
RemoteAddr: 127.0.0.1:34904
GET / HTTP/1.1
Host: localhost:8080
User-Agent: curl/7.81.0
Accept: */*
```

### Bước 4-2: sử dụng container ubuntu

Tạo template yaml cho Pod Ubuntu:22.04

```bash
$ k run ubuntu --image ubuntu:20.04 --dry-run=client -o yaml > template/ubuntu.yaml
```

Thực hiện cập nhật nội dung file yaml `template/ubuntu.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ubuntu
  name: ubuntu
spec:
  containers:
  - image: ubuntu:20.04
    name: ubuntu
    resources: {}
    // thêm dòng command phía dưới
    command: ["sleep", "1000"]
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Kiểm tra Pod ubuntu, chờ cho Pod hoàn tất khởi động và chuyển sang trạng thái RUNNING

```bash
$ k get pods ubuntu
NAME     READY   STATUS    RESTARTS   AGE
ubuntu   1/1     Running   0          81s
```

Sau khi xác nhận Pod ubuntu đã READY, thực hiện truy cập tới Pod

```bash
$ k exec -ti ubuntu -- bash
root@ubuntu:/# 
```

(Optional) thực hiện cài đặt công cụ curl trên container Ubuntu

```bash
root@ubuntu:/# apt update
root@ubuntu:/# apt install curl -y
```

Tại giao diện terminal (đã hiện thị `root@ubuntu:/#`), thực hiện kiểm tra kết nối tới service whoami và đánh giá kết quả

```bash
root@ubuntu:/# curl whoami:80
```

Tiếp đó, thực hiện tăng số lượng replica của deployment whoami lên 3 và tiếp thực hiện lại thao tác `curl whoami:80` và đánh giá kết quả được trả về

```bash
$ k scale --replicas=3 deployment/whoami
deployment.apps/whoami scaled
```

```bash
$ k get deploy whoami
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
whoami   3/3     3            3           44m

$ k get pods --selector app=whoami
NAME                      READY   STATUS    RESTARTS   AGE
whoami-85dd4d7ccb-dgpts   1/1     Running   0          57s
whoami-85dd4d7ccb-p7xkm   1/1     Running   0          57s
whoami-85dd4d7ccb-vt6bn   1/1     Running   0          45m
```

# Clean up

Sau khi hoàn thành bài lab, học viên thực hiện xóa các tài nguyên

```bash
k delete -f template/
```
