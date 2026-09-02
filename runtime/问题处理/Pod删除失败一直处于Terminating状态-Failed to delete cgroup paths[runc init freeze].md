# Pod删除失败一直处于Terminating状态-Failed to delete cgroup paths

参考文章

1. [Preventing containers from being unable to be deleted](https://github.com/opencontainers/runc/pull/4757)

## 场景描述

```log
root@yong-cluster-controller:~# kwd pod
NAME                   READY   STATUS        RESTARTS   AGE   IP               NODE            NOMINATED NODE   READINESS GATES
ice7-1                 0/1     Terminating   1          16h   <none>           yong-worker-1   <none>           <none>
```

```log
Events:
  Type     Reason             Age   From     Message
  ----     ------             ----  ----     -------
  Normal   Killing            18m   kubelet  Stopping container icr
  Warning  FailedPreStopHook  18m   kubelet  PreStopHook failed
```

PodID 为 f72055a8-691a-4c8f-acea-f0b7a679cb4c

```
$ runc --version
runc version 1.1.9+dev
commit: v1.1.9-2-g26a98ea2-dirty
spec: 1.0.2-dev
go: go1.20.12
libseccomp: 2.5.4
```

## 问题分析

kubelet日志如下

```log
Sep 01 10:20:30 yong-worker-1 kubelet[15326]: I0901 10:20:30.306344   15326 pod_container_manager_linux.go:210] "Failed to delete cgroup paths" cgroupName=["kubepods","podf72055a8-691a-4c8f-acea-f0b7a679cb4c"] err="unable to destroy cgroup paths for cgroup [kubepods podf72055a8-691a-4c8f-acea-f0b7a679cb4c] : Failed to remove paths: map[:/sys/fs/cgroup/unified/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice blkio:/sys/fs/cgroup/blkio/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice cpu:/sys/fs/cgroup/cpu,cpuacct/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice cpuacct:/sys/fs/cgroup/cpu,cpuacct/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice cpuset:/sys/fs/cgroup/cpuset/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice devices:/sys/fs/cgroup/devices/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice freezer:/sys/fs/cgroup/freezer/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice hugetlb:/sys/fs/cgroup/hugetlb/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice memory:/sys/fs/cgroup/memory/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice name=systemd:/sys/fs/cgroup/systemd/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice net_cls:/sys/fs/cgroup/net_cls,net_prio/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice net_prio:/sys/fs/cgroup/net_cls,net_prio/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice perf_event:/sys/fs/cgroup/perf_event/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice pids:/sys/fs/cgroup/pids/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice]"
Sep 01 10:20:30 yong-worker-1 kubelet[15326]: time="2026-09-01T10:20:30+08:00" level=error msg="Failed to remove cgroup" error="rmdir /sys/fs/cgroup/memory/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice/cri-containerd-2050d2711d962bd24b20aa68445afaef598f83ac32d84a4ae45396c25683f479.scope: device or resource busy"
Sep 01 10:20:30 yong-worker-1 kubelet[15326]: time="2026-09-01T10:20:30+08:00" level=error msg="Failed to remove cgroup" error="rmdir /sys/fs/cgroup/cpuset/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice/cri-containerd-2050d2711d962bd24b20aa68445afaef598f83ac32d84a4ae45396c25683f479.scope: device or resource busy"
Sep 01 10:20:30 yong-worker-1 kubelet[15326]: time="2026-09-01T10:20:30+08:00" level=error msg="Failed to remove cgroup" error="rmdir /sys/fs/cgroup/systemd/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice/cri-containerd-2050d2711d962bd24b20aa68445afaef598f83ac32d84a4ae45396c25683f479.scope: device or resource busy"
Sep 01 10:20:30 yong-worker-1 kubelet[15326]: time="2026-09-01T10:20:30+08:00" level=error msg="Failed to remove cgroup" error="rmdir /sys/fs/cgroup/pids/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice/cri-containerd-2050d2711d962bd24b20aa68445afaef598f83ac32d84a4ae45396c25683f479.scope: device or resource busy"
Sep 01 10:20:30 yong-worker-1 kubelet[15326]: time="2026-09-01T10:20:30+08:00" level=error msg="Failed to remove cgroup" error="rmdir /sys/fs/cgroup/cpu,cpuacct/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice/cri-containerd-2050d2711d962bd24b20aa68445afaef598f83ac32d84a4ae45396c25683f479.scope: device or resource busy"
Sep 01 10:20:30 yong-worker-1 kubelet[15326]: time="2026-09-01T10:20:30+08:00" level=error msg="Failed to remove cgroup" error="rmdir /sys/fs/cgroup/net_cls,net_prio/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice/cri-containerd-2050d2711d962bd24b20aa68445afaef598f83ac32d84a4ae45396c25683f479.scope: device or resource busy"
Sep 01 10:20:30 yong-worker-1 kubelet[15326]: time="2026-09-01T10:20:30+08:00" level=error msg="Failed to remove cgroup" error="rmdir /sys/fs/cgroup/perf_event/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice/cri-containerd-2050d2711d962bd24b20aa68445afaef598f83ac32d84a4ae45396c25683f479.scope: device or resource busy"
Sep 01 10:20:30 yong-worker-1 kubelet[15326]: time="2026-09-01T10:20:30+08:00" level=error msg="Failed to remove cgroup" error="rmdir /sys/fs/cgroup/hugetlb/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice/cri-containerd-2050d2711d962bd24b20aa68445afaef598f83ac32d84a4ae45396c25683f479.scope: device or resource busy"
```

登陆到yong-worker-1节点上, 使用`crictl ps -a`与`crictl pods`查看容器与sandbox, 都已经被清理了, 没有残留.

## 解决方法

runc的bug，runc会被freeze，导致无法kill，有两种解决方法，1是升级runc版本到1.31.1及以上，2是切成cgroup v2

```log
root@yong-worker-1:/sys/fs/cgroup/pids/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice/cri-containerd-2050d2711d962bd24b20aa68445afaef598f83ac32d84a4ae45396c25683f479.scope# ps -ef | grep -f
/sys/fs/cgroup/cpu,cpuacct/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice/cri-containerd-2050d2711d962bd24b20aa68445afaef598f83ac32d84a4ae45396c25683f479.scope/tasks
root     510968       1 0 8 Aug31 ?       00:00:00 runc init
```

> cgroup 目录还残留着, ps然后过滤 tasks 文件中的pid, 能看到 runc 进程.

临时解决方案，执行以下命令unfreeze

```
echo THAWED | sudo tee /sys/fs/cgroup/freezer/kubepods.slice/kubepods-podf72055a8_691a_4c8f_acea_f0b7a679cb4c.slice/cri-containerd-2050d2711d962bd24b20aa68445afaef598f83ac32d84a4ae45396c25683f479.scope/freezer.state
```
