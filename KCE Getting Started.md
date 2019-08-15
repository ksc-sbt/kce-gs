# 金山云容器引擎服务入门指南

本入门指南介绍如何利用金山云容器引擎服务快速搭建Kubernetes集群，并部署一个Nginx应用和Kubernetes Dashboard服务的过程。

部署架构参考如下：

![金山云容器引擎服务架构示意图](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/kce-architecture.png)

本指南包含如下内容：
* 准备网络环境
* 使用金山云容器镜像服务
* 创建Kubernetes容器引擎
* 部署Nginx应用
* 部署Kubernetes仪表盘
相关Kubernetes对象配置文件存储在Github: https://github.com/ksc-sbt/kce-gs.


# 1 准备网络环境
本指南所使用的环境位于金山云北京6区。下面是在使用金山云容器引擎服务所需要准备的网络配置信息。

## 1.1 VPC配置信息

|  网络资源   | 名称  | CIDR  |
|  ----  | ----  | ----  |
| VPC  | gs-vpc |	10.34.0.0/16 |

## 1.2 子网配置信息

| 子网名称 | 所属VPC |可用区 | 子网类型  | CIDR  | 说明|
|  ----  | ----  | ----  |----  |----  |----|
| public_a  | gs-vpc |	可用区A | 普通子网| 10.34.51.0/24|用于跳板机|
| private_a  | gs-vpc |	可用区A | 普通子网| 10.34.0.0/20|用于K8S集群主机和其他云服务器|
| endpoint  | gs-vpc |无需设置| 终端子网| 10.34.52.0/24|用于管理内网负载均衡器实例|

## 1.3 安全组配置信息
在创建K8S集群实例时，需要为集群的Node节点主机关联安全组。下面是安全组(k8s-sg)的配置规则：

|协议|行为|起始端口|结束端口|源IP|
|----|----|----|----|----|
|TCP|允许|30000|32768|0.0.0.0/0|
|ICMP|允许|全部协议|全部协议|0.0.0.0/0|
|TCP|允许|22|22|0.0.0.0/0|

## 1.4 NAT配置信息
为了保证K8S集群实例中的节点能和外网通信，需要配置NAT实例。下面是NAT实例的配置信息。

|名称|所属VPC|作用范围|类型|所绑定的子网|
|  ----  | ----  | ----  |----  |----  |
|Ksc_Nat|gs-vpc|绑定的子网|公网|private_a|


# 2 使用金山云容器镜像服务
金山云提供标准的Docker镜像服务，便于在金山云云服务器上启动容器实例时快速下载镜像。下面是使用金山云镜像服务创建私有镜像的过程。

## 2.1	重置镜像仓库登录密码
在使用金山云容器引擎的镜像仓库功能前，需要首先重置密码。该密码将在docker login命令中使用。其中用户名是当前金山云账户的账户ID。
![重置镜像服务器密码](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/reset-registry-password.png)

在完成密码重置后，可执行docker login命令完成金山云镜像仓库认证。下述命令中，XXXXXXXX替换为对应的账户ID。
```bash
docker login hub.kce.ksyun.com -u XXXXXXXX
```
该命令输出提示输入Password，在输入前面设置的镜像仓库登录口令后，显示login成功。
```text
Password: 
Login Succeeded
```
## 2.2 上传自定义镜像

通过docker push命令，可完成docker镜像上传。为了简单起见，重用docker.hub上的标准nginx镜像。首先执行docker pull命令下载标准nginx镜像。
```bash
docker pull nginx
```
命令输出为：
```text
Using default tag: latest
latest: Pulling from library/nginx
f5d23c7fed46: Pull complete 
918b255d86e5: Pull complete 
8c0120a6f561: Pull complete 
Digest: sha256:eb3320e2f9ca409b7c0aa71aea3cf7ce7d018f03a372564dbdb023646958770b
Status: Downloaded newer image for nginx:latest
```
然后执行docker images命令查看本地的镜像信息。
```bash
docker images
```
命令输出为：
```text
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              e445ab08b2be        2 weeks ago         126MB
```
为了在金山云容器镜像服务上存储私有镜像文件，金山云账户管理员需要在金山云控制台上创建镜像命令空间，比如为ksc-gs，并设置类型为私有。
![创建私有镜像空间](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/registry-namespace.png)

修改镜像tag，该tag包含金山云镜像注册服务器地址信息。
```bash
docker tag e445ab08b2be  hub.kce.ksyun.com/ksc-gs/nginx:latest
docker images
```
命令输出为：
```text
REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
nginx                            latest              e445ab08b2be        2 weeks ago         126MB
hub.kce.ksyun.com/ksc-gs/nginx   latest              e445ab08b2be        2 weeks ago         126MB
```

执行docker push命令，把镜像上传到hub.kce.ksyun.com镜像仓库。
```bash
docker push hub.kce.ksyun.com/ksc-gs/nginx:latest
```
命令输出为：
```text
The push refers to a repository [hub.kce.ksyun.com/ksc-gs/nginx]
fe6a7a3b3f27: Pushed 
d0673244f7d4: Pushed 
d8a33133e477: Pushed 
latest: digest: sha256:dc85890ba9763fe38b178b337d4ccc802874afe3c02e6c98c304f65b08af958f size: 948
```
此时访问金山云控制台，能看到镜像已成功上传。需要注意的是金山云镜像服务当前不支持docker的search命令。
![金山云容器镜像仓库镜像信息](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/registry-repository.png)
 
## 2.3	进行镜像安全扫描
金山云镜像仓库提供镜像安全扫描功能，通过选择特定的镜像版本，执行“安全扫描”操作，将获得该镜像的安全扫描结果。
![金山云容器镜像安全扫描结果](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/image-security-scan.png)
 
# 3 创建Kubernetes容器引擎实例
金山云容器引擎服务提供托管Kubernetes集群服务，用户可以通过控制台快速创建Kubernetes集群，并集成金山云负载均衡服务和云硬盘服务，并可根据需要快速增加集群节点。本节将介绍如何利用金山云容器引擎服务创建Kubernetes集群的过程。
## 3.1 配置Kubernetes集群实例网络
在金山云容器引擎管理控制台，点击"新建虚拟机集群"后的第一步就是配置集群网络。
![金山云容器集群网络配置](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/cluster-network.png)

下面是网络配置的参数说明：
* 集群网络：选择该集群所处的VPC。
* 终端子网：当集群外的云主机需要访问集群中Pod提供的服务时，需要创建一个Kubernetes Service，该Servie的Type是LoadBalancer，而且配置Service的annotations为service.beta.kubernetes.io/ksc-loadbalancer-type: "internal"。在创建Service时，将创建一个金山云内网负载均衡器。由于金山云内网负载均衡实例是创建在终端子网，因此集群的网络配置需要指定终端子网。
* Pod CIDR：配置Pod网络地址空间。
* Service CIDR: 配置Service的网络地址空间。

## 3.2 配置集群云服务器
配置集群云服务器明确了集群中Node节点配置信息。
![金山云容器集群云服务器配置](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/cluster-host.png)

具体的配置参数信息如下：
* 计费方式：可选择"按小时配置实时计费"和"按日配置计费"两种计费方式。金山云云服务器"按日配置计费"的价格约为"按小时配置实时计费"价格的一半。
* 节点网络：选择集群主机（包括Master和Node)节点所处的子网；
* 镜像：选择镜像类型，当前金山云支持CentOS和Ubuntu两种操作系统作为容器集群的节点操作系统；
* 数据盘：选择需要配置的本地数据盘或者云硬盘数据盘，数据盘将自动挂载到/data目录下；当Docker的日志文件比较大，缺省的20G系统盘空间不能满足存储需求时，需要创建更大存储容量的数据盘；
* 安全组：选择关联的安全组。该安全组的配置需求参考"1.3 安全组配置信息"。

## 3.3 配置节点
最后一步是配置容器Node节点的其他信息。
![金山云容器集群Node节点配置](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/cluster-host-key.png)

具体的配置参数信息如下：
* 所属项目：选择资源所属的项目，可选择"缺省项目"；
* 服务器名称：输入容器集群Node节点的名字；
* 登录方式：容器集群的Node节点可以通过ssh登录，可以设置登录的密码或者密钥文件。容器集群的Master节点是不允许ssh登录的。

## 3.4 验证集群创建结果
集群实例的创建过程大约5分钟，在创建完成后，能看到如下集群的详细信息。
![金山云容器集群详细信息](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/cluster-detail.png)

通过跳板机（处于同一个VPC，并关联一个弹性IP的云服务器），利用ssh密钥，可登录到集群的节点服务器(10.34.0.10)上。执行fdisk命令能查看该集群节点挂载的块设备，其中/dev/vdb就是添加的本地数据盘。
```bash
fdisk -l
```
命令输出为：
```text
Disk /dev/vda: 21.5 GB, 21474836480 bytes, 41943040 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x0009e73e

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048    41943039    20970496   83  Linux

Disk /dev/vdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
执行df -m, 能看到/dev/vdb数据盘挂载在/data目录下。
```bash
df -m
```
命令输出为：
```text
Filesystem     1M-blocks  Used Available Use% Mounted on
/dev/vda1          20030  2949     16042  16% /
devtmpfs             488     0       488   0% /dev
tmpfs                497     1       497   1% /dev/shm
tmpfs                497     7       490   2% /run
tmpfs                497     0       497   0% /sys/fs/cgroup
/dev/vdb            9952   505      8920   6% /data
```
执行docker ps命令，能看到在该服务上运行的容器实例。
```bash
docker ps
```
命令输出为：
```text
CONTAINER ID        IMAGE                                          COMMAND                  CREATED             STATUS              PORTS               NAMES
d955143d471b        hub.kce.ksyun.com/ksyun/traefik                "/traefik --api --ku…"   10 minutes ago      Up 10 minutes                           k8s_traefik-ingress-lb_traefik-ingress-controller-h695d_kube-system_018c6e60-bce4-11e9-881e-fa163e63dc05_0
a021c4d3bfcb        hub.kce.ksyun.com/ksyun/metrics-server-amd64   "/metrics-server --m…"   10 minutes ago      Up 10 minutes                           k8s_metrics-server_metrics-server-5ff7ffc54-kl8j5_kube-system_fddea69d-bce3-11e9-881e-fa163e63dc05_0
ef0dad9cae82        hub.kce.ksyun.com/ksyun/system-monitor         "/metrics-server --m…"   10 minutes ago      Up 10 minutes                           k8s_system-monitor_system-monitor-68b9bd8cbd-nlr9q_kube-system_fddd0cf5-bce3-11e9-881e-fa163e63dc05_0
addf7b5a8ba4        hub.kce.ksyun.com/ksyun/pause-amd64:3.0        "/pause"                 10 minutes ago      Up 10 minutes                           k8s_POD_traefik-ingress-controller-h695d_kube-system_018c6e60-bce4-11e9-881e-fa163e63dc05_0
6fa7865063bf        hub.kce.ksyun.com/ksyun/disk-provisioner       "/bin/disk-provision…"   10 minutes ago      Up 10 minutes                           k8s_disk-provisioner_disk-provisioner-57ddd6b8b8-58rnl_kube-system_fe83e677-bce3-11e9-881e-fa163e63dc05_0
b224f71b9086        hub.kce.ksyun.com/ksyun/pause-amd64:3.0        "/pause"                 11 minutes ago      Up 10 minutes                           k8s_POD_metrics-server-5ff7ffc54-kl8j5_kube-system_fddea69d-bce3-11e9-881e-fa163e63dc05_0
e2af972c6a8e        hub.kce.ksyun.com/ksyun/pause-amd64:3.0        "/pause"                 11 minutes ago      Up 10 minutes                           k8s_POD_disk-provisioner-57ddd6b8b8-58rnl_kube-system_fe83e677-bce3-11e9-881e-fa163e63dc05_0
f57440d1490e        hub.kce.ksyun.com/ksyun/pause-amd64:3.0        "/pause"                 11 minutes ago      Up 10 minutes                           k8s_POD_system-monitor-68b9bd8cbd-nlr9q_kube-system_fddd0cf5-bce3-11e9-881e-fa163e63dc05_0
7c5ac4106f65        hub.kce.ksyun.com/ksyun/ksc-flexvolume         "/root/entry.sh"         11 minutes ago      Up 11 minutes                           k8s_ksc-flexvolume-ds_ksc-flexvolume-ds-gq9r4_kube-system_fc7987cd-bce3-11e9-881e-fa163e63dc05_0
16ad4ec9c323        hub.kce.ksyun.com/ksyun/pause-amd64:3.0        "/pause"                 11 minutes ago      Up 11 minutes                           k8s_POD_ksc-flexvolume-ds-gq9r4_kube-system_fc7987cd-bce3-11e9-881e-fa163e63dc05_0
409af1fe8012        hub.kce.ksyun.com/ksyun/flanneld-cni           "/install-cni.sh"        11 minutes ago      Up 11 minutes                           k8s_install-cni_kube-flannel-k5cw6_kube-system_f4cbac90-bce3-11e9-881e-fa163e63dc05_0
eb2c3fcb3e14        hub.kce.ksyun.com/ksyun/flanneld               "/usr/local/bin/flan…"   11 minutes ago      Up 11 minutes                           k8s_kube-flannel_kube-flannel-k5cw6_kube-system_f4cbac90-bce3-11e9-881e-fa163e63dc05_0
bd1fbd124b64        hub.kce.ksyun.com/ksyun/pause-amd64:3.0        "/pause"                 11 minutes ago      Up 11 minutes                           k8s_POD_kube-flannel-k5cw6_kube-system_f4cbac90-bce3-11e9-881e-fa163e63dc05_0
60b37bf6f4ee        hub.kce.ksyun.com/ksyun/kube-proxy             "/usr/local/bin/kube…"   11 minutes ago      Up 11 minutes                           k8s_kube-proxy_kube-proxy-26qd4_kube-system_e2b97631-bce3-11e9-881e-fa163e63dc05_0
e942508f1bf5        hub.kce.ksyun.com/ksyun/pause-amd64:3.0        "/pause"                 11 minutes ago      Up 11 minutes                           k8s_POD_kube-proxy-26qd4_kube-system_e2b97631-bce3-11e9-881e-fa163e63dc05_0
```
## 3.5 利用kubectl客户端访问集群
通过金山云容器引擎服务创建的Kubernetes集群实例可以通过公网访问。因此，需要在能访问公网的机器上下载Kubernetes客户端kubectl。由于我们选择的Kubernetes版本是1.13.4，因此安装kubectl版本尽可能是1.13.4。

kubectl Linux版本下载地址：
https://storage.googleapis.com/kubernetes-release/release/v1.13.4/bin/linux/amd64/kubectl

kubectl MacOS版本下载地址：
https://storage.googleapis.com/kubernetes-release/release/v1.13.4/bin/darwin/amd64/kubectl

kubectl Windows版本下载地址：
https://storage.googleapis.com/kubernetes-release/release/v1.13.4/bin/windows/amd64/kubectl.exe

在安装完成后，可通过如下命令查看kubectl版本信息。
```bash
kubectl version --client
```
命令输出为：
```text
Client Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.4", GitCommit:"c27b913fddd1a6c480c229191a087698aa92f0b1", GitTreeState:"clean", BuildDate:"2019-03-01T23:34:27Z", GoVersion:"go1.12", Compiler:"gc", Platform:"darwin/amd64"}
```
通过金山云控制台，在所创建实例信息处点击"配置文件"下载集群的配置文件，并存储在客户端home目录下的.kube目录下。

![金山云容器集群配置文件](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/cluster-config.png)

下面是.kube目录下的config文件。
```bash
ls -al ~/.kube
```
命令输出为：
```text
total 16
drwxr-xr-x    3 myang  staff    96 Aug 12 18:23 .
drwxr-xr-x+ 186 myang  staff  5952 Aug 12 18:23 ..
-rw-r--r--@   1 myang  staff  5593 Aug 12 18:22 config
```
然后执行kubectl cluster-info命令，能看到当前集群的信息。
```bash
kubectl cluster-info
```
命令输出为：
```text
Kubernetes master is running at https://120.92.43.65:6443
CoreDNS is running at https://120.92.43.65:6443/api/v1/namespaces/kube-system/services/coredns:dns/proxy
Heapster is running at https://120.92.43.65:6443/api/v1/namespaces/kube-system/services/heapster/proxy
monitoring-influxdb is running at https://120.92.43.65:6443/api/v1/namespaces/kube-system/services/monitoring-influxdb/proxy
```
通过kubect get nodes获得集群的节点信息。
```bash
kubectl get nodes -o wide
```
命令输出为：
```text
NAME         STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION          CONTAINER-RUNTIME
10.34.0.10   Ready    node     57m   v1.13.4   10.34.0.10    <none>        CentOS Linux 7 (Core)   3.10.0-514.el7.x86_64   docker://18.9.2
10.34.0.11   Ready    master   57m   v1.13.4   10.34.0.11    <none>        CentOS Linux 7 (Core)   3.10.0-514.el7.x86_64   docker://18.9.2
10.34.0.16   Ready    master   57m   v1.13.4   10.34.0.16    <none>        CentOS Linux 7 (Core)   3.10.0-514.el7.x86_64   docker://18.9.2
10.34.0.6    Ready    master   57m   v1.13.4   10.34.0.6     <none>        CentOS Linux 7 (Core)   3.10.0-514.el7.x86_64   docker://18.9.2
```
在本机器运行kubectl proxy命令启动Proxy server，把在访问本机的请求转发到Kubernetes API Server。
```bash
kubectl proxy
```
命令输出为：
```text
Starting to serve on 127.0.0.1:8001
```
然后在另一个命令行窗口执行如下命令，能获得当前集群的版本信息。
```bash
curl 127.0.0.1:8001/version
```
命令输出为：
```text
{
  "major": "1",
  "minor": "13",
  "gitVersion": "v1.13.4",
  "gitCommit": "c27b913fddd1a6c480c229191a087698aa92f0b1",
  "gitTreeState": "clean",
  "buildDate": "2019-02-28T13:30:26Z",
  "goVersion": "go1.11.5",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```
# 4. 部署Nginx应用
本节介绍如何在新创建的集群上部署一个简单Nginx应用，并可通过公网地址访问。
## 4.1 创建Namespace
为了管理方便，新建一个Namespace（名字为nginx-app)来管理与该应用相关的资源。
```bash
michaeldembp-2:~ myang$ kubectl create namespace nginx-app
```
然后执行kubectl get namespaces获得新建的Namespace。
```bash
kubectl get namespaces
```
命令输出为：
```text
NAME          STATUS   AGE
default       Active   76m
kube-public   Active   76m
kube-system   Active   76m
nginx-app     Active   7s
```
## 4.2 创建secret对象
由于nginx镜像是存储在金山云镜像仓库中的私有镜像，在pull镜像时需要用户名和口令，因此创建一个secret对象，其中命令中的"XXXXXXXX"需要替换为在"2.1 重置镜像仓库登录密码"中重置的密码。
```bash
kubectl create secret docker-registry gs-registry-key --docker-server=hub.kce.ksyun.com --docker-username=73403251 --docker-password=XXXXXXXX --docker-email=obsicn@hotmail.com -n nginx-app
secret/gs-registry-key created
```
获得新创建的secret对象。
```bash
kubectl get secrets -n nginx-app
```
命令输出为：
```text
NAME                  TYPE                                  DATA   AGE
default-token-bxbdd   kubernetes.io/service-account-token   3      4m9s
gs-registry-key       kubernetes.io/dockerconfigjson        1      45s
ksyunregistrykey      kubernetes.io/dockerconfigjson        1      4m7s
```
## 4.3 创建Deployment，同时创建Pods

编辑nginx-deployment.yaml文件如下。其中image参数指定了需要获取的Docker镜像地址，imagePullSecrets参数指定了前一步创建的secret对象。
```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      run: nginx-app
  replicas: 2
  template:
    metadata:
      labels:
        run: nginx-app
    spec:
      containers:
      - name: nginx
        image: hub.kce.ksyun.com/ksc-gs/nginx:latest
        ports:
        - containerPort: 80
      imagePullSecrets:
        - name: gs-registry-key
```
通过执行kubectl apply命令完成Deployment对象的创建。
```bash
kubectl apply -f https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/nginx/nginx-deployment.yaml -n nginx-app
```

获得新创建的deployment对象信息。
```bash
michaeldembp-2:kce-gs myang$ kubectl get deployments -n nginx-app -o wide
```
命令输出为：
```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                                  SELECTOR
nginx-deployment   2/2     2            2           71s   nginx        hub.kce.ksyun.com/ksc-gs/nginx:latest   run=nginx-app
```
获得新创建的Pod对象（因为replicas值为2，所以会创建两个Pod对象）。
```bash
kubectl get pods -n nginx-app -o wide -L run
```
命令输出为：
```text
NAME                                READY   STATUS    RESTARTS   AGE     IP          NODE         NOMINATED NODE   READINESS GATES   RUN
nginx-deployment-857bc75d4d-bxn2p   1/1     Running   0          9m30s   10.0.1.13   10.34.0.10   <none>           <none>            nginx-app
nginx-deployment-857bc75d4d-qkddf   1/1     Running   0          9m30s   10.0.1.14   10.34.0.10   <none>           <none>            nginx-app
```
在显示的Pod信息可以看到两个Pod的Run Label设置为"nginx-app"，该Label将在后续创建Service对象时用到。
kubectl port-forward命令可在本地启动一个Proxy，把针对本地的请求forward给Pod。
```bash
kubectl port-forward pod/nginx-deployment-857bc75d4d-bxn2p 8888:80 -n nginx-app
```
命令输出为：
```text
Forwarding from 127.0.0.1:8888 -> 80
Forwarding from [::1]:8888 -> 80
```
然后在另一个命令行窗口中访问127.0.0.1:8888，可发现Pod能正常提供服务。
```bash
curl -I 127.0.0.1:8888
```
命令输出为：
```text
HTTP/1.1 200 OK
Server: nginx/1.17.2
Date: Mon, 12 Aug 2019 15:11:08 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 23 Jul 2019 11:45:37 GMT
Connection: keep-alive
ETag: "5d36f361-264"
Accept-Ranges: bytes
```
## 4.4 创建Kubernetes Service对象，通过公网访问Nginx服务
首先编辑nginx-service.yaml文件如下。
```yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx-service
  labels:
    run: nginx-app
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
  selector:
     run: nginx-app
```
其中type为LoadBalancer，将创建一个负载均衡器，同时会创建一个Listener，该Listener的侦听端口为80（port: 80), 后端服务器的端口也是80(targetPort: 80)。此外，selector的配置为"run: nginx-app"，也就是说把请求转发给Label为"run: nginx-app"的Pod上。
执行如下命令将完成Service对象创建。
```bash
kubectl apply -f https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/nginx/nginx-service.yaml -n nginx-app
```
执行如下命令获得新创建的Service对象信息。
```bash
kubectl get services -o wide -n nginx-app
```
命令输出信息如下：
```text
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE   SELECTOR
nginx-service   LoadBalancer   10.254.213.197   120.92.79.86   80:30244/TCP   97s   run=nginx-app
```
其中120.92.79.86是公网访问地址，通过执行如下命令验证Nginx服务可以通过外网访问。
```bash
curl  -I 120.92.79.86 
```
命令输出信息如下：
```text
HTTP/1.1 200 OK
Server: nginx/1.17.2
Date: Mon, 12 Aug 2019 15:53:07 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 23 Jul 2019 11:45:37 GMT
Connection: keep-alive
ETag: "5d36f361-264"
Accept-Ranges: bytes
```
此时访问金山云控制台，能看到创建Kubernetes Service对象nginx-service时创建的负载均衡实例和监听器信息。

![负载均衡实例信息](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/slb-listener.png)


# 5 部署Kubernetes仪表盘

## 5.1 创建Dashboard相关对象
部署Kubernetes Dashboard通常基于github的指南，并推荐采用该yaml文件https://github.com/kubernetes/dashboard/blob/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml。
但该部署配置文件有如下三个问题：
* 和ServiceAccount对象kubernetes-dashboard绑定的权限不够，导致在Dashboard中某些对象不能访问；
* Dashboard容器镜像地址是k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1，该地址访问有时不畅。
针对上述两个问题，对yaml文件进行如下修改：
* 创建的Dashboard Service只能集群内访问。

下面针对上述三个问题对Dashboard进行相应修改。
* 给kubernetes-dashboard绑定cluster-admin ClusterRole。
```yaml
# -------------------  Role Binding ------------------- #

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
```
* 修改Dashboard容器镜像地址为金山云镜像仓库中对应镜像的支持。
```yaml
      containers:
      - name: kubernetes-dashboard
        image: hub.kce.ksyun.com/kubernetes-google/kubernetes-dashboard-amd64:v1.10.1
        ports:
        - containerPort: 8443
          protocol: TCP
```
* Service类型从缺省的ClusterIP修改为LoadBalancer。
```yaml
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: LoadBalancer
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
```
执行如下命令创建Dashboard相关对象。
```bash
kubectl apply -f https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/dashboard/kubernetes-dashboard.yaml
```
命令输出为：
```text
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-admin created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
```
执行如下命令查看Service信息：
```bash
kubectl get services -n kube-system
```
命令输出为：
```text
NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                   AGE
coredns                   ClusterIP      10.254.0.10      <none>           53/UDP,53/TCP,9153/TCP    16h
heapster                  ClusterIP      10.254.150.172   <none>           80/TCP                    16h
kubernetes-dashboard      LoadBalancer   10.254.172.103   120.92.209.133   443:32400/TCP             51s
metrics-server            ClusterIP      10.254.67.107    <none>           443/TCP                   16h
monitoring-influxdb       ClusterIP      10.254.190.9     <none>           8086/TCP                  16h
traefik-ingress-service   ClusterIP      10.254.208.103   <none>           80/TCP,443/TCP,8080/TCP   16h
```
通过Service对象kubernetes-dashboard可获得Kubernetes Dashboard的访问地址是：[https://120.92.209.133/](https://120.92.209.133/)。

## 5.2 获得访问Dashboard的Token
在创建Dashboard相关对象时，会创建一个类型为kubernetes.io/service-account-token的Secret。首先执行如下命令列出Secret。
```bash
kubectl get secrets -n kube-system
```
命令输出为：
```text
NAME                                     TYPE                                  DATA   AGE
cluster-autoscaler-token-lcrhd           kubernetes.io/service-account-token   3      16h
coredns-token-b8swl                      kubernetes.io/service-account-token   3      16h
default-token-cgzxq                      kubernetes.io/service-account-token   3      16h
flannel-token-v27jn                      kubernetes.io/service-account-token   3      16h
heapster-token-rjxv4                     kubernetes.io/service-account-token   3      16h
ksyunregistrykey                         kubernetes.io/dockerconfigjson        1      16h
kube-proxy-token-7cr7z                   kubernetes.io/service-account-token   3      16h
kubernetes-dashboard-certs               Opaque                                0      6m11s
kubernetes-dashboard-key-holder          Opaque                                2      10m
kubernetes-dashboard-token-5qk86         kubernetes.io/service-account-token   3      6m11s
metrics-server-token-99282               kubernetes.io/service-account-token   3      16h
traefik-ingress-controller-token-frfmz   kubernetes.io/service-account-token   3      16h
```
其中"kubernetes-dashboard-token-5qk86" Secret对象中存储了Token。通过如下命令获得token信息。
```bash
kubectl describe secret/kubernetes-dashboard-token-5qk86 -n kube-system
```
命令输出为：
```text
Name:         kubernetes-dashboard-token-5qk86
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard
              kubernetes.io/service-account.uid: 809eb749-bd6f-11e9-94cc-fa163e09c4bc

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1029 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC10b2tlbi01cWs4NiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjgwOWViNzQ5LWJkNmYtMTFlOS05NGNjLWZhMTYzZTA5YzRiYyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.jwER9T9_s3OgxJLE5l-fYdWTHFpmxcthJVpIjfg-LMBj0XsT0JYumMZgnh5drM3ulq9HtQ7buR7UjR6meHEMjHLVekNOlop6trbGflc3Js7pGuakz6zjqmv8oWKwa4zSZvGZkrLOLgBDTrPGv157A77Pc8QxXnKrdQYwTabLVsrrdP7Ln0ykpuvweucelhu15gn7ImR8CZUmyy03DCshlVXDOlvEGR1I4SLPgtVE3kJ1Gq22-5UpVhnXhiB8pMb7UCU2BuWlALmMB5UEipwDF4hpjoVBArtndzlQPK2eZ8-KZ-ingZEh0eN8nMi497lxkCjBbXeYSFHkkjgMsLW-6A
```
其中token字段后面的值就是访问Dashboard的Token。

## 5.3 通过浏览器访问Dashboard
通过浏览器访问地址[https://120.92.209.133/](https://120.92.209.133/)。在忽略一些安全告警信息后，进入如下界面：

![Dashboar登录](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/dashboard-login.png)
在"Enter token"处输入"5.2 获得访问Dashboard的Token"获得的Token信息，进入Dashboard界面。
![Dashboar登录](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/dashboard-main.png)

# 6 小结
本文介绍了如何利用金山云容器引擎服务管理容器镜像和Kubernetes集群，并部署一个简单的Nginx应用和标准仪表盘的过程。金山云容器引擎服务不仅仅简化Kubernetes集群的安装和管理，还可以和金山云其它IaaS（云主机、云硬盘、负载均衡等）和PaaS服务整合，为客户提供了高弹性、高安全、易管理并按量计费的弹性应用支撑平台。

# 7 参考资料
* 金山云容器引擎产品介绍：https://www.ksyun.com/post/product/KCE
* 金山云容器引擎产品帮助：https://docs.ksyun.com/products/46


