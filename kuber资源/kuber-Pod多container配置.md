# kuber-Pod多container配置

参考文章

1. [Kubernetes 多container组成的Pod](https://blog.csdn.net/liumiaocn/article/details/52490444)
2. [多容器POD及Kubernetes容器通信](https://www.kubernetes.org.cn/2767.html)
3. [List All Container Images Running in a Cluster](https://kubernetes.io/zh/docs/tasks/access-application-cluster/list-all-running-container-images/)
    - 针对容器的操作. kubernetes里container并不是一种资源, 所以`describe`, `log`等命令不能指定某一容器, 这篇文章是用`get pod`子命令加`-o`选项输出格式化信息得到的.

突然发现pod配置中`metadata`下有`name`字段, 而`containers`竟然接受数组, 查了一下, 发现一个pod真的可以包含多个container, 见参考文章1中`sonar.yml`文件.

注意: 在一个pod内部的container共享所有资源, 包括共享pod的ip:port和磁盘.

`myltipod.yml`

```yml
apiVersion: v1
kind: Pod
metadata:
  name: project
  labels:
    app: web-project
spec:
  containers:
    - name: centos7
      image: centos:7
      command: ["tail", "-f", "/etc/profile"]
    - name: postgres
      image: postgres
      env:
        - name: "POSTGRES_USER"
          value: "mydb"
        - name: "POSTGRES_PASSWORD"
          value: "mydb"
```

```
$ k create -f multipod.yml 
pod "project" created
$ k get pod
NAME      READY     STATUS    RESTARTS   AGE
project   2/2       Running   0          6s
```

## 对指定容器执行操作

```
$ k log -f project
log is DEPRECATED and will be removed in a future version. Use logs instead.
Error from server (BadRequest): a container name must be specified for pod project, choose one of: [centos7 postgres]
```

`log`命令需要指定容器名称, 不能直接用pod名称查看, 正确命令应该是

```
$ k logs -f project -c centos7
```

进入指定容器

```
$ k exec -it project -c centos7 -- /bin/bash
```

`kubectl`没有针对容器的操作, 因为在kubernetes里container并不是标准资源. 参考文章3倒是介绍了通过格式化输出, 得到容器和镜像等信息.

查看pod内的容器列表, 可以用`describe`子命令完成, 不过输出的信息有点多, 起码能用.

## 同一个pod内容器的关系

这两个pod之间貌似没有联系, 无法通过container name连接???

```console
$ k exec -it project -c centos7 -- /bin/bash
[root@project /]# ping postgres
ping: postgres: Name or service not known
```

但是网络栈貌似是相通的, 同一个端口多个container不能相同, 毕竟service绑定的对象是pod, 出现相同端口的话怎么做映射?

参考文章2对单pod多container的应用场景和共享机制进行了详细解释, 同一个pod下的多个container的确共享网络空间, ta们无法通过name通信, 但是都可以使用localhost表示自己.

```console
[root@project /]# nc -l 5432
Ncat: bind to :::5432: Address already in use. QUITTING.
[root@project /]# netstat -anp
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:5432            0.0.0.0:*               LISTEN      -
tcp6       0      0 :::5432                 :::*                    LISTEN      -
udp        0      0 127.0.0.1:43116         127.0.0.1:43116         ESTABLISHED -
```

> 如果你了解docker的4种网络模式的话就会发现, kubernetes的多container的配置, 就是docker网络的`container`模式.
