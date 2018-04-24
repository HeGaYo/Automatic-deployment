[TOC]

## 简介
Docker 是一个开源的应用容器引擎——能够把开发的应用程序自动部署到容器，开发者可以利用Docker打包自己的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的Linux机器上，也可以实现虚拟化。
技术：
Docker基于LXC(LinuX Containers)，使用AUFS
优势：
1. 提供一个简单、轻量的建模方式
Docker依赖于“写时复制”模型，使得修改应用程序也非常迅速
轻量级的虚拟化技术。与传统的VM相比，更轻量，启动速度更快。单台硬件上可以同时运行成百上千个容器，时刻在业务高峰期通过启动大量容器进行横向拓展。
2. 可移植（跨平台）
3. 职责的逻辑分离：交付的东西是整个应用程序运行环境：OS+各种中间件、类库+应用程序代码。开发者关心容器中运行的应用程序；运维者关心如何管理容器
4. 快速、高效的开发生命周期。缩短代码从开发、测试到部署、上限运行的周期，让应用程序具备可移植性，已与构建，并易于协作。

## 核心组件
- Docker客户端和服务器
Docker是一个客户-服务器（C/S）架构的程序。Docker客户端只需向Docker服务器或者守护进程发送请求，服务器或守护进程将完成所有的工作并返回结果。
Docker提供命令行工具docker以及一整套的RestFul API

- Docker镜像
用户基于镜像来运行自己的容器
- Registry
Docker利用Registry来保存用户构建的镜像，分为公共和私有两种。
- Docker容器
容器是基于镜像启动起来的，容器中可以运行一个或多个进程
    - 一个镜像格式
    - 一系列标准的操作
    - 一个执行环境

容器可以为各种测试提供很好的沙盒环境，并且具备“标准性”的特征，非常适合为服务创建构建快

## Docker的安装
此处介绍ubuntu的安装（结合自身安装的经历）
我是通过安装脚本进行安装
```
$curl https://get.docker.com/ | sudo sh
$sudo reboot （重启后docker才会启动，奇怪--||）
$docker info （出现一串东西就对了 containers:0 Running:0之类的）
```
Ps：如果出现了这段文字,说明守护进程没有开启，我的解决方法就是重启机器
```
cannot connect to the Docker daemon. Is 'docker -d' running on this host?
```
启动/停止docker
```
sudo start/stop docker
```

## Docker的基本工作原理

**检查docker是否正常运行：**
```
$docker info
```
这个命令会返回所有容器和镜像的数量，Docker使用的额执行驱动和存储驱动以及Docker的基本配置

**创建新容器**
```
$sudo docker run -i -t ubuntu /bin/bash
```
`-i`标志保证容器中的STDIN是开启的
`-t`标志告诉Docker为要创建的容器分配一个伪tty终端
这里选择`ubuntu`作为基础镜像

- Docker会先检查本地是否存在ubuntu镜像，不存在就回去Docker Hub Registry查看。一旦找到该镜像，就会下载镜像并将其保存到本地宿主机中
- Docker利用这个镜像创建一个新容器，该容器拥有自己的网络、ip地址以及一个和宿主机进行通信的桥接网络接口。
- 最后我们告诉Docker要在容器中运行什么命令 这里启动了一个bash shell

**容器的命名**
```
$sudo docker run --name bob_the_container -I -t ubuntu /bin/bash
```
**退出容器（退出容器及停止容器）**
```
$exit
```
**重新启动已经停止的容器**
```
$sudo docker start bob_the_container(也可以换为容器ID)
```
**附着到容器上（进入到容器中）**
```
$sudo docker attach bob_the_container
```
**创建守护式容器（长期运行的容器，没有交互式会话，非常适合运行应用程序和服务）**
```
$sudo docker run --name daemon_dave -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
```
**获取容器的日志**
```
$sudo docker logs daemon_dave 输出几条日志项并返回
$sudo docker logs -f daemon_dave 一直监控Docker的日志
$sudo docker logs -ft daemon_dave 加上时间戳
```
**查看容器内的进程**
```
$sudo docker top daemon_dave
```
**在容器内部额外启动新进程**
```
$sudo docker exec -d daemon_dave touch /etc/new_config_file
```

**获取更多容器的信息**
```
$sudo docker inspect daemon_dave
$sudo docker inspect --format='{{.Name}}' daemon_dave
$sudo docker inspect --format '{{.Name}} {{.State.Running}}' daemon_dave
```
**删除的前提是停止或者关闭容器**
```
$sudo docker stop daemon_dave
```
或
```
$sudo docker kill daemon_dave
```
**删除容器**
```
$sudo docker rm daemon_dave
```
**删除所有的容器**
```
$sudo rm 'docker ps -a -q'
```

Docker镜像是由文件系统叠加而成，最低端是一个引导文件系统bootfs
当一个容器启动之后，他会被移到内存中，引导文件系统会被卸载，以留下更多的内存供intrid磁盘镜像使用。
Docker镜像的第二层是root文件系统rootfs，位于引导系统之上。在docker中，root文件系统永远只是只读状态，并且Docker利用联合加载(union mount)技术又会在root文件系统层加载更多的只读文件系统。

Docker是写时复制

**列出镜像**
```
$sudo docker images
拉去ubuntu仓库中的所有内容，会下载一系列的ubuntu镜像（只包含最低限度的支持系统运行的组件，用tag来区分不同的镜像 用[仓库名:标签名]来指定某一镜像）
$sudo docker pull ubuntu
$sudo docker run -t -I --name new_container ubuntu:12.04 /bin/bash
```
**查找镜像**
$sudo docker search puppet

**构建镜像**
两种方法。`docker commit`就不说了 主要针对——编写`Dockerfile`之后使用`docker build`命令
```
$mkdir static_web
$cd static_web
$touch Dockerfile
```
写入`Dockerfile`内容后，用`docker build`编译
```
$cd static_web
$sudo docker build -t="jamtur01/static_web" .
```
**设置标签**
```
$sudo docker build -t="jamtur01/static_web:v1" .
```
**忽略Dockerfile的构建缓存**
```
$sudo docker build --no-cache -t="jamtur01/static_web"
```
**基于构建缓存的Dockerfile模板**
```
ENV REFRESHED_AT 2017-01-08
```
`REFRESHED_AT`的环境变量用来表明该镜像模板最后的更新时间,若要刷新一个构建，只需要修改ENV指令中的日期即可

**探求镜像如何构建出来**
```
$sudo docker history image_id
```
**从新镜像启动容器**
```
Sudo docker run -d -p 80 --name static_web jamtur01/static_web nginx -g "daemon off;"
```
`-d` 以分离的方式在后台运行
`Nginx -g "daemon off;"`需要在容器中运行的命令 将以前台的方式启动Nginx，来作为web服务器
`-p 80` 端口分配映射 docker会在宿主机上随机选择一个位于49153-65535的一个比较大的端口号来映射到容器中的80端口上
**查看端口映射**
```
$sudo docker ps -l
$sudo docker port image_id 80
```

## Dockerfile常用语法
给个例子
```
version:0.0.1
FROM ubuntu
MAINTAINER ann "ann@example.com"
RUN apt-get update
RUN apt-get install -y nginx
RUN echo 'Hi, I am in your container' > /usr/share/nginx/html/index.html
EXPOSE 80
```
- **RUN** 指定镜像被构建时要运行的命令。会在shell里使用命令包装器/bin/sh -c来执行
	RUN apt-get update
- **EXPOSE** 告诉docker该容器内的应用程序将会使用容器的指定端口
- **CMD** 用于指定一个容器启动时要运行的命令。指定容器被启动时要运行的命令
	CMD ["/bin/bash","-l"] 将-l标志传递给了/bin/bash命令
	使用docker run命令可以覆盖CMD指令
	在Dockerfile中只能指定一条CMD指令。最后一条CMD会覆盖前面的命令
- **ENTRYPOINT** 提供的命令不容易在启动容器时被覆盖。Docker run中的任何参数都会被当做参数再次传递给ENTRYPOINT指定的命令。
- **WORKDIR** 用来在从镜像创建一个新容器时，在容器内部设置一个工作目录，ENTRYPOINT和/或CMD都会在这个目录中执行
	WORKDIR /opt/webapp
	Docker run命令行用-w标志来设置工作目录
- **ENV** 用来在镜像构建过程中设置环境变量
	ENV RVM_PATH /home/rvm/
	Docker run命令行用-e标志来传递环境变量
- **ADD**用来将构建环境下的文件和目录复制到镜像中
	ADD software.lic /opt/application/softwar/lic
	将构建目录下的文件复制到镜像中的/opt/application/目录下。如果目的地址末尾字符为/，那么docker认定原位置指向是目录。否则，是文件。
	ADD在处理归档文件（gzip,bzip2,xz），docker会自动将归档文件解开(unpack)
- **COPY** 在构建上下文中复制本地文件，不做文件提取与解压工作。复制的路径必须是一个与当前构建环境相对的文件或目录，本地文件要放在Dockerfile同一个目录中，不能复制目录外的任何文件
- **VOLUME** 用来向基于镜像创建的容器添加卷。一个卷是可以存在于一个或者多个容器内的特定的目录，这个目录绕过联合文件系统，并提供如下共享数据或者对数据进行是九华的功能。
	VOLUME["/opt/project"]这条指令会基于此镜像创建的任何容器创建一个名为/opt/project的挂载点
- **USER**用来指定该镜像会以什么样的用户去执行
- **ONBUILD**为镜像添加触发器
