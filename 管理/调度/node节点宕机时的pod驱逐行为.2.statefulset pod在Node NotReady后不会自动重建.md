# node节点宕机时的pod驱逐行为.2.statefulset pod在Node NotReady后不会自动重建

参考文章

1. [Pods stuck in Terminating state when worker node is down (never redeployed on healthy nodes), how to fix this?](https://stackoverflow.com/questions/68979835/pods-stuck-in-terminating-state-when-worker-node-is-down-never-redeployed-on-he)
2. [Statefulset should be able to evicted if the worker node goes down #74947](https://github.com/kubernetes/kubernetes/issues/74947)
3. [Add new Taint Effect: ForceEviction #719](https://github.com/kubernetes/enhancements/pull/719)
4. [跟我学 K8S--代码: Kubernetes StatefulSet 代码分析与Unknown 状态处理](https://segmentfault.com/a/1190000019488735)

官方设计如此, 不想让 statefulset pod 自动迁移, 吵了好多年.

```log
root@black-cluster-controller:~# kwd node
NAME                 STATUS     ROLES           AGE     VERSION           INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                                                                                 KERNEL-VERSION                 CONTAINER-RUNTIME
black-master-1       NotReady   control-plane   28m     v1.28.13-supcon   192.168.203.100   <none>        Nyx NyxOS-V1.0.0-250113-M (264)                                                          6.1.78-rt18-nyxos-preempt-rt   containerd://1.7.7-5-g5e21abb18.m
root@black-cluster-controller:~# kwd pod
datacenter-dc-0        2/2     Terminating   0          13h     10.244.137.65   black-master-1   <none>           <none>
ice1-1                 1/1     Terminating   0          19m     10.244.137.69   black-master-1   <none>           <none>
```

```log
root@black-cluster-controller:~# k logs -f kube-controller-manager-black-master-2  --tail=20
I0827 02:05:45.661972       1 event.go:307] "Event occurred" object="supcon/ice1-1" fieldPath="" kind="Pod" apiVersion="v1" type="Warning" reason="NodeNotReady" message="Node is not ready"
I0827 02:05:45.864333       1 event.go:307] "Event occurred" object="supcon/datacenter-dc-0" fieldPath="" kind="Pod" apiVersion="v1" type="Warning" reason="NodeNotReady" message="Node is not ready"
I0827 02:10:50.989653       1 taint_manager.go:106] "NoExecuteTaintManager is deleting pod" pod="supcon/ice1-1"
I0827 02:10:50.989653       1 taint_manager.go:106] "NoExecuteTaintManager is deleting pod" pod="supcon/datacenter-dc-0"
I0827 02:10:50.989848       1 event.go:307] "Event occurred" object="supcon/ice1-1" fieldPath="" kind="Pod" apiVersion="" type="Normal" reason="TaintManagerEviction" message="Marking for deletion Pod supcon/ice1-1"
I0827 02:10:50.989862       1 event.go:307] "Event occurred" object="supcon/datacenter-dc-0" fieldPath="" kind="Pod" apiVersion="" type="Normal" reason="TaintManagerEviction" message="Marking for deletion Pod supcon/datacenter-dc-0"

I0827 02:33:38.913976       1 topologycache.go:237] "Can't get CPU or zone information for node" node="black-worker-kp920"
I0827 02:33:42.745964       1 event.go:307] "Event occurred" object="supcon/ice1" fieldPath="" kind="StatefulSet" apiVersion="apps/v1" type="Normal" reason="SuccessfulCreate" message="create Pod ice1-1 in StatefulSet ice1 successful"
I0827 02:34:32.891296       1 event.go:307] "Event occurred" object="supcon/datacenter-dc" fieldPath="" kind="StatefulSet" apiVersion="apps/v1" type="Normal" reason="SuccessfulCreate" message="create Pod datacenter-dc-0 in StatefulSet datacenter-dc successful"
```

但是terminating状态的pod会被添加`deletionTimestamp`与`deletionGracePeriodSeconds`. 

当出问题的节点重启Ready时, Pod会被删除重建(pod uid会变化), 自然也会重新调度, 有可能会被调度到其他节点上.
