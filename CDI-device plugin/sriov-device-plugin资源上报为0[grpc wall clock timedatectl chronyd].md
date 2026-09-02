参考文章

1. [The Unix domain socket EOF between kubelet and device plugin is disconnected. Currently, kubelet does not have the retry mechanism](https://github.com/kubernetes/kubernetes/issues/131013)
2. [VKE 组件优化-应对 Kubernetes Device Plugin 框架资源上报异常](https://docs.volcengine.com/docs/6460/2525949?lang=zh)
    - device-plugin 框架的设计缺陷, 连接断开后不会重新注册.

kubernetes: v1.28.13
sriov-device-plugin: [v3.6.2](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin/tree/v3.6.2)

## 场景描述

sriov-device-plugin 一段时间没有新申请资源后, 偶现 vf 资源上报变成 0.

```log
Capacity:
  cpu:                                  64
  ephemeral-storage:                    435302676Ki
  hugepages-2Mi:                        0
  intel.com/sriov_net_A:                0
  intel.com/sriov_net_B:                0
Allocatable:
  cpu:                                  58
  ephemeral-storage:                    401174945538
  hugepages-2Mi:                        0
  intel.com/sriov_net_A:                0
  intel.com/sriov_net_B:                0
```

device-plugin 端, 在 sriov 资源变成0后就不再有日志输出.

查看 kubelet 端日志, 显示 device-plugin 与 kubelet 之间的连接断开了.

```log
Jul 09 18:04:50 yong-worker-1 kubelet[14491]: E0709 18:04:50.164615   14491 manager.go:419] "Unexpected: unhealthyDevices and endpoints are out of sync"
Jul 09 18:04:50 yong-worker-1 kubelet[14491]: E0709 18:04:50.164595   14491 manager.go:419] "Unexpected: unhealthyDevices and endpoints are out of sync"
Jul 09 18:04:48 yong-worker-1 kubelet[14491]: E0709 18:04:48.135913   14491 manager.go:419] "Unexpected: unhealthyDevices and endpoints are out of sync"
Jul 09 18:04:48 yong-worker-1 kubelet[14491]: E0709 18:04:48.135887   14491 manager.go:419] "Unexpected: unhealthyDevices and endpoints are out of sync"
Jul 09 17:59:48 yong-worker-1 kubelet[14491]: E0709 17:59:48.900232   14491 client.go:88] "ListAndWatch ended unexpectedly for device plugin" err="rpc error: code = Unavailable desc = error reading from server: EOF" resource="supcon.com/io_bandwidth_sriov_net_A"
Jul 09 17:59:48 yong-worker-1 kubelet[14491]: E0709 17:59:48.900203   14491 client.go:88] "ListAndWatch ended unexpectedly for device plugin" err="rpc error: code = Unavailable desc = error reading from server: EOF" resource="supcon.com/io_bandwidth_sriov_net_B"
Jul 09 17:59:47 yong-worker-1 kubelet[14491]: E0709 17:59:47.220562   14491 client.go:88] "ListAndWatch ended unexpectedly for device plugin" err="rpc error: code = Unavailable desc = error reading from server: EOF" resource="intel.com/sriov_net_A"
Jul 09 17:59:47 yong-worker-1 kubelet[14491]: E0709 17:59:47.220529   14491 client.go:88] "ListAndWatch ended unexpectedly for device plugin" err="rpc error: code = Unavailable desc = error reading from server: EOF" resource="intel.com/sriov_net_B"
```

只有重启 device-plugin 或是 kubelet 才能恢复.

## 原因分析

使用`lsof`命令查看 device-plugin 与 kubelet 之间的 socket 连接, 组件正常工作时如下.

```log
$ ps -ef | grep sriovdp
root      352078  351411  0 10:24 pts/0    00:00:00 grep sriovdp
root      696571  696205  0 Jul20 ?        00:02:07 /usr/bin/sriovdp -v 10 --log_dir /var/log/sriovdp --alsologtostderr
$ lsof -p 696571
## 省略
COMMAND    PID USER   FD      TYPE             DEVICE SIZE/OFF    NODE NAME
sriovdp 696571 root    7u     unix 0x0000000000000000      0t0 7375325 /var/lib/kubelet/plugins_registry/intel.com_sriov_net_A.sock type=STREAM (LISTEN)
sriovdp 696571 root    8u     unix 0x0000000000000000      0t0 7371703 /var/lib/kubelet/plugins_registry/intel.com_sriov_net_B.sock type=STREAM (LISTEN)
sriovdp 696571 root   11u     unix 0x0000000000000000      0t0 7430215 /var/lib/kubelet/plugins_registry/intel.com_sriov_net_A.sock type=STREAM (CONNECTED)
sriovdp 696571 root   12u     unix 0x0000000000000000      0t0 7428484 /var/lib/kubelet/plugins_registry/intel.com_sriov_net_B.sock type=STREAM (CONNECTED)
```

出现该问题时如下, CONNECTED 类型的 socket 消失了.

```log
$ ps -ef | grep sriovdp
root      352078  351411  0 10:24 pts/0    00:00:00 grep sriovdp
root      696571  696205  0 Jul20 ?        00:02:07 /usr/bin/sriovdp -v 10 --log_dir /var/log/sriovdp --alsologtostderr
$ lsof -p 696571
COMMAND    PID USER   FD      TYPE             DEVICE SIZE/OFF    NODE NAME
## 省略
sriovdp 696571 root    7u     unix 0x0000000000000000      0t0 7375325 /var/lib/kubelet/plugins_registry/intel.com_sriov_net_A.sock type=STREAM (LISTEN)
sriovdp 696571 root    8u     unix 0x0000000000000000      0t0 7371703 /var/lib/kubelet/plugins_registry/intel.com_sriov_net_B.sock type=STREAM (LISTEN)
```

AI分析是 kubelet 与 device-plugin 之间在 grpc 连接断开, 且 kubelet 没有再重试, device-plugin 也没有重新注册. 

问题点出在 device-plugin 的 [ListAndWatch](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin/blob/v3.6.2/pkg/resources/server.go#L155) 代码.

为了验证这个猜测, 修改 ListAndWatch 代码如下.

```golang
func (rs *resourceServer) ListAndWatch(empty *pluginapi.Empty, stream pluginapi.DevicePlugin_ListAndWatchServer) error {
	// listen for events: if updateSignal send new list of devices
	for {
		select {
		case <-rs.termSignal:
      ## 省略
		case <-rs.updateSignal:
      ## 省略
		case <-stream.Context().Done():     ## 新增此行
			glog.Infof("%s: stream context cancelled", methodID)
			return nil
		}
	}
}
```

再次出现该问题时, 打印了该行日志, 于是确认猜测.

该问题不难修复, 只要捕获到双方连接断开就退出, 让 device-plugin 重新注册即可, 但是没搞清楚原因, 测试同学无法验证.

但由于双方采用 unix domain 模式建立连接, 无法使用类似 tcpkill 等方法模拟双方断开的场景, 极难复现.

------

后面某次测试人员在node节点上执行了时钟跳变命令, 与出现该问题的时间点重合, 于是AI有如下猜测.

> 任何导致 wall clock 回拨的操作（手动 timedatectl 或 chronyd 同步）都会触发 kubelet device plugin ListAndWatch 的 gRPC keepalive 超时，导致连接断开且不自动重连 。

> tcp 连接有一种 keepalive 机制, 在双方没有数据通信时开始计时(重新有数据通信时清零), 如果超过`net.ipv4.tcp_keepalive_time`定义的时间, server端会怀疑对端是不是掉线(掉电)了, 尝试向client端发送ping包, 如果对端回应, 则表示连接仍正常. 
> 假如连续`net.ipv4.tcp_keepalive_probes`次, 每次间隔`net.ipv4.tcp_keepalive_intvl`, 都没有回应, server端会单方面断开连接.
> 每次发送 keepalive 包的超时时间无法配置, 只与`Retransmission Timeout`重传时间有关.

如果在 device-plugin 端发送 Keep-Alive 探测包时修改node节点的时间, 那么 device-plugin 收到 ACK 包的时间与发送时间跨度太大, 就有可能触发该问题.

gRPC没有使用操作系统的TCP keepalive超时判断, 而是实现了类似的机制.

device-plugin 的 gRPC server初始化在[这里](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin/blob/v3.6.2/pkg/resources/server.go#L70)

gRPC的tcp_keepalive_time默认也是2小时, 而ACK超时时间默认则是20秒, 且只要超时一次就判断连接异常而关闭, 不会重试. 

添加如下环境变量打印 ping 包日志, 可以验证这个想法.

```yaml
        env:
        - name: GRPC_GO_LOG_VERBOSITY_LEVEL
          value: "99"
        - name: GRPC_GO_LOG_SEVERITY_LEVEL
          value: info
        - name: GODEBUG
          value: http2debug=2           ## 可以打印 ping 包日志
```

```log
2026/09/02 03:05:01 http2: Framer 0xc00049e000: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/02 03:05:01 http2: Framer 0xc000ace2a0: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/02 03:05:01 http2: Framer 0xc00049e000: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/02 03:05:01 http2: Framer 0xc000ace2a0: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/02 05:05:02 http2: Framer 0xc000ace2a0: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/02 05:05:02 http2: Framer 0xc00049e000: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/02 05:05:02 http2: Framer 0xc000ace2a0: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/02 05:05:02 http2: Framer 0xc00049e000: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
```

修改方案为将 keepalive 检测间隔改为60秒, 提高异常出现的几率.

```golang
    grpc.NewServer(grpc.KeepaliveParams(keepalive.ServerParameters{
        Time:    60 * time.Second,
        Timeout: 20 * time.Second,
    }))
```

在下一次发送 Keep-Alive 包前3秒, 执行如下命令调整node节点时间.

```bash
kill -STOP $(pidof kubelet); sleep 5; systemctl restart chronyd; sleep 1; kill -CONT $(pidof kubelet);
```

其中`kill -STOP`可以冻结 kubelet , 让其不再得到CPU时间片, `kill -CONT`恢复. 具体时间线如下

```
device-plugin端

第N-1次ping包                     第N次ping包      收到ACK回包  第N+1次ping包
────────┴─────────────────┬─────────┴─────────┬─────┬──┴──────────┴──────>
                         STOP              时间跳变 CONT
                                  间隔5秒      间隔1秒
kubelet端
```

```log
## 原本每隔1分钟发送一次ping包
2026/09/01 10:21:00 http2: Framer 0xc0003a4000: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/01 10:21:00 http2: Framer 0xc0003a4000: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/01 10:21:00 http2: Framer 0xc0009a60e0: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/01 10:21:00 http2: Framer 0xc0009a60e0: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/01 10:22:00 http2: Framer 0xc0003a4000: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/01 10:22:00 http2: Framer 0xc0003a4000: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/01 10:22:00 http2: Framer 0xc0009a60e0: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/01 10:22:00 http2: Framer 0xc0009a60e0: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"

## 此处冻结 kubelet 并时间跳变
2026/09/01 10:23:00 http2: Framer 0xc0009a60e0: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/01 10:23:00 http2: Framer 0xc0003a4000: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
## ACK回包的时间戳就"超时"了
2026/09/01 09:01:22 http2: Framer 0xc0003a4000: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2026/09/01 09:01:22 http2: Framer 0xc0009a60e0: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
## 这里的 timeout 日志有误导, 应该是代码的bug, 把 Timeout 写成 Time 字段了.
2026/09/01 09:01:37 INFO: [transport] [server-transport 0xc0003b2000] Closing: keepalive ping not acked within timeout 1m0s
2026/09/01 09:01:37 INFO: [transport] [server-transport 0xc0004341a0] Closing: keepalive ping not acked within timeout 1m0s
I0901 09:01:37.373358       1 server.go:208] ListAndWatch(sriov_net_A): stream context cancelled
I0901 09:01:37.373379       1 server.go:209] ListAndWatch(sriov_net_A) goroutine exiting
2026/09/01 09:01:37 INFO: [transport] [server-transport 0xc0003b2000] loopyWriter exiting with error: transport closed by client
2026/09/01 09:01:37 INFO: [transport] [server-transport 0xc0004341a0] loopyWriter exiting with error: transport closed by client
I0901 09:01:37.373450       1 server.go:208] ListAndWatch(sriov_net_B): stream context cancelled
I0901 09:01:37.373476       1 server.go:209] ListAndWatch(sriov_net_B) goroutine exiting
```

## 解决方案

解决方法很简单, 捕获到连接断开, 就重新注册.

但还有个问题没解决, 在2小时的间隔中, 发送出 Keep-Alive 包的瞬间进行时钟跳变, 真实场景中真有这么巧合的事吗? 
