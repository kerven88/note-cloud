# flannel-hostgw网络模型

参考文章

1. [深入浅出Kubernetes网络：跨节点网络通信之Flannel](https://cloud.tencent.com/developer/article/1450296)
    - `host-gw`模式`flannel`的唯一作用就是负责主机上路由表的动态更新, 不过缺陷是需要宿主机处于同一子网, 否则路由无法直达. 
2. [Kubernetes学习之路（二十一）之网络模型和网络策略](https://www.cnblogs.com/linuxk/p/10517055.html)
    - host-gw: 要求各节点必须在同一个2层网络, 对报文转发性能要求较高的场景使用.

host-gw真的是十分简单的网络模型, 简单到可以直接使用docker原生网络, 加上几条宿主机路由就可以实现.

## 环境搭建

- vm1: `172.16.91.128/24`, docker: `172.17.1.1/24`
- vm2: `172.16.91.129/24`, docker: `172.17.2.1/24`

我们需要修改`docker`桥接网络, 使用`/etc/docker/daemon.json`中的`"bip": "172.17.1.1/24"`, 然后重启docker服务即可. 

> `bip`参数必须要是`docker0`的IP地址, docker服务会自动解析容器网络为`172.17.1.0/24`.

此时就是典型的`docker`网络, 只不过网段不是默认的`172.17.0.0/16`而已, 跨主机容器间无法相互通信.

> 我们借鉴了`flannel`的做法, 为不同宿主机节点上的容器划分掩码位为24的小子网, 实际所有容器都存在于一个掩码位为16大子网中.

在两台虚拟机上启动容器

```bash
docker run -it --name test generals/centos7 /bin/bash
```

vm1上的容器获得IP `172.17.1.2`, vm2上的容器获得IP `172.17.2.2`.

以vm1为例, 默认路由如下

```log
$ ip r 
default via 172.16.91.2 dev ens33 proto static metric 100
172.16.91.0/24 dev ens33 proto kernel scope link src 172.16.91.128 metric 100
172.17.1.0/24 dev docker0 proto kernel scope link src 172.17.1.1
```

网络拓扑如下

```log
+-------------------------------------+          +-------------------------------------+
|  172.17.1.2/24     172.17.1.x/24    |          |    172.17.2.2/24     172.17.2.x/24  |
|   +--------+        +--------+      |          |     +--------+        +--------+    |
|   |  eht0  |        |  eht0  |      |          |     |  eht0  |        |  eht0  |    |
|   +----┬---+        +----┬---+      |          |     +----┬---+        +----┬---+    |
|        └────────┬────────┘          |          |          └────────┬────────┘        |
|           +-----┴-----+             |          |             +-----┴-----+           |
|           |  docker0  |             |          |             |  docker0  |           |
|           +-----┬-----+             |          |             +-----┬-----+           |
|  172.17.1.1/24  |     +--------+    |          |    +--------+     |  172.17.2.1/24  |
|                 └────>|  ens33 |    |          |    | ens33  |<────┘                 |
|                       +----┬---+    |          |    +----┬---+                       |
|          172.16.91.128/24  |        |          |         |  172.16.91.129/24         |
+----------------------------|--------+          +---------|---------------------------+
                             |     +----------------+      |                             
                             └────>| 172.16.91.1/24 |<─────┘                             
                                   +----------------+
                                       网关/路由器
```

## 部署实验网络

以下所有操作都在宿主机上执行, 无需在容器内操作.

### step 1

`vm1`宿主机上执行

```log
ip r add 172.17.2.0/24 dev ens33 via 172.16.91.129
```

`vm2`宿主机上执行

```log
ip r add 172.17.1.0/24 dev ens33 via 172.16.91.128
```

> `ip route`中的`via`选项等同于`route`命令中的`gateway`选项.

以vm1上的命令来说, 是把来自`docker0`的, 目标是`172.17.2.0/24`的请求, 通过物理网卡`ens33`发出, 且下一跳(即传统意义上的"网关")为`172.16.91.129`. 

### step 2

但此请求还不能顺利抵达目标容器, 只添加路由是不够的. 由于我们需要**把宿主机节点作为路由器**, 把请求转发到容器网络, 还需要开启`ip_forward`.

以下命令在`vm1`和`vm2`中都要执行.

```log
sysctl -w net.ipv4.ip_forward=1
```

### step 3

另外, 可能还需要`iptables`做一些事情. 由于访问目标是容器, 不在宿主机自身, 所以数据包到达宿主机后, 会进入`filter`表的`FORWARD`链, 需要查看该链的默认规则是否为`ACCEPT`.

```log
$ iptables -nvL | grep FORWARD
Chain FORWARD (policy DROP 0 packets, 0 bytes)
```

如果为`DROP/REJECT`, 还需要执行如下命令.

```log
iptables -P FORWARD ACCEPT
```

上面的命令是全部放行, 也许不太合适, 可以单独对容器大子网开放.

```log
iptables -A FORWARD -s 172.17.0.0/16 -j ACCEPT
iptables -A FORWARD -d 172.17.0.0/16 -j ACCEPT
```

------

此时, 不同宿主机的容器与容器之间, 容器与宿主机之间都可以相互通信了.

在`flannel`的`host-gw`模型中, 需要监听node的节点的增删操作, 每增加一个node, 就需要为其划分一个小子网`172.17.x.0/24`, 并在现有node上添加一条与上面类似的路由, 这样才能实现各节点间容器能够相互通信.
