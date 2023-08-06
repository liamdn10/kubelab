# Giới thiệu

bài lab hướng dẫn lập lịch workload dựa trên `nodeSelector` nhằm giúp deployment lựa chọn đúng các node thỏa mãn để thực hiện thao tác tạo pod trên các node đó

> **Note**
>
> - Toàn bộ các nội dung về lập lịch cho Kubernetes workload được mô tả tại tài liệu chính thống: [Kubernetes doc: Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
> - Học viên có thể chủ động mở rộng bài lab bằng cách sử dụng Pod và StatefulSet,...

# Mục tiêu bài lab

# Chuẩn bị

Để hoàn thành bài lab, người học cần chuẩn bị:

- Cài đặt kubectl trên client
- Cluster Kubernetes đang hoạt động và người học có khả năng kết nối tới cluster này
- Có thể sử dụng các công cụ như minikube, docker-desktop kubernetes, KinD,... trên môi trường lab
- thực hiện cấu hình alias trên terminal bằng command "alias k=kubectl"

# Các bước thực hiện

#### Bước 0: (optional) tạo cluster với KinD

> **Note** 
> 
> Học viên sử dụng công cụ KinD có thể tái sử dụng kubernetes cluster hoặc tạo cluster mới dựa trên file cấu hình `kind.conf` trong cùng thư mục:

```bash
$ kind create cluster --name <cluster-name> --config kind.conf
```

Trước khi thực hiện bài lab, hãy đảm bảo các worker node đang ở trạng thái `Ready`

```bash
$ k get nodes
NAME                STATUS   ROLES           AGE   VERSION
lab-control-plane   Ready    control-plane   60s   v1.26.3
lab-worker          Ready    <none>          31s   v1.26.3
lab-worker2         Ready    <none>          44s   v1.26.3
lab-worker3         Ready    <none>          31s   v1.26.3
```

#### Bước 1: tạo deployment - chưa sử dụng nodeSelector

Trên giao diện terminal, thực hiện nhập command sau để tạo deployment template:

```bash
$ k create deploy whoami --image containous/whoami --replicas=10 --dry-run=client -o yaml > templates/whoami-deploy.yaml
```

Kiểm tra thông tin template mới được tạo

```bash
$ cat templates/whoami-deploy.yaml
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
  replicas: 10
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

Học viên nhận thấy rằng trong template trên hoàn toàn không có cấu hình nào liên quan tới việc lựa chọn các Pod whoami sẽ được chạy ở đâu. Như vậy, theo lý thuyết, các Pod whoami nằm trong deployment có khả năng được triển khai trên toàn bộ các worker node.

Để kiểm nghiệm, học viên thực hiện tạo deployment:

```bash
$ k create -f template/whoami-deploy.yaml
```

Sau đó kiểm tra thông tin các pod đã được tạo và kubernetes node mà pod đó đang chạy và nhận thấy rằng các Pod được khởi tạo trên các worker node bất kì:

```bash
$ k get pod whoami-deploy -o wide
$ k get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
whoami-85dd4d7ccb-478hc   1/1     Running   0          21s   10.244.3.2   lab-worker    <none>           <none>
whoami-85dd4d7ccb-4g2vd   1/1     Running   0          21s   10.244.1.2   lab-worker2   <none>           <none>
whoami-85dd4d7ccb-7dmkw   1/1     Running   0          21s   10.244.3.4   lab-worker    <none>           <none>
whoami-85dd4d7ccb-8mg5b   1/1     Running   0          21s   10.244.2.2   lab-worker3   <none>           <none>
whoami-85dd4d7ccb-fvdhc   1/1     Running   0          21s   10.244.2.3   lab-worker3   <none>           <none>
whoami-85dd4d7ccb-j9t5k   1/1     Running   0          21s   10.244.1.3   lab-worker2   <none>           <none>
whoami-85dd4d7ccb-lnnv6   1/1     Running   0          21s   10.244.1.4   lab-worker2   <none>           <none>
whoami-85dd4d7ccb-vxjrx   1/1     Running   0          21s   10.244.3.5   lab-worker    <none>           <none>
whoami-85dd4d7ccb-wm75k   1/1     Running   0          21s   10.244.2.4   lab-worker3   <none>           <none>
whoami-85dd4d7ccb-x7s4r   1/1     Running   0          21s   10.244.3.3   lab-worker    <none>           <none>
```

#### Bước 2: sử dụng nodeSelector nhằm xác định các node mà Pod sẽ được khởi tạo trên đó

Lựa chọn một node trong danh sách các node trong cluster và thực hiện gắn label `lab=node-selector`

```bash
$ k label nodes lab-worker lab=node-selector
```

Để lấy ra thông tin các Node kèm theo các label đã được gán cho Node, học viên thực hiện command:

```bash
$ k get nodes --show-labels
NAME                STATUS   ROLES           AGE   VERSION   LABELS
lab-control-plane   Ready    control-plane   23m   v1.26.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=lab-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
lab-worker          Ready    <none>          23m   v1.26.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=lab-worker,kubernetes.io/os=linux,lab=node-selector
lab-worker2         Ready    <none>          23m   v1.26.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=lab-worker2,kubernetes.io/os=linux
lab-worker3         Ready    <none>          23m   v1.26.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=lab-worker3,kubernetes.io/os=linux$ k get nodes
```

Học viên thêm `--selector lab=node-selector` vào câu lệnh để lấy ra danh sách node có chứa label `lab=node-selector`

```bash
$ k get nodes --show-labels --selector lab=node-selector
NAME         STATUS   ROLES    AGE   VERSION   LABELS
lab-worker   Ready    <none>   23m   v1.26.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=lab-worker,kubernetes.io/os=linux,lab=node-selector
```

Sau đó tiến hành cập nhật `template/whoami-deploy.yaml` và thêm phần cấu hình `.spec.template.spec.nodeSelector` như dưới đây

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: whoami
  name: whoami
spec:
  replicas: 10
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
      # thêm thông tin nodeSelector tại phần .spec.template.spec
      nodeSelector:
        lab: node-selector
status: {}
```

Quan sát sự thay đổi của các Pod và đưa ra đánh giá

```bash
$ k get pods -o wide
NAME                      READY   STATUS              RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
whoami-6b4995b4d6-27fbk   1/1     Running             0          14s   10.244.3.11   lab-worker    <none>           <none>
whoami-6b4995b4d6-4kwkf   1/1     Running             0          18s   10.244.3.6    lab-worker    <none>           <none>
whoami-6b4995b4d6-6g7tr   0/1     ContainerCreating   0          7s    <none>        lab-worker    <none>           <none>
whoami-6b4995b4d6-7ck77   1/1     Running             0          18s   10.244.3.9    lab-worker    <none>           <none>
whoami-6b4995b4d6-fpqnv   1/1     Running             0          18s   10.244.3.10   lab-worker    <none>           <none>
whoami-6b4995b4d6-fqmsv   1/1     Running             0          18s   10.244.3.8    lab-worker    <none>           <none>
whoami-6b4995b4d6-m2v8t   1/1     Running             0          18s   10.244.3.7    lab-worker    <none>           <none>
whoami-6b4995b4d6-mdprk   0/1     ContainerCreating   0          9s    <none>        lab-worker    <none>           <none>
whoami-6b4995b4d6-mhxvj   1/1     Running             0          12s   10.244.3.12   lab-worker    <none>           <none>
whoami-6b4995b4d6-t78tc   0/1     ContainerCreating   0          5s    <none>        lab-worker    <none>           <none>
whoami-85dd4d7ccb-4g2vd   1/1     Running             0          26m   10.244.1.2    lab-worker2   <none>           <none>
whoami-85dd4d7ccb-8mg5b   1/1     Terminating         0          26m   10.244.2.2    lab-worker3   <none>           <none>
```

Kết quả cuối cùng sau một thời gian chờ, toàn bộ pod đã được chuyển về worker node lab-worker dựa trên cấu hình nodeSelector

```bash
$ k get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
whoami-6b4995b4d6-27fbk   1/1     Running   0          92s   10.244.3.11   lab-worker   <none>           <none>
whoami-6b4995b4d6-4kwkf   1/1     Running   0          96s   10.244.3.6    lab-worker   <none>           <none>
whoami-6b4995b4d6-6g7tr   1/1     Running   0          85s   10.244.3.14   lab-worker   <none>           <none>
whoami-6b4995b4d6-7ck77   1/1     Running   0          96s   10.244.3.9    lab-worker   <none>           <none>
whoami-6b4995b4d6-fpqnv   1/1     Running   0          96s   10.244.3.10   lab-worker   <none>           <none>
whoami-6b4995b4d6-fqmsv   1/1     Running   0          96s   10.244.3.8    lab-worker   <none>           <none>
whoami-6b4995b4d6-m2v8t   1/1     Running   0          96s   10.244.3.7    lab-worker   <none>           <none>
whoami-6b4995b4d6-mdprk   1/1     Running   0          87s   10.244.3.13   lab-worker   <none>           <none>
whoami-6b4995b4d6-mhxvj   1/1     Running   0          90s   10.244.3.12   lab-worker   <none>           <none>
whoami-6b4995b4d6-t78tc   1/1     Running   0          83s   10.244.3.15   lab-worker   <none>           <none>
```

### Bước 3: Thử nghiệm nodeSelector với label chưa được cấu hình

Học viên thực hiện tạo một deployment `whoami-2` với nội dung phần `.spec.template.spec.nodeSelector` chỉ định vào một label chưa được cấu hình

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
spec:
  ...
  template:
    ...
    spec:
      ...
      nodeSelector:
        lab-3: node-selector-3
status: {}
```

Sau đó tạo deployment và đánh giá kết quả thu được

### Bước 4: Thử nghiệm nodeSelector với nhiều label

Thực hiện gắn label `lab-3=node-selector-3` vào một node và `lab4=node-selector-4` vào một node khác. Sau đó, học viên tạo deployment template với thông tin `.spec.template.spec.nodeSelector` chứa cả 2 label như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
spec:
  ...
  template:
    ...
    spec:
      ...
      nodeSelector:
        lab-3: node-selector-3
        lab-4: node-selector-4
status: {}
```

Thực hiện tạo deployment dựa trên template này và đánh giá kết quả thu được. 

> **Note**
>
> Để trực quan hơn, học viên có thể thực hiện thao tác scale deployment để số lượng Pod được tạo ra nhiều hơn

# Clean up

Sau khi hoàn thành bài lab, học viên thực hiện xóa các tài nguyên

```bash
$ k delete -f templates/
```
