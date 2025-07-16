参考文章

1. [【博客593】k8s为pod进行cpu绑核以进一步提高性能](https://blog.csdn.net/qq_43684922/article/details/128721232)
    - 确认kubelet开启绑核后，pod 不需要重启，也会自动绑核
    - taskset 查看cpu绑定情况

kubelet 需要配置`--cpu-manager-policy="static"`

较新版本的kubelet已经将这个参数放到`/var/lib/kubelet/config.yaml`配置文件里了, 而且`cpuManagerPolicy`要求预留cpu槽位, 必须同时指定`reservedSystemCPUs`.

```yml
cpuManagerPolicy: static
reservedSystemCPUs: 1-7
```
