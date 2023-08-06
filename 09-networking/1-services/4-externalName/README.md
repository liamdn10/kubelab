# Giới thiệu

Bài lab tạo service externalName

# Mục tiêu bài lab

Bài lab giúp học viên nắm được mô hình hoạt động của externalName service

Sau bài lab, học viên cần nắm được các nội dung:
- Hiểu được cơ chế hoạt động cơ bản của externalName service
- Có khả năng tạo được externalName service phục vụ truy cập tới ứng dụng nằm trên các namespace khác nhau

# Chuẩn bị

Để hoàn thành bài lab, người học cần chuẩn bị:

- Cài đặt kubectl trên client
- Cluster Kubernetes đang hoạt động và người học có khả năng kết nối tới cluster này
- Có thể sử dụng các công cụ như minikube, docker-desktop kubernetes, KinD,... trên môi trường lab
- thực hiện cấu hình alias trên terminal bằng command "alias k=kubectl"

# Các bước thực hiện

### Bước 1: Tạo namespace demo-01 và demo-02

Nhập các command dưới đây để tạo namespace demo-01 và namespace demo-02

```bash
$ k create namespace demo-01
$ k create namespace demo-02
```

### Bước 1: tạo deployment với image whoami trên namespace demo-01

Trên giao diện terminal, nhập command sau để tạo deployment template với image whoami:

```bash
$ k -n demo-01 create deployment whoami --image containous/whoami --dry-run=client -o yaml > template/whoami-deploy.yaml
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
  namespace: demo-01
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
$ k -n demo-01 get deployment whoami
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
whoami   1/1     1            1           33s
```

Học viên cũng có thể sử dụng command tắt để kiểm tra:

```bash
$ k -n demo-01 get deploy whoami
```

### Bước 2: tạo clusterIp service trên namespace demo-02

Trên giao diện terminal, nhập command sau để tạo clusterIp service whoami::

```bash
$ k -n demo-01 create service clusterip whoami --tcp=80:80 --dry-run=client -o yaml > template/whoami-svc.yaml
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
  namespace: demo-01
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
$ k -n demo-01 get service whoami
NAME     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
whoami   ClusterIP   10.96.222.244   <none>        80/TCP    3s
```

> Học viên có thể sử dụng command tắt để kiểm tra:

```bash
$ k -n demo-01 get svc whoami
```

### Bước 4: tạo externalService trên namespace demo-02

Trên giao diện terminal, nhập command sau để tạo template externalName service trên namespace demo-02

```bash
$ k -n demo-02 create service externalname whoami --external-name whoami.demo-01.svc.cluster.local --tcp=80:80 --dry-run=client -o yaml > template/external-whoami-svc.yaml
```

Trong đó, `whoami.demo-01.svc.cluster.local` là dns được trỏ tới service `whoami` nằm trên namespace `demo-01` thuộc cùng cluster k8s

Kiểm tra nội dung template

```bash
$ cat template/external-whoami-svc.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: whoami
  name: whoami
  namespace: demo-02
spec:
  externalName: whoami.demo-01.svc.cluster.local
  ports:
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: whoami
  type: ExternalName
status:
  loadBalancer: {}
```

Thực hiện thao tác tạo externalName service:

```bash
$ k create -f template/external-whoami-svc.yaml
```

Kiểm tra trạng thái của service sau khi tạo:

```bash
$ k -n demo-02 get service whoami
NAME     TYPE           CLUSTER-IP   EXTERNAL-IP                        PORT(S)   AGE
whoami   ExternalName   <none>       whoami.demo-01.svc.cluster.local   80/TCP    2m51s
```

### Bước 4: thực hiện truy cập tới ứng dụng whoami thông qua service


Tạo template yaml cho Pod Ubuntu:22.04

```bash
$ k -n demo-02 run ubuntu --image ubuntu:20.04 --dry-run=client -o yaml > template/ubuntu.yaml
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
  namespace: demo-02
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
$ k -n demo-02 get pod ubuntu
NAME     READY   STATUS    RESTARTS   AGE
ubuntu   1/1     Running   0          14s
```

Sau khi xác nhận Pod ubuntu đã READY, thực hiện truy cập tới Pod

```bash
$ k -n demo-02 exec -ti ubuntu -- bash
root@ubuntu:/# 
```

(Optional) thực hiện cài đặt công cụ curl trên container Ubuntu

```bash
root@ubuntu:/# apt update
root@ubuntu:/# apt install curl -y
```

Tại giao diện terminal (đã hiện thị `root@ubuntu:/#`), thực hiện kiểm tra kết nối tới service whoami và đánh giá kết quả

```bash
root@ubuntu:/# curl whoami
root@ubuntu:/# curl whoami.demo-01
```

# Clean up

Sau khi hoàn thành bài lab, học viên thực hiện xóa các tài nguyên

```bash
k delete -f template/
```
