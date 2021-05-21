# Centos7.8 安装 K8S

## 准备三台Centos7.8的虚拟机

- 每台机器 2 GB 或更多的 RAM (如果少于这个数字将会影响您应用的运行内存)

- 2 CPU 核或更多

- 集群中的所有机器的网络彼此均能相互连接(公网和内网都可以)

- 节点之中不可以有重复的主机名、MAC 地址或 product_uuid。请参见[这里](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#verify-the-mac-address-and-product-uuid-are-unique-for-every-node) 了解更多详细信息。

  ```bash
  # 查看MAC地址
  $ cat /etc/class/net/ens33/address
  # 查看UUID
  $ cat /sys/class/dmi/id/product_uuid
  ```

- 开启机器上的某些端口。请参见[这里](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports) 了解更多详细信息。

- 禁用交换分区。为了保证 kubelet 正常工作，您 **必须** 禁用交换分区

	```bash
	$ free -m 
	
	# 关闭交换分区
	$ swapoff -a
	
	# 备份配置文件
	$ cp /etc/fstab /etc/fstab_bak
	
	# 修改配置文件 - /etc/fstab
	# 删除swap相关行 /mnt/swap swap swap defaults 0 0 这一行或者注释掉这一行
	$ vim /etc/fstab
	
	# 确认swap已经关闭
	$ free -m 
	# 若swap行都显示 0 则表示关闭成功
	
	# 调整 swappiness 参数
	# $ echo 0 > /proc/sys/vm/swappiness # 临时生效
	
	$ vim /etc/sysctl.conf # 修改系统配置，使之永久生效
	#修改 vm.swappiness 的修改为 0
	# vm.swappiness=0
	$sysctl -p # 使配置生效
	```
	
- 放行一些端口或者关闭防火墙

  ```bash
  $ systemctl disable firewalld.service && systemctl stop firewalld.service
  ```

- 关闭 selinux
  ```bash
  $ vim /etc/selinux/config
  # 修改 SELINUX=disabled
	$ setenforce 0
	```
	
## 修改 `/etc/hosts`添加以下
```
192.168.19.98 centos7-1
192.168.19.101 centos7-2
192.168.19.80 centos7-3

192.168.175.128 VM1
192.168.175.130 VM2
192.168.175.131 VM3
```

  

## 修改三台主机名

```bash
$ hostnamctl set-hostname centos7-1
$ hostnamctl set-hostname centos7-2
$ hostnamctl set-hostname centos7-3
```

## 升级系统内核 `https://www.cnblogs.com/xzkzzz/p/9627658.html`

```bash
# 查看内核
$ uname -r

# 导入ELRepo仓库的公共密钥 `http://elrepo.org/tiki/HomePage`
$ rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

# 安装ELRepo仓库的yum源
$ yum install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm

# 安装最新版本内核
$ yum --enablerepo=elrepo-kernel install kernel-ml

# 查看系统上的所有可用内核：
$ sudo awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg

# 方法1、通过 grub2-set-default 0 命令设置，其中 0 是上面查询出来的可用内核
$ grub2-set-default 0

# 重启
$ reboot

# 验证
$ uname -r

```

## 安装docker： `https://docs.docker.com/engine/install/centos/`

```bash
# 删除旧版docker
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
                  
$ sudo yum install -y yum-utils

$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    
# 安装docker
$ sudo yum install docker-ce docker-ce-cli containerd.io
# docker 开机启动
$ sudo systemctl enable docker.service
```

启动docker后查看 运行命令`docker info` 会有以下报警提示

```
WARNING: bridge-nf-call-iptables is disabled
WARNING: bridge-nf-call-ip6tables is disabled
```

```bash
$ cd /proc/sys/net/bridge/
$ ll
# 查看 bridge-nf-call-arptables 和 bridge-nf-call-ip6tables，如果都是 0 就需要开启

# 修改系统参数
$ vim /etc/sysctl.conf # 添加以下两行
# net.bridge.bridge-nf-call-iptables = 1
# net.bridge.bridge-nf-call-ip6tables = 1

# 重新加载系统参数
$ sysctl -p

$ docker info # 发现 Warning 没有了
```



## 安装K8S `https://developer.aliyun.com/mirror/kubernetes`

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

setenforce 0

yum install -y kubelet kubeadm kubectl

# systemctl enable kubelet && systemctl start kubelet
```

## 拉取K8S所需镜像

```bash
# 检查版本
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.3", GitCommit:"1e11e4a2108024935ecfcb2912226cedeafd99df", GitTreeState:"clean", BuildDate:"2020-10-14T12:47:53Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}

# 这里需要查找对应 v1.19.3 版本的镜像
$ kubeadm config images list --kubernetes-version=v1.19.3
k8s.gcr.io/kube-apiserver:v1.19.3
k8s.gcr.io/kube-controller-manager:v1.19.3
k8s.gcr.io/kube-scheduler:v1.19.3
k8s.gcr.io/kube-proxy:v1.19.3
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns:1.7.0

# 拉取阿里云的google镜像
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.19.3 && \
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.19.3 && \
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.19.3 && \
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.19.3 && \
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 && \
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0 && \
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0

# 重命名镜像
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.19.3 k8s.gcr.io/kube-apiserver:v1.19.3 && \
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.19.3 k8s.gcr.io/kube-controller-manager:v1.19.3  && \
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.19.3 k8s.gcr.io/kube-scheduler:v1.19.3  && \
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.19.3 k8s.gcr.io/kube-proxy:v1.19.3  && \
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2  && \
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0  && \
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0

# 删除阿里云镜像
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.19.3 && \
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.19.3 && \
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.19.3 && \
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.19.3 && \
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 && \
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0 && \
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0
```

1.21.1版本

```bahs
$ kubeadm config images list --kubernetes-version=v1.21.1
k8s.gcr.io/kube-apiserver:v1.21.1
k8s.gcr.io/kube-controller-manager:v1.21.1
k8s.gcr.io/kube-scheduler:v1.21.1
k8s.gcr.io/kube-proxy:v1.21.1
k8s.gcr.io/pause:3.4.1
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns/coredns:v1.8.0

# 拉取阿里云的google镜像
docker pull registry.aliyuncs.com/k8sxio/kube-apiserver:v1.21.1 && \
docker pull registry.aliyuncs.com/k8sxio/kube-controller-manager:v1.21.1 && \
docker pull registry.aliyuncs.com/k8sxio/kube-scheduler:v1.21.1 && \
docker pull registry.aliyuncs.com/k8sxio/kube-proxy:v1.21.1 && \
docker pull registry.aliyuncs.com/k8sxio/pause:3.4.1 && \
docker pull registry.aliyuncs.com/k8sxio/etcd:3.4.13-0 && \
docker pull swr.cn-east-2.myhuaweicloud.com/coredns/coredns:1.8.0

# 重命名镜像
docker tag registry.aliyuncs.com/k8sxio/kube-apiserver:v1.21.1 k8s.gcr.io/kube-apiserver:v1.21.1 && \
docker tag registry.aliyuncs.com/k8sxio/kube-controller-manager:v1.21.1 k8s.gcr.io/kube-controller-manager:v1.21.1  && \
docker tag registry.aliyuncs.com/k8sxio/kube-scheduler:v1.21.1 k8s.gcr.io/kube-scheduler:v1.21.1  && \
docker tag registry.aliyuncs.com/k8sxio/kube-proxy:v1.21.1 k8s.gcr.io/kube-proxy:v1.21.1  && \
docker tag registry.aliyuncs.com/k8sxio/pause:3.4.1 k8s.gcr.io/pause:3.4.1  && \
docker tag registry.aliyuncs.com/k8sxio/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0  && \
docker tag swr.cn-east-2.myhuaweicloud.com/coredns/coredns:1.8.0 k8s.gcr.io/coredns/coredns:v1.8.0

# 删除阿里云镜像
docker rmi registry.aliyuncs.com/k8sxio/kube-apiserver:v1.21.1 && \
docker rmi registry.aliyuncs.com/k8sxio/kube-controller-manager:v1.21.1 && \
docker rmi registry.aliyuncs.com/k8sxio/kube-scheduler:v1.21.1 && \
docker rmi registry.aliyuncs.com/k8sxio/kube-proxy:v1.21.1 && \
docker rmi registry.aliyuncs.com/k8sxio/pause:3.4.1 && \
docker rmi registry.aliyuncs.com/k8sxio/etcd:3.4.13-0 && \
docker rmi swr.cn-east-2.myhuaweicloud.com/coredns/coredns:1.8.0

```



## 拉取flannel镜像（这步不要做）

```bash
# https://quay.io/repository/coreos/flannel?tab=tags
docker pull quay.io/coreos/flannel:v0.13.0-arm64
docker pull quay.io/coreos/flannel:v0.14.0-rc1-arm64

# 配置
mkdir -p /etc/cni/net.d
vim /etc/cni/net.d/10-flannel.conf # 内容如下：
# {"name":"cbr0","type":"flannel","delegate":{"isDefaultGateway":true}}
mkdir -p /usr/share/oci-umount/oci-umount.d
mkdir -p /run/flannel
vim /run/flannel/subnet.env # 内容如下：
# FLANNEL_NETWORK=172.100.0.0/16
# FLANNEL_SUBNET=172.100.1.0/24
# FLANNEL_MTU=1450
# FLANNEL_IPMASQ=true

# 重启服务
systemctl daemon-reload && \
systemctl restart kubelet.service && \
systemctl restart docker.service
```

## 三台虚拟机关机并拍摄快照

## Master主机初始化kubeadm

```bash
kubeadm init --apiserver-advertise-address=192.168.175.128 --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v1.21.1 --ignore-preflight-errors=Swap

# 成功后，要使用集群要按照提示运行下面命令
mkdir -p $HOME/.kube && \
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && \
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# You should now deploy a pod network to the cluster.
# Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
#  https://kubernetes.io/docs/concepts/cluster-administration/addons/
# 
# 其实就是 下载 https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml(https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml) 文件内容到到服务器
# 然后运行 `kubectl apply -f kube-flannel.yml`

```

journalctl -f -u kubelet 查看日志发现，报错如下：

kubelet cgroup driver: “cgroupfs” is different from docker cgroup driver: “systemd”

这个里要修改3个配置文件为systemd
1.vim /etc/docker/daemon.json

“exec-opts”: [“native.cgroupdriver=systemd”]
2.

```bash
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

在KUBELET_KUBECONFIG_ARGS 后面追加 --cgroup-driver=systemd

Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --cgroup-driver=systemd"

```
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --cgroup-driver=systemd"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env

# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS

```



3.vim /var/lib/kubelet/kubeadm-flags.env

KUBELET_KUBEADM_ARGS="--cgroup-driver=systemd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.2"

systemctl daemon-reload
systemctl restart kubelet

## 两台Node主机加入Master

```bash
# 可执行 `kubeadm token create --print-join-command` 获取node节点加入master的命令

# 按提示在另外两台主机运行
kubeadm join 192.168.19.98:6443 --token me8pf2.e88l7mvwaty517kr \
    --discovery-token-ca-cert-hash sha256:d8a2926d6ee7ee5c17e1790bd807ca3a6217f08472a5e54b4d0b65accd51094c
    
# master主机运行，查看这两个node是否加入成功
kubectl get nodes
```

## 更改设置，令容器运行时和 kubelet 使用 `systemd` 作为 cgroup 驱动，以此使系统更为稳定

https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/

```shell
# 三台主机 设置 daemon。
cat > /etc/docker/daemon.json <<EOF
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
```

如果已有这个文件就要复制内容粘贴进去

## kubectl使用tab键设置

```bash
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
```

## kubectl get cs 出现不健康的问题

```bash
kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused   
etcd-0               Healthy     {"health":"true"} 
```

出现这种情况，是/etc/kubernetes/manifests下的kube-controller-manager.yaml和kube-scheduler.yaml设置的默认端口是0，在文件中注释掉就可以了

kube-controller-manager.yaml文件修改：注释掉27行

kube-scheduler.yaml配置修改：注释掉19行

然后重启kubelet

```bash
systemctl restart kubelet.service
```

## 问题

```bash
kubectl get pods --all-namespaces
AMESPACE     NAME                                READY   STATUS              RESTARTS   AGE
kube-system   coredns-f9fd979d6-gv9t9             0/1     ContainerCreating   0          113m
kube-system   coredns-f9fd979d6-wrpjt             0/1     ContainerCreating   0          113m
kube-system   etcd-centos7-1                      1/1     Running             0          114m
kube-system   kube-apiserver-centos7-1            1/1     Running             0          113m
kube-system   kube-controller-manager-centos7-1   1/1     Running             0          31m
kube-system   kube-flannel-ds-8srsz               1/1     Running             0          85m
kube-system   kube-flannel-ds-cf9xs               1/1     Running             0          88m
kube-system   kube-flannel-ds-q8q86               1/1     Running             0          81m
kube-system   kube-proxy-kkbg9                    1/1     Running             0          113m
kube-system   kube-proxy-rlgmj                    1/1     Running             0          85m
kube-system   kube-proxy-wbkh9                    1/1     Running             0          81m
kube-system   kube-scheduler-centos7-1            1/1     Running             0          31m
# 前两个一直处于容器创建状态
kubectl describe pod coredns-f9fd979d6-gv9t9 --namespace=kube-system 
kubectl describe pod coredns-f9fd979d6-wrpjt --namespace=kube-system 
```

这里我就是从步骤`拉取flannel`开始重启执行了一遍，运行`kubeadm init`前先执行 `kubeadm reset`，其他两个node需要重新加入也需要选运行 `kubeadm reset`

如果出现 `failed to set bridge addr: "cni0" already has an IP address different from 10.244.0.1/24` 类似的错误，可能是 cni0的网卡配置（`ifconfig`查看）和报错提示的ip不一致导致，可将这个错误的网卡删掉，它会自己重建：

```ba
ifconfig cni0 down    
ip link delete cni0
```



```bash
# 可能跟fannel网络有关的配置删除
rm -rf /etc/cni/net.d/
rm -rf /var/lib/cni/
```



## 其他教程

http://blog.hungtcs.top/2019/11/27/23-K8S%E5%AE%89%E8%A3%85%E8%BF%87%E7%A8%8B%E7%AC%94%E8%AE%B0/  （没有测试）