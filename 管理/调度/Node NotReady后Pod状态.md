Node NotReady后, 使用 kubectl delete 删除pod为什么删不掉, 删除行为需要 kubelet 回应吗?

确实是, kubelet 完成删除操作后，会向 API Server 报告：“这个Pod我已经成功删除了”。API Server 收到kubelet的报告后，才会最终从 etcd 中完全移除该Pod的元数据记录。此时，kubectl get pods 就看不到这个Pod了。

强制删除就是跳过了 kubelet 回应这个过程.

```
root@black-cluster-controller:~# kwd node
NAME                 STATUS     ROLES           AGE     VERSION           INTERNAL-IP       EXTERNAL-IP
black-master-1       NotReady   control-plane   4d18h   v1.28.13-supcon   192.168.203.100   <none>     
```

```log
kube-apiserver-black-master-1              1/1     Terminated   6 (42h ago)     42h     192.168.203.100   black-master-1
kube-scheduler-black-master-1              0/1     Completed    5 (45h ago)     42h     192.168.203.100   black-master-1
kube-controller-manager-black-master-1     0/1     Error        5 (45h ago)     42h     192.168.203.100   black-master-1
```

```
  containerStatuses:
  - name: kube-apiserver
    lastState:
      terminated:
        exitCode: 255
        finishedAt: "2025-08-23T08:34:12Z"
        reason: Unknown
        startedAt: "2025-08-23T03:09:09Z"
    ready: true
    restartCount: 6
    started: true
    state:
      running:
        startedAt: "2025-08-23T08:43:23Z"
  message: Pod was terminated in response to imminent node shutdown.
  reason: Terminated
  startTime: "2025-08-23T08:43:23Z"
```

```
$ kya pod kube-controller-manager-black-master-1
  containerStatuses:
  - name: kube-controller-manager
    lastState:
      terminated:
        exitCode: 2
        finishedAt: "2025-08-23T05:41:52Z"
        reason: Error
        startedAt: "2025-08-23T03:09:09Z"
    ready: false
    restartCount: 5
    started: false
    state:
      terminated:
        exitCode: 2
        finishedAt: "2025-08-25T01:30:24Z"
        reason: Error
        startedAt: "2025-08-23T08:43:23Z"
  message: Pod was terminated in response to imminent node shutdown.
  phase: Failed
  reason: Terminated
  startTime: "2025-08-23T08:43:23Z"
```

```
kya pod kube-scheduler-black-master-1
  containerStatuses:
  - name: kube-scheduler
    lastState:
      terminated:
        exitCode: 0
        finishedAt: "2025-08-23T05:41:52Z"
        reason: Completed
        startedAt: "2025-08-23T03:09:09Z"
    ready: false
    restartCount: 5
    started: false
    state:
      terminated:
        exitCode: 0
        finishedAt: "2025-08-25T01:30:24Z"
        reason: Completed
        startedAt: "2025-08-23T08:43:23Z"
  message: Pod was terminated in response to imminent node shutdown.
  phase: Failed
  reason: Terminated
  startTime: "2025-08-23T08:43:23Z"
```
