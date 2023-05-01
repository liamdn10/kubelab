# Giới thiệu

Bài lab tạo service nodePort đơn giản

> **Note**
> Trong trường hợp sử dụng KinD (Kubernetes in Docker), học viên lưu ý thiết lập port mapping cho docker container
> Học viên cũng có thể sử dụng file kind.conf được đính kèm trong thư mục

# Mục tiêu bài lab

Bài lab giúp học viên nắm được mô hình hoạt động của nodePort service

Sau bài lab, học viên cần nắm được các nội dung:
- Hiểu được cơ chế hoạt động cơ bản của nodePort service
- Có khả năng tạo được nodePort service phục vụ truy cập tới ứng dụng

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

Trên giao diện terminal, nhập command sau để tạo template nodeport service whoami:

```bash
$ k create service nodeport whoami --tcp=80:80 --dry-run=client -o yaml > template/whoami-svc.yaml
```

Kiểm tra service template đã được tạo:

> **Note**
> - Học viên lưu ý kiểm tra phần spec.selector của service phải đúng với ít nhất một label được khai báo trong phần spec.selector.mathLabels của deployment. Nếu thông tin trên không đúng, hãy sửa lại
> - Thêm thông tin `nodePort=32002` vào phần cấu hình `spec.ports`

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
    # thêm dòng này nếu sử dụng KinD
    nodePort: 32002
  selector:
    app: whoami
  type: NodePort
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
NAME     TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
whoami   NodePort   10.96.225.173   <none>        80:32002/TCP   6m37s
```

> Học viên có thể sử dụng command tắt để kiểm tra:

```bash
k get svc whoami
```

### Bước 3: Kiểm tra các thông tin trên service

Thực hiện command sau nhằm lấy ra thông tin mô tả của service whoami

```bash
$ k describe service whoami
Name:                     whoami
Namespace:                default
Labels:                   app=whoami
Annotations:              <none>
Selector:                 app=whoami
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.96.225.173
IPs:                      10.96.225.173
Port:                     80-80  80/TCP
TargetPort:               80/TCP
NodePort:                 80-80  32002/TCP
Endpoints:                10.244.1.2:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

> **Note**
> Lưu lại thông tin Nodeport (32002) nhằm phục vụ mục đích truy cập ở bước tiếp theo
> Lưu lại thông tin trường "Endpoints" nhằm đối chiếu với địa chỉ ip của các pod nằm trong deployment whoami.

```bash
$ k get pod --selector app=whoami -o wide
NAME                      READY   STATUS    RESTARTS   AGE    IP           NODE               NOMINATED NODE   READINESS GATES
whoami-85dd4d7ccb-mtzbl   1/1     Running   0          8m1s   10.244.1.2   node-port-worker   <none>           <none>
```

Học viên thực hiện đối chiếu danh sách địa chỉ IP của service và địa chỉ IP của các pod nằm trong deployment whoami

Sau đó, học viên tiếp tục kiểm tra thông tin "endpoint" với command sau và cho đánh giá

```bash
k describe endpoints whoami
```

### Bước 4: thực hiện truy cập tới ứng dụng whoami thông qua service

> **Note**
> Học viên thực hiện bước 4 theo một trong các trường hợp thực hành dưới đây:
> - 4-1: với cluster thông thường
> - 4-2: với cluster được tạo bằng KinD
> - 4-3: với cluster được tạo bằng minikube

#### 4-1: Với cluster Kubernetes thông thường

Lấy thông tin IP của các node

``` bash
$ k get nodes -o wide
NAME                      STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION                      CONTAINER-RUNTIME
node-port-control-plane   Ready    control-plane   10m     v1.26.3   172.18.0.3    <none>        Ubuntu 22.04.2 LTS   5.15.90.1-microsoft-standard-WSL2   containerd://1.6.19-46-g941215f49
node-port-worker          Ready    <none>          9m55s   v1.26.3   172.18.0.2    <none>        Ubuntu 22.04.2 LTS   5.15.90.1-microsoft-standard-WSL2   containerd://1.6.19-46-g941215f49
```

Thực hiện truy cập tới service thông qua nodeport (đã được lưu ở bước 2):

```bash
$ curl <node-ip>:<nodePort>
```

VD:

```bash
$ curl 172.18.0.2:32002
```

#### 4-2: Với KinD

thực hiện lấy thông tin port được mapping với port 32002

```bash
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS          PORTS                                                 NAMES
709d19a8025c   kindest/node:v1.26.3   "/usr/local/bin/entr…"   13 minutes ago   Up 13 minutes   0.0.0.0:32002->32002/tcp, 127.0.0.1:39169->6443/tcp   node-port-control-plane
5c00e3322225   kindest/node:v1.26.3   "/usr/local/bin/entr…"   13 minutes ago   Up 13 minutes   0.0.0.0:32003->32002/tcp                              node-port-worker
```

Thực hiện truy cập tới service

```bash
$ curl 127.0.0.1:32002
$ curl 127.0.0.1:32003
```

#### 4-3: Với Minikube

thực hiện lấy địa chỉ IP của minikube

```bash
$ minikube ip
```

Sau đó thực hiện truy cập tới service whoami thông qua node port

```bash
$ curl <minikube-ip>:<nodePort-port>
```

### Bước 5: thử nghiệm khả năng cân bằng tải (load balance) của nodePort

Thực hiện tăng số lượng replica của deployment whoami lên 3 và tiếp thực hiện lại thao tác được mô tả tại bước số 4

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
