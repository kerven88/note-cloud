# UDP网络模型

- kube: 1.17.2
- flannel: 0.11.0

Node A 172.16.91.10(Master)

Node B 172.16.91.14


将flannel部署到集群后, Node A 上的路由如下

```log
10.254.0.0/24 dev cni0 proto kernel scope link src 10.254.0.1
10.254.0.0/16 dev flannel0
```

> `cni0`为`bridge`设备, `flannel0`为`tun`设备.

`UDP`模型中每个宿主机上的路由只有2条, 并不像`hostgw`模型那样, 每加入/删除一个节点, 原本已经在在集群内部的节点, 就会将新节点划分的网段所属的路由进行新增/删除.

## 网络拓扑

```log
                            Node A 172.16.91.10                                                                   Node B 172.16.91.14
+----------------------------------------------------------------------------+    +----------------------------------------------------------------------------+
|  +--------------+ +--------------+                                         |    |                                         +--------------+ +--------------+  |
|  |10.254.0.11/24| |10.254.0.12/24|                                         |    |                                         |10.254.1.11/24| |10.254.1.12/24|  |
|  |     Pod A    | |     Pod B    |                                         |    |                                         |     Pod C    | |     Pod D    |  |
|  +-------↓------+ +------↓-------+                                         |    |                                         +-------↑------+ +------↑-------+  |
|          |               |                                                 |    |                                                 |               |          |
|          └-------┬-------┘                                                 |    |                                                 └-------┬-------┘          |
|                  |                 +-------------------------------+       |    |       +-------------------------------+                 |                  |
|                  |                 |       flannel controller      |       |    |       |       flannel controller      |                 |                  |
|                  | 1               +-------↑---------------↓-------+       |    |       +-------↑---------------↓-------+                 | 11               |
|                  |                         |               | 4             |    |               | 8             |                         |                  |
|..................|.........................↑...............|...............|    |...............|...............↓.........................|..................|
|                  |                         |               |               |    |               |               |                         |                  |
|                  |                         |        +------↓-----+         |    |        +------↑-----+         |                         |                  |
|                  |                       3 |        | UDP socket |         |    |        | UDP socket |         | 9                       |                  |
|                  |                         |        +------↓-----+         |    |        +------↑-----+         |                         |                  |
|                  |                         |               |               |    |               |               |                         |                  |
|..................|.........................↑...............|...............|    |...............|...............↓.........................|..................|
|                  |                         |               |               |    |               |               |                         |                  |
|                  |                    /dev/net/tun         |               |    |               |          /dev/net/tun                   |                  |
|                  |                     +-------------------↓----+          |    |          +------------------------+                     |                  |
|                  |                     | Newwork Protocol Stack |          |    |          | Newwork Protocol Stack |                     |                  |
|                  |                     +----↑--------------↓----+          |    |          +----↑--------------↓----+                     |                  |
|                  |                          |              | 5             |    |             7 |              |                          |                  |
|..................|..........................|..............|...............|    |...............|..............|..........................|..................|
|                  |                          |              |               |    |               |              |                          |                  |
|         +--------↓--------+    +------------↑----+    +----↓------------+  |    |  +------------↑----+    +----↓------------+    +--------↑--------+         |
|         |  10.254.0.1/24  | 2  |  10.254.0.0/32  |    | 172.16.91.10/24 |  |    |  | 172.16.91.14/24 |    |  10.254.1.0/32  | 10 |  10.254.0.1/24  |         |
|         |      cni0       | -> |    flannel0     |    |      eth0       |  |    |  |      eth0       |    |    flannel0     | -> |      cni0       |         |
|         +-----------------+    +-----------------+    +--------↓--------+  |    |  +--------↑--------+    +-----------------+    +-----------------+         |
+----------------------------------------------------------------|-----------+    +-----------|----------------------------------------------------------------+
                                                                 |              6             |                                                                 
                                                                 |    +------------------+    |                                                                 
                                                                 |    |                  |    |                                                                 
                                                                 └───>|  172.16.91.1/24  |────┘                                                                 
                                                                      |     Gateway      |                                                                      
                                                                      +------------------+                                                                      
```

## 数据包流向

假设在Pod A(`10.254.0.11`)内部访问Pod C(`10.254.1.11`), 则数据包所经如下

1. 数据包通过Pod A内部的`eth0`网卡与宿主机网桥设备`cni0`连接的veth pair, 进入宿主机;
2. 目标地址`10.254.1.11`匹配上面的第2条路由, 会由内核网络层(是内核还是协议栈???)转交给`flannel0`;
3. `flannel0`的另一头连接到了`/dev/net/tun`, 由内核协议栈进行读取;
    - `flannel0`是一个`tun`设备, 由`flannel controller`在启动时创建, 并且连接到`/dev/net/tun`字符设备, 并开始监听(监听的地址是宿主机IP, 如`172.16.91.10`, `172.16.91.14`).
4. 内核协议栈从数据包的header中(按IP数据包的结构体进行解析)读取到目标地址为`10.254.1.11`, 然后根据路由, 得到目标Pod所属的主机IP`172.16.91.14`, 使用UDP socket将数据包原样发送出去;
    - `UDP`模型中每个宿主机上的路由只有2条, 并不像`hostgw`模型那样, 每加入/删除一个节点, 原本已经在在集群内部的节点, 就会将新节点划分的网段所属的路由进行新增/删除.
    - 但是`UDP`模型仍然会监听节点的加入/删除, 由`flannel controller`自己维护主机和ta划分的网段.
5. UDP socket发出的数据包, 由宿主机按照本机路由, 通过物理网卡`eth0`流出;
6. 由物理网络中的交换机/路由器将数据包转发到目标主机;
7. 数据包进入Node B`172.16.91.14`的物理网卡, 由该机器上的UDP socket进行处理;
8. Node B上的内核协议栈从UDP socket读取到的数据包, 直接写入到`/dev/net/tun`文件中;
9. 数据包写入到`/dev/net/tun`, 就会流入`flannel0`, 而所有写入`flannel0`网卡的数据, 都会根据该主机上的路由再次转发; 
10. `flannel0`将数据包再次路由, 将目标为`10.254.1.11`的数据包交由`cni0`处理; 
    - 注意: 此时写入`flannel0`的数据包已不再拥有`172.16.91.14`那种UDP socket包, 而是像第1步那种纯粹的3层IP包.
11. `cni0`会将数据包直接发送给Pod C.

至于响应包, 按照上面的流程反过来走一遍就可以了.

------

## 为什么用UDP而不是TCP

所以为啥要用UDP而不是TCP, 这下有点理解了吧? 

因为UDP不分服务端/客户端, 如果要用TCP, 一台主机要把数据包发送到其他主机, 就需要与其他所有主机都建立起TCP连接, 当集群中有100个节点时, 每台主机就要维护1个服务端和99个socket客户端...

## 为什么`flannel0`上的地址掩码为32

一个网络设备(如veth, tun)如果拥有IP地址, 然后通过`ip link set xxx up`启动, 主机上就会拥有相应地址的路由.

如: veth01 `192.168.0.2/24`, 启动后就会拥有

```
192.168.0.0/24 dev veth01 proto kernel scope link src 192.168.0.2
```

但是`flannel0`上的地址是`10.254.0.0/32`, 一方面IP是网络号, 根本不可用, 另一方面掩码是32, 这样启动起来是不会有路由生成的...`10.254.0.0/16 dev flannel0`这条是`flannel controller`在代码里生成的.

不过为啥要用32呢? 把`flannel0`地址赋值成`10.254.0.0/16`, 不行吗???

不过如果是这样的样, 得到的路由将是这样的

```
10.254.0.0/16 dev flannel0 proto kernel scope link src 10.254.0.0
```

那看来是因为`10.254.0.0/16 dev flannel0`这上面的那个有区别

关键是, 有啥区别呢???
