
## 1. 准备(所有节点)

```bash
#### 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

#### 关闭selinux
setenforce 0
sed -i --follow-symlinks "s/^SELINUX=enforcing/SELINUX=disabled/g"  /etc/selinux/config
sed -i --follow-symlinks "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/selinux/config

#### 关闭swap, 将fstab中swap相关的配置注释掉.
swapoff -a; sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

#### 更新时间(etcd 对时间一致性要求高)
#### 以下两条适用于 centos7
timedatectl set-timezone Asia/Shanghai
echo '8 * * * * /usr/sbin/ntpdate asia.pool.ntp.org && /sbin/hwclock --systohc' >> /var/spool/cron/root
#### 以下3句适用于 centos 8
## systemctl start chronyd
## systemctl enable chronyd
## chronyc sources

hostnamectl set-hostname --static k8s-master-01

#### 修改hosts(如果节点结构与下面的不同, 最好现在就修改, 同时修改nginx.conf, 否则nginx无法正常运行, 各组件就无法与apiserver通信)
cat <<EOF >> /etc/hosts
127.0.0.1 kube-apiserver.generals.space

192.168.0.101 k8s-master-01
192.168.0.102 k8s-master-02
192.168.0.103 k8s-master-03
192.168.0.104 k8s-worker-01
192.168.0.105 k8s-worker-02
EOF

#### 配置内核选项

cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.may_detach_mounts = 1
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720
EOF
## 使其生效
sysctl --system

#### 加载内核模块并设置为开机启动

cat <<EOF > /etc/sysconfig/modules/k8s.modules
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4
modprobe br_netfilter
EOF
## 保证可执行
chmod 755 /etc/sysconfig/modules/k8s.modules

#### 安装docker及kuber

curl -o /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y docker-ce

mkdir /etc/docker
cat <<EOF > /etc/docker/daemon.json
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m"
    },
    "storage-driver": "overlay2",
    "storage-opts": [
        "overlay2.override_kernel_check=true"
    ]
}
EOF

systemctl start docker
systemctl enable docker

yum install -y kubelet kubeadm kubectl ipvsadm
systemctl start kubelet
systemctl enable kubelet
```

> `dockerd`使用的 cgroup driver 默认为`cgroupfs`.

------

安装指定版本的kube组件(需要与`kubeadm-config.yaml`中的`kubernetesVersion`字段匹配)

```
yum install -y kubernetes-cni-0.6.0 kubelet-1.13.2 kubeadm-1.13.2 kubectl-1.13.2
yum install -y kubelet-1.16.2 kubeadm-1.16.2 kubectl-1.16.2
```

## 2. apiserver高可用

~~在所有master节点执行如下命令.~~ 不对, 应该是在所有节点都要部署nginx, 因为在worker节点上, kubelet服务也需要连接api server进行通信.

```bash
docker run -d --name kube-apiserver.generals.space \
--restart always --net host \
-v /etc/kubernetes/k8s-nginx-lb.conf:/etc/nginx/nginx.conf \
nginx:latest
```

> nginx 容器使用的是宿主机网络, 监听的端口是 8443, 这里为了与 api server 监听的 6443 区分开. 并且由于写在 kubeadm-config.yaml 文件中, 最终生成的 kubectl 配置文件 `/etc/kubernetes/admin.conf`, 及各节点的 kubelet 的通信地址, 都将是 `kube-apiserver.generals.space:8443`.

## 3. 创建初始master节点

```bash
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> /etc/profile
kubeadm init --config kubeadm-config.yaml
```

初始master节点完成后, 会得到2条命令分别用于添加 master/worker 节点.

有如下常用命令以作备忘

### reset重置集群

```bash
kubeadm reset
ipvsadm --clear
rm -rf /var/lib/etcd
```

### 单纯拉取镜像

```bash
kubeadm config images pull --config kubeadm-config.yaml
```

## 4. 加入其他master节点

```bash
#!/bin/bash
USER=root
CONTROL_PLANE_IPS="k8s-master-02 k8s-master-03"
for host in ${CONTROL_PLANE_IPS}; do
    ## 如果etcd是内嵌到kuber集群中的, 那么还需要拷贝etcd的证书
    ssh "${USER}"@$host mkdir -p /etc/kubernetes/pki/etcd

    scp /etc/kubernetes/pki/ca.crt "${USER}"@$host:/etc/kubernetes/pki/
    scp /etc/kubernetes/pki/ca.key "${USER}"@$host:/etc/kubernetes/pki/
    scp /etc/kubernetes/pki/sa.key "${USER}"@$host:/etc/kubernetes/pki/
    scp /etc/kubernetes/pki/sa.pub "${USER}"@$host:/etc/kubernetes/pki/
    scp /etc/kubernetes/pki/front-proxy-ca.crt "${USER}"@$host:/etc/kubernetes/pki/
    scp /etc/kubernetes/pki/front-proxy-ca.key "${USER}"@$host:/etc/kubernetes/pki/
    scp /etc/kubernetes/pki/etcd/ca.crt "${USER}"@$host:/etc/kubernetes/pki/etcd/
    scp /etc/kubernetes/pki/etcd/ca.key "${USER}"@$host:/etc/kubernetes/pki/etcd/
    scp /etc/kubernetes/admin.conf "${USER}"@$host:/etc/kubernetes/
done
```

### 单点集群

如果希望只部署一个单点集群, 那么上面的步骤就没有必要进行了. 在第一个 master 节点上执行如下命令, 允许 master 节点可以运行 pod 就可以了.

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

> k taint node k8s-master-01 node-role.kubernetes.io/master=:NoSchedule

## 5. 网络插件

此时加入的新节点, 包括初始的master节点, 状态都是 NotReady, 可以通过`kubectl get node`查看. 并且 kube-system 命名空间下的 core-dns pod 也是 Pending 状态(`FailedScheduling`: 0/3 nodes are available: 3 node(s) had taints that the pod didn't tolerate.). 

部署网络插件后, 一切都会变得正常.
