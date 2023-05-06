# Giới thiệu

bài lab hướng dẫn truy cập vào pod thông qua command `kubectl exec`

> **Note**
>
> Học viên có thể thử nghiệm nhiều hơn dựa trên các tài liệu chính thống của Kubernetes: https://kubernetes.io/docs/tasks/debug/debug-application/get-shell-running-container/

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

#### Bước 1: tạo template yaml

Trên giao diện terminal, thực hiện nhập command sau để tạo pod template:

```bash
$ k run ubuntu --image ubuntu:20.04 --dry-run=client -o yaml > templates/ubuntu.yaml
```

Kiểm tra thông tin trên pod template vừa được tạo và tiến hành phân tích nội dung:

```bash
$ cat templates/ubuntu.yaml
```

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
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

#### Bước 2: tạo pod ubuntu

Trên terminal, thực hiện command:

```bash
$ k create -f templates/ubuntu.yaml
```

Sau khi hoàn thành tạo Pod, học viên thực hiện thao tác `get pod` và sẽ sớm nhận thấy rằng pod ubuntu sẽ được đưa về trạng thái `CrashLoopbackOff`

```bash
$ k get pod ubuntu
NAME     READY   STATUS              RESTARTS   AGE
ubuntu   0/1     ContainerCreating   0          15s

$ k get pod ubuntu
NAME     READY   STATUS             RESTARTS      AGE
ubuntu   0/1     CrashLoopBackOff   1 (11s ago)   29s
```

Nguyên nhân mà pod ubuntu không thể khởi động là do không có tiến trình nào duy trì trạng thái running trong pod ubuntu, để giải quyết vấn đề này, hãy thêm cấu hình `command: ["sleep", "1000"]` nhằm giữ pod trong trạng thái sleep 1000 giây

```bash
$ vim templates/ubuntu.yaml
```

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
    # thêm dòng cấu hình command:
    command: ["sleep", "1000"]
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Sau khi hoàn thành thay đổi, thực hiện thao tác tạo lại pod ubuntu và đảm bảo pod sẽ ở trạng thái `Running`:

```bash
$ k delete pod ubuntu
pod "ubuntu" deleted

$ k create -f templates/ubuntu.yaml
pod/ubuntu created

$ k get pods ubuntu
NAME     READY   STATUS    RESTARTS   AGE
ubuntu   1/1     Running   0          4s
```

#### Bước 3: thực hiện exection vào container bên trong pod ubuntu

Để thực hiện một command bên trong container ubuntu, học viên sử dụng câu lệnh:

```bash
$ k exec ubuntu -- <ubuntu-command>
```

Ví dụ, để xem nội dung file `/etc/passwd` bên trong container ubuntu:

```bash
$ k exec ubuntu -- cat /etc/passwd
```
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
```

Để thực hiện truy cập trực tiếp vào bên trong container ubuntu, học viên thực hiện command sau (tương tự như `docker exec`)

```bash
$ k exec -ti ubuntu -- bash
root@ubuntu:/#
```

Sau đó, học viên có thể thực hiện các thao tác bên trong container ubuntu. Ví dụ: đọc file `/etc/passwd`

```bash
root@ubuntu:/# cat /etc/passwd
```
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
```

# Clean up

Sau khi hoàn thành bài lab, học viên thực hiện xóa các tài nguyên

```bash
$ k delete -f templates/
```
