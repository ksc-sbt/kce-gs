本入门指南介绍如何利用金山云容器引擎服务快速搭建Kubernetes集群，并部署一个Nginx应用的过程。同时介绍如何快速搭建和配置仪表盘服务和基于EFK的日志服务。

部署架构参考如下：


![金山云容器引擎服务架构示意图](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/kce-architecture.png)
界面参数具体描述如下：


本指南包含如下内容：
* 准备网络环境
* 使用金山云容器镜像服务
* 创建Kubernetes容器引擎
* 部署Nginx应用
* 部署Kubernetes仪表盘服务
* 部署基于EFK的容器日志服务



# 1 网络环境
本指南所使用的环境位于金山云北京6区。下面是在使用金山云容器引擎服务所需要准备的网络配置信息。

## 1.1 VPC配置信息

|  网络资源   | 名称  | CIDR  |
|  ----  | ----  | ----  |
| VPC  | gs-vpc |	10.34.0.0/16 |

## 1.2 子网配置信息

|  子网名称   | 所属VPC |可用区 | 子网类型  | CIDR  | 说明|
|  ----  | ----  | ----  |----  |----  |----|
| public_a  | gs-vpc |	可用区A | 普通子网| 10.34.51.0/24|用于跳板机|
| private_a  | gs-vpc |	可用区A | 普通子网| 10.34.0.0/20|用于K8S集群主机和其他云服务器|
| endpoint  | gs-vpc |	 | 终端子网| 10.34.52.0/24|用于管理内网负载均衡器实例|

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
michaeldembp-2:~ myang$ docker login hub.kce.ksyun.com -u XXXXXXXX
Password: 
Login Succeeded
```
##2.2	上传自定义镜像

通过docker push命令，可完成docker镜像上传。为了简单起见，重用docker.hub上的标准nginx镜像。首先执行docker pull命令下载标准nginx镜像。
```bash
michaeldembp-2:~ myang$ docker pull nginx
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
michaeldembp-2:~ myang$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              e445ab08b2be        2 weeks ago         126MB
```
为了在金山云容器镜像服务上存储私有镜像文件，金山云账户管理员需要在金山云控制台上创建镜像命令空间，比如为ksc-gs，并设置类型为私有。
![创建私有镜像空间](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/registry-namespace.png)

修改镜像tag，该tag包含金山云镜像注册服务器地址信息。
```bash
michaeldembp-2:~ myang$ docker tag e445ab08b2be  hub.kce.ksyun.com/ksc-gs/nginx:latest
michaeldembp-2:~ myang$ docker images
REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
nginx                            latest              e445ab08b2be        2 weeks ago         126MB
hub.kce.ksyun.com/ksc-gs/nginx   latest              e445ab08b2be        2 weeks ago         126MB

执行docker push命令，把镜像上传到hub.kce.ksyun.com镜像仓库。
```bash
michaeldembp-2:~ myang$ docker push hub.kce.ksyun.com/ksc-gs/nginx:latest
The push refers to a repository [hub.kce.ksyun.com/ksc-gs/nginx]
fe6a7a3b3f27: Pushed 
d0673244f7d4: Pushed 
d8a33133e477: Pushed 
latest: digest: sha256:dc85890ba9763fe38b178b337d4ccc802874afe3c02e6c98c304f65b08af958f size: 948
```
此时访问金山云控制台，能看到镜像已成功上传。需要注意的是金山云镜像服务当前不支持docker的search命令。
![金山云容器镜像仓库镜像信息](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/registry-repository.png)
 
## 2.3	进行镜像安全扫描
金山云镜像仓库提供镜像安全扫码功能，通过选择特定的镜像版本，执行“安全扫描”操作，将获得该镜像的安全扫描结果。
![金山云容器镜像仓库镜像信息](https://raw.githubusercontent.com/ksc-sbt/kce-gs/master/images/image-security-scan.png)
 
# 3 创建Kubernetes容器引擎