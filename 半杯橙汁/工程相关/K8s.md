> 使用脚本管理大量容器时效率不高，K8s主要用来管理容器集群，主要用于容器化应用程序的编排，部署和扩展。

# 核心组件

可以使用一个简单的例子来了解Kubernetes的核心组件：

> 从一个**节点**开始， 节点就是一个物理机或者虚拟机；在节点中可以有多个**pod**，是Kubernetes最小调度单元，可以由一个或者多个容器组合而成。同一个pod的容器可以共享资源。
>
> 假设一个系统，包括一个应用程序和一个数据库。我们可以将应用程序和数据库分别放在两个不同的pod中。应用程序和数据库之间通过IP地址通信，这个IP地址是pod创建时分配的，集群内部的IP地址。  这里就有两个问题：
>
> + IP地址是一个内部的IP地址，集群外无法访问
> + pod不是一个稳定的实体，非常容易创建和销毁。比如发生故障时，Kubernetes自动销毁故障pod并新建一个，此时pod的IP地址会发生变化 
>
> 为了解决这个IP地址不稳定的问题，Kubernetes提供了**服务**(Service)组件，可以将一组pod封装为一个服务，这个服务可以通过一个统一的入口来访问 。在示例中，可以将应用程序和数据库封装为两个Service，这样的话，应用程序就可以通过Service的IP地址来访问数据库了。此时，即使Service管理的某个pod发生了故障，Service会自动将请求转发到健康的pod中，即对外提供的服务没有收到影响。
>
> 服务又分为内部服务和外部服务，类似C++类的public和private。外部服务有几种类型，其中一种就是NodePort，它会在节点上开放一个端口，然后将该端口映射到Servie的IP地址和端口上，此时就可以通过节点的IP地址和端口来访问服务了。在开发，测试时使用这种方式是没有问题的，但是生产环境中都是通过域名访问服务的，此时需要使用另一个资源对象，**Ingress**
>
> Ingress是用来管理从集群外部访问集群内部服务的入口和方式的，可以通过Ingress来配置不同的转发规则。 Ingress也可用来管理负载均衡，SSL证书等。
>
> 现在程序和数据库已经可以在Kubernetes运行并对外提供服务了。但是程序访问数据库时，如果将url，用户密码等写在配置文件中，两者就会有耦合问题，一旦数据库信息发生了变化，就要重新编译应用程序并部署到集群中。这对7×24的服务器来说是不可接收的。为了解决该问题，Kubernetes提供了**ConfigureMap**的组件，可以将配置信息封装起来，然后就可以在应用程序中读取和使用了。这样就可以将配置信息和应用程序的镜像分离开来，保证了容器化应用程序的可移植性。此时，数据库信息发生变化时，我们仅需要修改ConfigureMap中的信息，然后重新加载数据库Pod即可。需要注意的是，ConfigureMap中的信息都是明文的，一些敏感信息不建议存储在该组件中，而是建议存储在**Secret**组件中。
>
> Secret和ConfigureMap类似，可以存储配置信息提供应用程序使用。虽然不是明文存储，但是也只是做了一层Base64编码而已，并非加密机制，因此还需要Kubernetes其他如访问控制等功能来提高安全性。
>
> 当pod销毁或者重启时，容器中的数据也会消失。这对需要持久化存储的应用程序是不行的。为此，Kubernetes提供了**Volumes**组件，可以将一些持久化的资源挂载到集群中的本地磁盘上或者集群外的远程存储上。
>
> 此时应用程序已经有了大多基本功能，但是仍没有高可用性。比如应用程序所在的节点发生了故障，或者节点需要升级和更新维护的时候，应用程序就会停止服务。 解决方法很简单，就是增加节点，简单来说就是将所有东西都复制几份放在另外的节点上，这样当一个节点发生故障时，Service会自动将请求转发到另一个节点上来继续提供服务。Kubernetes提供了**Deployment**组件就是来解决这个问题的。
>
> Deployment可以定义和管理应用程序的副本数量 以及应用程序的更新策略，可以简化应用程序的部署和更新操作。Deployment和pod的关系就像pod对容器的关系，即高度抽象的管理者。Deployment的功能：
>
> + 副本控制：定义和管理应用程序的副本数量。如果某个副本发生故障，会自动创建新副本保持指定数量的副本。
> + 滚动更新：定义和管理应用程序的更新策略，就是为了保证应用程序的平滑升级
>
> 对于数据库来说，也存在高可用的问题，但是数据库一般不使用deployment的策略，因为数据库之间的多副本是有状态的（即每个副本有自己的状态）。最直白的就是数据库中持久化存储的数据需要保证多个副本之间是一致的。这就要求额外的机制，比如将数据写入共享内存中或副本数据之间的同步。对于这样有状态的程序，Kubernetes提供了**StatefulSet**的组件来管理
>
> StatefulSet和Deployment非常相似，也提供了定义和管理应用程序副本数量或动态扩缩容等功能。此外，还保证了每个副本都有自己稳定的网络标识符和持久化存储。但是也并非万能的，推荐做法是将数据库这样的组件剥离出Kubernetes单独部署和管理，这样可以简化架构和方便部署以及避免不需要的问题。

# 架构

简单来说就是Master和Worker的模式架构

对于每个工作Node，需要三个核心组件：

+ container-runtime:可以理解为一个运行容器的软件，负责拉取容器镜像，创建容器启动或者停止容器等。所有的应用程序都需要container-runtime来运行。所以每个工作节点上都必须要安装

+ kubelet:负责管理和维护节点上的pod，确保按照预期运行。也会定期从apiServer组件接收新的或修改后的Pod规范。同时也会监控工作节点的运行情况，并将信息汇报给apiServer。

+ kube-proxy:负责为pod对象提供网络代理和负载均衡服务。

  > 通常情况下，不同的node通过service进行通信。这时需要一个负载均衡器来接收请求，然后将请求发送到不同的节点。kube-proxy就是负责该项功能的，它会在每个node上启动一个网络代理，使发往service的流量以一种高效的方式路由到正确的pod中

 对于Master节点，有四个核心组件：

+ kube-apiserver:负责提供Kubernetes集群的API接口服务，所有的组件都会通过这个接口来通信。比如作为用户，需要在集群中部署一个新应用时，可以通过使用客户端，如kubectl命令行工具，来和apiserver进行交互。apiserver就像一个集群的网关。是整个系统的入口，所有的请求都会经过它并验证请求的合法性。

+ Scheduler:调度器，负责监控集群中所有节点的资源使用情况，然后根据调度策略，将pod调度到合适的节点上运行。比如新建pod，就会根据node的资源使用状况自动的选择合适的节点

+ ControllerManger:控制器管理器，负责管理集群中各种资源对象的状态。比如当集群中任何一个节点发生故障的时候，ControllerManager就会选择合适的策略来响应，确保集群中的各种资源对象都按照预期进行。如何获取资源状态，则是通过etcd获取

+ etcd:一个高可用的键值存储系统。用来存储集群中所有资源对象的状态信息。比如pod崩溃了，或者新建了pod等。可以理解为集群的大脑，是整个集群的数据存储中心

+ CloudControllerManager：当使用云厂商的Kubernetes集群时，提供的云平台相关的控制器。负责与云平台的API交互，可以提供一致的管理接口。
# 搭建环境

minikube是一个轻量级的Kubernetes实现，可在本地计算机上创建虚拟机，并部署仅包含一个节点的简单集群。

```bash
# 安装
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
# 启动
minikube start
```

## 使用Multipass和k3s可以搭建一个多节点的kubernetes集群环境

### Multipass

Multipass是一个轻量级的虚拟机管理工具，可以用来在本地快速创建和管理虚拟机。
相比于VirtualBox或者VMware这样的虚拟机管理工具，Multipass更加轻量快速，而且它还提供了一些命令行工具来方便我们管理虚拟机。

```bash
# 安装
sudo snap install multipass
# 常用命令
# 查看帮助
multipass help
multipass help <command>
# 创建一个名字叫做k3s的虚拟机
multipass launch --name k3s --cpus 2 --memory 4G -- disk 10G
# 查看虚拟机列表
multipass list
# 进入虚拟机并执行shell
multipass shell k3s
# 在虚拟机中执行命令
multipass exec k3s -- ls -l
# 查看虚拟机的信息
multipass info k3s
# 停止虚拟机
multipass stop k3s
# 启动虚拟机
multipass start k3s
# 删除虚拟机
multipass delete k3s
# 清理虚拟机
multipass purge
# 挂载目录（将本地的~/kubernetes/master目录挂载到虚拟机中的~/master目录）
multipass mount ~/kubernetes/master master:~/master
```

创建虚拟机后，使虚拟机支持ssh登陆

```bash
# 进入虚拟机并执行shell
multipass shell k3s
# 为虚拟机创建密码
sudo passwd ubuntu #注意这个名字是shell提示符@前面的名字
# 更改ssh配置文件
sudo vim /etc/ssh/sshd_config # 将PasswordAuthentication改为yes，PermintRootLogind改为yes
# 重启ssh服务
sudo service ssh restart
# 退出虚拟机就可以使用ssh登陆虚拟机了
exit
ssh name@IP
# 配置免密登陆
ssh-keygen -t rsa -b 4096
cd .ssh
ssh-copy-id name@IP
# 为命令设置别名,仅在当前终端生效，永久生效要更改.bashrc文件
alias master='ssh name@IP'
```

### K3s

轻量级Kubernetes

```bash
# 安装
curl -sfL https://get.k3s.io | sh -
# 国内安装
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
# 安装完成后验证
sudo kubectl get nodes
# 获取master的token，用来创建worker节点
sudo cat /var/lib/rancher/k3s/server/node-token
# 将TOKEN保存在环境变量中
TOKEN=$(multipass exec k3s sudo cat /var/lib/rancher/k3s/server/node-token)
# 保存master的几点地址
MASTER_IP=$(multipass info k3s | grep IPv4 | awk '{print $2}')
# 创建两个worker节点的虚拟机
multipass launch --name worker1 --cpus 2 --memory 8G --disk 10G
multipass launch --name worker2 --cpus 2 --memory 8G --disk 10G

# 在worker节点虚拟机上安装k3s
 for f in 1 2; do
     multipass exec worker$f -- bash -c "curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=\"https://$MASTER_IP:6443\" K3S_TOKEN=\"$TOKEN\" sh -"
 done
```

## 在线环境

https://killercoda.com/

https://labs.play-with-k8s.com/

# 常用命令

## 基础使用

```bash
# 查看帮助
kubectl --help

# 查看API版本
kubectl api-versions

# 查看集群信息
kubectl cluster-info
```

## 资源创建和运行

> replicaset是处于deployment和pod之间的组件，一般不需要我们关心，是用来管理副本集的
>
> 一般情况下我们也不直接创建pod，而是创建一个deployment然后指定名称和使用镜像来创建，Kubernetes会自动帮我们创建对应的pod，其他操作可以直接通过edit资源deployment的配置文件来自动完成。

```bash
# 创建并运行一个指定的镜像
kubectl run NAME --image=image [params...]
# e.g. 创建并运行一个名字为nginx的Pod
kubectl run nginx --image=nginx

# 根据YAML配置文件或者标准输入创建资源
kubectl create RESOURCE
# e.g.
# 创建一个nginx镜像的deployment
kubectl create deployment nginx-deployment --image=nginx
# 根据nginx.yaml配置文件创建资源
kubectl create -f nginx.yaml
# 根据URL创建资源
kubectl create -f https://k8s.io/examples/application/deployment.yaml
# 根据目录下的所有配置文件创建资源
kubectl create -f ./dir

# 通过文件名或标准输入配置资源
kubectl apply -f (-k DIRECTORY | -f FILENAME | stdin)
# e.g.
# 根据nginx.yaml配置文件创建资源
kubectl apply -f nginx.yaml
```

## 查看资源信息

```bash
# 查看集群中某一类型的资源
kubectl get RESOURCE
# 其中，RESOURCE可以是以下类型：
kubectl get pods / po         # 查看Pod
kubectl get svc               # 查看Service
kubectl get deploy            # 查看Deployment
kubectl get rs                # 查看ReplicaSet
kubectl get cm                # 查看ConfigMap
kubectl get secret            # 查看Secret
kubectl get ing               # 查看Ingress
kubectl get pv                # 查看PersistentVolume
kubectl get pvc               # 查看PersistentVolumeClaim
kubectl get ns                # 查看Namespace
kubectl get node              # 查看Node
kubectl get all               # 查看所有资源

# 后面还可以加上 -o wide 参数来查看更多信息
kubectl get pods -o wide

# 查看某一类型资源的详细信息
kubectl describe RESOURCE NAME
# e.g. 查看名字为nginx的Pod的详细信息
kubectl describe pod nginx
```

## 资源修改、删除和清理

```bash
# 编辑某个资源的配置文件
kubectl edit RESOURCETYPE RESOURCENAME
# e.g. 编辑名为nginx-deployment的deployment组件配置文件
kubectl edit deployment nginx-deployment

# 更新某个资源的标签
kubectl label RESOURCE NAME KEY_1=VALUE_1 ... KEY_N=VALUE_N
# e.g. 更新名字为nginx的Pod的标签
kubectl label pod nginx app=nginx

# 删除某个资源
kubectl delete RESOURCE NAME
# e.g. 删除名字为nginx的Pod
kubectl delete pod nginx

# 删除某个资源的所有实例
kubectl delete RESOURCE --all
# e.g. 删除所有Pod
kubectl delete pod --all

# 根据YAML配置文件删除资源
kubectl delete -f FILENAME
# e.g. 根据nginx.yaml配置文件删除资源
kubectl delete -f nginx.yaml

# 设置某个资源的副本数
kubectl scale --replicas=COUNT RESOURCE NAME
# e.g. 设置名字为nginx的Deployment的副本数为3
kubectl scale --replicas=3 deployment/nginx

# 根据配置文件或者标准输入替换某个资源
kubectl replace -f FILENAME
# e.g. 根据nginx.yaml配置文件替换名字为nginx的Deployment
kubectl replace -f nginx.yaml
```

## 调试和交互

```bash
# 进入某个Pod的容器中
kubectl exec [-it] POD [-c CONTAINER] -- COMMAND [args...]
# e.g. 进入名字为nginx的Pod的容器中，并执行/bin/bash命令
kubectl exec -it nginx -- /bin/bash

# 查看某个Pod的日志
kubectl logs [-f] [-p] [-c CONTAINER] POD [-n NAMESPACE]
# e.g. 查看名字为nginx的Pod的日志
kubectl logs nginx

# 将某个Pod的端口转发到本地
kubectl port-forward POD [LOCAL_PORT:]REMOTE_PORT [...[LOCAL_PORT_N:]REMOTE_PORT_N]
# e.g. 将名字为nginx的Pod的80端口转发到本地的8080端口
kubectl port-forward nginx 8080:80

# 连接到现有的某个Pod（将某个Pod的标准输入输出转发到本地）
kubectl attach POD -c CONTAINER
# e.g. 将名字为nginx的Pod的标准输入输出转发到本地
kubectl attach nginx

# 运行某个Pod的命令
kubectl run NAME --image=image -- COMMAND [args...]
# e.g. 运行名字为nginx的Pod
kubectl run nginx --image=nginx -- /bin/bash 
```

## 配置文件

```bash
# 创建一个deployment的配置文件，配置文件名无要求，但是最好使用yaml
vim nginx-deployment.yaml
```

## 对外公开服务

> 一般是通过配置文件来创建的，但是也可以通过命令行创建

```bash
# 创将一个新的服务
kubectl create service nginx-service
# 或者将已创建的deployment对外暴露，即提供服务
kubectl expose deployment nginx-deployment
# 查看服务的详细信息
kubectl describe service nginx-deployment
```

Service的种类：

+ ClusterIP:默认类型，集群内部的服务
+ NodePort:节点端口类型，将服务公开到集群节点上
+ LoadBalancer:负载均衡类型，经服务公开到外部负载均衡器上
+ ExternalName:外部名称类型，将服务映射到一个外部域名上
+ Headless:无头类型，主要用于DNS解析和服务发现

# Portainer的安装和使用

[Portainer](https://www.portainer.io/) 是一个轻量级的容器管理工具，可以用来管理Docker和Kubernetes。
它提供了一个Web界面来方便我们管理容器。

```bash
# 创建一个名字叫做portainer的虚拟机
multipass launch --name portainer --cpus 2 --memory 8G --disk 10G
# 或者在master节点上安装portainer，并将其暴露在NodePort 30777上
kubectl apply -n portainer -f https://downloads.portainer.io/ce2-19/portainer.yaml
# 然后直接访问 https://localhost:30779/ 或者 http://localhost:30777/ 就可以了
```

