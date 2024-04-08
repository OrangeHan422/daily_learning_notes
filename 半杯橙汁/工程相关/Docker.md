# Docker

## Docker和虚拟机的区别

+ 虚拟机会通过Hypervisor(虚拟机管理系统，就是VMware,VirtualBox这些)，虚拟出网卡，cpu，内存等虚拟硬件，再在上面建立起虚拟机，每个虚拟机是独立的操作系统，有自己的系统内核
+ 容器则是利用**namespace**将文件系统，进程，网络，设备等资源进行隔离；利用**cgroup**对权限、cpu资源进行限制，最终可以使容器之间互不影响

## 容器的整体流程

+ 命名空间隔离：容器启动时，他的进程以及相关的资源都被置于独立的命名空间中，这样容器中的进程只能看到自己的命名空间，而不会感知到主机或者其他容器的存在
+ Cgroup控制：使用cgroup，哦容器的资源使用可以被限制和控制，例如可以限制容器使用的CPU百分比，内存量等。

## Docker的优势

> docker就相当于一个进程

+ 直接使用宿主机的硬件资源，因此在cpu、内存、利用率上，Docker将会在效率上有更大的优势
+ Docker直接利用宿主机的系统内核，避免了虚拟机启动时所需要的系统引导时间和操作系统运行的资源消耗，利用Docker可以在几秒钟之内启动大量的容器，是虚拟机无法半岛的。快速启动，低资源消耗的优点，使Docker在弹性云平台的自动运维系统方面具有很好的应用场景(结合K8s集群管理)
+ 容器的启动是秒级的，大量节约开发、测试、部署的时间。更关键的是，Docker可以更高效的部署和扩容，Docker几乎可以在任何平台运行，包括虚拟机、物理机、公有云、私有云、个人电脑、服务器等，这种兼容性，可以让用户把一个应用程序从一个平台直接迁移到另外一个平台
+ 但是，虚拟机的安全性比容器好一些，docker与宿主机共享内核、文件系统等资源，可能对其他容器和宿主机造成影响

## Docker的基本组成

> 相关概念：
>
> + 镜像：镜像类似于一个模板，或者说是容器的静态版本。镜像运行起来就是容器
> + 容器：可以简单理解成微型虚拟机，但是没有自己的资源，都依赖宿主机的资源。（类似进程与线程的关系）

+ 客户端
+ 服务器
+ 仓库：存储镜像文件

## Docker的常用命令

### 帮助命令：

```shell
docker version		# 显示docker版本信息
docker info			#显示docker的系统信息，包括镜像和容器的数量
docker 命令 --help   #帮助命令
```

### 镜像常用命令：

#### docker images：查看本地镜像

```shell
[root@test /]# docker images
RESPOSITORY		TAG		IMAGE ID		CREATED		SIZE
hello-world		latest	salskdfj		1 days ago	13.3kB

# 解释
RESPOSITORY 镜像的仓库源
TAG			镜像的标签
IMAGE ID	镜像ID
CREATED		镜像的创建时间
SIZE		镜像的大小

# 可选项
-a,--all	# 列出所有镜像
-q,--quiet	# 只显示镜像的id
```

#### docker search：搜索镜像

```shell
[root@test /]# docker search mysql
NAME	DESCRIPITION	STARS	OFFICIAL
mysql	...				9494	[OK]		
mariadb	...				3441	[OK]
...

# 可选项，过滤条件
--filter=STARS=3000 #搜索STARS大于3000的
# 示例
[root@test /]# docker search mysql --filter=STARS=3000
NAME	DESCRIPITION	STARS	OFFICIAL
mysql	...				9494	[OK]		
mariadb	...				3441	[OK]

[root@test /]# docker search mysql --filter=STARS=5000
NAME	DESCRIPITION	STARS	OFFICIAL
mysql	...				9494	[OK]		
```

#### docker pull:拉取镜像

```shell
# docker pull name[:tag=latest]
[root@test /]# docker pull mysql
using default tag:latest	# 默认tag为latest
...
Digest:sha256:...			# 签名
Status:Downloaded newer image for mysql:latest
docker.io/library/mysql:latest # 真实地址

# 以下命令等价
docker pull mysql
docker pull docker.io/library/mysql:latest
```

#### docker rmi:删除镜像

```shell
[root@test /]# docker rmi -f 容器ID
[root@test /]# docker rmi -f 容器ID 容器ID 容器ID
[root@test /]# docker rmi -f $(docker images -aq) #删除全部容器
```

#### docker tag：镜像打标签

```shell
# docker tag 镜像ID 命名容器
[root@test /]#docker tag docker.io/centos docker.io/centos:v1
```

### 容器常用命令

#### docker run:新建容器并启动

```shell
# docker run [param] image

#参数说明
--name=""	# 容器名称
-d			#	后台方式运行
-it			# 使用交互方式运行，进入容器查看内容
[root@test /]#docker run -it centos /bin/bash			#使用it交互方式打开centos镜像，使用/bin/bash解释命令

-p			# 指定容器端口 -p 8080:8080
	-p	ip:主机端口:容器端口
	-p	主机端口:容器端口（常用，即端口映射）
	-p	容器端口
	容器端口
-p			# 随机指定端口
```

#### docker ps：查看容器

```shell
# docker ps
-a		# 列出当前正在运行的容器+历史运行过的容器
-n=?	# 显示最近创建的容器
-q		# 只显示容器编号

#示例：
[root@test /]#docker ps
CONTAINER ID   IMAGE           COMMAND       CREATED      STATUS                  PORTS     NAMES

[root@test /]#docker ps -a
CONTAINER ID   IMAGE           COMMAND       CREATED      STATUS                  PORTS     NAMES
1157c35256ab   linux:monitor   "/bin/bash"   2 days ago   Exited (0) 2 days ago             linux_monitor
```

#### docker rm：删除容器

```shell
docker rm ID						# 删除指定容器，不能删除正在运行的容器，如果要强制删除，使用rm -f
docker -rm -f $(docker ps -aq)		# 删除所有容器	 
docker ps -a -q|xargs docker rm		# 删除所有容器
```

#### 启动和停止容器（类似服务）

```shell
docker start id		# 启动容器
docker restart id	# 重启容器
docker stop id		# 停止当前正在运行的容器
docker kill id		# 强制停止当前的容器

#	后台启动容器：docker run -d name
[root@test /]#docker run -d centos
#注意，docker容器使用后台运行，就必须要有一个前台进程，docker发现没有应用，就会自动停止。（似乎可以使用screen解决？）
```

#### 进入后台运行的容器

```shell
# 方式一：docker -it id bashShell

[root@test /]#docker exec -it id /bin/bash    进入容器后开启一个/bin/bash的终端，可以在bash中操作（常用）

# 方式二：docker attach id
[root@test /]#docker attach id			进入程序正在执行的终端，不会启动新的进程
```

#### 日志相关

```shell
#docker logs -tf --tail n id
# -tf 显示日志
# --tail n 显示日志的条数

# 示例：
rosnoetic@rosnoetic-VirtualBox:~$ docker ps -a
CONTAINER ID   IMAGE           COMMAND       CREATED      STATUS                  PORTS     NAMES
1157c35256ab   linux:monitor   "/bin/bash"   2 days ago   Exited (0) 2 days ago             linux_monitor

rosnoetic@rosnoetic-VirtualBox:~$ docker logs -tf --tail 10 1157c35256ab
2024-03-08T07:50:31.928023507Z root@rosnoetic-VirtualBox:/# exit
```

#### 查看容器进程

```shell
# docker top id
[root@test /]#docker top id
UID		PID		PPID		C		STIME		TTY
```

#### 查看镜像的元数据

> 元数据：

```shell
# docker inspect id
[root@test /]#docker inspect id
```

#### 容器拷贝文件到宿主机

```shell
# docker cp id:docker_path host_path

# 示例：
[root@test /]#docker cp id:/home/test /home
```

#### 退出容器

```shell
exit		#容器停止并退出
ctrl+P+Q	#容器不停止并退出
```

### docker命令概述

```shell
attach		# 当前shell下依附指定运行镜像
build		# 通过Dockerfile定制镜像
commit		# 提交当前容器为新的镜像
cp			# 从容器中拷贝文件或目录到宿主机
create		# 创建一个新的容器，和run的区别是，仅创建，不启动,即run = create + start
diff		# 查看docker容器变化
events		# 从docker服务获取容器实时事件
exec		# 在已存在的容器上运行命令
export		# 到处容器的内容流作为一个tar归档文件  和import对应
history		# 展示一个镜像的历史记录
images		# 列出镜像
import		# 从tar包中的内容太创建一个新的文件系统
info		# 显示系统相关信息
inspect		# 查看容器的的底层信息
kill		# 关闭正在运行的容器
load		# 从一个tar包中加载一个镜像
login		# 注册或者登陆一个Docker注册（源）服务器
logout		# 从Docker注册（源）服务器中退出
logs		# 获取容器的日志
port		# 查看映射端口对应的容器内存源端口
pause		# 暂停容器内的所有进程
ps			# 列出容器
pull		# 从Docker注册（源）服务器拉去指定镜像或者仓库
push		# 推送指定镜像或者仓库到Docker注册（源）服务器
restart		# 重启运行中的容器
rm			# 移除一个或多个容器
rmi			# 移除一个或多个镜像（镜像需要使关闭状态，否则需要加-f）
run			# 在一个新容器中执行命令
save		# 保存一个镜像到tar包中
search		# 在Docker hub中搜索镜像
start		# 启动一个停止了的容器
stop		# 停止一个正在运行的容器
tag			# 将镜像打标签放入源（repository）中
top			# 查询容器中运行的进程
unpause		# 取消暂停容器
version		# 查看docker版本
wait		# 等待容器停止并且阻塞，然后输出退出代码(exit code)
```

## 容器数据卷

### 什么是数据卷

数据卷是一个虚拟目录，将宿主机目录映射到容器内目录，方便我们操作容器内的文件，或者方便迁移容器产生的数据  （本地目录<-----map--->数据卷<-----------map------->容器目录）

### 如何挂载数据卷

+ 在创建容器的时候，使用`-v 数据卷名:容器内目录`完成挂载

  > 该命令会在本地创建docker/volumns/数据卷名/_data,使得虚拟目录与本地目录相关联。由此，本地目录也和容器目录进行了间接的关联

+ 容器创建时，如果发现挂载的数据卷不存在时，会自动创建

### 数据卷常见命令

+ docker volume ls：查看数据卷
+ docker volume rm：删除数据卷
+ docker volume inspect：查看数据卷详情
+ docker volume prune：删除未使用的数据卷

> 卷技术也可以用来解决持久化的问题，即将容器中的目录挂载到本地目录下：
>
> + 在执行docker run 时，使用`-v 本地目录:容器目录`可以完成本地目录的挂载
> + 本地目录必须以"/"或"./"揩油，如果直接使用名称开头，会被识别为数据卷，而非本地目录。比如：
>   - -v mysql:/var/lib/mysql 会将容器目录映射到数据卷mysql中
>   - -v mysql:/var/lib/mysql 会将容器目录挂载到本地当前目录下的mysql目录中

利用数据卷可以实现宿主机与容器的双向映射。

## Dockerfile

> 通过dockerfile可以构建出所需的镜像，构建步骤：
>
> + 编写dockerfile文件
> + docker build 构建成为一个镜像
> + docker run 运行镜像
> + docker push 发布镜像

Dockerfile就相当于一层一层的构建镜像,dockerfile实例：

```dockerfile
FROM centos

VOLUME ["volume01","volume02"]

CMD echo "----end----"

CMD /bin/bash
```

通过dockerfile构建镜像的命令：

```shell
docker build -f /home/path/dockfile -t path/name:tag .
```

### dockerfile的编写

基础知识：

+ 每个保留关键字都必须是大写字母

+ 执行顺序从上到下

+ \#表示注释

+ 每一个指令都会创建一个新的镜像层，并提交。实例：

  > ​			可写容器(container)
  >
  > ​			镜像(tomcat)
  >
  > ​			镜像(jdk)
  >
  > ​			rootfs基础镜像(centos/ubuntu)
  >
  > ​			bootfs

dockerfile中的关键字：

```dockerfile
FROM			# 基础镜像，一切从这里开始构建
MAINTAINER		# 镜像作者，一般姓名+邮箱
RUN				# 镜像构建时需要运行的命令
ADD				# 添加内容：如压缩包
WORKDIR			# 进行的工作目录
VOLUME			# 挂载的目录
EXPOSE			# 保留端口配置
CMD				# 指定这个容器启动时候要运行的命令，仅最后一个生效，可以被覆盖
ENTRYPOINT		# 指定这个容器启动时候要运行的命令，可以追加命令
ONBUILD			# 当构建一个被继承的DockerFile，这个时候就会运行OMBUILD的指令。触发指令
COPY			# 类似ADD，将本地文件拷贝到镜像中
ENV				# 构建时候设置的环境变量
```

