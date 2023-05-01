# Giới thiệu

bài lab hướng dẫn tạo một workload đơn giản với pod

# Mục tiêu bài lab

# Chuẩn bị

Để hoàn thành bài lab, người học cần chuẩn bị:

- Cài đặt kubectl trên client
- Cluster Kubernetes đang hoạt động và người học có khả năng kết nối tới cluster này
- Có thể sử dụng các công cụ như minikube, docker-desktop kubernetes, KinD,... trên môi trường lab
- thực hiện cấu hình alias trên terminal bằng command "alias k=kubectl"

# Các bước thực hiện

#### Bước 0: (optional) tạo cluster với KinD

Chạy command dưới đây để tạo kubernetes cluster với công cụ KinD. Học viên bỏ qua bước này nếu đã có cluster sử dụng engine khác

```bash
$ kind create cluster --name create-simple-pod --config kind.conf                                                                                                       Creating cluster "create-simple-pod" ...
 ✓ Ensuring node image (kindest/node:v1.26.3) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-create-simple-pod"
You can now use your cluster with:

kubectl cluster-info --context kind-create-simple-pod

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

Sau đó, thực hiện kết nối tới cluster:

```bash
k config use-context kind-create-simple-pod
```

#### Bước 1: tạo template yaml

Trên giao diện terminal, thực hiện nhập command sau để tạo pod template:

```bash
$ k run nginx --image nginx --dry-run=client -o yaml > pod.yaml
```

Kiểm tra thông tin trên pod template vừa được tạo và tiến hành phân tích nội dung:

```bash
$ cat pod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

#### Bước 2: tạo pod

Trên terminal, thực hiện command:

```bash
$ k create -f pod.yaml
```

Khi pod được tạo thành công, thông báo sẽ được thể hiện trên terminal

```bash
$ k create -f pod.yaml
pod/nginx created
```

#### Bước 3: xác nhận pod đã chạy

Để kiểm tra danh sách pod đang chạy (STATUS=Running), nhập command:

```bash
$ k get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          2m34s
```

Trong trường hợp có quá nhiều pod đang chạy, học viên có thể thêm filter trên command nhằm lấy ra thông tin của một trường cụ thể trong pod (VD: metadata.name - tương ứng trong file pod.yaml):

```bash
$ k get pods --field-selector metadata.name=nginx
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          5m40s
```

Học viên cũng có thể đưa ra các output dựa trên định dạng mà mình mong muốn:

- Với output chứa nhiều hơn các thông tin cơ bản của pod:

```bash
$ k get pod nginx -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE                       NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          9m    10.244.1.3   create-simple-pod-worker   <none>           <none>
```

- Với output dưới định dạng yaml
```bash
$ k get pod nginx -o yaml
```

# Clean up

Sau khi hoàn thành bài lab, học viên thực hiện xóa các tài nguyên

```bash
$ k delete -f pod.nginx
```
