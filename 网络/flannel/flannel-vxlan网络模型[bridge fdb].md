A: 172.12.0.21
B: 172.12.0.22

```
┌------------------------------------------------┐
|         10.254.1.2/24     10.254.1.3/24        |
|          ┌--------┐        ┌--------┐          |
|          |  eht0  |        |  eht0  |          |
|          └----┬---┘        └----┬---┘          |
|               └────────┬────────┘              |
|                        |                       |
|                  ┌-----┴-----┐ 10.254.1.1/24   |
|                  |    cni0   |   (bridge)      |
|                  └-----------┘                 |
|                                                |
|                  ┌-----------┐ 10.254.1.0/32   |
|                  | flannel.1 |    (vxlan)      |
|                  └-----------┘                 |
|                                                |
|                    ┌--------┐                  |
|                    |  eth0  |  172.12.0.21/24  |
|                    └----┬---┘                  |
|                         |                      |
└-------------------------|----------------------┘
                          |
                          |     +----------------+      |
                          └────>|  172.12.0.1/24 |<─────┘
                                +----------------+
                                    网关/路由器
```

`cni0`是bridge设备, 同主机的所有Pod都通过 veth pair 接入该 bridge, **但是`flannel.1`并未接入**.

在 Pod01 中 ping Pod11, 然后在 eth0 上用 tcpdump 抓 icmp 包是没有结果的, 因为 ping 被封装成了 udp 包, 如下

```log
[root@kube-master-01 ~]# tcpdump -nnn -i eth0 udp
20:03:24.786321 IP 172.12.0.21.47795 > 172.12.0.22.8472: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.254.0.3 > 10.254.2.6: ICMP echo request, id 24, seq 1, length 64
20:03:24.786368 IP 172.12.0.22.47795 > 172.12.0.21.8472: OTV, flags [I] (0x08), overlay 0, instance 1
IP 10.254.2.6 > 10.254.0.3: ICMP echo reply, id 24, seq 1, length 64
```

Pod01 的数据包到了`flannel.1`, 就会被封装成 udp 包, 发送到 vm2 的8472端口(172.12.0.22:8472), 而监听该端口的正是 vm2 的`flannel.1`设备, 解包之后抛给内核协议栈, 按照路由流入 Pod11. 回包也是相同的流程.

###

```log
$ ip -d link show flannel.1
6: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/ether aa:a3:01:5b:23:6a brd ff:ff:ff:ff:ff:ff promiscuity 0 
    vxlan id 1 local 172.12.0.21 dev eth0 srcport 0 0 dstport 8472 nolearning ageing 300 noudpcsum noudp6zerocsumtx noudp6zerocsumrx addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
```

- `dstport 8472`表示vxlan数据包通过本机的8472 udp端口转发出去;
- `group default`多播组, default默认为`239.1.1.1`;

```log
[root@kube-worker-01 /]# netstat -anopu
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address  State  PID/Program name  Timer
udp        0      0 0.0.0.0:8472            0.0.0.0:*               -                 off (0.00/0/0)
```

### vm1(172.12.0.21)

#### 路由

```log
$ ip r
default via 172.12.0.1 dev eth0 
10.254.0.0/24 via 10.254.0.0 dev flannel.1 onlink 
10.254.1.0/24 dev cni0 proto kernel scope link src 10.254.1.1 
10.254.2.0/24 via 10.254.2.0 dev flannel.1 onlink 
172.12.0.0/24 dev eth0 proto kernel scope link src 172.12.0.21 
```

跨主机通信时, 本机数据包需要经过 flannel.1 这个 vxlan 设备转发出去.

#### 转发表

```log
$ bridge fdb show dev flannel.1
e2:34:8b:db:db:95 dev flannel.1 dst 172.12.0.22 self permanent
62:ae:83:d5:62:ea dev flannel.1 dst 172.12.0.11 self permanent
```

`e2:34:8b:db:db:95`是vm2上`flannel.1`接口的mac地址.

### vm2(172.12.0.22)

#### 路由

```log
$ ip r
default via 172.12.0.1 dev eth0 
10.254.0.0/24 via 10.254.0.0 dev flannel.1 onlink 
10.254.1.0/24 via 10.254.1.0 dev flannel.1 onlink 
10.254.2.0/24 dev cni0 proto kernel scope link src 10.254.2.1 
172.12.0.0/24 dev eth0 proto kernel scope link src 172.12.0.22 
```

#### 转发表

```log
$ bridge fdb show dev flannel.1
62:ae:83:d5:62:ea dev flannel.1 dst 172.12.0.11 self permanent
aa:a3:01:5b:23:6a dev flannel.1 dst 172.12.0.21 self permanent
```

`aa:a3:01:5b:23:6a`是vm2上`flannel.1`接口的mac地址.
