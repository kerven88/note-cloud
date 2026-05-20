kubernetes: 1.28.13

## 场景描述

k8s集群中, 有个statefulset的pod中的网卡, 出现了dadfailed(IPv6地址冲突).

```bash
root@k8s-cluster-controller:~# kex datacenter1-dc-0
root@datacenter1-dc-0:/# ip a
## 省略...
76: net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 06:6e:a0:41:16:4b brd ff:ff:ff:ff:ff:ff
    inet 172.16.254.2/16 brd 172.16.255.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fd00::2:2:0:0:fe02/64 scope global dadfailed tentative 
       valid_lft forever preferred_lft forever
    inet6 fe80::46e:a0ff:fe41:164b/64 scope link 
       valid_lft forever preferred_lft forever
## 省略...
```

> 该pod使用了固定ip.

排查过程中, 发现该pod存在一个残留的sandbox未删除, 该sandbox仍然持有IP.

```
root@k8s-master-1:~# crictl pods | grep datacenter1-dc-0
5aa5d66d2ccdb       20 hours ago        Ready               datacenter1-dc-0                         -1            0                   (default)
14c1004655303       3 days ago          Ready               datacenter1-dc-0                         -1            0                   (default)
root@k8s-master-1:~# crictl ps -a | grep datacenter1-dc-0
c3596af4db131       cf96868952590       20 hours ago        Running             container-postgres           0                   5aa5d66d2ccdb       datacenter1-dc-0
26946f5f0392b       891427520d516       20 hours ago        Running             container-monitor-backend    0                   5aa5d66d2ccdb       datacenter1-dc-0
c74ac63fe8eb1       cf96868952590       3 days ago          Exited              container-postgres           1                   14c1004655303       datacenter1-dc-0
9e57b897fc1a4       891427520d516       3 days ago          Exited              container-monitor-backend    0                   14c1004655303       datacenter1-dc-0
```

`14c1004655303`就是残留的sandbox. 测试人员反馈, 3天前删除该Pod一直删不掉, 卡在Terminating状态, 于是就使用了`--force`强删.

时间能对上, 强删导致这样的情况也正常, 现在的问题就是`14c1004655303`为什么删不掉?

## 排查过程

### 1. sandbox清理异常

查看kubelet/containerd日志, 有如下输出

```log
May 19 16:36:25 k8s-master-1 kubelet[1857]: E0519 16:36:25.524480    1857 remote_runtime.go:222] "StopPodSandbox from runtime service failed" err="rpc error: code = DeadlineExcee
ded desc = context deadline exceeded" podSandboxID="14c1004655303a70a983452efb2bc04b390d8f4a4ad8b65de6e27a762de80e1a"
May 19 16:36:24 k8s-master-1 kubelet[1857]: E0519 16:36:24.518870    1857 pod_workers.go:1300] "Error syncing pod, skipping" err="failed to \"KillPodSandbox\" for \"0c98df45-982b
-4a22-a977-7dac36a014c7\" with KillPodSandboxError: \"rpc error: code = DeadlineExceeded desc = context deadline exceeded\"" pod=-1/datacenter1-dc-0" podUID="0c98df45-982b-4a
22-a977-7dac36a014c7"
May 19 16:36:24 k8s-master-1 kubelet[1857]: E0519 16:36:24.518848    1857 kubelet.go:2009] failed to "KillPodSandbox" for "0c98df45-982b-4a22-a977-7dac36a014c7" with KillPodSandb
oxError: "rpc error: code = DeadlineExceeded desc = context deadline exceeded"
May 19 16:36:24 k8s-master-1 kubelet[1857]: E0519 16:36:24.518799    1857 kuberuntime_manager.go:1390] "Failed to stop sandbox" podSandboxID={"Type":"containerd","ID":"5aa5d66d2c
cdb271caa294a413e56c6dccdcb647a8b123c58413bc1525467a26"}
May 19 16:36:24 k8s-master-1 kubelet[1857]: E0519 16:36:24.518750    1857 remote_runtime.go:222] "StopPodSandbox from runtime service failed" err="rpc error: code = DeadlineExcee
ded desc = context deadline exceeded" podSandboxID="5aa5d66d2ccdb271caa294a413e56c6dccdcb647a8b123c58413bc1525467a26"
May 19 16:36:10 k8s-master-1 kubelet[1857]: E0519 16:36:10.445473    1857 pod_workers.go:1300] "Error syncing pod, skipping" err="failed to \"KillPodSandbox\" for \"c1745bf9-e9e2
-49db-a5ba-bdb54e216040\" with KillPodSandboxError: \"rpc error: code = DeadlineExceeded desc = failed to stop sandbox container \\\"14c1004655303a70a983452efb2bc04b390d8f4a4ad8b65d
e6e27a762de80e1a\\\" in \\\"SANDBOX_READY\\\" state: wait sandbox container \\\"14c1004655303a70a983452efb2bc04b390d8f4a4ad8b65de6e27a762de80e1a\\\": context deadline exceeded\"" po
d=-1/datacenter1-dc-0" podUID="c1745bf9-e9e2-49db-a5ba-bdb54e216040"
```

```log
May 19 16:40:11 k8s-master-1 containerd[808]: time="2026-05-19T16:40:11.884245005+08:00" level=info msg="StopPodSandbox for \"14c1004655303a70a983452efb2bc04b390d8f4a4ad8b65de6e27a762de80e1a\""
May 19 16:40:11 k8s-master-1 containerd[808]: time="2026-05-19T16:40:11.802547569+08:00" level=error msg="StopPodSandbox for \"14c1004655303a70a983452efb2bc04b390d8f4a4ad8b65de6e27a762de80e1a\" failed" error="rpc error: code = Canceled desc = failed to stop sandbox container \"14c1004655303a70a983452efb2bc04b390d8f4a4ad8b65de6e27a762de80e1a\" in \"SANDBOX_READY\" state: wait sandbox container \"14c1004655303a70a983452efb2bc04b390d8f4a4ad8b65de6e27a762de80e1a\": context canceled"
```

kubelet/containerd一直尝试清理残留的sandbox, 但一直超时失败.

### 2. preStop

```log
Events:
  Type     Reason             Age                   From     Message
  ----     ------             ----                  ----     -------
  Normal   Killing            7m11s                 kubelet  Stopping container container-postgres
  Normal   Killing            4m9s (x3 over 7m11s)  kubelet  Stopping container container-monitor-backend
  Warning  FailedKillPod      4m9s (x2 over 5m41s)  kubelet  error killing pod: [failed to "KillContainer" for "container-monitor-backend" with KillContainerError: "rpc error: code = DeadlineExceeded desc = context deadline exceeded", failed to "KillPodSandbox" for "0c98df45-982b-4a22-a977-7dac36a014c7" with KillPodSandboxError: "rpc error: code = DeadlineExceeded desc = context deadline exceeded"]
  Warning  FailedPreStopHook  4m1s (x2 over 5m30s)  kubelet  PreStopHook failed
  Warning  FailedKillPod      85s                   kubelet  error killing pod: failed to "KillPodSandbox" for "0c98df45-982b-4a22-a977-7dac36a014c7" with KillPodSandboxError: "rpc error: code = DeadlineExceeded desc = failed to stop sandbox container \"5aa5d66d2ccdb271caa294a413e56c6dccdcb647a8b123c58413bc1525467a26\" in \"SANDBOX_READY\" state: wait sandbox container \"5aa5d66d2ccdb271caa294a413e56c6dccdcb647a8b123c58413bc1525467a26\": context deadline exceeded"
  Warning  FailedKillPod      13s (x9 over 2m41s)   kubelet  error killing pod: failed to "KillPodSandbox" for "0c98df45-982b-4a22-a977-7dac36a014c7" with KillPodSandboxError: "rpc error: code = DeadlineExceeded desc = context deadline exceeded"
```

```log
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/bash
          - -c
          - |
            mv /opt/package/Platform/Base/BsStorageServer /opt/package/Platform/Base/BsStorageServer.bak
            mv /opt/package/Log/LogService /opt/package/Log/LogService.bak
            mv /opt/package/Log/LogBackup /opt/package/Log/LogBackup.bak
            kill -15 $(pidof BsStorageServer)
            kill -15 $(pidof LogService)
            kill -15 $(pidof LogBackup)
            sleep 50
```

本来以为是`preStop`执行时间太长, 超过了30秒的宽限期, 但是就算超过30秒, kubelet也没放弃, 早就该删掉了.

### 发现僵尸进程

按照AI的说法, containerd会等待容器内所有进程都退出后才清理sandbox, 所以查看了下pause下是否存在关联子进程.

```log
root@k8s-master-1:~# crictl inspectp 14c1004655303 | grep pid
          "pid": "POD",
    "pid": 764880,
            "type": "pid"
                "getpid",
                "getppid",
                "pidfd_open",
                "pidfd_send_signal",
                "waitpid",
root@k8s-master-1:~# ps -ef | grep 764880
65535     764880  764860  0 May15 ?        00:00:00 [pause]
root      778711  764880  0 May15 ?        00:13:20 [BaseBackup] <defunct>
root     1272974  622024  0 11:20 pts/2    00:00:00 grep 764880
root@k8s-master-1:~# ps aux | grep 764880
65535     764880  0.0  0.0      0     0 ?        Ss   May15   0:00 [pause]
root     2504132  0.0  0.0   3116  1720 pts/2    S+   19:53   0:00 grep 764880
## Zl表示僵尸进程, l表示其为多线程进程(注意pause不是僵尸进程).
root@k8s-master-1:~# ps aux | grep 778711
root      778711  0.2  0.0      0     0 ?        Zl   May15  13:20 [BaseBackup] <defunct>
root     2503091  0.0  0.0   3116  1628 pts/2    S+   19:53   0:00 grep 778711
root@k8s-master-1:~# pstree -aps 764880
systemd,1
  `-containerd-shim,764860 -namespace k8s.io -id 14c1004655303a70a983452efb2bc04b390d8f4a4ad8b65de6e27a762de80e1a -address /var/run/containerd/containerd.sock
      `-(pause,764880)
          `-(BaseBackup,778711)
              |-{BaseBackup},778875
              |-{BaseBackup},778878
              `-{BaseBackup},778886
```

### 阻塞点

由于delete阻塞是一个偶现现象, 所以我想找到是什么原因引起的, 僵尸进程被卡在了哪里(系统调用)?

查看子进程的栈空间.

```log
root@k8s-master-1:~# ls /proc/778711/task
778711  778875  778878  778886
root@k8s-master-1:~# cat /proc/778711/stack
root@k8s-master-1:~# cat /proc/778875/stack
[<0>] nfs_wait_on_request+0x48/0x60 [nfs]
[<0>] nfs_page_group_lock_head+0x24/0x90 [nfs]
[<0>] nfs_lock_and_join_requests+0xb4/0x350 [nfs]
[<0>] nfs_updatepage+0x164/0xc70 [nfs]
[<0>] nfs_write_end+0x110/0x310 [nfs]
[<0>] generic_perform_write+0x11a/0x200
[<0>] nfs_file_write+0x1a2/0x2a0 [nfs]
[<0>] vfs_write+0x242/0x410
[<0>] ksys_write+0x73/0xf0
[<0>] __x64_sys_write+0x19/0x20
[<0>] do_syscall_64+0x38/0x90
[<0>] entry_SYSCALL_64_after_hwframe+0x64/0xce
```

`/proc/778875/stack`显示的是内核栈, 不是用户态栈. 即: 线程当前正在执行哪些 kernel function. 所以能显示出函数名称, 否则一个二进制程序的执行过程反编译出来都很难读的.

> 我还以为要用strace呢.

```log
root@k8s-master-1:# ps -L -o pid,tid,state,wchan:40,cmd -p 778711
    PID     TID S WCHAN                                    CMD
 778711  778711 Z -                                        [BaseBackup] <defunct>
 778711  778875 D nfs_wait_on_request                      /opt/package/BaseBackup -h -id DataService.BaseBackup
 778711  778878 D nfs_wait_on_request                      /opt/package/BaseBackup -h -id DataService.BaseBackup
 778711  778886 D nfs_wait_on_request                      /opt/package/BaseBackup -h -id DataService.BaseBackup
```

看来是卡在nfs操作上了, 使用`mount | grep ${POD_UID}`查看, 确实存在挂载项, 但是这个pod早就不存在了.

与测试同学复盘时, 发现确实是在发起备份任务后就开始删除pod了, 导致了卡死.

### socket是否还在?

按理说如果僵尸进程在等待nfs响应, 那一定还持有与nfs server的socket连接, 怎么查看?

```log
## 主线程中没有socket
root@cherry-master-1:~# ls -l /proc/778711/fd
total 0
## 子线程才有
root@cherry-master-1:~# ls -l /proc/778875/fd | grep socket
lrwx------ 1 root root 64 May 20 09:33 11 -> socket:[3092221073]
lrwx------ 1 root root 64 May 20 09:33 12 -> socket:[3092228364]
lrwx------ 1 root root 64 May 20 09:33 13 -> socket:[3089188747]
```

但是接下来就没办法了, `netstat`, `/proc/net/tcp`都找不到对应的inode. 虽然我知道nfs server的地址, 但是两边对应不上.
