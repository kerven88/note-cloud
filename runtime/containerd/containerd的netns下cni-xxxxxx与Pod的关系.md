# containerd的netns下cni-xxxxxx与Pod的关系

与docker不同, kube在使用containerd为runtime时, 使用`ip netns ls`可以查看到当前节点上各容器使用的网络空间.

```log
$ ip netns ls
cni-ee061e8c-5a3b-5826-01e4-2c295839cd1d (id: 6)
cni-079566fd-4d42-2128-04b9-7acd0abe01c1 (id: 5)
cni-7ef37cd4-6c61-eb35-1879-d4fa5b7f5d0f (id: 4)
cni-cde9706d-299f-283f-d002-d1c46e8d7afc (id: 3)
cni-debe2fce-4dde-ea85-0bf0-1e5234292f87 (id: 2)
cni-01b2994c-2c35-1a66-c3bd-77be5773ac25 (id: 1)
cni-f327421a-20a1-7aac-1c3b-a679362e8c6c (id: 0)
```

对应到`/var/run/netns`目录.

```log
$ ls /var/run/netns -al
total 0
drwxr-xr-x  2 root root 180 Jun 26 16:35 .
drwxr-xr-x 25 root root 820 Jun 25 14:04 ..
-r--r--r--  1 root root   0 Jun 25 14:04 cni-01b2994c-2c35-1a66-c3bd-77be5773ac25
-r--r--r--  1 root root   0 Jun 25 20:20 cni-079566fd-4d42-2128-04b9-7acd0abe01c1
-r--r--r--  1 root root   0 Jun 25 14:04 cni-7ef37cd4-6c61-eb35-1879-d4fa5b7f5d0f
-r--r--r--  1 root root   0 Jun 25 14:04 cni-cde9706d-299f-283f-d002-d1c46e8d7afc
-r--r--r--  1 root root   0 Jun 25 14:04 cni-debe2fce-4dde-ea85-0bf0-1e5234292f87
-r--r--r--  1 root root   0 Jun 26 16:35 cni-ee061e8c-5a3b-5826-01e4-2c295839cd1d
-r--r--r--  1 root root   0 Jun 25 14:04 cni-f327421a-20a1-7aac-1c3b-a679362e8c6c
```

> 注意, 这些命名空间都是属于非hostNetwork类型的容器.

cni-xxx后面的id部分貌似是随机的, 无法追溯ta们各自属于哪个Pod.

可以进到这些pod里, 查看ta们的IP, 查询使用该IP的Pod即可.

```log
$ ip netns exec cni-ee061e8c-5a3b-5826-01e4-2c295839cd1d ip a show eth0
3: eth0@if1087: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether de:22:ff:6d:6c:9c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.29.86/32 scope global eth0
       valid_lft forever preferred_lft forever
```

或者

```log
$ nsenter --net=/var/run/netns/cni-ee061e8c-5a3b-5826-01e4-2c295839cd1d ip a show eth0
3: eth0@if1087: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether de:22:ff:6d:6c:9c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.29.86/32 scope global eth0
       valid_lft forever preferred_lft forever
```
