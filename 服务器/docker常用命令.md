# docker常用命令

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

