# Docker底层原理

## Docker基础

作为大神或者准架构师/架构师，一定要了解一下docker的底层原理。

但是，首先还是简单说明一下docker的简介。

### Docker 简介

![img](../images/docker/docker01.png)

Docker 是一个开源的应用容器引擎，基于 [Go 语言](https://www.runoob.com/go/go-tutorial.html) 并遵从 Apache2.0 协议开源。

Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。

容器是完全使用沙箱机制，相互之间不会有任何接口（类似 iPhone 的 app）,更重要的是容器性能开销极低。

Docker 从 17.03 版本之后分为 CE（Community Edition: 社区版） 和 EE（Enterprise Edition: 企业版），我们用社区版就可以了

### Docker的应用场景

-   Web 应用的自动化打包和发布。
-   自动化测试和持续集成、发布。
-   在服务型环境中部署和调整数据库或其他的后台应用。
-   从头编译或者扩展现有的 OpenShift 或 Cloud Foundry 平台来搭建自己的 PaaS 环境。

### Docker 架构

Docker 包括三个基本概念:

-   **镜像（Image）**：Docker 镜像（Image），就相当于是一个 root 文件系统。比如官方镜像 ubuntu:16.04 就包含了完整的一套 Ubuntu16.04 最小系统的 root 文件系统。
-   **容器（Container）**：镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。
-   **仓库（Repository）**：仓库可看成一个代码控制中心，用来保存镜像。

Docker 使用客户端-服务器 (C/S) 架构模式，使用远程API来管理和创建Docker容器。

Docker 容器通过 Docker 镜像来创建。

| 概念                   | 说明                                                         |
| :--------------------- | :----------------------------------------------------------- |
| Docker 镜像(Images)    | Docker 镜像是用于创建 Docker 容器的模板，比如 Ubuntu 系统。  |
| Docker 容器(Container) | 容器是独立运行的一个或一组应用，是镜像运行时的实体。         |
| Docker 客户端(Client)  | Docker 客户端通过命令行或者其他工具使用 Docker SDK (https://docs.docker.com/develop/sdk/) 与 Docker 的守护进程通信。 |
| Docker 主机(Host)      | 一个物理或者虚拟的机器用于执行 Docker 守护进程和容器。       |
| Docker Registry        | Docker 仓库用来保存镜像，可以理解为代码控制中的代码仓库。Docker Hub([https://hub.docker.com](https://hub.docker.com/)) 提供了庞大的镜像集合供使用。一个 Docker Registry 中可以包含多个仓库（Repository）；每个仓库可以包含多个标签（Tag）；每个标签对应一个镜像。通常，一个仓库会包含同一个软件不同版本的镜像，而标签就常用于对应该软件的各个版本。我们可以通过 **<仓库名>:<标签>** 的格式来指定具体是这个软件哪个版本的镜像。如果不给出标签，将以 **latest** 作为默认标签。 |

### 最为常用的几个命令

#### docker的守护进程查看

systemctl status docker

#### docker 镜像查看

docker image ls

#### docker 容器查看

docker ps

#### Docker Registry配置和查看

cat /etc/docker/daemon.json

    配置私有仓库
    
    cat>/etc/docker/daemon.json<<EOF
    
    {
    
      "registry-mirrors":["http://10.24.2.30:5000","https://tnxkcso1.mirrors.aliyuncs.com"],
    
      "insecure-registries":["10.24.2.30:5000"]
    
    }
    
    EOF

## Docker 的发展历史

Docker 公司前身是 DotCloud，由 Solomon Hykes 在2010年成立，2013年更名 Docker。同年发布了 Docker-compose 组件提供容器的编排工具。

2014年 Docker 发布1.0版本，2015年Docker 提供 Docker-machine，支持 windows 平台。

在此期间，Docker 项目在开源社区大受追捧，同时也被业界诟病的是 Docker 公司对于 Docker 发展具有绝对的话语权，比如 Docker 公司推行了 libcontainer 难以被社区接受。

为了防止 Docker 这项开源技术被Docker 公司控制，在几个核心贡献的厂商，于是在一同贡献 Docker 代码的公司诸如 Redhat，谷歌的倡导下，成立了 OCI 开源社区。

OCI 开源社区旨在于将 Docker 的发展权利回归社区，当然反过来讲，Docker 公司也希望更多的厂商安心贡献代码到Docker 项目，促进 Docker 项目的发展。

于是通过OCI建立了 runc 项目，替代 libcontainer，这为开发者提供了除 Docker 之外的容器化实现的选择。

OCI 社区提供了 runc 的维护，而 runc 是基于 OCI 规范的运行容器的工具。换句话说，你可以通过 runc，提供自己的容器实现，而不需要依赖 Docker。当然，Docker 的发行版底层也是用的 runc。在 Docker 宿主机上执行 runc，你会发现它的大多数命令和 Docker 命令类似，感兴趣的读者可以自己实践如何用 runc 启动容器。

至2017年，Docker 项目转移到 Moby 项目，基于 Moby 项目，Docker 提供了两种发行版，Docker CE 和 Docker EE， Docker CE 就是目前大家普遍使用的版本，Docker EE 成为付费版本，提供了容器的编排，Service 等概念。Docker 公司承诺 Docker 的发行版会基于 Moby 项目。这样一来，通过 Moby 项目，你也可以自己打造一个定制化的容器引擎，而不会被 Docker 公司绑定。

## Docker 与虚拟机有何区别

> Docker 的误解：Docker 是轻量级的虚拟机。

很多人将docker理解为， Docker 实现了类似于虚拟化的技术，能够让应用跑在一些轻量级的容器里。这么理解其实是错误的。

### 到底什么是docker：

Docker是一个Client-Server结构的系统，Docker守护进程运行在主机上， 然后通过Socket连接从客户端访问Docker守护进程。

Docker守护进程从客户端接受命令，并按照命令，管理运行在主机上的容器。

**一个docker 容器，是一个运行时环境，可以简单理解为进程运行的集装箱。**


### docker和kvm都是虚拟化技术，它们的主要差别：

1、Docker有着比虚拟机更少的抽象层

2、docker利用的是宿主机的内核，VM需要的是Guest OS

![img](../images/docker/1881010-20210402133626070-1344548588.png)

二者的不同：

-   VM(VMware)在宿主机器、宿主机器操作系统的基础上创建虚拟层、虚拟化的操作系统、虚拟化的仓库，然后再安装应用；
-   Container(Docker容器)，在宿主机器、宿主机器操作系统上创建Docker引擎，在引擎的基础上再安装应用。

所以说，新建一个容器的时候，docker不需要像虚拟机一样重新加载一个操作系统，避免引导。docker是利用宿主机的操作系统，省略了这个复杂的过程，秒级！

虚拟机是加载Guest OS ，这是分钟级别的

### 与传统VM特性对比：

作为一种轻量级的虚拟化方式，Docker在运行应用上跟传统的虚拟机方式相比具有显著优势：

-   Docker 容器很快，启动和停止可以在秒级实现，这相比传统的虚拟机方式要快得多。
-   Docker 容器对系统资源需求很少，一台主机上可以同时运行数千个Docker容器。
-   Docker 通过类似Git的操作来方便用户获取、分发和更新应用镜像，指令简明，学习成本较低。
-   Docker 通过Dockerfile配置文件来支持灵活的自动化创建和部署机制，提高工作效率。
-   Docker 容器除了运行其中的应用之外，基本不消耗额外的系统资源，保证应用性能的同时，尽量减小系统开销。
-   Docker 利用Linux系统上的多种防护机制实现了严格可靠的隔离。从1.3版本开始，Docker引入了安全选项和镜像签名机制，极大地提高了使用Docker的安全性。

| 特性       | 容器               | 虚拟机     |
| :--------- | :----------------- | :--------- |
| 启动速度   | 秒级               | 分钟级     |
| 硬盘使用   | 一般为MB           | 一般为GB   |
| 性能       | 接近原生           | 弱于原生   |
| 系统支持量 | 单机支持上千个容器 | 一般几十个 |

## Docker的技术底座：

Linux 命名空间、控制组和 UnionFS 三大技术支撑了目前 Docker 的实现，也是 Docker 能够出现的最重要原因。

![在这里插入图片描述](../images/docker/53de044ef67d4b1e9592a065645f778a.png)

-   namespace，命名空间

命名空间，容器隔离的基础，保证A容器看不到B容器.  
6个命名空间：User，Mnt，Network，UTS，IPC，Pid

-   cgroups，Cgroups 是 Control Group 的缩写，控制组

cgroups 容器资源统计和隔离

主要用到的cgroups子系统：cpu，blkio，device，freezer，memory

实际上 Docker 是使用了很多 Linux 的隔离功能，让容器看起来像一个轻量级虚拟机在独立运行，容器的本质是被限制了的 Namespaces，cgroup，具有逻辑上独立文件系统，网络的一个进程。

-   unionfs 联合文件系统

典型：aufs/overlayfs，分层镜像实现的基础

## UnionFS 联合文件系统

### 什么是镜像

那么问题来了，没有操作系统，怎么运行程序？

可以在Docker中创建一个centos的镜像文件，这样就能将centos系统集成到Docker中，运行的应用就都是centos的应用。

Image 是 Docker 部署的基本单位，一个 Image 包含了我们的程序文件，以及这个程序依赖的资源的环境。Docker Image 对外是以一个文件的形式展示的（更准确的说是一个 mount 点）。

### UnionFS 与AUFS

UnionFS 其实是一种为 Linux 操作系统设计的用于把多个文件系统『联合』到同一个挂载点的文件系统服务。

AUFS 即 Advanced UnionFS 其实就是 UnionFS 的升级版，它能够提供更优秀的性能和效率。

AUFS 作为先进联合文件系统，它能够将不同文件夹中的层联合（Union）到了同一个文件夹中，这些文件夹在 AUFS 中称作分支，整个『联合』的过程被称为联合挂载（Union Mount）。

概念理解起来比较枯燥，最好是有一个真实的例子来帮助我们理解：

首先，我们建立 company 和 home 两个目录，并且分别为他们创造两个文件

    # tree .
    .
    |-- company
    |   |-- code
    |   `-- meeting
    `-- home
        |-- eat
        `-- sleep


然后我们将通过 mount 命令把 company 和 home 两个目录「联合」起来，建立一个 AUFS 的文件系统，并挂载到当前目录下的 mnt 目录下：

    # mkdir mnt
    # ll
    total 20
    drwxr-xr-x 5 root root 4096 Oct 25 16:10 ./
    drwxr-xr-x 5 root root 4096 Oct 25 16:06 ../
    drwxr-xr-x 4 root root 4096 Oct 25 16:06 company/
    drwxr-xr-x 4 root root 4096 Oct 25 16:05 home/
    drwxr-xr-x 2 root root 4096 Oct 25 16:10 mnt/
    
    # mount -t aufs -o dirs=./home:./company none ./mnt
    # ll
    total 20
    drwxr-xr-x 5 root root 4096 Oct 25 16:10 ./
    drwxr-xr-x 5 root root 4096 Oct 25 16:06 ../
    drwxr-xr-x 4 root root 4096 Oct 25 16:06 company/
    drwxr-xr-x 6 root root 4096 Oct 25 16:10 home/
    drwxr-xr-x 8 root root 4096 Oct 25 16:10 mnt/
    root@rds-k8s-18-svr0:~/xuran/aufs# tree ./mnt/
    ./mnt/
    |-- code
    |-- eat
    |-- meeting
    `-- sleep
    
    4 directories, 0 files


通过 ./mnt 目录结构的输出结果，可以看到原来两个目录下的内容都被合并到了一个 mnt 的目录下。

默认情况下，如果我们不对「联合」的目录指定权限，内核将根据从左至右的顺序将第一个目录指定为可读可写的，其余的都为只读。那么，当我们向只读的目录做一些写入操作的话，会发生什么呢？

    # echo apple > ./mnt/code
    # cat company/code
    # cat home/code
    apple


​    

通过对上面代码段的观察，我们可以看出，当写入操作发生在 company/code 文件时， 对应的修改并没有反映到原始的目录中。

而是在 home 目录下又创建了一个名为 code 的文件，并将 apple 写入了进去。

看起来很奇怪的现象，其实这正是 Union File System 的厉害之处：

> Union File System 联合了多个不同的目录，并且把他们挂载到一个统一的目录上。

在这些「联合」的子目录中， 有一部分是可读可写的，但是有一部分只是可读的。

> 当你对可读的目录内容做出修改的时候，其结果只会保存到可写的目录下，不会影响只读的目录。

比如，我们可以把我们的服务的源代码目录和一个存放代码修改记录的目录「联合」起来构成一个 AUFS。前者设置只读权限，后者设置读写权限。

那么，一切对源代码目录下文件的修改都只会影响那个存放修改的目录，不会污染原始的代码。

在 AUFS 中还有一个特殊的概念需要提及一下：

branch – 就是各个要被union起来的目录。

Stack 结构 - AUFS 它会根据branch 被 Union 的顺序形成一个 Stack 的结构，从下至上，最上面的目录是可读写的，其余都是可读的。如果按照我们刚刚执行 aufs 挂载的命令来说，最左侧的目录就对应 Stack 最顶层的 branch。

所以：下面的命令中，最为左侧的为 home，而不是 company

    mount -t aufs -o dirs=./home:./company none ./mnt


### 什么是 Docker 镜像分层机制？

首先，让我们来看下 Docker Image 中的 Layer 的概念：

![在这里插入图片描述](../images/docker/f96ac59bd3d64229b818fe2e3209b63a.png)

Docker Image 是有一个层级结构的，最底层的 Layer 为 BaseImage（一般为一个操作系统的 ISO 镜像），然后顺序执行每一条指令，生成的 Layer 按照入栈的顺序逐渐累加，最终形成一个 Image。

直观的角度来说，是这个图所示：

![在这里插入图片描述](../images/docker/dcb8554b2e934a97a5ed36a70976a5d8.png)

每一次都是一个被联合的目录，从目录的角度来说，大致如下图所示：

![在这里插入图片描述](../images/docker/f96ac59bd3d64229b818fe2e3209b63a.png)

### Docker Image 如何而来呢？

简单来说，一个 Image 是通过一个 DockerFile 定义的，然后使用 docker build 命令构建它。

DockerFile 中的每一条命令的执行结果都会成为 Image 中的一个 Layer。

这里，我们通过 Build 一个镜像，来观察 Image 的分层机制：

Dockerfile:

    # Use an official Python runtime as a parent image
    FROM python:2.7-slim
    
    # Set the working directory to /app
    WORKDIR /app
    
    # Copy the current directory contents into the container at /app
    COPY . /app
    
    # Install any needed packages specified in requirements.txt
    RUN pip install --trusted-host pypi.python.org -r requirements.txt
    
    # Make port 80 available to the world outside this container
    EXPOSE 80
    
    # Define environment variable
    ENV NAME World
    
    # Run app.py when the container launches
    CMD ["python", "app.py"]


构建结果：

    root@rds-k8s-18-svr0:~/xuran/exampleimage# docker build -t hello ./
    Sending build context to Docker daemon  5.12 kB
    Step 1/7 : FROM python:2.7-slim
     ---> 804b0a01ea83
    Step 2/7 : WORKDIR /app
     ---> Using cache
     ---> 6d93c5b91703
    Step 3/7 : COPY . /app
     ---> Using cache
     ---> feddc82d321b
    Step 4/7 : RUN pip install --trusted-host pypi.python.org -r requirements.txt
     ---> Using cache
     ---> 94695df5e14d
    Step 5/7 : EXPOSE 81
     ---> Using cache
     ---> 43c392d51dff
    Step 6/7 : ENV NAME World
     ---> Using cache
     ---> 78c9a60237c8
    Step 7/7 : CMD python app.py
     ---> Using cache
     ---> a5ccd4e1b15d
    Successfully built a5ccd4e1b15d


通过构建结果可以看出，构建的过程就是执行 Dockerfile 文件中我们写入的命令。构建一共进行了7个步骤，每个步骤进行完都会生成一个随机的 ID，来标识这一 layer 中的内容。 最后一行的 a5ccd4e1b15d 为镜像的 ID。由于我贴上来的构建过程已经是构建了第二次的结果了，所以可以看出，对于没有任何修改的内容，Docker 会复用之前的结果。

如果 DockerFile 中的内容没有变动，那么相应的镜像在 build 的时候会复用之前的 layer，以便提升构建效率。并且，即使文件内容有修改，那也只会重新 build 修改的 layer，其他未修改的也仍然会复用。

通过了解了 Docker Image 的分层机制，我们多多少少能够感觉到，Layer 和 Image 的关系与 AUFS 中的联合目录和挂载点的关系比较相似。

而 Docker 也正是通过 AUFS 来管理 Images 的。

## Namespaces

在Linux系统中，Namespace是在内核级别以一种抽象的形式来封装系统资源，通过将系统资源放在不同的Namespace中，来实现资源隔离的目的。不同的Namespace程序，可以享有一份独立的系统资源。

命名空间（namespaces）是 Linux 为我们提供的用于分离进程树、网络接口、挂载点以及进程间通信等资源的方法。在日常使用 Linux 或者 macOS 时，我们并没有运行多个完全分离的服务器的需要，但是如果我们在服务器上启动了多个服务，这些服务其实会相互影响的，每一个服务都能看到其他服务的进程，也可以访问宿主机器上的任意文件，这是很多时候我们都不愿意看到的，我们更希望运行在同一台机器上的不同服务能做到完全隔离，就像运行在多台不同的机器上一样。

Linux 的命名空间机制提供了以下七种不同的命名空间，包括 :

-   CLONE\_NEWCGROUP、
-   CLONE\_NEWIPC、
-   CLONE\_NEWNET、
-   CLONE\_NEWNS、
-   CLONE\_NEWPID、
-   CLONE\_NEWUSER 和
-   CLONE\_NEWUTS，

通过这七个选项, 我们能在创建新的进程时, 设置新进程应该在哪些资源上与宿主机器进行隔离。具体如下：

| **Namespace** |      **Flag**       |        **Page**        |                   **Isolates**                    |
| :-----------: | :-----------------: | :--------------------: | :-----------------------------------------------: |
|    Cgroup     | **CLONE_NEWCGROUP** | **cgroup_namespaces**  |               Cgroup root directory               |
|      IPC      |  **CLONE_NEWIPC**   |   **ipc_namespaces**   | System V IPC,POSIX message queues 隔离进程间通信  |
|    Network    |  **CLONE_NEWNET**   | **network_namespaces** | Network devices,stacks, ports, etc. 隔离网络资源  |
|     Mount     |   **CLONE_NEWNS**   |  **mount_namespaces**  |          Mount points 隔离文件系统挂载点          |
|      PID      |  **CLONE_NEWPID**   |   **pid_namespaces**   |             Process IDs 隔离进程的ID              |
|     Time      |  **CLONE_NEWTIME**  |  **time_namespaces**   |             Boot and monotonic clocks             |
|     User      |  **CLONE_NEWUSER**  |  **user_namespaces**   |      User and group IDs 隔离用户和用户组的ID      |
|      UTS      |  **CLONE_NEWUTS**   |   **uts_namespaces**   | Hostname and NIS domain name 隔离主机名和域名信息 |

对于docker来说，最为直接的，是PID隔离

### PID隔离

如果需要了解 PID的命名空间隔离，我们从基础的 linux 进程的 fork函数开始，尽管，系统调用函数fork()并不属于namespace的API。

> 当程序调用fork（）函数时，系统会创建新的进程，为其分配资源，例如存储数据和代码的空间。然后把原来的进程的所有值都复制到新的进程中，只有少量数值与原来的进程值不同，相当于克隆了一个自己。那么程序的后续代码逻辑要如何区分自己是新进程还是父进程呢？

fork()的神奇之处在于它仅仅被调用一次，却能够返回两次（父进程与子进程各返回一次），通过返回值的不同就可以进行区分父进程与子进程。它可能有三种不同的返回值：

- 在父进程中，fork返回新创建子进程的进程ID

- 在子进程中，fork返回0

- 如果出现错误，fork返回一个负值  
  下面给出一段实例代码，命名为fork\_example.c。


  int main (){
      pid_t fpid; //fpid表示fork函数返回的值
      int count=0;
      fpid=fork();
      if (fpid < 0)printf("error in fork!");
      else if (fpid == 0) {
          printf("I am child. Process id is %d/n",getpid());
      }
      else {
          printf("i am parent. Process id is %d/n",getpid());
      }
      return 0;
  }

编译并执行，结果如下。

    root@local:~# gcc -Wall fork_example.c && ./a.out
    I am parent. Process id is 28365
    I am child. Process id is 28366


> 使用fork()后，父进程有义务监控子进程的运行状态，并在子进程退出后自己才能正常退出，否则子进程就会成为“孤儿”进程。

这里提出一个问题，在宿主机上启动两个容器，在这两个容器内都各有一个 PID=1的进程，众所周知，Linux 里 PID 是唯一的，既然 Docker 不是跑在宿主机上的两个虚拟机，那么它是如何实现在宿主机上运行两个相同 PID 的进程呢？

这里就用到了 PID Namespaces，它其实是 Linux 创建新进程时的一个可选参数，在 Linux 系统中创建进程的系统调用是 clone()方法。

    int clone(int (*fn) (void *)，void *child stack,
              int flags， void *arg， . . .
             /* pid_ t *ptid, void *newtls, pid_ t *ctid */ ) ;


![在这里插入图片描述](../images/docker/525181d7950a45c3b7fac5bc450b5f91.png)

通过调用这个方法，这个进程会获得一个独立的进程空间，它的 pid 是1，并且看不到宿主机上的其他进程，这也就是在容器内执行 PS 命令的结果。

下面我们通过运行代码来感受一下PID namespace的隔离效果。下面的代码，使用clone创建子进程，并且加入PID namespace的标识位CLONE\_NEWPID，并把程序命名为pid.c。

    ...
    #define _GNU_SOURCE
    #define STACK_SIZE (1024 * 1024)
    
    static char child_stack[STACK_SIZE];
    char* const child_args[] = {
      "/bin/bash",
      NULL
    };
    
    int child_main(void* args) {
      printf("在子进程中!\n");
      execv(child_args[0], child_args);
      return 1;
    }
    
    int main() {
      printf("程序开始: \n");
      int child_pid = clone(child_main, child_stack+STACK_SIZE,
                 CLONE_NEWPID | CLONE_NEWIPC | CLONE_NEWUTS 
                 | SIGCHLD, NULL);
      waitpid(child_pid, NULL, 0);
      printf("已退出\n");
      return 0;
    }


编译运行可以看到如下结果。

> root@local:~# gcc -Wall pid.c -o pid.o && ./pid.o


    程序开始:
    
    在子进程中!


此时的控制台的 命名空间，已经变了，变成了NewNamespace，并且可以输入命令。  
来一个简单的，输入 echo $$ 查看 shell id

    root@NewNamespace:~# echo $$
    
    1                      <<-- 注意此处看到 shell 的 PID 变成了 1


退出 子进程

    root@NewNamespace:~# exit
    
    exit
    
    已退出   <<-- 父进程的输出来了


退出之后，再一次打印 $$ 可以看到 shell 的 PID，退出后如果再次执行可以看到效果如下。

    root@local:~# echo $$
    
    17542             <<-- 已经回到了正常状态


已经回到了正常状态。

可能有的读者在子进程的shell中执行了ps aux/top之类的命令，发现还是可以看到所有父进程的PID，那是因为我们还没有对文件系统进行隔离，ps/top之类的命令调用的是真实系统下的/proc文件内容，看到的自然是所有的进程。

此外，与其他的namespace不同的是，为了实现一个稳定安全的容器，PID namespace还需要进行一些额外的工作才能确保其中的进程运行顺利。

#### 详解进程隔离

进程是 Linux 以及现在操作系统中非常重要的概念，它表示一个正在执行的程序，也是在现代分时系统中的一个任务单元。

在每一个 \*nix 的操作系统上，我们都能够通过 ps 命令打印出当前操作系统中正在执行的进程，比如在 Ubuntu 上，使用该命令就能得到以下的结果：

    |$ ps -ef
    UID		PID		PPID	C 	STIME 	TTY          TIME CMD
    root     1       0  	0   Apr08 	 ?      00:00:09 /sbin/init
    root     2       0  	0   Apr08 	 ?      00:00:00 [kthreadd]
    root     3       2  	0   Apr08	 ?      00:00:05 [ksoftirqd/0]
    root     5       2  	0   Apr08 	 ?      00:00:00 [kworker/0:0H]
    root     7       2  	0   Apr08 	 ?     	00:07:10 [rcu_sched]
    root    39       2  	0   Apr08 	 ?      00:00:00 [migration/0]
    root    40       2  	0   Apr08 	 ?      00:01:54 [watchdog/0]


​    

当前机器上有很多的进程正在执行，在上述进程中有两个非常特殊，一个是 pid 为 1 的 /sbin/init 进程，另一个是 pid 为 2 的 kthreadd 进程，这两个进程都是被 Linux 中的上帝进程 idle 创建出来的，其中前者负责执行内核的一部分初始化工作和系统配置，也会创建一些类似 getty 的注册进程，而后者负责管理和调度其他的内核进程。

![在这里插入图片描述](../images/docker/2019013011014888.png)

如果我们在当前的 Linux 操作系统下运行一个新的 Docker 容器，并通过 exec 进入其内部的 bash 并打印其中的全部进程，我们会得到以下的结果：

    UID        PID  PPID  C STIME TTY          TIME CMD
    root     29407     1  0 Nov16 ?        00:08:38 /usr/bin/dockerd --raw-logs
    root      1554 29407  0 Nov19 ?        00:03:28 docker-containerd -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --metrics-interval=0 --start-timeout 2m --state-dir /var/run/docker/libcontainerd/containerd --shim docker-containerd-shim --runtime docker-runc
    root      5006  1554  0 08:38 ?        00:00:00 docker-containerd-shim b809a2eb3630e64c581561b08ac46154878ff1c61c6519848b4a29d412215e79 /var/run/docker/libcontainerd/b809a2eb3630e64c581561b08ac46154878ff1c61c6519848b4a29d412215e79 docker-runc


​    

在新的容器内部执行 ps 命令打印出了非常干净的进程列表，只有包含当前 ps -ef 在内的三个进程，在宿主机器上的几十个进程都已经消失不见了。

当前的 Docker 容器成功将容器内的进程与宿主机器中的进程隔离，如果我们在宿主机器上打印当前的全部进程时，会得到下面三条与 Docker 相关的结果：

在当前的宿主机器上，可能就存在由上述的不同进程构成的进程树：![在这里插入图片描述](../images/docker/20190130110906474.png)

实际上，docker容器的pid隔离，就是在使用 clone(2) 创建新进程时传入 CLONE\_NEWPID 实现的，也就是使用 Linux 的命名空间实现进程的隔离，Docker 容器内部的任意进程都对宿主机器的进程一无所知。

    containerRouter.postContainersStart
    └── daemon.ContainerStart
    └── daemon.createSpec
        └── setNamespaces
            └── setNamespace


​    

Docker 的容器就是使用上述技术实现与宿主机器的进程隔离，当我们每次运行 docker run 或者 docker start 时，都会在下面的方法中创建一个用于设置进程间隔离的 Spec：

    func (daemon *Daemon) createSpec(c *container.Container) (*specs.Spec, error) {
    s := oci.DefaultSpec()
    
    // ...
    if err := setNamespaces(daemon, &s, c); err != nil {
        return nil, fmt.Errorf("linux spec namespaces: %v", err)
    }
    
    return &s, nil
    } 


​    

在 setNamespaces 方法中不仅会设置进程相关的命名空间，还会设置与用户、网络、IPC 以及 UTS 相关的命名空间：

    func setNamespaces(daemon *Daemon, s *specs.Spec, c *container.Container) error {
    // user
    // network
    // ipc
    // uts
    
    // pid
    if c.HostConfig.PidMode.IsContainer() {
        ns := specs.LinuxNamespace{Type: "pid"}
        pc, err := daemon.getPidContainer(c)
        if err != nil {
            return err
        }
        ns.Path = fmt.Sprintf("/proc/%d/ns/pid", pc.State.GetPID())
        setNamespace(s, ns)
    } else if c.HostConfig.PidMode.IsHost() {
        oci.RemoveNamespace(s, specs.LinuxNamespaceType("pid"))
    } else {
        ns := specs.LinuxNamespace{Type: "pid"}
        setNamespace(s, ns)
    }
    
    return nil
    } 


​    

所有命名空间相关的设置 Spec 最后都会作为 Create 函数的入参在创建新的容器时进行设置：

     daemon.containerd.Create(context.Background(), container.ID, spec, createOptions)


​    

所有与命名空间的相关的设置都是在上述的两个函数中完成的，Docker 通过命名空间成功完成了与宿主机进程和网络的隔。

#### PID namespace隔离非常实用

PID namespace隔离非常实用，它对进程PID重新标号，即两个不同namespace下的进程可以有同一个PID。  
每个PID namespace都有自己的计数程序。内核为所有的PID namespace维护了一个树状结构，最顶层的是系统初始时创建的，我们称之为root namespace。他创建的新PID namespace就称之为child namespace（树的子节点），而原先的PID namespace就是新创建的PID namespace的parent namespace（树的父节点）。  
通过这种方式，不同的PID namespaces会形成一个等级体系。所属的父节点可以看到子节点中的进程，并可以通过信号等方式对子节点中的进程产生影响。反过来，子节点不能看到父节点PID namespace中的任何内容。由此产生如下结论

-   每个PID namespace中的第一个进程“PID 1“，都会像传统Linux中的init进程一样拥有特权，起特殊作用。

-   一个namespace中的进程，不可能通过kill或ptrace影响父节点或者兄弟节点中的进程，因为其他节点的PID在这个namespace中没有任何意义。

-   如果你在新的PID namespace中重新挂载/proc文件系统，会发现其下只显示同属一个PID namespace中的其他进程。

-   在root namespace中可以看到所有的进程，并且递归包含所有子节点中的进程。

到这里，可能你已经联想到一种在外部监控Docker中运行程序的方法了，就是监控Docker Daemon所在的PID namespace下的所有进程即其子进程，再进行删选即可。

#### 其他的操作系统基础组件隔离

不仅仅是 PID，当你启动启动容器之后，Docker 会为这个容器创建一系列其他 namespaces。

这些 namespaces 提供了不同层面的隔离。容器的运行受到各个层面 namespace 的限制。

Docker Engine 使用了以下 Linux 的隔离技术:

The pid namespace: 管理 PID 命名空间 (PID: Process ID).

The net namespace: 管理网络命名空间(NET: Networking).

The ipc namespace: 管理进程间通信命名空间(IPC: InterProcess Communication).

The mnt namespace: 管理文件系统挂载点命名空间 (MNT: Mount).

The uts namespace: Unix 时间系统隔离. (UTS: Unix Timesharing System).

通过这些技术，运行时的容器得以看到一个和宿主机上其他容器隔离的环境。

### 网络隔离

如果 Docker 的容器通过 Linux 的命名空间完成了与宿主机进程的网络隔离，但是却有没有办法通过宿主机的网络与整个互联网相连，就会产生很多限制。

所以 Docker 虽然可以通过命名空间创建一个隔离的网络环境，但是 Docker 中的服务仍然需要与外界相连才能发挥作用。  
每一个使用 docker run 启动的容器其实都具有单独的网络命名空间，Docker 为我们提供了四种不同的网络模式，Host、Container、None 和 Bridge 模式。![在这里插入图片描述](../images/docker/20190130111155374.png)

在这一部分，我们将介绍 Docker 默认的网络设置模式：网桥模式。

在这种模式下，除了分配隔离的网络命名空间之外，Docker 还会为所有的容器设置 IP 地址。

当 Docker 服务器在主机上启动之后会创建新的虚拟网桥 docker0，随后在该主机上启动的全部服务在默认情况下都与该网桥相连。

![在这里插入图片描述](../images/docker/20190130111221850.png)

在默认情况下，每一个容器在创建时都会创建一对虚拟网卡，两个虚拟网卡组成了数据的通道，其中一个会放在创建的容器中，会加入到名为 docker0 网桥中。

我们可以使用如下的命令来查看当前网桥的接口：

    $ brctl show
    bridge name bridge id       STP enabled interfaces
    docker0     8000.0242a6654980   no      veth3e84d4f
     veth9953b75


​    

docker0会为每一个容器分配一个新的 IP 地址并将 docker0 的 IP 地址设置为默认的网关。

网桥 docker0 通过 iptables 中的配置与宿主机器上的网卡相连，所有符合条件的请求都会通过 iptables 转发到 docker0 并由网桥分发给对应的机器。

    $ iptables -t nat -L
    Chain PREROUTING (policy ACCEPT)
    target     prot opt source               destination
    DOCKER     all  --  anywhere             anywhere             ADDRTYPE match dst-type LOCAL
    
    Chain DOCKER (2 references)
    target     prot opt source               destination
    RETURN     all  --  anywhere             anywhere


​    

我们在当前的机器上使用 docker run -d -p 6379:6379 redis 命令启动了一个新的 Redis 容器，在这之后我们再查看当前 iptables 的 NAT 配置就会看到在 DOCKER 的链中出现了一条新的规则：

    DNAT       tcp  --  anywhere             anywhere             tcp dpt:6379 to:192.168.0.4:6379


​    

上述规则会将从任意源发送到当前机器 6379 端口的 TCP 包转发到 192.168.0.4:6379 所在的地址上。

这个地址其实也是 Docker 为 Redis 服务分配的 IP 地址，如果我们在当前机器上直接 ping 这个 IP 地址就会发现它是可以访问到的：

    $ ping 192.168.0.4
    PING 192.168.0.4 (192.168.0.4) 56(84) bytes of data.
    64 bytes from 192.168.0.4: icmp_seq=1 ttl=64 time=0.069 ms
    64 bytes from 192.168.0.4: icmp_seq=2 ttl=64 time=0.043 ms
    ^C
    --- 192.168.0.4 ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 999ms
    rtt min/avg/max/mdev = 0.043/0.056/0.069/0.013 ms


​    

从上述的一系列现象，我们就可以推测出 Docker 是如何将容器的内部的端口暴露出来并对数据包进行转发的了；当有 Docker 的容器需要将服务暴露给宿主机器，就会为容器分配一个 IP 地址，同时向 iptables 中追加一条新的规则。

![在这里插入图片描述](../images/docker/20190130111631392.png)  
当我们使用 redis-cli 在宿主机器的命令行中访问 127.0.0.1:6379 的地址时，经过 iptables 的 NAT PREROUTING 将 ip 地址定向到了 192.168.0.4，重定向过的数据包就可以通过 iptables 中的 FILTER 配置，最终在 NAT POSTROUTING 阶段将 ip 地址伪装成 127.0.0.1，到这里虽然从外面看起来我们请求的是 127.0.0.1:6379，但是实际上请求的已经是 Docker 容器暴露出的端口了。

    $ redis-cli -h 127.0.0.1 -p 6379 ping
    PONG


​    

Docker 通过 Linux 的命名空间实现了网络的隔离，又通过 iptables 进行数据包转发，让 Docker 容器能够优雅地为宿主机器或者其他容器提供服务。

### Libnetwork

整个网络部分的功能都是通过 Docker 拆分出来的 libnetwork 实现的，它提供了一个连接不同容器的实现，同时也能够为应用给出一个能够提供一致的编程接口和网络层抽象的容器网络模型。  
The goal of libnetwork is to deliver a robust Container Network Model that provides a consistent programming interface and the required network abstractions for applications.

libnetwork 中最重要的概念，容器网络模型由以下的几个主要组件组成，分别是 Sandbox、Endpoint 和 Network：![在这里插入图片描述](../images/docker/20190130111815551.png)  
在容器网络模型中，每一个容器内部都包含一个 Sandbox，其中存储着当前容器的网络栈配置，包括容器的接口、路由表和 DNS 设置，Linux 使用网络命名空间实现这个 Sandbox，每一个 Sandbox 中都可能会有一个或多个 Endpoint，在 Linux 上就是一个虚拟的网卡 veth，Sandbox 通过 Endpoint 加入到对应的网络中，这里的网络可能就是我们在上面提到的 Linux 网桥或者 VLAN。

**挂载点**  
虽然我们已经通过 Linux 的命名空间解决了进程和网络隔离的问题，在 Docker 进程中我们已经没有办法访问宿主机器上的其他进程并且限制了网络的访问，但是 Docker 容器中的进程仍然能够访问或者修改宿主机器上的其他目录，这是我们不希望看到的。

在新的进程中创建隔离的挂载点命名空间需要在 clone 函数中传入 CLONE\_NEWNS，这样子进程就能得到父进程挂载点的拷贝，如果不传入这个参数子进程对文件系统的读写都会同步回父进程以及整个主机的文件系统。

如果一个容器需要启动，那么它一定需要提供一个根文件系统（rootfs），容器需要使用这个文件系统来创建一个新的进程，所有二进制的执行都必须在这个根文件系统中。![在这里插入图片描述](../images/docker/20190130111910940.png)  
想要正常启动一个容器就需要在 rootfs 中挂载以上的几个特定的目录，除了上述的几个目录需要挂载之外我们还需要建立一些符号链接保证系统 IO 不会出现问题。

为了保证当前的容器进程没有办法访问宿主机器上其他目录，我们在这里还需要通过 libcotainer 提供的 pivor\_root 或者 chroot 函数改变进程能够访问个文件目录的根节点。

    // pivor_root
    put_old = mkdir(...);
    pivot_root(rootfs, put_old);
    chdir("/");
    unmount(put_old, MS_DETACH);
    rmdir(put_old);
    
    // chroot
    mount(rootfs, "/", NULL, MS_MOVE, NULL);
    chroot(".");
    chdir("/");


​    

到这里我们就将容器需要的目录挂载到了容器中，同时也禁止当前的容器进程访问宿主机器上的其他目录，保证了不同文件系统的隔离。

### **Chroot**

在这里不得不简单介绍一下 chroot（change root），在 Linux 系统中，系统默认的目录就都是以 / 也就是根目录开头的，chroot 的使用能够改变当前的系统根目录结构，通过改变当前系统的根目录，我们能够限制用户的权利，在新的根目录下并不能够访问旧系统根目录的结构个文件，也就建立了一个与原系统完全隔离的目录结构。

## CGroups物理资源限制分组

我们通过 Linux 的命名空间为新创建的进程隔离了文件系统、网络并与宿主机器之间的进程相互隔离，但是命名空间并不能够为我们提供物理资源上的隔离，比如 CPU 或者内存，如果在同一台机器上运行了多个对彼此以及宿主机器一无所知的『容器』，这些容器却共同占用了宿主机器的物理资源。  
![在这里插入图片描述](../images/docker/20190130112231930.png)

如果其中的某一个容器正在执行 CPU 密集型的任务，那么就会影响其他容器中任务的性能与执行效率，导致多个容器相互影响并且抢占资源。如何对多个容器的资源使用进行限制就成了解决进程虚拟资源隔离之后的主要问题，而 Control Groups（简称 CGroups）就是能够隔离宿主机器上的物理资源，例如 CPU、内存、磁盘 I/O 和网络带宽。

每一个 CGroup 都是一组被相同的标准和参数限制的进程，不同的 CGroup 之间是有层级关系的，也就是说它们之间可以从父类继承一些用于限制资源使用的标准和参数。  
Linux 的 CGroup 能够为一组进程分配资源，也就是我们在上面提到的 CPU、内存、网络带宽等资源，通过对资源的分配。  
Linux 使用文件系统来实现 CGroup，我们可以直接使用下面的命令查看当前的 CGroup 中有哪些子系统：

    $ lssubsys -m
    cpuset /sys/fs/cgroup/cpuset
    cpu /sys/fs/cgroup/cpu
    cpuacct /sys/fs/cgroup/cpuacct
    memory /sys/fs/cgroup/memory
    devices /sys/fs/cgroup/devices
    freezer /sys/fs/cgroup/freezer
    blkio /sys/fs/cgroup/blkio
    perf_event /sys/fs/cgroup/perf_event
    hugetlb /sys/fs/cgroup/hugetlb


​    

大多数 Linux 的发行版都有着非常相似的子系统，而之所以将上面的 cpuset、cpu 等东西称作子系统，是因为它们能够为对应的控制组分配资源并限制资源的使用。

如果我们想要创建一个新的 cgroup 只需要在想要分配或者限制资源的子系统下面创建一个新的文件夹，然后这个文件夹下就会自动出现很多的内容，如果你在 Linux 上安装了 Docker，你就会发现所有子系统的目录下都有一个名为 Docker 的文件夹：

    $ ls cpu
    cgroup.clone_children  
    ...
    cpu.stat  
    docker  
    notify_on_release 
    release_agent 
    tasks
    
    $ ls cpu/docker/
    9c3057f1291b53fd54a3d12023d2644efe6a7db6ddf330436ae73ac92d401cf1 
    cgroup.clone_children  
    ...
    cpu.stat  
    notify_on_release 
    release_agent 
    tasks


​    

9c3057xxx 其实就是我们运行的一个 Docker 容器，启动这个容器时，Docker 会为这个容器创建一个与容器标识符相同的 CGroup，在当前的主机上 CGroup 就会有以下的层级关系：![在这里插入图片描述](../images/docker/20190130112403516.png)  
每一个 CGroup 下面都有一个 tasks 文件，其中存储着属于当前控制组的所有进程的 pid，作为负责 cpu 的子系统，cpu.cfs\_quota\_us 文件中的内容能够对 CPU 的使用作出限制，如果当前文件的内容为 50000，那么当前控制组中的全部进程的 CPU 占用率不能超过 50%。

如果系统管理员想要控制 Docker 某个容器的资源使用率就可以在 docker 这个父控制组下面找到对应的子控制组并且改变它们对应文件的内容，当然我们也可以直接在程序运行时就使用参数，让 Docker 进程去改变相应文件中的内容。

当我们使用 Docker 关闭掉正在运行的容器时，Docker 的子控制组对应的文件夹也会被 Docker 进程移除，Docker 在使用 CGroup 时其实也只是做了一些创建文件夹改变文件内容的文件操作，不过 CGroup 的使用也确实解决了我们限制子容器资源占用的问题，系统管理员能够为多个容器合理的分配资源并且不会出现多个容器互相抢占资源的问题。

## linux namespace的API

接下来，介绍一下如何使用namespace的API，本文所讨论的namespace实现针对的均是Linux内核3.8及其以后的版本。

namespace的API包括clone()、setns()以及unshare()，还有/proc下的部分文件。为了确定隔离的到底是哪种namespace，在使用这些API时，通常需要指定以下六个常数的一个或多个，通过|（位或）操作来实现。你可能已经在上面的表格中注意到，这六个参数分别是CLONE\_NEWIPC、CLONE\_NEWNS、CLONE\_NEWNET、CLONE\_NEWPID、CLONE\_NEWUSER和CLONE\_NEWUTS。

### 通过clone()创建新进程的同时创建namespace

使用clone()来创建一个独立namespace的进程是最常见做法，它的调用方式如下。

    int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);


clone()实际上是传统UNIX系统调用fork()的一种更通用的实现方式，它可以通过flags来控制使用多少功能。一共有二十多种CLONE\_\*的flag（标志位）参数用来控制clone进程的方方面面（如是否与父进程共享虚拟内存等等），下面外面逐一讲解clone函数传入的参数。

-   参数child\_func传入子进程运行的程序主函数

-   参数child\_stack传入子进程使用的栈空间

-   参数flags表示使用哪些CLONE\_\*标志位

-   参数args则可用于传入用户参数

### 查看/proc/\[pid\]/ns文件

从3.8版本的内核开始，用户就可以在/proc/\[pid\]/ns文件下看到指向不同namespace号的文件，效果如下所示，形如\[4026531839\]者即为namespace号。

    $ ls -l /proc/$$/ns         <<-- 0="" 1="" 8="" $$="" 表示应用的pid="" total="" lrwxrwxrwx.="" mtk="" jan="" 04:12="" ipc="" -=""> ipc:[4026531839]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 mnt -> mnt:[4026531840]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 net -> net:[4026531956]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 pid -> pid:[4026531836]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 user->user:[4026531837]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 uts -> uts:[4026531838]


如果两个进程指向的namespace编号相同，就说明他们在同一个namespace下，否则则在不同namespace里面。/proc/\[pid\]/ns的另外一个作用是，一旦文件被打开，只要打开的文件描述符（fd）存在，那么就算PID所属的所有进程都已经结束，创建的namespace就会一直存在。那如何打开文件描述符呢？把/proc/\[pid\]/ns目录挂载起来就可以达到这个效果，命令如下。

    # touch ~/uts
    # mount --bind /proc/27514/ns/uts ~/uts


如果你看到的内容与本文所描述的不符，那么说明你使用的内核在3.8版本以前。该目录下存在的只有ipc、net和uts，并且以硬链接存在。

### 通过setns()加入一个已经存在的namespace

上文刚提到，在进程都结束的情况下，也可以通过挂载的形式把namespace保留下来，保留namespace的目的自然是为以后有进程加入做准备。通过setns()系统调用，你的进程从原先的namespace加入我们准备好的新namespace，使用方法如下。

    int setns(int fd, int nstype);


​    

-   参数fd表示我们要加入的namespace的文件描述符。上文已经提到，它是一个指向/proc/\[pid\]/ns目录的文件描述符，可以通过直接打开该目录下的链接或者打开一个挂载了该目录下链接的文件得到。

-   参数nstype让调用者可以去检查fd指向的namespace类型是否符合我们实际的要求。如果填0表示不检查。

为了把我们创建的namespace利用起来，我们需要引入execve()系列函数，这个函数可以执行用户命令，最常用的就是调用/bin/bash并接受参数，运行起一个shell，用法如下。

    fd = open(argv[1], O_RDONLY);   /* 获取namespace文件描述符 */
    setns(fd, 0);                   /* 加入新的namespace */
    execvp(argv[2], &argv[2]);      /* 执行程序 */


​    

假设编译后的程序名称为setns。

    # ./setns ~/uts /bin/bash   # ~/uts 是绑定的/proc/27514/ns/uts


​    

至此，你就可以在新的命名空间中执行shell命令了，在下文中会多次使用这种方式来演示隔离的效果。

### 通过unshare()在原先进程上进行namespace隔离

最后要提的系统调用是unshare()，它跟clone()很像，不同的是，unshare()运行在原先的进程上，不需要启动一个新进程，使用方法如下。

    int unshare(int flags);


​    

调用unshare()的主要作用就是不启动一个新进程就可以起到隔离的效果，相当于跳出原先的namespace进行操作。这样，你就可以在原进程进行一些需要隔离的操作。Linux中自带的unshare命令，就是通过unshare()系统调用实现的，有兴趣的读者可以在网上搜索一下这个命令的作用。

## 总之：dockers=LXC+AUFS

docker为LXC+AUFS组合：

-   LXC负责资源管理
-   AUFS负责镜像管理；

而LXC包括cgroup，namespace，chroot等组件，并通过cgroup资源管理，那么，从资源管理的角度来看，Docker，Lxc,Cgroup三者的关系是怎样的呢？

cgroup是在底层落实资源管理，LXC在cgroup上面封装了一层，随后，docker有在LXC封装了一层；

![img](../images/docker/1561918-20190504235645679-116967018.png)

**Cgroup其实就是linux提供的一种限制，记录，隔离进程组所使用的物理资源管理机制；也就是说，Cgroup是LXC为实现虚拟化所使用资源管理手段，我们可以这样说，底层没有cgroup支持，也就没有lxc，更别说docker的存在了，这是我们需要掌握和理解的，三者之间的关系概念**

 我们在把重心转移到LXC这个相当于中间件上，上述我们提到LXC是建立在cgroup基础上的，我们可以粗略的认为**LXC=Cgroup+namespace+Chroot+veth+用户控制脚本；LXC利用内核的新特性（cgroup）来提供用户空间的对象，用来保证资源的隔离和对应用系统资源的限制；**

 Docker容器的文件系统最早是建立在Aufs基础上的，Aufs是一种Union FS，简单来说就**是支持将不同的目录挂载到同一个虚拟文件系统之下**

**并实现一种laver的概念,**

由于Aufs未能加入到linux内核中，考虑到兼容性的问题，便加入了Devicemapper的支持，Docker目前默认是建立在Devicemapper基础上，

**devicemapper用户控件相关部分主要负责配置具体的策略和控制逻辑，**比如逻辑设备和哪些物理设备建立映射，怎么建立这些映射关系等，而具体过滤和重定向IO请求的工作有内核中相关代码完成，因此整个device mapper机制由两部分组成--内核空间的device mapper驱动，用户控件的device mapper库以及它提供的dmsetup工具；