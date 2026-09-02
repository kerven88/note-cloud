# client-go断网检测

## 场景描述

使用如下代码连接apiserver, 定时查询本地缓存中的 pod 列表.

```go
package main

import (
	"fmt"
	"log"
	"time"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/util/runtime"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/klog"
)

/*
	各种处理函数中, obj即为watch接口响应得到的资源对象.
*/

func onAdd(obj interface{}) {
	pod := obj.(*corev1.Pod)
	klog.Infof("add a pod: %+v", pod.Name)
}

// onUpdate // 此处省略 workqueue 的使用
func onUpdate(oldObj interface{}, newObj interface{}) {
	klog.Infof("update a pod")
	oldPod := oldObj.(*corev1.Pod)
	newPod := newObj.(*corev1.Pod)

	klog.Infof("old pod: %+v\n", oldPod.Name)
	klog.Infof("new pod: %+v\n", newPod.Name)
}

func onDelete(obj interface{}) {
	pod := obj.(*corev1.Pod)
	klog.Infof("delete a pod: +v", pod.Name)
}

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "/root/data/admin.conf")
	if err != nil {
		panic(err)
	}
	// config.Timeout = time.Second * 5
	// 初始化 client
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Panic(err.Error())
	}

	stopCh := make(chan struct{})
	defer close(stopCh)

	klog.Infof("初始化 informer...")
	// Shared指的是多个 lister 共享同一个cache, 而且资源的变化会同时通知到cache和listers.
	factory := informers.NewSharedInformerFactory(clientset, 60*time.Second)

	// podInformer 拥有两个方法: Informer, Lister.
	// 其实可以把 Informer 看作是 watch 操作.
	podInformer := factory.Core().V1().Pods()
	informer := podInformer.Informer()
	defer runtime.HandleCrash()

	// 启动 informer, 开始 list & watch 流程(不需要使用 go func() 模式)
	factory.Start(stopCh)

	// 从 apiserver 同步某种资源的全部对象, 即 list.
	// 之后就可以使用watch这种资源, 维护这份缓存.
	if !cache.WaitForCacheSync(stopCh, informer.HasSynced) {
		errTimeout := fmt.Errorf("初次同步缓存超时失败")
		runtime.HandleError(errTimeout)
		return
	}

	// 使用自定义 handler, 处理 watch 响应的各种事件.
	// 具体的维护操作在informer内部执行, 这里挂载的是额外的触发器.
	// 需要注意的是, 在上面的list过程中, 会不断触发onAdd事件, 相当于服务发现了.
	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    onAdd,
		UpdateFunc: onUpdate,
		DeleteFunc: onDelete,
	})

	go func() {
		for {
			// 从informer对象创建lister, 不过这里的代码没有特殊的目的,
			// 应该只是展示一下通过informer的接口得到list资源的方法.
			podLister := podInformer.Lister()
			// 从 lister 中获取所有 items
			podList, err := podLister.List(labels.Everything())
			if err != nil {
				klog.Errorf("获取Pod列表失败: %s", err)
			}
			klog.Infof("获取Pod列表")
			for _, pod := range podList {
				if pod.Namespace != "default" {
					continue
				}
				klog.Infof("%s", pod.Name)
			}

			time.Sleep(time.Second * 1)
		}
	}()

	<-stopCh
}

```

使用如下命令断开本地与apiserver的路由表模拟断网

```
ip r add blackhole 172.31.249.8/32
```

> 172.31.249.8 为 admin.conf 中的 apiserver 地址.

```log
I1214 17:20:20.496916   20403 main.go:97] 获取Pod列表
I1214 17:20:20.497061   20403 main.go:102] test-8494555654-fbwxk
W1214 17:20:20.853943   20403 reflector.go:474] /home/k8s.io/client-go/informers/factory.go:171: watch of *v1.Pod ended with: an error on the server ("unable to decode an event from the watch stream: read tcp 172.17.0.2:46892->172.31.249.8:6443: read: connection timed out") has prevented the request from succeeding
I1214 17:20:21.497664   20403 main.go:97] 获取Pod列表
I1214 17:20:21.497697   20403 main.go:102] test-8494555654-fbwxk
E1214 17:20:21.855285   20403 reflector.go:245] /home/k8s.io/client-go/informers/factory.go:171: Failed to list *v1.Pod: Get "https://172.31.249.8:6443/api/v1/pods?limit=500&resourceVersion=0": dial tcp 172.31.249.8:6443: connect: invalid argument
I1214 17:20:22.498689   20403 main.go:97] 获取Pod列表
I1214 17:20:22.498722   20403 main.go:102] test-8494555654-fbwxk
E1214 17:20:22.856511   20403 reflector.go:245] /home/k8s.io/client-go/informers/factory.go:171: Failed to list *v1.Pod: Get "https://172.31.249.8:6443/api/v1/pods?limit=500&resourceVersion=0": dial tcp 172.31.249.8:6443: connect: invalid argument
```

默认超时5分钟，才会报错, 而且在超时过程中, 也不影响本地缓存的遍历.

## 问题分析

这里的5分钟是golang代码与操作系统共同作用的结果.

client-go与apiserver之间的连接, 由于tls的原因, 一定会使用 tcp Transport, 该配置在`transport/cache.go`中的`tlsTransportCache.get()`

```go
	dial := config.Dial
	if dial == nil {
		dial = (&net.Dialer{
			Timeout:   30 * time.Second,
			// KeepAlive 可以调整网络断开的检测时间.
			// 关于断网检测, 操作系统一般有3个配置:
			// net.ipv4.tcp_keepalive_intvl = 75
			// net.ipv4.tcp_keepalive_probes = 9
			// net.ipv4.tcp_keepalive_time = 7200
			//
			// KeepAlive 在应用层面取代了 tcp_keepalive_intvl 系统配置, 
			// 即断网后, 操作系统会每隔30s对该长连接进行检测, 连续检测 9 次.
			// 如果都无响应, 则表示连接断开, 会触发到如下异常
			// apimachinery/pkg/watch/streamwatcher.go -> StreamWatcher.receive()
			// action, obj, err := sw.source.Decode() 返回 err 
			// (TODO: 其实网络断开的异常肯定是 net 标准库中报出的, 后续再查)
			// 
			KeepAlive: 30 * time.Second,
		}).DialContext
	}
```

异常抛出的流程如下.

```log
apimachinery:pkg/watch/streamwatcher.go -> StreamWatcher.receive()
action, obj, err := sw.source.Decode() 将异常写入 sw.result 通道
|
-- client-go:tools/cache/reflector.go -> Reflector.watchHandler()
    event, ok := <-w.ResultChan() 接收断网异常并返回
    |
    -- client-go:tools/cache/reflector.go -> Reflector.ListAndWatch
        "%s: watch of %v ended with: %v"
```

