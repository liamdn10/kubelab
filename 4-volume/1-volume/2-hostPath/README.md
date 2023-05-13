# Giới thiệu

Bài lab tạo hostPath kubernetes volume

- Toàn bộ nội dung về hostPath được mô tả tại tài liệu chính thống của Kubernetes: [Kubernetes doc: Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)
- Học viên có thể mở rộng bằng cách so sánh hostPath trong bài lab này và hostPath khi triển khai persistent volume

# Mục tiêu bài lab

Bài lab giúp học viên nắm được mô hình hoạt động cơ bản của hostPath volume

Sau bài lab, học viên cần nắm được các nội dung:
- Hiểu được cơ chế hoạt động cơ bản của hostPath volume
- Có khả năng tạo được hostPath volume nhằm tạo ra một phân vùng lưu trữ dữ liệu gắn với host filesystem

# Chuẩn bị

Để hoàn thành bài lab, người học cần chuẩn bị:

- Cài đặt kubectl trên client
- Cluster Kubernetes đang hoạt động và người học có khả năng kết nối tới cluster này
- Có thể sử dụng các công cụ như minikube, docker-desktop kubernetes, KinD,... trên môi trường lab
- thực hiện cấu hình alias trên terminal bằng command "alias k=kubectl"

# Các bước thực hiện

### Bước 1: thiết lập mã nguồn cho static website

trên giao diện terminal, thực hiện truy cập tới Kubernetes worker node

> - với minikube, thực hiện ssh vào Node dựa trên hướng dẫn https://minikube.sigs.k8s.io/docs/commands/ssh/:
> ```bash
> $ minikube ssh
> ```

> - với KinD, thực hiện exec vào container chạy workerNode
> ```bash
> # lấy danh sách các các node đang chạy (KinD)
> $ docker ps | grep kind
> d27a10098291   kindest/node:v1.26.3   "/usr/local/bin/entr…"   10 minutes ago   Up 10 minutes                               host-path-worker2
> 42b63998e058   kindest/node:v1.26.3   "/usr/local/bin/entr…"   10 minutes ago   Up 10 minutes                               host-path-worker
> 42a1f1be2287   kindest/node:v1.26.3   "/usr/local/bin/entr…"   10 minutes ago   Up 10 minutes   127.0.0.1:40209->6443/tcp   host-path-control-plane
> 
> # thực hiện exec tới 1 trong các worker node trên Kubernetes
> $ docker exec -ti host-path-worker bash
> root@host-path-worker:/#
> ```

Sau khi thành công truy cập kubernetes worker node, thực hiện tạo một thư mục và cài đặt static web vào thư mục đó:

> **Note**
>
> - Học viên cần cài đặt wget và unzip nếu worker node chưa có
> - Hướng dẫn cài đặt wget có thể tham khảo tại đường dẫn: https://www.tecmint.com/install-wget-in-linux/
> - Hướng dẫn cài đặt unzip có thể tham khảo tại đường dẫn: https://www.tecmint.com/install-zip-and-unzip-in-linux/
> - Học viên có thể lấy template web tĩnh bất kì theo sở thích tại trang web https://www.free-css.com/ 

```bash
$ mkdir hostPathLab
$ cd hostPathLab
$ wget https://www.free-css.com/assets/files/free-css-templates/download/page291/dozecafe.zip
$ unzip dozecafe.zip
```

sau khi hoàn thành các bước trên, kiểm tra thưc mục output của trang web tĩnh và lưu lại đường dẫn của trang web này.

Với ví dụ mẫu dưới đây, đường dẫn thư mục chứa mã nguồn của trang web tĩnh cần lưu lại là `/hostPathLab/html`, học viên cũng cần lưu lại thông tin về tên của node đã thực hiện thao tác này

```bash
$ ls -l
total 7688
-rw-r--r-- 1 root root 7867702 Aug 19  2021 dozecafe.zip
drwxr-xr-x 5 root root    4096 May 13 00:52 html

$ pwd
/hostPathLab
```

Sau khi hoàn thành các hướng dẫn của bước 1, thoát khỏi workerNode

### Bước 2: tạo pod cho static website

Trên giao diện terminal, nhập command sau để tạo deployment template chạy ứng dụng web tĩnh

```bash
$ k run static-web1 --image nginx --dry-run=client -o yaml > templates/static-web1.yaml
```

Kiểm tra nội dung template đã được generate và thêm một số cấu hình như sau:

```bash
$ cat templates/static-web1.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: static-web1
  name: static-web1
spec:
  containers:
  - image: nginx
    name: static-web1
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

### Bước 3: Thực hiện thay đổi templates/static-web1.yaml và thêm các cấu hình volume

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

Thực hiện tạo pod static-web1:

```bash
$ k create -f templates/static-web1.yaml
pod/static-web1 created
```

Kiểm tra trạng thái của pod và xác nhận `STATUS=Running` của pod:

```bash
$ k get pod static-web1
NAME          READY   STATUS    RESTARTS   AGE
static-web1   1/1     Running   0          55s
```

### Bước 3: truy cập tới trang web tĩnh 

Trên terminal, học viên thực hiện thao tác tạo thiết lập port-forward cho phép truy cập tới deployment từ bên ngoài thông qua cổng 8080

```bash
$ k port-forward --address 0.0.0.0 pod/static-web1 8080:80
```

Sau đó, học viên thực hiện truy cập tới địa chỉ http://\<client-ip-address>:8080 thông qua command `curl` hoặc qua giao diện web

![image](/4-volume/1-emptyDir//images/Screenshot_42.png)

> **Note**
>
> Với học viên sử dụng WSL, WSL2 mà không truy cập được trang web, có thể học viên cần thực hiện thêm các thao tác mapping port từ WSL ra localhost
> 
> lấy thông tin địa chỉ IP của WSL trên file `/etc/resolv.conf`
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

### Bước 4 (optional)

Học viên thực hiện tạo static-web2 bằng cách copy template của static-web1

```bash
$ cp templates/static-web1.yaml templates/static-web2.yaml
```

Sau đó thực hiện sửa thông tin `nodeName` thành tên của một node khác mà trên node đó chưa được thực hiện các thao tác cài đặt trang web ở bước 1

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: static-web2
  name: static-web2
spec:
  volumes:
  # thêm cấu hình hostPath volume, trong đó, học viên chú ý thay đổi phần hostPath.path thành địa chỉ thư mục đã lưu lại sau bước 1
  - name: static-web-src
    hostPath:
      path: /hostPathLab/html
  containers:
  - image: nginx
    name: static-web2
    resources: {}
    # thêm cấu hình mount volume /usr/share/nginx/html vào volume static-web-src
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: static-web-src
  # thêm thông tin về nodeName nhằm đảm bảo pod sẽ chạy lên không nằm ở node đã được cấu hình hostPath ở bước 1
  nodeName: host-path-worker1
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Sau đó tạo pod static-web2, thiết lập port-forward như bước 3 và đánh giá kết quả

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