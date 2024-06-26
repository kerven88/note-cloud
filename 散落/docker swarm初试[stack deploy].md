参考文章

1. [Where is the docker swarm token stored?](https://stackoverflow.com/questions/33035303/where-is-the-docker-swarm-token-stored)

一个拥有docker服务的节点可以使用`docker swarm init`将自己初始化成`manager`节点, 也可以通过`docker swarm join`加入一个已经存在的集群.

`docker swarm join-token [manager|worker]`可以查看如何向一个已存在的集群中添加`manager`节点和`worker`节点.

`docker swarm join`组成集群后可以通过`docker node`进行管理.

拥有`manager`和`worker`节点的集群可以使用`docker stack deploy`部署docker-compose那一套东西, docker swarm自动完成负载均衡和高可用的工作.

```
$ docker stack deploy -c ./swarm.yml celery
Creating network celery_default
Creating service celery_celery-worker
```

`swarm.yml`的内容如下

```yaml
version: '3'

services:
  celery-worker:
    image: generals-space/wuhou-spider
    deploy:
      mode: replicated
      replicas: 4
```

> `celery`只是个名称, 可以随便取, 相当于命名空间.

**注意**

1. `worker`节点上不能执行`docker stack deploy`, 只能在`manager`节点上执行.
2. `manager`节点上不会部署容器, 通过`deploy`部署的容器只能分发到各`worker`节点.

之后可以通过`docker stack`查看和管理运行的容器(只能在`manager`节点上管理).

## 查看当前集群中的stack(命名空间)

```log
$ docker stack ls
NAME                SERVICES            ORCHESTRATOR
celery              1                   Swarm
```

## 查看名为celery的stack中运行的任务

```log
$ docker stack services celery
ID                  NAME                   MODE                REPLICAS            IMAGE                                                                  PORTS
ibzem7b6r32c        celery_celery-worker   replicated          4/4                 registry.cn-hangzhou.aliyuncs.com/generals-space/wuhou-spider:latest   
```

## 查看名为celery的stack中的容器在各worker节点上的运行状态(本例中只有一个worker节点, 复制了4份)

```log
$ docker stack ps celery
ID                  NAME                         IMAGE                                NODE                    DESIRED STATE       CURRENT STATE             PORTS
v1xqjnuop3g3        celery_celery-worker.1       generals-space/wuhou-spider:latest   linuxkit-00155de8010d   Running             Running 11 minutes ago
g5upnfph0962        celery_celery-worker.2       generals-space/wuhou-spider:latest   linuxkit-00155de8010d   Running             Running 11 minutes ago
timyjzt3ufr3        celery_celery-worker.3       generals-space/wuhou-spider:latest   linuxkit-00155de8010d   Running             Running 11 minutes ago
rpejuxy81iek        celery_celery-worker.4       generals-space/wuhou-spider:latest   linuxkit-00155de8010d   Running             Running 11 minutes ago
```

## 移除stack, 各节点上的容器也会被停止

```log
$ docker stack rm celery
Removing service celery_celery-worker
Removing network celery_default
```

------

**关于缩容与扩容**, 如果你觉得4份太多或太少, 可以直接修改`replicas`的值, 然后再次运行`docker stack deploy -c ./swarm.yml celery`, Docker将会做`in-place`替换, 不用先停服务, 或者kill容器.
