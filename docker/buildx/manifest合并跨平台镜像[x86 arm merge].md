# manifest合并跨平台镜像[x86 arm]

x86主机上也可以拉取arm镜像, 只是不能启动.

A  双平面地址
B  x86地址
C  arm地址

docker manifest rm A
docker manifest create A B C
docker manifest push A

由于 manifest 只做元数据的合并, 因此要求 BC 事先已存在于远程仓库, 否则在合并时会报如下错误

```
$ docker manifest create A B C
unknown: artifact B not found
```
