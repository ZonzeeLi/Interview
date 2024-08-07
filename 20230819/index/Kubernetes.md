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

在计算机领域的隔离是出于系统安全的角度来考虑。对于Linux操作系统来说，一个不受任何限制的应用程序是十分危险的。这个进程能够看到系统里所有的文件、所有的进程、所有的网络流量，访问内存里的任何数据，那么恶意程序很容易就会把系统搞瘫痪，正常程序也可能会因为无意的 Bug 导致信息泄漏或者其他安全事故。虽然 Linux 提供了用户权限控制，能够限制进程只访问某些资源，但这个机制还是比较薄弱的，和真正的“隔离”需求相差得很远。而现在，使用容器技术，我们就可以让应用程序运行在一个有严密防护的“沙盒”（Sandbox）环境之内。

另外，在计算机里有各种各样的资源，CPU、内存、硬盘、网卡，虽然目前的高性能服务器都是几十核 CPU、上百 GB 的内存、数TB的硬盘、万兆网卡，但这些资源终究是有限的，而且考虑到成本，也不允许某个应用程序无限制地占用。

容器技术的另一个本领就是为应用程序加上资源隔离，在系统里切分出一部分资源，让它只能使用指定的配额，比如只能使用一个 CPU，只能使用 1GB 内存等等，这样就可以避免容器内进程的过度系统消耗，充分利用计算机硬件，让有限的资源能够提供稳定可靠的服务。

### 容器与虚拟机的区别

![K8s 容器与虚拟机的对比](../picture/K8s 容器与虚拟机的对比.png)

Docker 官网的图示其实并不太准确，容器并不直接运行在 Docker 上，Docker 只是辅助建立隔离环境，让容器基于 Linux 操作系统运行。

首先，容器和虚拟机的目的都是隔离资源，保证系统安全，然后是尽量提高资源的利用率。

从实现的角度来看，虚拟机虚拟化出来的是硬件，需要在上面再安装一个操作系统后才能够运行应用程序，而硬件虚拟化和操作系统都比较“重”，会消耗大量的 CPU、内存、硬盘等系统资源，但这些消耗其实并没有带来什么价值，属于“重复劳动”和“无用功”，不过好处就是隔离程度非常高，每个虚拟机之间可以做到完全无干扰。

容器是直接利用了下层的计算机硬件和操作系统。因为比虚拟机少了一层，所以自然就会节约 CPU 和内存，显得非常轻量级，能够更高效地利用硬件资源。不过，因为多个容器共用操作系统内核，应用程序的隔离程度就没有虚拟机那么高了。

运行效率，可以说是容器相比于虚拟机最大的优势，在这个对比图中就可以看到，同样的系统资源，虚拟机只能跑3个应用，其他的资源都用来支持虚拟机运行了，而容器则能够把这部分资源释放出来，同时运行6个应用。

### 什么是容器化应用

在其他场合中也曾经见到过“镜像”这个词，比如最常见的光盘镜像，重装电脑时使用的硬盘镜像，还有虚拟机系统镜像。这些“镜像”都有一些相同点：只读，不允许修改，以标准格式存储了一系列的文件，然后在需要的时候再从中提取出数据运行起来。容器技术里的镜像也是同样的道理。因为容器是由操作系统动态创建的，那么必然就可以用一种办法把它的初始环境给固化下来，保存成一个静态的文件，相当于是把容器给“拍扁”了，这样就可以非常方便地存放、传输、版本化管理了。

从功能上来看，镜像和常见的 tar、rpm、deb 等安装包一样，都打包了应用程序，但最大的不同点在于它里面不仅有基本的可执行文件，还有应用运行时的整个系统环境。这就让镜像具有了非常好的跨平台便携性和兼容性，能够让开发者在一个系统上开发（例如Ubuntu），然后打包成镜像，再去另一个系统上运行（例如CentOS），完全不需要考虑环境依赖的问题，是一种更高级的应用打包方式。

任何应用都能够用这种形式打包再分发后运行，这也是无数开发者梦寐以求的“一次编写，到处运行（Build once, Run anywhere）”的至高境界。所以，所谓的“容器化的应用”，或者“应用的容器化”，就是指应用程序不再直接和操作系统打交道，而是封装成镜像，再交给容器环境去运行。

镜像就是静态的应用容器，容器就是动态的应用镜像，两者互相依存，互相转化，密不可分。

### 镜像的内部机制是什么

镜像就是一个打包文件，里面包含了应用程序还有它运行所依赖的环境，例如文件系统、环境变量、配置参数等等。

环境变量、配置参数这些东西还是比较简单的，随便用一个 manifest 清单就可以管理，真正麻烦的是文件系统。为了保证容器运行环境的一致性，镜像必须把应用程序所在操作系统的根目录，也就是 rootfs，都包含进来。

虽然这些文件里不包含系统内核（因为容器共享了宿主机的内核），但如果每个镜像都重复做这样的打包操作，仍然会导致大量的冗余。可以想象，如果有一千个镜像，都基于Ubuntu系统打包，那么这些镜像里就会重复一千次Ubuntu根目录，对磁盘存储、网络传输都是很大的浪费。

很自然的，我们就会想到，应该把重复的部分抽取出来，只存放一份Ubuntu根目录文件，然后让这一千个镜像以某种方式共享这部分数据。

这个思路，也正是容器镜像的一个重大创新点：分层，术语叫“Layer”。

容器镜像内部并不是一个平坦的结构，而是由许多的镜像层组成的，每层都是只读不可修改的一组文件，相同的层可以在镜像之间共享，然后多个层像搭积木一样堆叠起来，再使用一种叫“Union FS联合文件系统”的技术把它们合并在一起，就形成了容器最终看到的文件系统。

可以用命令`docker inspect`来查看镜像的分层信息，比如：

```bash
docker inspect nginx:alpine
```

它的分层信息在`RootFS`部分：

![K8s docker layer分层信息](../picture/K8s docker layer分层信息.png)

所以使用`docker pull`，`docker rmi`等命令操作镜像的时候，有一些奇怪的输出信息，就是镜像里的各个 Layer。Docker 会检查是否有重复的层，如果本地已经存在就不会重复下载，如果层被其他镜像共享就不会删除，这样就可以节约磁盘和网络成本。

### Dockerfile

最简单的 Dockerfile 实例：

```dockerfile
# Dockerfile.busybox
FROM busybox                  # 选择基础镜像
CMD echo "hello world"        # 启动容器时默认运行的命令
```

第一条指令是 FROM，所有的 Dockerfile 都要从它开始，表示选择构建使用的基础镜像，相当于“打地基”，这里我们使用的是 busybox。

第二条指令是 CMD，它指定`docker run`启动容器时默认运行的命令，这里我们使用了 echo 命令，输出“hello world”字符串。

用`docker build`命令来创建出镜像：

```bash
docker build -f Dockerfile.busybox .

Sending build context to Docker daemon   7.68kB
Step 1/2 : FROM busybox
 ---> d38589532d97
Step 2/2 : CMD echo "hello world"
 ---> Running in c5a762edd1c8
Removing intermediate container c5a762edd1c8
 ---> b61882f42db7
Successfully built b61882f42db7
```

用`-f`参数指定Dockerfile文件名，后面必须跟一个文件路径，叫做“构建上下文”（build’s context），这里只是一个简单的点号，表示当前路径的意思。

### 如何编写正确、高效的 Dockerfile

首先因为构建镜像的第一条指令必须是`FROM`，所以基础镜像的选择非常关键。如果关注的是镜像的安全和大小，那么一般会选择Alpine；如果关注的是应用的运行稳定性，那么可能会选择Ubuntu、Debian、CentOS。

```dockerfile
FROM alpine:3.15                # 选择Alpine镜像
FROM ubuntu:bionic              # 选择Ubuntu镜像
```

我们在本机上开发测试时会产生一些源码、配置等文件，需要打包进镜像里，这时可以使用`COPY`命令，它的用法和 Linux 的`cp`差不多，不过拷贝的源文件必须是“构建上下文”路径里的，不能随意指定文件。也就是说，如果要从本机向镜像拷贝文件，就必须把这些文件放到一个专门的目录，然后在`docker build`里指定“构建上下文”到这个目录才行。

```dockerfile
COPY ./a.txt  /tmp/a.txt    # 把构建上下文里的a.txt拷贝到镜像的/tmp目录
COPY /etc/hosts  /tmp       # 错误！不能使用构建上下文之外的文件
```

Dockerfile 里最重要的一个指令`RUN`，它可以执行任意的 Shell 命令，比如更新系统、安装应用、下载文件、创建目录、编译程序等等，实现任意的镜像构建步骤，非常灵活。`RUN`通常会是 Dockerfile 里最复杂的指令，会包含很多的 Shell 命令，但 Dockerfile 里一条指令只能是一行，所以有的`RUN`指令会在每行的末尾使用续行符`\`，命令之间也会用`&&`来连接，这样保证在逻辑上是一行，就像下面这样：

```dockerfile
RUN apt-get update \
    && apt-get install -y \
        build-essential \
        curl \
        make \
        unzip \
    && cd /tmp \
    && curl -fSL xxx.tar.gz -o xxx.tar.gz\
    && tar xzf xxx.tar.gz \
    && cd xxx \
    && ./config \
    && make \
    && make clean
```

有的时候在 Dockerfile 里写这种超长的`RUN`指令很不美观，而且一旦写错了，每次调试都要重新构建也很麻烦，所以你可以采用一种变通的技巧：把这些 Shell 命令集中到一个脚本文件里，用`COPY`命令拷贝进去再用`RUN`来执行：

```dockerfile
COPY setup.sh  /tmp/                # 拷贝脚本到/tmp目录

RUN cd /tmp && chmod +x setup.sh \  # 添加执行权限
    && ./setup.sh && rm setup.sh    # 运行脚本然后再删除
```

`RUN`指令实际上就是 Shell 编程，如果你对它有所了解，就应该知道它有变量的概念，可以实现参数化运行，这在 Dockerfile 里也可以做到，需要使用两个指令`ARG`和`ENV`。

它们区别在于`ARG`创建的变量只在镜像构建过程中可见，容器运行时不可见，而`ENV`创建的变量不仅能够在构建镜像的过程中使用，在容器运行时也能够以环境变量的形式被应用程序使用。

下面是一个简单的例子，使用`ARG`定义了基础镜像的名字（可以用在`FROM`指令里），使用 ENV 定义了两个环境变量：

```dockerfile
ARG IMAGE_BASE="node"
ARG IMAGE_TAG="alpine"

ENV PATH=$PATH:/tmp
ENV DEBUG=OFF
```

还有一个重要的指令是`EXPOSE`，它用来声明容器对外服务的端口号，对现在基于 Node.js、Tomcat、Nginx、Go 等开发的微服务系统来说非常有用：

```dockerfile
EXPOSE 443           # 默认是tcp协议
EXPOSE 53/udp        # 可以指定udp协议
```

每个指令都会生成一个镜像层，所以 Dockerfile 里最好不要滥用指令，尽量精简合并，否则太多的层会导致镜像臃肿不堪。

### docker build 是怎么工作的

由上面的 Docker 架构图可知，命令行`docker`是一个简单的客户端，真正的镜像构建工作是由服务端的`Docker daemon`完成的，所以`docker`客户端就只能把“构建上下文”目录打包上传（显示信息`Sending build context to Docker daemon`），这样服务器才能够获取本地的这些文件。

“构建上下文”其实与 Dockerfile 并没有直接的关系，它其实指定了要打包进镜像的一些依赖文件，而`COPY`命令也只能使用基于“构建上下文”的相对路径，因为“Docker daemon”看不到本地环境，只能看到打包上传的那些文件。

但这个机制也会导致一些麻烦，如果目录里有的文件（例如 readme/.git/.svn 等）不需要拷贝进镜像，docker 也会一股脑地打包上传，效率很低。

为了避免这种问题，可以在“构建上下文”目录里再建立一个`.dockerignore`文件，语法与`.gitignore`类似，排除那些不需要的文件。

下面是简单的示例，表示不打包上传后缀是`swp``sh`的文件：

```dockerfile
# docker ignore
*.swp
*.sh
```

另外一般应该在命令行里使用`-f`来显式指定。但如果省略这个参数，`docker build`就会在当前目录下找名字是 Dockerfile 的文件。所以，如果只有一个构建目标的话，文件直接叫“Dockerfile”是最省事的。

还可以加上一个`-t`参数，也就是指定镜像的标签（tag），这样 Docker 就会在构建完成后自动给镜像添加名字。当然，名字必须要符合上节课里的命名规范，用 `:`分隔名字和标签，如果不提供标签默认就是“latest”。

### 如何拷贝容器内的数据

`cp`这个命令，它可以在宿主机和容器之间拷贝文件，是最基本的一种数据交换功能，先用`docker run`启动一个容器：

```bash
docker run -d -rm redis
```

![K8s docker cp 举例1](../picture/K8s docker cp 举例1.png)

注意这里使用了`-d、--rm`两个参数，表示运行在后台，容器结束后自动删除，然后使用`docker ps`命令可以看到 Redis 容器正在运行。

`docker cp`用法很简单，类似于 Linux 的`cp``scp`，指定源路径（src path）和目标路径（dest path）就可以了。如果源路径是宿主机那么就是把文件拷贝进容器，如果源路径是容器那么就是把文件拷贝出容器，主要需要用容器名或者容器 ID 来指明是哪个容器的路径。

假设当前目录下有一个“面经.jpg”的图片，现在要拷贝进 Redis 容器的“/tmp”目录，如果使用容器 ID，命令如下：

```bash
docker cp 面经.jpg 811:/tmp
```

然后使用`docker exec`命令，进入容器查看文件是否已经正确拷贝了：

```bash
docker exec -it 811 sh
```

![K8s docker cp 举例2](../picture/K8s docker cp 举例2.png)

这里由于是中文所以显示的字符格式不对，而同样可以从容器中导出：

```bash
docker cp 811:/tmp/面经.jpg	./面经导出.jpg
```

### 如何共享主机上的文件

`docker cp`的用法模仿了操作系统的拷贝命令，偶尔一两次的文件共享还可以应付，如果容器运行时经常有文件来往互通，这样反复地拷来拷去就显得很麻烦，也很容易出错。

虚拟机有一种“共享目录”的功能，它可以在宿主机上开一个目录，然后把这个目录“挂载”进虚拟机，这样就实现了两者共享同一个目录，一边对目录里文件的操作，另一边立刻就能看到，没有了数据拷贝，效率自然也会高很多。

容器也提供了这样的共享宿主机目录第的功能，效果也和虚拟机几乎一样，只需要在`docker run`命令启动容器的时候使用`-v`参数就行，具体的格式就是“宿主机路径：容器内路径”。

以 Redis 为例，启动容器，使用`-v`参数把本机的“/tmp”目录挂载到容器里的“/tmp”目录，也就是说让容器共享宿主机的“/tmp”目录：

```bash
docker run -d --rm -v /tmp:/tmp redis
```

然后我们再用`docker exec`进入容器，查看一下容器内的“/tmp”目录，应该就可以看到文件与宿主机是完全一致的。

![K8s docker cp 举例3](../picture/K8s docker cp 举例3.png)

也可以在容器里的“/tmp”目录下随便做一些操作，比如删除文件、建立新目录等等，再回头观察一下宿主机，会发现修改会即时同步，这就表明容器和宿主机确实已经共享了这个目录。

`-v`参数挂载宿主机目录的这个功能，对于我们日常开发测试工作来说非常有用，我们可以在不变动本机环境的前提下，使用镜像安装任意的应用，然后直接以容器来运行我们本地的源码、脚本，非常方便。

### 如何实现网络互通

现在使用`docker cp`和`docker run -v`可以解决容器与外界的文件互通问题，但是对于Nginx、Redis 这些服务器来说，网络互通才是更要紧的问题。

网络互通的关键在于“打通”容器内外的网络，而处理网络通信无疑是计算机系统里最棘手的工作之一，Docker 提供了三种网络模式，分别是 null、host 和 bridge。

null 是最简单的模式，也就是没有网络，但允许其他的网络插件来自定义网络连接。

host 的意思是直接使用宿主机网络，相当于去掉了容器的网络隔离（其他隔离依然保留），所有的容器会共享宿主机的 IP 地址和网卡。这种模式没有中间层，自然通信效率高，但缺少了隔离，运行太多的容器也容易导致端口冲突。

host 模式需要在`docker run`时使用`--net=host`参数，以启动 Nginx 为例：

```bash
docker run -d --rm --net=host nginx:alpine
```

为了验证效果，可以在本机和容器里分别执行`ip addr`命令，查看网卡信息：

```bash
ip addr	// 本机查看网卡
docker exec xxx ip addr // 容器查看网卡
```

第三种 bridge，也就是桥接模式，它有点类似现实世界里的交换机、路由器，只不过是由软件虚拟出来的，容器和宿主机再通过虚拟网卡接入这个网桥（下图中的 docker0），那么它们之间也就可以正常的收发网络数据包了。不过和 host 模式相比，bridge 模式多了虚拟网桥和网卡，通信效率会低一些。

![K8s docker bridge模式](../picture/K8s docker bridge模式.png)

和 host 模式一样，我们也可以用`--net=bridge`来启用桥接模式，但其实并没有这个必要，因为 Docker 默认的网络模式就是 bridge，所以一般不需要显式指定。

下面启动两个容器 Nginx 和 Redis，没有特殊制定就是使用 bridge 模式：

```bash
docker run -d --rm nginx:alpine    # 默认使用桥接模式
docker run -d --rm redis           # 默认使用桥接模式
```

然后同样在本机和容器里执行`ip addr`命令，会发现容器里的网卡设置与宿主机完全不同。还可以使用`docker inspect`直接查看容器的 ip 地址。

```bash
docker inspect xxx | grep IPAddress
```

### 如何分配服务端口号

使用 host 模式或者 bridge 模式，容器就有了 ip 地址，建立了与外部世界的网络连接，接下来要解决的就是网络服务的端口号问题。

