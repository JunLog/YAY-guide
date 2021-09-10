# 进军docker之docker安装

## 什么是docker

> Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。
>
> 容器是完全使用沙箱机制，相互之间不会有任何接口（类似 iPhone 的 app）,更重要的是容器性能开销极低。

你可以理解`docker`封装后，就可以到处运行程序了

因为docker解决了应用环境的问题，安装了docker的平台就能跑“docker包”，这样就决绝了“开发环境能跑，一上线就崩”的尴尬。

## docker的应用场景

- Web 应用的自动化打包和发布。
- 自动化测试和持续集成、发布。
- 在服务型环境中部署和调整数据库或其他的后台应用。
- 从头编译或者扩展现有的 OpenShift 或 Cloud Foundry 平台来搭建自己的 PaaS 环境。

## Docker 的优点

Docker 是一个用于开发，交付和运行应用程序的开放平台。Docker 使您能够将应用程序与基础架构分开，从而可以快速交付软件。借助 Docker，您可以与管理应用程序相同的方式来管理基础架构。通过利用 Docker 的方法来快速交付，测试和部署代码，您可以大大减少编写代码和在生产环境中运行代码之间的延迟。

**1、快速，一致地交付您的应用程序**

Docker 允许开发人员使用您提供的应用程序或服务的本地容器在标准化环境中工作，从而简化了开发的生命周期。

容器非常适合持续集成和持续交付（CI / CD）工作流程，请考虑以下示例方案：

- 您的开发人员在本地编写代码，并使用 Docker 容器与同事共享他们的工作。
- 他们使用 Docker 将其应用程序推送到测试环境中，并执行自动或手动测试。
- 当开发人员发现错误时，他们可以在开发环境中对其进行修复，然后将其重新部署到测试环境中，以进行测试和验证。
- 测试完成后，将修补程序推送给生产环境，就像将更新的镜像推送到生产环境一样简单。

 **2、响应式部署和扩展**

Docker 是基于容器的平台，允许高度可移植的工作负载。Docker 容器可以在开发人员的本机上，数据中心的物理或虚拟机上，云服务上或混合环境中运行。

Docker 的可移植性和轻量级的特性，还可以使您轻松地完成动态管理的工作负担，并根据业务需求指示，实时扩展或拆除应用程序和服务。

 **3、在同一硬件上运行更多工作负载**

Docker 轻巧快速。它为基于虚拟机管理程序的虚拟机提供了可行、经济、高效的替代方案，因此您可以利用更多的计算能力来实现业务目标。Docker 非常适合于高密度环境以及中小型部署，而您可以用更少的资源做更多的事情。

## 安装docker

根据官方文档来安装！！

官方网站：https://www.docker.com/

**根据以下步骤完成即可**

![image-20210910210038267](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210910210038.png)

![image-20210910210051038](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210910210051.png)

选择安装的版本，因为我是服务器上面装，所以选择linux

![image-20210910210151477](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210910210151.png)

我选择centos（按照自己的服务器的系统选择）

![image-20210910210234264](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210910210234.png)

跑 命令。（在下面安装docker时候需要用到`yum-utils` 这个包）

```bash
sudo yum install -y yum-utils
```

不建议你跑

```bash
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

因为这个网站在国外，下面很慢，一般都是改`yum`源为阿里的（以下命令）

```bash
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/dockerce/linux/centos/docker-ce.repo
```

然后跑docker的安装命令

```bash
sudo yum install docker-ce docker-ce-cli containerd.io
```



![image-20210909214929185](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210909214929.png)

![image-20210909215034248](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210909215034.png)

> 在安装途中你可以看到一个key，如果匹配的上的话，那么这说明这个阿里的安装源没有取篡改redis，是官方的。

## 检测docker安装是否成功

方案一：

你可以看docker的版本

![image-20210909215144749](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210909215144.png)

方案二（**推荐**）：

官方的方案检测docker是否安装成功

跑命令

```bash
sudo docker run hello-world
```

![image-20210909215207477](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210909215207.png)

![image-20210909215255586](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210909215255.png)

以上如果你都对的上，那么恭喜你docker安装成功！！！🎉🎉🎉

## 配置镜像加速

**Docker 从 Docker Hub 拉取镜像，因为是从国外获取，所以速度较慢，**

为什么之前拉`hello-world` 很快？

> 因为他很小，即使在国外也可以，拉取，但是你去拉tomcat，nignx 都会比较慢，甚至不成功

Docker 官方和国内很多云服务商都提供了国内加速器服务，例如：

- 科大镜像：**https://docker.mirrors.ustc.edu.cn/**
- 网易：**https://hub-mirror.c.163.com/**
- 阿里云：**https://<你的ID>.mirror.aliyuncs.com**
- 七牛云加速器：**https://reg-mirror.qiniu.com**



第一步：

跑命令，创建文件

```bash
vi /etc/docker/daemon.json
```

第二步：

输入以下代码：

```bash
{ 
 "registry-mirrors": ["http://hub-mirror.c.163.com", "https://docker.mirrors.ustc.edu.cn"]
}
```

保存并退出

第三步：

重启文件或者docker

```bash
# 重新加载某个服务的配置文件 
sudo systemctl daemon-reload 
# 重新启动 docker
sudo systemctl restart docker
```

## 拉取hello-world的流程

通过运行 `hello-world`  镜像来验证` Docker Engine ` 是否已正确安装。

```bash
[root@localhost ~]# docker run hello-world 
Unable to find image 'hello-world:latest' locally # 本地找不到 hello-world 镜像 

latest: Pulling from library/hello-world # 拉取最新版本的 hello-world 镜像 
0e03bdcc26d7: Pull complete Digest: sha256:49a1c8800c94df04e9658809b006fd8a686cab8028d33cfba2cc049724254202 Status: Downloaded newer image for hello-world:latest
# 看到此消息表示您已正常安装。
Hello from Docker!
This message shows that your installation appears to be working correctly. 
To generate this message, Docker took the following steps: 
1. The Docker client contacted the Docker daemon. 
2. The Docker daemon pulled the "hello-world" image from the Docker Hub. (amd64)
3. The Docker daemon created a new container from that image which runs the executable that produces the output you are currently reading.
4. The Docker daemon streamed that output to the Docker client, which sent it to your terminal.
To try something more ambitious, you can run an Ubuntu container with: 
$ docker run -it ubuntu bash
Share images, automate workflows, and more with a free Docker ID: 
https://hub.docker.com/
For more examples and ideas, visit:
https://docs.docker.com/get-started/

```

 ![image-20210910213410699](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210910213410.png)

