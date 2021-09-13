#docker常用命令

官方文档：https://docs.docker.com/reference/

## 镜像相关命令

### 查看镜像

```bash
[root@iZf8z9gs5xht6267o9en7mZ ~]# docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
nginx         latest    ad4c705f24d3   43 hours ago   133MB
hello-world   latest    d1165f221234   6 months ago   13.3kB

```

* `REPOSITORY`:镜像在仓库中的名称
* `TAG`:镜像标签 一般指版本
* `IMAGE ID` ：镜像ID
* `CREATED`：镜像创建日期
* `SIZE` ：镜像大小

这些镜像都是存储在 Docker 宿主机的 `/var/lib/docker `目录下。

### 搜索镜像

```bash
[root@iZf8z9gs5xht6267o9en7mZ ~]# docker search redis
NAME                             DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
redis                            Redis is an open source key-value store that…   9907      [OK]       
sameersbn/redis                                                                  83                   [OK]
grokzen/redis-cluster            Redis cluster 3.0, 3.2, 4.0, 5.0, 6.0, 6.2      79                   
rediscommander/redis-commander   Alpine image for redis-commander - Redis man…   65                   [OK]
redislabs/redisearch             Redis With the RedisSearch module pre-loaded…   38                   
redislabs/redisinsight           RedisInsight - The GUI for Redis                35                   
redislabs/redis                  Clustered in-memory database engine compatib…   31                   
oliver006/redis_exporter          Prometheus Exporter for Redis Metrics. Supp…   29                   
redislabs/rejson                 RedisJSON - Enhanced JSON data type processi…   27                   
arm32v7/redis                    Redis is an open source key-value store that…   24                   
redislabs/redisgraph             A graph database module for Redis               16                   [OK]
arm64v8/redis                    Redis is an open source key-value store that…   15                   
redislabs/redismod               An automated build of redismod - latest Redi…   15                   [OK]
redislabs/rebloom                A probablistic datatypes module for Redis       14                   [OK]
webhippie/redis                  Docker images for Redis                         11                   [OK]
redislabs/redistimeseries        A time series database module for Redis         10                   
s7anley/redis-sentinel-docker    Redis Sentinel                                  10                   [OK]
insready/redis-stat              Docker image for the real-time Redis monitor…   10                   [OK]
goodsmileduck/redis-cli          redis-cli on alpine                             9                    [OK]
centos/redis-32-centos7          Redis in-memory data structure store, used a…   5                    
clearlinux/redis                 Redis key-value data structure server with t…   3                    
wodby/redis                      Redis container image with orchestration        1                    [OK]
tiredofit/redis                  Redis Server w/ Zabbix monitoring and S6 Ove…   1                    [OK]
runnable/redis-stunnel           stunnel to redis provided by linking contain…   1                    [OK]
xetamus/redis-resource           forked redis-resource                           0                    [OK]

```

* `NAME` :镜像名称
* `DESCRIPTION`：镜像描述
* `STARS` :用户评价  （类似github的stars）
* `OFFICIAL ` ：是否为官方构建
* `AUTOMATED`：自动构建

### 拉取镜像

```bash
docker pull 镜像名称[:tag]
```

如果不声明tag镜像标签，则默认拉取latest版本



### 删除镜像

```bash
docker rmi 镜像ID或者镜像名称
```

> 注意：如果通过某个镜像创建了容器，则该镜像无法删除。
>
>  解决办法：先删除镜像中的容器，再删除该镜像。
>
> ```bash
> docker rmi `docker images -q`
> ```

## 容器相关命令

### 查看容器

```bash
[root@iZf8z9gs5xht6267o9en7mZ ~]# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

* `CONTAINER ID ` :容器ID
* `IMAGE` ：所属镜像
* `COMMAND` ：启动容器时运行的命令。
* `CREATED`：容器创建时间
* `STATUS`：容器状态
  * created(已创建)
  * restarting(重启中)
  * running(运行中)
  * removing(迁移中)
  * paused(暂停)
  * exited(停止)
  * dead(死亡)
* `PORTS`：容器的端口信息和使用的连接类型(tcp\udp)。
* `NAMES`：自动分配的容器名称。

### 创建与启动容器

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

* `-i`:表示运行容器
* `-t`：表示容器启动后进入其命令行。加入`-it` 容器创建就会登录进入。分配一个伪终端
* `–-name` ：为创建的容器命名
* `-v`：表示目录映射关系（前者是宿主机目录，后者是映射到宿主机上的目录），可以使用多个 -v 做多个目录或文件映射。注意：最好做目录映射，在宿主机上做修改，然后共享到容器上；
* `-d`：在 run 后面加上 -d 参数，则会创建一个守护式容器在后台运行（这样创建容器后不会自动登 录容器，如果只加 -i -t 两个参数，创建容器后就会自动进容器里）；
* `-p`：表示端口映射，前者是宿主机端口，后者是容器内的映射端口。可以使用多个 -p 做多个端口 映射。（小写的p）
* `-P`：随机使用宿主机的可用端口与容器内暴露的端口映射。（大写的P）

### 创建并进入容器

```bash
docker run -it --name 容器名称 镜像名称:标签 /bin/bash
```

### 守护式方式创建容器

```bash
docker run -di --name 容器名称 镜像名称:标签
```

就是将容器永远在后台运行



### 登录守护式容器方式

```bash
docker exec -it 容器名称|容器ID /bin/bash
```

### 停止与启动容器

```bash 
# 停止容器 
docker stop 容器名称|容器ID 
# 启动容器
docker start 容器名称|容器ID
```

### 文件拷贝

宿主机-》容器

```bash
docker cp 需要拷贝的文件或目录 容器名称:容器目录
```

容器-》宿主机

```bash
docker cp 容器名称:容器目录 需要拷贝的文件或目录
```

### 目录挂载

创建容器添加 `-v` 参数，格式为`宿主机目录:容器目录`，例如：

```bash
docker run -di -v /mydata/docker_centos/data:/usr/local/data --name centos7-01 centos:7 
# 多目录挂载
docker run -di -v /宿主机目录:/容器目录 -v /宿主机目录2:/容器目录2 镜像名
```

> 目录挂载操作可能会出现权限不足的提示。这是因为 CentOS7 中的安全模块 SELinux 把权限禁 掉了，在 docker run 时通过privileged=true 给该容器加权限来解决挂载的目录没有权限的问题。

### 匿名挂载

匿名挂载只需要写容器目录即可，容器外对应的目录会在 `/var/lib/docker/volumes` 中生成。

```bash
# 匿名挂载 
docker run -di -v /usr/local/data --name centos7-02 centos:7 
# 查看 volume 数据卷信息
docker volume ls
```

### 具名挂载

具名挂载就是给数据卷起了个名字，容器外对应的目录会在 `/var/lib/docker/volume` 中生成。

```bash
# 匿名挂载 
docker run -di -v docker_centos_data:/usr/local/data --name centos7-03 centos:7 
# 查看 volume 数据卷信息
docker volume ls
```

### 查看目录挂载关系

通过 `docker volume inspect 数据卷名称` 可以查看该数据卷对应宿主机的目录地址。

通过 `docker inspect 容器ID或名称 `，在返回的 JSON 节点中找到 `Mounts`，可以查看详细的数 据挂载信息

### 只读/只写

```bash
# 只读。只能通过修改宿主机内容实现对容器的数据管理。 
docker run -it -v /宿主机目录:/容器目录:ro 镜像名 
# 读写，默认。宿主机和容器可以双向操作数据。
docker run -it -v /宿主机目录:/容器目录:rw 镜像名
```

### 继承

```bash
# 容器 centos7-01 指定目录挂载 
docker run -di -v /mydata/docker_centos/data:/usr/local/data --name centos7-01 centos:7 
# 容器 centos7-04 和 centos7-05 相当于继承 centos7-01 容器的挂载目录 
docker run -di --volumes-from centos7-01:ro --name centos7-04 centos:7
docker run -di --volumes-from centos7-01:rw --name centos7-05 centos:7
```

### 查看容器IP地址

```bash
docker inspect 容器名称|容器ID
docker inspect --format='{{.NetworkSettings.IPAddress}}' 容器名称|容器ID
```

### 删除容器

```bash
# 删除指定容器 
docker rm 容器名称|容器ID 
# 删除多个容器
docker rm 容器名称|容器ID 容器名称|容器ID
```

