# deployment pod残留error completed状态排查

参考文章

1. [Pod 干扰状况](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/disruptions/#pod-disruption-conditions)
    - TerminationByKubelet: Pod 由于节点压力驱逐、 节点体面关闭 或系统关键 Pod的抢占而被 kubelet 终止。
2. [Pod 的垃圾收集](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/#pod-garbage-collection)
    - Pod 的垃圾收集器（PodGC）是控制平面的控制器，它会在 Pod 个数超出所配置的阈值 （根据 kube-controller-manager 的 terminated-pod-gc-threshold 设置）时删除已终止的 Pod（阶段值为 Succeeded 或 Failed）。 这一行为会避免随着时间演进不断创建和终止 Pod 而引起的资源泄露问题。
    - 来自"通义千问": 
        - PodGC不会清理仍在有效 Node 上的 Terminated Pod；
        - PodGC不会清理由 ReplicaSet/Deployment 管理的已完成或失败 Pod
    - bug中的残留pod不满足上面的两个条件
3. [Graceful node drain results in never-cleaned-up pods in various Error/Completed phases #122122](https://github.com/kubernetes/kubernetes/issues/122122)
    - 官方issue, 已关闭, 与我的情况基本相符
    - WAI术语解释: Work As Intended（按预期工作）
    - [mimowo on Feb 16, 2024](https://github.com/kubernetes/kubernetes/issues/122122#issuecomment-1946445146)
        - 社区开发者的回复
        - statefulset pod 出现 completed 需要被修复, 因为会阻塞正常pod的创建, 别无选择;
            - 修复版本为 1.29.0, 并向后移植到 1.28.5 和 1.27.9, 修复提交[Resubmit "Make StatefulSet restart pods with phase Succeeded](https://github.com/kubernetes/kubernetes/pull/121389)
        - 而 deployment pod 则不用, 一方面ta不会影响新pod的创建, 另一方面管理员可能需要根据 completed 状态的 pod 排查问题;
        - 集群中出现的 completed pod 会在达到 --terminated-pod-gc-threshold=12500 选项表示的阈值时被清理(注意, 这个选项表示的是累计的数量而不是出现的时间).
    - [tuibeovince 的评论](https://github.com/kubernetes/kubernetes/issues/122122#issuecomment-2051376319)
        - 关机与重启的行为不同, 重启会造成 completed, 而关机后再重启不会.
    - [mimowo on Aug 30, 2023](https://github.com/kubernetes/kubernetes/issues/118310#issuecomment-1698823460)
        - 重现方案(要改成 deployment)
4. [Graceful node shutdown](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/#graceful-node-shutdown)

k8s: v1.28.13

voliet环境

kubelet 配置文件

```yaml
shutdownGracePeriod: 60s
shutdownGracePeriodCriticalPods: 30s
```

## 场景描述

执行reboot关机, 该节点上的 deployment pod 变成`Completed`或是`Error`状态, 一直保留不清理, 新的 pod 会在其他节点上重建, 也不影响 deployement 实际的副本数, **必现**.


```log
root@violet-cluster-controller:~# kwd pod -A | grep -v Run
NAMESPACE        NAME                                           READY   STATUS      RESTARTS    AGE     IP        NODE           
kube-storage     nfs-client-provisioner-95df5c979-5sm82         0/1     Error       0           4d22h   <none>    violet-master-2
kube-storage     nyx-pvc-backup-5d7d5ff855-q97xf                0/1     Completed   0           4d23h   <none>    violet-master-2
monitoring       kube-state-metrics-56bd6c47d4-pjlw4            0/1     Error       0           4d23h   <none>    violet-master-2
```

3个异常残留, 都是 deployment 类型的 Pod, 但3者都没有 rs 残留.

奇怪的是, statefulset 和 daemonset 的 pod 都没有残留.

## 排查记录

可以确定与节点的重启操作有关.

2025-12-31 17:05开始重启, 07分启动后 pod 与 container 都出现残留.

```log
root@violet-master-2:~# journalctl --list-boots
IDX BOOT ID                          FIRST ENTRY                 LAST ENTRY                 
 -3 c4e9e7d52d414b57990c6a1929d0e592 Wed 2025-12-31 15:20:41 CST Wed 2025-12-31 15:24:43 CST
 -2 9f352435631e4cb69f578b49ade8033f Wed 2025-12-31 15:26:44 CST Wed 2025-12-31 17:06:01 CST
 -1 d23c2cf4a94a46598d96d0651b0d14f2 Wed 2025-12-31 17:07:50 CST Wed 2025-12-31 17:21:29 CST
  0 a68d5822622b46348674914b02e984a5 Wed 2025-12-31 17:23:05 CST Mon 2026-01-05 15:27:59 CST
```

```yaml
$ kya pod kube-state-metrics-56bd6c47d4-pjlw4
## 没有 deletionTimestamp
status:
  conditions:
  - lastTransitionTime: "2025-12-31T09:05:29Z"
    message: Pod was terminated in response to imminent node shutdown.
    reason: TerminationByKubelet
    status: "True"
    type: DisruptionTarget
  - lastTransitionTime: "2025-12-31T09:05:29Z"
    reason: PodFailed
    status: "False"
    type: Ready
  - lastTransitionTime: "2025-12-31T09:05:29Z"
    reason: PodFailed
    status: "False"
    type: ContainersReady
  containerStatuses:
  - containerID: containerd://840841ba335f395f8be87f619212d042693ca5930d5e8da6029a6c014a4b5352
    image: registry.nyx.supcon.com/kube-state-metrics/kube-state-metrics:v2.10.0
    imageID: registry.nyx.supcon.com/kube-state-metrics/kube-state-metrics@sha256:b3d6cd5e9c2f707e5c3aded9d08e4fee0315a2fcb36d1ca8723d8734d85e657c
    lastState: {}
    name: kube-state-metrics
    ready: false
    restartCount: 0
    started: false
    state:
      terminated:
        containerID: containerd://840841ba335f395f8be87f619212d042693ca5930d5e8da6029a6c014a4b5352
        exitCode: 2
        finishedAt: "2025-12-31T09:05:29Z"
        reason: Error
        startedAt: "2025-12-31T07:43:57Z"
  hostIP: 192.168.203.101
  message: Pod was terminated in response to imminent node shutdown.
  phase: Failed
  qosClass: Burstable
  reason: Terminated
  startTime: "2025-12-31T07:43:56Z"
```

### ephemeral-storage

由于 ephemeral-storage 不足被驱逐也会出现残留, 如果是 Completed 状态, 那 phase 会是 Succeeded, 而不是 Failed.

```log
root@power-cluster-controller:~# kwd pod
NAME                                           READY   STATUS      RESTARTS        AGE    IP               NODE             NOMINATED NODE   READINESS GATES
nyx-pvc-backup-574f7dbcc4-gpdkn                0/1     Completed   0               132d   10.244.134.182   power-master-2   <none>           <none>
$ kya pod nyx-pvc-backup-574f7dbcc4-gpdkn
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2025-09-22T08:46:11Z"
    message: 'The node was low on resource: ephemeral-storage. Threshold quantity:
      67007895757, available: 59739228Ki. Container nyx-pvc-backup was using 284Ki,
      request is 0, has larger consumption of ephemeral-storage. '
    reason: TerminationByKubelet
    status: "True"
    type: DisruptionTarget
  containerStatuses:
  - containerID: containerd://9984298d8a02bb17b4415a1c8581514a9121f59ecd3e5f9942ed940c43034416
    image: registry.nyx.supcon.com/nyxos-k8s/nyx-backup-service:V1.0.0-250815
    lastState: {}
    name: nyx-pvc-backup
    ready: false
    restartCount: 0
    started: false
    state:
      terminated:
        containerID: containerd://9984298d8a02bb17b4415a1c8581514a9121f59ecd3e5f9942ed940c43034416
        exitCode: 0
        finishedAt: "2025-09-22T08:46:10Z"
        reason: Completed
        startedAt: "2025-08-26T08:29:13Z"
  hostIP: 192.168.203.101
  phase: Succeeded
  podIP: 10.244.134.182
  podIPs:
  - ip: 10.244.134.182
  qosClass: Burstable
  startTime: "2025-08-26T08:29:06Z"
```

### 与 readiness/liveness 探针无关

kube-state-metrics有探针, nfs-client-provisioner 和 nyx-pvc-backup 没探针.

### 与 crictl pods/ps 的残留无关

master-2上的残留信息

```log
root@violet-master-2:~# crictl pods
POD ID              CREATED             STATE               NAME                                      NAMESPACE     
3f4a5e1355193       4 days ago          NotReady            nfs-client-provisioner-95df5c979-5sm82    kube-storage  
d99b83a31dad0       5 days ago          NotReady            kube-scheduler-violet-master-2            kube-system   
de75e6c514bad       5 days ago          NotReady            kube-controller-manager-violet-master-2   kube-system   
dc2584e86e910       5 days ago          NotReady            nyx-pvc-backup-5d7d5ff855-q97xf           kube-storage  
bf33f2ec89096       5 days ago          NotReady            kube-state-metrics-56bd6c47d4-pjlw4       monitoring    
6e5a277707ef5       5 days ago          NotReady            nyx-monitor-fq97t                         nyx-monitoring
d60897ee41b24       5 days ago          NotReady            speaker-2zjfc                             metallb-system
00e27023b7b3b       5 days ago          NotReady            coredns-7576659d87-765wd                  kube-system   
457c88a769821       5 days ago          NotReady            supconipam-r9khn                          kube-system   
9cde0db39eb3a       5 days ago          NotReady            kube-sriov-cni-ds-amd64-4cm8c             kube-system   
f41849faf7559       5 days ago          NotReady            kube-multus-ds-mxt96                      kube-system   
0937a0f0bb38c       5 days ago          NotReady            calico-node-hlzks                         kube-system   
b7b0fa3a9fc11       5 days ago          NotReady            kube-proxy-9kxt2                          kube-system   
fc62b9bb16e6b       5 days ago          NotReady            nyx-operator-797d5885db-xkrc7             nyx-operator  
```

> coredns和nyx-operator也是 deployment 的 pod, 但是没有残留, 说明 crictl pods 的 NotReady 与 kubectl get pod 的残留并不完全相关.

```
root@violet-master-2:~# crictl ps -a  | grep -v Run
CONTAINER        IMAGE            CREATED         STATE     NAME                         ATTEMPT    POD ID           POD
874cf725f2db4    8065b798a4d67    4 days ago      Exited    mount-bpffs                  0          5140f12a2f7c2    calico-node-hlzks
55904a054e546    c02cab285709b    4 days ago      Exited    controller                   1          7e6b3541f663e    controller-54795fd8b5-2wd2r
bdf0bc72e2b57    de977d562e969    4 days ago      Exited    init-copy-files              0          729db57ccccfd    doc-manager-0
5344c5b1e6d35    a608c686bac93    4 days ago      Exited    metrics-server               1          58005d607cc05    metrics-server-58df99d4b9-gl4wk
29b66f176cbd7    f027cb9672f7e    4 days ago      Exited    install-supconipam           1          2626430996e91    supconipam-r9khn
810e5533b22a9    9dee260ef7f59    4 days ago      Exited    install-cni                  0          5140f12a2f7c2    calico-node-hlzks
88439b6627b9a    9dee260ef7f59    4 days ago      Exited    upgrade-ipam                 1          5140f12a2f7c2    calico-node-hlzks
fae986b1b5f61    3e763566b00bb    4 days ago      Exited    install-multus-binary        1          a97648626affb    kube-multus-ds-mxt96
0ba5b38934415    f0f94c8434681    4 days ago      Exited    kube-apiserver               10         79489a56e75d5    kube-apiserver-violet-master-2
2d86b1749c3b1    22aaebb38f4a9    4 days ago      Exited    kube-vip                     10         3d5bb61d33640    kube-vip-violet-master-2
54b40a43753f3    b61ff63c958cd    4 days ago      Exited    etcd                         6          2afd7a963d693    etcd-violet-master-2
e8543c8209ce4    932b0bface75b    4 days ago      Exited    nfs-client-provisioner       0          3f4a5e1355193    nfs-client-provisioner-95df5c979-5sm82
193f00696a089    f027cb9672f7e    5 days ago      Exited    supconipamd                  1          457c88a769821    supconipam-r9khn
d42164168c369    78fe016520c91    5 days ago      Exited    kube-scheduler               1          d99b83a31dad0    kube-scheduler-violet-master-2
9ae353633da7a    33fa2b1dcaa1f    5 days ago      Exited    kube-controller-manager      1          de75e6c514bad    kube-controller-manager-violet-master-2
ef5ae1fc3b9ef    69a8ee2e444d6    5 days ago      Exited    nyx-pvc-backup               0          dc2584e86e910    nyx-pvc-backup-5d7d5ff855-q97xf
840841ba335f3    8e0f85b91e3b0    5 days ago      Exited    kube-state-metrics           0          bf33f2ec89096    kube-state-metrics-56bd6c47d4-pjlw4
405b6d8149160    657ee177e1224    5 days ago      Exited    nyx-monitor                  0          6e5a277707ef5    nyx-monitor-fq97t
38d789d0f0864    ad7edcf788909    5 days ago      Exited    speaker                      0          d60897ee41b24    speaker-2zjfc
7ac0f035ee917    254ed72220290    5 days ago      Exited    coredns                      0          00e27023b7b3b    coredns-7576659d87-765wd
369b4449bd3cf    f027cb9672f7e    5 days ago      Exited    cert-manager                 0          457c88a769821    supconipam-r9khn
e794cf1e7ef25    e2a56bd85ec79    5 days ago      Exited    kube-sriov-cni               0          9cde0db39eb3a    kube-sriov-cni-ds-amd64-4cm8c
8cb43a2b32a7e    3e763566b00bb    5 days ago      Exited    kube-multus                  0          f41849faf7559    kube-multus-ds-mxt96
9f89718175b88    8065b798a4d67    5 days ago      Exited    calico-node                  0          0937a0f0bb38c    calico-node-hlzks
4ed831b5c65a1    96cda566c5df0    5 days ago      Exited    kube-proxy                   0          b7b0fa3a9fc11    kube-proxy-9kxt2
81937c21d8a86    470df6c90cb79    5 days ago      Exited    nyx-opeartor                 0          fc62b9bb16e6b    nyx-operator-797d5885db-xkrc7
```

## 关机流程记录

```log
root@green-cluster-controller:~/huangjiale# kwd pod -w
NAME                              READY   STATUS    RESTARTS   AGE   IP              NODE          
completed-test-5c64dbffdc-6qw49   1/1     Running   0          97m   10.244.41.160   green-worker-1
completed-test-5c64dbffdc-8cbs9   1/1     Running   0          97m   10.244.3.12     green-master-3
completed-test-5c64dbffdc-b2gxq   1/1     Running   0          97m   10.244.3.11     green-master-3
completed-test-5c64dbffdc-b6smt   1/1     Running   0          97m   10.244.41.159   green-worker-1
completed-test-5c64dbffdc-fxp27   1/1     Running   0          97m   10.244.173.78   green-master-2
completed-test-5c64dbffdc-g2l27   1/1     Running   0          97m   10.244.44.142   green-master-1
completed-test-5c64dbffdc-khkd5   1/1     Running   0          97m   10.244.173.77   green-master-2
completed-test-5c64dbffdc-wxlns   1/1     Running   0          97m   10.244.44.141   green-master-1
```

```log
root@green-master-3:~# crictl ps -a | grep comple
b4f5d92c1da91       ad7edcf788909       2 hours ago         Running             test                          0                   c67918ba34992       completed-test-5c64dbffdc-8cbs9
d1c110d10703a       ad7edcf788909       2 hours ago         Running             test                          0                   99eb360fd450f       completed-test-5c64dbffdc-b2gxq
```

10:56:20 执行 reboot 命令重启 master-3

```
Jan 12 10:56:20 green-master-3 systemd-logind[742]: The system will reboot now!
```

kubelet感知到重启行为, 立即停止 container.

```log
Jan 12 10:56:20 green-master-3 containerd[9759]: time="2026-01-12T10:56:20.221309517+08:00" level=info msg="StopContainer for \"d1c110d10703ab5e1f7ff00ea6d2033d95f8894b9a427f278f6582b2bead1992\" with timeout 30 (s)"
Jan 12 10:56:20 green-master-3 containerd[9759]: time="2026-01-12T10:56:20.221824540+08:00" level=info msg="Stop container \"d1c110d10703ab5e1f7ff00ea6d2033d95f8894b9a427f278f6582b2bead1992\" with signal terminated"
Jan 12 10:56:20 green-master-3 kubelet[3112080]: I0112 10:56:20.222397 3112080 setters.go:552] "Node became not ready" node="green-master-3" condition={"type":"Ready","status":"False","lastHeartbeatTime":"2026-01-12T02:56:20Z","lastTransitionTime":"2026-01-12T02:56:20Z","reason":"KubeletNotReady","message":"node is shutting down"}
Jan 12 10:56:20 green-master-3 containerd[9759]: time="2026-01-12T10:56:20.225406028+08:00" level=info msg="StopContainer for \"b4f5d92c1da9124c9e64dc8161700e10803ec9d453feb1f95234c89237525ba3\" with timeout 30 (s)"
Jan 12 10:56:20 green-master-3 containerd[9759]: time="2026-01-12T10:56:20.225741151+08:00" level=info msg="Stop container \"b4f5d92c1da9124c9e64dc8161700e10803ec9d453feb1f95234c89237525ba3\" with signal terminated"
```

> 日志中的`with timeout 30 (s)`, 这个时间就是pod的`.spec.terminationGracePeriodSeconds`字段的值.

> 从先后顺序来看, kubelet 应该是先停止业务服务, 再停止系统组件, 按照优先级反着来的.

```log
root@green-cluster-controller:~/huangjiale# kwd pod
NAME                              READY   STATUS              RESTARTS   AGE   IP              NODE          
completed-test-5c64dbffdc-6qw49   1/1     Running             0          98m   10.244.41.160   green-worker-1
completed-test-5c64dbffdc-8cbs9   0/1     Completed           0          98m   10.244.3.12     green-master-3
completed-test-5c64dbffdc-b2gxq   0/1     Completed           0          98m   10.244.3.11     green-master-3
completed-test-5c64dbffdc-b6smt   1/1     Running             0          98m   10.244.41.159   green-worker-1
completed-test-5c64dbffdc-fxp27   1/1     Running             0          98m   10.244.173.78   green-master-2
completed-test-5c64dbffdc-g2l27   1/1     Running             0          98m   10.244.44.142   green-master-1
completed-test-5c64dbffdc-h47z9   0/1     ContainerCreating   0          1s    <none>          green-master-1
completed-test-5c64dbffdc-khkd5   1/1     Running             0          98m   10.244.173.77   green-master-2
completed-test-5c64dbffdc-wxlns   1/1     Running             0          98m   10.244.44.141   green-master-1
completed-test-5c64dbffdc-xtq4v   0/1     ContainerCreating   0          1s    <none>          green-worker-1
```

```yaml
## kya pod completed-test-5c64dbffdc-8cbs9
status:
  conditions:
  - lastTransitionTime: "2026-01-12T02:56:28Z"
    message: Pod was terminated in response to imminent node shutdown.
    reason: TerminationByKubelet
    status: "True"
    type: DisruptionTarget
  - lastTransitionTime: "2026-01-12T02:56:20Z"
    reason: PodCompleted
    status: "False"
    type: Ready
  - lastTransitionTime: "2026-01-12T02:56:20Z"
    reason: PodCompleted
    status: "False"
    type: ContainersReady
```

看着像是 kubelet 只修改了本节点上 pod 的 status 状态, replicas controller 感知到后, 重新创建 pod 以补全副本数量.

```log
root@green-cluster-controller:~# k logs kube-controller-manager-green-master-1
## 发现 deployment 副本数不足, 创建新 pod
I0112 02:56:28.430657       1 event.go:307] "Event occurred" object="hjl-test/completed-test-5c64dbffdc" fieldPath="" kind="ReplicaSet" apiVersion="apps/v1" type="Normal" reason="SuccessfulCreate" message="Created pod: completed-test-5c64dbffdc-h47z9"
## 忽略 complete 的 pod(completed的pod没有 deleteTimestamp, 因此不自动删除)
I0112 02:58:57.659677       1 event.go:307] "Event occurred" object="hjl-test/completed-test-5c64dbffdc-b2gxq" fieldPath="" kind="Pod" apiVersion="" type="Normal" reason="TaintManagerEviction" message="Cancelling deletion of Pod hjl-test/completed-test-5c64dbffdc-b2gxq"
I0112 02:58:57.659727       1 event.go:307] "Event occurred" object="hjl-test/completed-test-5c64dbffdc-8cbs9" fieldPath="" kind="Pod" apiVersion="" type="Normal" reason="TaintManagerEviction" message="Cancelling deletion of Pod hjl-test/completed-test-5c64dbffdc-8cbs9"
```

同一时间查看 Daemonset 和 Statefulset, 发现都有明确的 delete 行为, 而 deployment 没有.

```go
I0112 02:56:28.412966       1 event.go:307] "Event occurred" object="kube-system/nyx-device-plugin" fieldPath="" kind="DaemonSet" apiVersion="apps/v1" type="Normal" reason="Succe
ededDaemonPod" message="Found succeeded daemon pod kube-system/nyx-device-plugin-k9s4g on node green-master-3, will try to delete it"
I0112 02:56:28.421246       1 event.go:307] "Event occurred" object="metallb-system/speaker" fieldPath="" kind="DaemonSet" apiVersion="apps/v1" type="Normal" reason="SucceededDae
monPod" message="Found succeeded daemon pod metallb-system/speaker-nkdbf on node green-master-3, will try to delete it"
I0112 02:56:28.421996       1 event.go:307] "Event occurred" object="supcon-1/designer" fieldPath="" kind="StatefulSet" apiVersion="apps/v1" type="Normal" reason="SuccessfulDelet
e" message="delete Pod designer-0 in StatefulSet designer successful"
I0112 02:56:28.422238       1 event.go:307] "Event occurred" object="supcon-1/designer-0" fieldPath="" kind="Pod" apiVersion="" type="Normal" reason="TaintManagerEviction" messag
e="Cancelling deletion of Pod supcon-1/designer-0"
I0112 02:56:28.422694       1 event.go:307] "Event occurred" object="kube-system/nyx-device-plugin" fieldPath="" kind="DaemonSet" apiVersion="apps/v1" type="Normal" reason="Succe
ssfulDelete" message="Deleted pod: nyx-device-plugin-k9s4g"
I0112 02:56:28.430657       1 event.go:307] "Event occurred" object="hjl-test/completed-test-5c64dbffdc" fieldPath="" kind="ReplicaSet" apiVersion="apps/v1" type="Normal" reason=
"SuccessfulCreate" message="Created pod: completed-test-5c64dbffdc-h47z9"
I0112 02:56:28.440852       1 replica_set.go:676] "Finished syncing" kind="ReplicaSet" key="hjl-test/completed-test-5c64dbffdc" duration="18.521018ms"
```
