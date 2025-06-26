# kuber-核心概念总结(转)

原文链接

[Kubernetes核心概念总结](https://www.cnblogs.com/zhenyuyaodidiao/p/6500720.html)

## 1. 基础架构

![](https://gitee.com/generals-space/gitimg/raw/master/9a3ce4b73794277bac82ec69dcca592d.png)

## 4. Service

### 4.2 Service代理外部服务

Service不仅可以代理Pod, 还可以代理任意其他后端, 比如运行在Kubernetes外部Mysql、Oracle等. 这是通过定义两个同名的service和endPoints来实现的. 示例如下: 

`redis-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  ports:
  - port: 6379
    targetPort: 6379
    protocol: TCP
```

`redis-endpoints.yaml`

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: redis-service
subsets:
  - addresses:
    - ip: 10.0.251.145
    ports:
    - port: 6379
      protocol: TCP
```

基于文件创建完Service和Endpoints之后, 在Kubernetes的Service中即可查询到自定义的Endpoints:

```
[root@k8s-master demon]# kubectl describe service redis-service
Name:            redis-service
Namespace:        default
Labels:            <none>
Selector:        <none>
Type:            ClusterIP
IP:            10.254.52.88
Port:            <unset>    6379/TCP
Endpoints:        10.0.251.145:6379
Session Affinity:    None
No events.
[root@k8s-master demon]# etcdctl get /skydns/sky/default/redis-service
{"host":"10.254.52.88","priority":10,"weight":10,"ttl":30,"targetstrip":0}
```
