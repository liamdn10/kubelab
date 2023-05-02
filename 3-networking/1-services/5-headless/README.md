# Giới thiệu

Bài lab tạo service headless

# Mục tiêu bài lab

Bài lab giúp học viên nắm được mô hình hoạt động của headless service và so sánh với clusterIp service

Sau bài lab, học viên cần nắm được các nội dung:
- Hiểu được cơ chế hoạt động cơ bản của headless service
- Có khả năng tạo được headless service phục vụ truy cập trực tiếp tới IP của pod

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
```

```yaml
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
whoami   1/1     1            1           10s
```

Học viên cũng có thể sử dụng command tắt để kiểm tra:

```bash
$ k get deploy whoami
```

### Bước 2: tạo clusterIp service để kết nối tới whoami

Trên giao diện terminal, nhập command sau để tạo clusterIp service whoami:

```bash
$ k create service clusterip whoami --tcp=80:80 --dry-run=client -o yaml > template/whoami-svc.yaml
```

Kiểm tra service template đã được tạo:

> **Note**
> Học viên lưu ý kiểm tra phần spec.selector của service phải đúng với ít nhất một label được khai báo trong phần spec.selector.mathLabels của deployment.
> Nếu thông tin trên không đúng, hãy sửa lại

```bash
$ cat template/whoami-svc.yaml
```

```yaml
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
$ k get svc whoami
NAME     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
whoami   ClusterIP   10.96.84.165   <none>        80/TCP    5s
```

> Học viên có thể sử dụng command tắt để kiểm tra:

```bash
$ k get svc whoami
```

### Bước 3: tạo clusterIp service để kết nối tới whoami

Do kubectl không support tạo headless service từ terminal, vì vậy học viên cần tạo clusterIp service template trước, sau đó sửa template thành template của headless service

```bash
$ k create service -h
Create a service using a specified subcommand.

Aliases:
service, svc

Available Commands:
  clusterip      Create a ClusterIP service
  externalname   Create an ExternalName service
  loadbalancer   Create a LoadBalancer service
  nodeport       Create a NodePort service

Usage:
  kubectl create service [flags] [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).
```

Trên giao diện terminal, nhập command sau để tạo clusterIp service whoami-headless:

```bash
$ k create service clusterip whoami-headless --tcp=80:80 --dry-run=client -o yaml > template/whoami-headless-svc.yaml
```

Để tạo được headless service, thực hiện thay đổi thông tin cấu hình trên file `template/whoami-headless-svc.yaml` và thêm dòng cấu hình `.spec.clusterIp: None` và cập nhật đúng thông tin `.spec.selector.app: whoami`

```bash
$ vim template/whoami-headless-svc.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: whoami-headless
  name: whoami-headless
spec:
  # thêm dòng cấu hình `clusterIp: None`
  clusterIP: None
  ports:
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    # cập nhật `app: whoami-headless` thành `app:whoami`
    # app: whoami-headless
    app: whoami
  type: ClusterIP
status:
  loadBalancer: {}
```


Kiểm tra service template đã được tạo:

> **Note**
> Học viên lưu ý kiểm tra phần spec.selector của service phải đúng với ít nhất một label được khai báo trong phần spec.selector.mathLabels của deployment.
> Nếu thông tin trên không đúng, hãy sửa lại


Tạo service:

```bash
$ k create -f template/whoami-svc.yaml
service/whoami created
```

Kiểm tra trạng thái của service sau khi tạo và so sánh thông tin hiển thị giữa service "whoami-headless" và "whoami":

```bash
$ k get service whoami-headless
NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
whoami-headless   ClusterIP   None         <none>        80/TCP    27s
```

> Học viên có thể sử dụng command tắt để kiểm tra:

```bash
$ k get svc whoami-headless
```


### Bước 4: thực hiện truy cập tới ứng dụng whoami thông qua service


Tạo template yaml cho Pod Ubuntu:22.04

```bash
$ k run ubuntu --image ubuntu:20.04 --dry-run=client -o yaml > template/ubuntu.yaml
```

Thực hiện cập nhật nội dung file yaml `template/ubuntu.yaml`

```yaml
$ cat template/ubuntu.yaml
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
    # thêm dòng `command: ["sleep", "1000"]` phía dưới
    command: ["sleep", "1000"]
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Thực hiện tạo Pod ubuntu:

```bash
$ k create -f template/ubuntu.yaml
```

Kiểm tra Pod ubuntu, chờ cho Pod hoàn tất khởi động và chuyển sang trạng thái RUNNING

```bash
$ k get pod ubuntu
NAME     READY   STATUS    RESTARTS   AGE
ubuntu   1/1     Running   0          14s
```

Sau khi xác nhận Pod ubuntu đã READY, thực hiện truy cập tới Pod

```bash
$ k exec -ti ubuntu -- bash
root@ubuntu:/# 
```

(Optional) thực hiện cài đặt công cụ curl trên container Ubuntu

```bash
root@ubuntu:/# apt update
root@ubuntu:/# apt install dnsutils iputils-ping -y
```

Tại giao diện terminal (đã hiện thị `root@ubuntu:/#`), thực hiện thao tác ping tới service whoami và whoami-headless, sau đó đánh giá kết quả

```bash
root@ubuntu:/# ping whoami
root@ubuntu:/# ping whoami-headless
```

Tiếp tục thực hiện kiểm tra với `nslookup`

```bash
root@ubuntu:/# nslookup whoami
root@ubuntu:/# nslookup whoami-headless
```

Học viên có thể thực hiện scale demployment whoami lên 3 và thử lại kết quả nslookup. Sau đó đánh giá về kết quả thu về, địa chỉ IP nhận được là địa chỉ của thành phần nào?

# Clean up

Sau khi hoàn thành bài lab, học viên thực hiện xóa các tài nguyên

```bash
k delete -f template/
```
