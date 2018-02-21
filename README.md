# 使用kubeadm离线部署kubernetes v1.9.1#

2018-02-12

kubeadm是一个kubernetes官方提供的工具，kubeadm默认情况下并不会安装一个网络解决方案，所以用kubeadm安装完之后需要自己来安装一个网络的插件，本文选用flannel插件。

## 1 环境 ##

3个CentOS 节点：
master: 10.85.136.220
node:   10.85.136.228
node:   10.85.136.229

### 1.1 关闭swap ###

执行swapoff -a 关闭swap分区

### 1.2关闭selinux ###

编辑/etc/sysconfig/selinux，将SELINUX改成disabled，然后重启机器

## 2 安装docker ##

由于我的节点已经安装了docker,这里暂时不记录docker安装步骤,可以使用yum在线安装，
安装后启动docker：

```
systemctl enable docker && systemctl start docker
```


## 3 安装kubectl,kubelet,kubeadm ##

所有节点都需要安装这三个工具，先把如下几个软件包下载到本地：

* kubeadm-1.9.1-0.x86_64.rpm
* kubectl-1.9.1-0.x86_64.rpm
* kubelet-1.9.1-0.x86_64.rpm
* kubernetes-cni-0.6.0-0.x86_64.rpm
* socat-1.7.3.2-2.el7.x86_64.rpm

kubernetes-cni 和 socat 是安装kubelet,kubeadm所需要依赖的包，执行安装：

```
rpm -ivh kubectl-1.9.1-0.x86_64.rpm
rpm -ivh kubernetes-cni-0.6.0-0.x86_64.rpm kubelet-1.9.1-0.x86_64.rpm  socat-1.7.3.2-2.el7.x86_64.rpm
rpm -ivh kubeadm-1.9.1-0.x86_64.rpm
```

## 4 修改kubelet配置 ##

确保kubelet使用的cgroup驱动程序与Docker使用的相同,修改kubernetes的配置文件/etc/systemd/system/kubelet.service.d/10-kubeadm.conf，将“--cgroup-driver=systemd”修改成为“--cgroup-driver=cgroupfs”，重新启动kubelet

```
systemctl restart kubelet
```

## 5 下载安装k8s依赖镜像 ##

kubenetes初始化启动会依赖很多镜像，本文档采用离线安装，因为所有镜像都在google，所以提前下载所有镜像之后打包成tar，然后执行docker命令加载

```
docker load -i k8s-1-9-1.tar
```

此处列出master节点所需要的所有镜像列表：

* quay.io/coreos/flannel
* gcr.io/google_containers/kube-apiserver-amd64 
* gcr.io/google_containers/kube-controller-manager-amd64
* gcr.io/google_containers/kube-scheduler-amd64 
* gcr.io/google_containers/kube-proxy-amd64
* gcr.io/google_containers/kubernetes-dashboard-amd64
* gcr.io/google_containers/k8s-dns-sidecar-amd64
* gcr.io/google_containers/k8s-dns-kube-dns-amd64
* gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64
* gcr.io/google_containers/etcd-amd64 
* gcr.io/google_containers/pause-amd6

node节点也是需要执行docker load命令加载镜像的，node节点需要的镜像：

* quay.io/coreos/flannel
* gcr.io/google_containers/kube-proxy-amd64
* gcr.io/google_containers/pause-amd64

## 6 使用kubeadm初始化master ##

初始化的时候指定一下kubernetes版本，并设置一下pod-network-cidr，

```
kubeadm init --kubernetes-version=v1.9.1 --pod-network-cidr=10.244.0.0/16
```

这里pod-network-cidr根据使用的网络插件不同设置不同的值，本文档使用的flannel插件，执行init后输出如下：

```
You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token xxxxxxx 192.168.80.28:6443
```

kubeadm join这行需要拷贝下来，用来在node节点执行，这样就把node节点加入集群

## 7 安装网络插件flannel ##

安装flannel插件，执行

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

这里获取flannel配置文件也可能需要翻墙，我是下载的本地后执行

```
kubectl apply -f kube-flannel.yml
```

安装完网络插件后，通过kubectl get pods --all-namespaces来查看kube-dns是否在running来判断network是否安装成功

## 8 node节点加入集群 ##

将kubeadm init命令输出的kubeadm join那一行在node节点执行，即可把node节点加入集群，执行join后输出如下：

```
Node join complete:
* Certificate signing request sent to master and response
  received.
* Kubelet informed of new secure connection details.

Run 'kubectl get nodes' on the master to see this machine join.
```

## 9 验证安装 ##

在master节点上执行kubectl get nodes即可查看节点状态是否正确，安装是否成功，输出如下：

```
[root@centos7-base-ok]# kubectl get nodes
NAME      STATUS    AGE       VERSION
k8s-1     Ready     3d        v1.9.1
k8s-2     Ready     3d        v1.9.1
k8s-3     Ready     3d        v1.9.1
```

## 10 安装过程遇到的问题 ##

### 问题1 node not ready ###

第一次安装后发现node节点状态是not ready，原因是node节点镜像没有安装完全，缺少镜像gcr.io/google_containers/pause-amd64

### 问题2 dns pod状态是ContainerCreating ###

反复kubeadm init，kubeadm reset会遇到kube dns状态异常，通过执行kubectl describe pod kube-dns查看事件，
发现有event FailedCreatePodSandBox，解决办法：

```
执行docker rmi删除kube-dns镜像，重新load
```


