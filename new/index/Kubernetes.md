# Kubernetes

## 学前准备

### 虚拟机软件

[VirtualBox](https://www.virtualbox.org/wiki/Downloads)

[VMWare Fusion（需付费，有试用期300天）](https://communities.vmware.com/t5/Fusion-for-Apple-Silicon-Tech/ct-p/3022)

>注：VirtualBox 支持 Windows 和 macOS，但有一个小缺点，它只能运行在Intel（x86_64）芯片上，不支持Apple新出的 M1（arm64/aarch64）芯片。

### Linux 环境

[Ubuntu](https://ubuntu.com/download/desktop)

[Centos（推荐阿里云镜像）](https://developer.aliyun.com/mirror/)

### 配置

最低要求是 2c2g，推荐 4g，硬盘容量 40G 以上。另外，一些对于服务器来说不必要的设备也可以禁用或者删除，比如声卡、摄像头、软驱等等，可以节约一点系统资源。

网络配置

待更新。。。

## 容器

### Docker 安装

```bash
sudo apt install -y docker.io #安装Docker Engine
```

```bash
sudo service docker start         #启动docker服务
sudo usermod -aG docker ${USER}   #当前用户加入docker组
```

第一个 `service docker start` 是启动Docker的后台服务，第二个 `usermod -aG` 是把当前的用户加入Docker的用户组。这是因为操作Docker必须要有root权限，而直接使用root用户不够安全，加入Docker用户组是一个比较好的选择，这也是Docker官方推荐的做法。当然，如果只是为了图省事，你也可以直接切换到root用户来操作Docker。

上面的三条命令执行完之后，我们还需要退出系统（命令 `exit` ），再重新登录一次，这样才能让修改用户组的命令 `usermod` 生效。

现在我们就可以来验证Docker是否安装成功了，使用的命令是 `docker version` 和 `docker info`。

### Docker 命令

```bash
docker ps  # 列出当前系统里运行的容器

# 可选项
	-a			：所有容器，也会带上历史运行过的容器
	-n			：显示最近创建的容器
	-q			：只显示容器的编号

docker pull [imageName]:tag # 从外部镜像仓库拉取镜像

docker images  # 列出当前存储的所有镜像
docker image ls 

# 可选项
	-a,--all			：列出所有镜像
	-q,--quiet		：只显示镜像的id

docker image rm [imageName] # 删除 image 文件
docker rmi -f [imageId...]
docker rm [imageId]			#删除指定的容器，不能删除正在运行的容器，如果要强制删除加上 -f
docker rm -f $(docker ps -aq)	#删除所有容器
docker ps -a -q|xargs docker rm	#删除所有容器

docker search [imageName] # 查找镜像

# 可选项
	-f;--filter		    ：过滤

docker run [参数] [imageName] # 新建容器并启动

# 可选项
	--name="Name"		：容器名称，如tomcat01、tomcat02，用来区分容器
	-d					：后台方式运行
	-it					：使用交互方式运行，进入容器查看内容
	-p					：指定容器的端口，如-p 8080:8080
		-p ip:主机端口:容器端口
		-p 主机端口:容器端口（常用）
		-p 容器端口:容器端口
	-P					：随机指定端口

docker logs  # 查看容器日志

# 可选项
	-tf		：显示日志
	-tail number	：显示日志条数
	
docker top [containerId]	# 查看容器中进程信息

docker inspect [imageId]	# 查看镜像中的元数据

docker exec -it [containerId] bashShell  # 进入容器后开启一个新的终端，可以在里面操作
docker attach [containerId]  # 进入容器正在执行的终端，不会启动新的进程

docker stop [containerId] # 停止运行的容器
docker start [containerId] # 再次运行停止的容器
```

### Docker 架构

![K8s Docker 架构图](../picture/K8s Docker 架构图.png)

命令行的`docker`实际上是一个客户端 client，它会与 Docker Engine 里的后台服务 Docker daemon 通信，而镜像则存储在远端的仓库 Registry 里，客户端并不能直接访问镜像仓库。

### 容器隔离

在计算机领域的隔离是出于系统安全的角度来考虑。对于Linux操作系统来说，一个不受任何限制的应用程序是十分危险的。这个进程能够看到系统里所有的文件、所有的进程、所有的网络流量，访问内存里的任何数据，那么恶意程序很容易就会把系统搞瘫痪，正常程序也可能会因为无意的 Bug导 致信息泄漏或者其他安全事故。虽然 Linux 提供了用户权限控制，能够限制进程只访问某些资源，但这个机制还是比较薄弱的，和真正的“隔离”需求相差得很远。而现在，使用容器技术，我们就可以让应用程序运行在一个有严密防护的“沙盒”（Sandbox）环境之内。

另外，在计算机里有各种各样的资源，CPU、内存、硬盘、网卡，虽然目前的高性能服务器都是几十核CPU、上百GB的内存、数TB的硬盘、万兆网卡，但这些资源终究是有限的，而且考虑到成本，也不允许某个应用程序无限制地占用。

容器技术的另一个本领就是为应用程序加上资源隔离，在系统里切分出一部分资源，让它只能使用指定的配额，比如只能使用一个 CPU，只能使用 1GB 内存等等，这样就可以避免容器内进程的过度系统消耗，充分利用计算机硬件，让有限的资源能够提供稳定可靠的服务。

### 容器与虚拟机的区别

![K8s 容器与虚拟机的对比](../picture/c.png)

Docker 官网的图示其实并不太准确，容器并不直接运行在 Docker 上，Docker 只是辅助建立隔离环境，让容器基于 Linux 操作系统运行。

首先，容器和虚拟机的目的都是隔离资源，保证系统安全，然后是尽量提高资源的利用率。

从实现的角度来看，虚拟机虚拟化出来的是硬件，需要在上面再安装一个操作系统后才能够运行应用程序，而硬件虚拟化和操作系统都比较“重”，会消耗大量的 CPU、内存、硬盘等系统资源，但这些消耗其实并没有带来什么价值，属于“重复劳动”和“无用功”，不过好处就是隔离程度非常高，每个虚拟机之间可以做到完全无干扰。

容器是直接利用了下层的计算机硬件和操作系统。因为比虚拟机少了一层，所以自然就会节约 CPU 和内存，显得非常轻量级，能够更高效地利用硬件资源。不过，因为多个容器共用操作系统内核，应用程序的隔离程度就没有虚拟机那么高了。

运行效率，可以说是容器相比于虚拟机最大的优势，在这个对比图中就可以看到，同样的系统资源，虚拟机只能跑3个应用，其他的资源都用来支持虚拟机运行了，而容器则能够把这部分资源释放出来，同时运行6个应用。

### 什么是容器化应用

在其他场合中也曾经见到过“镜像”这个词，比如最常见的光盘镜像，重装电脑时使用的硬盘镜像，还有虚拟机系统镜像。这些“镜像”都有一些相同点：只读，不允许修改，以标准格式存储了一系列的文件，然后在需要的时候再从中提取出数据运行起来。容器技术里的镜像也是同样的道理。因为容器是由操作系统动态创建的，那么必然就可以用一种办法把它的初始环境给固化下来，保存成一个静态的文件，相当于是把容器给“拍扁”了，这样就可以非常方便地存放、传输、版本化管理了。

从功能上来看，镜像和常见的tar、rpm、deb等安装包一样，都打包了应用程序，但最大的不同点在于它里面不仅有基本的可执行文件，还有应用运行时的整个系统环境。这就让镜像具有了非常好的跨平台便携性和兼容性，能够让开发者在一个系统上开发（例如Ubuntu），然后打包成镜像，再去另一个系统上运行（例如CentOS），完全不需要考虑环境依赖的问题，是一种更高级的应用打包方式。

任何应用都能够用这种形式打包再分发后运行，这也是无数开发者梦寐以求的“一次编写，到处运行（Build once, Run anywhere）”的至高境界。所以，所谓的“容器化的应用”，或者“应用的容器化”，就是指应用程序不再直接和操作系统打交道，而是封装成镜像，再交给容器环境去运行。

镜像就是静态的应用容器，容器就是动态的应用镜像，两者互相依存，互相转化，密不可分。

### 镜像的内部机制是什么


