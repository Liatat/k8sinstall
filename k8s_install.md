# 一、k8s安装记录

[重置kubernetes](https://www.jianshu.com/p/31f7dda9ccf7)

ubuntu 18.04.5

docker 19.03.12

kubernetes 1.18.8

重点参考：官方文档！！！

kubernetes官方网站的[kubeadm安装文档](https://kubernetes.io/docs/setup/independent/install-kubeadm/)以及[利用kubeadm创建集群](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)这两个文档。

## 安装docker

1、安装

```shell
$apt-get update 

$apt-get install curl 

$curl -fsSL get.docker.com -o get-docker.sh 
$sh get-docker.sh --mirror Aliyun
```

2、配置加速器

```shell
$ curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://573b0ee5.m.daocloud.io

$ systemctl restart docker.service
```

```shell
# 设置 daemon
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

```shell
mkdir -p /etc/systemd/system/docker.service.d
```

```shell
# 重启 docker.
systemctl daemon-reload
systemctl restart docker
```



## 禁用交换分区

[参考](https://blog.csdn.net/u013164931/article/details/105548102/)

验证交换分区有没有关闭

> free -m

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200416001652534.png)

1.注释/etc/fstab关于swap的配置
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200416001726490.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMxNjQ5MzE=,size_16,color_FFFFFF,t_70)
 2.执行如下命令

```shell
echo vm.swappiness=0 >> /etc/sysctl.conf
```

3.重启

```shell
reboot
```

4.验证(Swap行均为0)

```shell
free -m
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200416001815414.png)

## 关闭防火墙

1.查看防火墙状态

```shell
root@s01-H61H2-CM:~# ufw status
状态：不活动
```

表示防火墙是关闭状态

## 修改k8s.conf文件

```shell
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### 关闭ipv6

编辑配置文件/etc/sysctl.conf

添加：

```shell
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

然后重启操作系统或者执行sysctl -p命令。最后查看ifconfig如下：ipv6信息消失

```shell
......
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:f5:30:c3:37  txqueuelen 0  (以太网)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp5s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.128  netmask 255.255.255.0  broadcast 192.168.1.255
        ether ec:a8:6b:a6:e3:3b  txqueuelen 1000  (以太网)
        RX packets 148372  bytes 36255623 (36.2 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 104273  bytes 41191724 (41.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (本地环回)
        RX packets 1810128  bytes 432725501 (432.7 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1810128  bytes 432725501 (432.7 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
   .......

```







## 确保每个节点上 MAC 地址和 product_uuid 的唯一性

- 您可以使用命令 `ip link` 或 `ifconfig -a` 来获取网络接口的 MAC 地址
- 可以使用 `sudo cat /sys/class/dmi/id/product_uuid` 命令对 product_uuid 校验

> 一般来讲，硬件设备会拥有唯一的地址，但是有些虚拟机的地址可能会重复。Kubernetes 使用这些值来唯一确定集群中的节点。 如果这些值在每个节点上不唯一，可能会导致安装[失败](https://github.com/kubernetes/kubeadm/issues/31)。



## 检查网络适配器

> 如果您有一个以上的网络适配器，同时您的 Kubernetes 组件通过默认路由不可达，我们建议您预先添加 IP 路由规则，这样 Kubernetes 集群就可以通过对应的适配器完成连接。

**单网卡跳过**



## 准备工作

- 配置主机名映射，每个节点

```
# cat /etc/hosts
127.0.0.1	localhost
192.168.0.200   Ubuntu-master
192.168.0.201   Ubuntu-1
192.168.0.202   Ubuntu-2
192.168.0.203   Ubuntu-3
```



## 在所有节点上安装kubeadm

查看apt安装源如下配置，使用阿里云的系统和kubernetes的源。

```
$ cat /etc/apt/sources.list
# 系统安装源
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
# kubeadm及kubernetes组件安装源
deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
```

## 安装kubeadm，kubectl，kubelet

master节点安装kubeadm，kubectl，kubelet

[参考](https://blog.csdn.net/M82_A1/article/details/95635705)

官网上安装过程的全部命令：

```shell
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**1.更新源并下载工具**

```
apt-get update && apt-get install -y apt-transport-https curl
```

![img](https://img-blog.csdnimg.cn/20190802155849879.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L004Ml9BMQ==,size_16,color_FFFFFF,t_70)

**2.添加公钥**

```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```

如果Linux网络无法访问，此时会报错

点击此链接 https://packages.cloud.google.com/apt/doc/apt-key.gpg （非国内资源）获取pgp文件，然后

通过 `apt-key add apt-key.gpg`来加载。

无法下载的自行在网盘中提取。

网盘地址：链接：https://pan.baidu.com/s/1aHtwOveSt0-QLPw9SYS8xw 
         提取码：uqjf 


**3.添加kubernetes源**

官方的源（非国内源）

```shell
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

国内的源（也可以直接使用阿里的源，前面已经配置过了--># kubeadm及kubernetes组件安装源）

```shell
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
EOF
```

再次更新源

```
apt-get update
```

![img](https://img-blog.csdnimg.cn/20190802161512111.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L004Ml9BMQ==,size_16,color_FFFFFF,t_70)

 

**4.安装最新kubelet、kubeadm、kubectl**(默认安装最新版本)

```
apt-get install -y kubelet kubeadm kubectl
```

![img](https://img-blog.csdnimg.cn/20190802161624621.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L004Ml9BMQ==,size_16,color_FFFFFF,t_70)

==如果要安装指定版本，先查看版本：==

```
apt-cache madison  kubeadm kubelet kubectl
```

==安装指定版本：==

```
apt-get install -y kubelet=1.18.8-00 kubeadm=1.18.8-00 kubectl=1.18.8-00
```

 

**5.设置不随系统更新而更新**

```
apt-mark hold kubelet kubeadm kubectl
```

![img](https://img-blog.csdnimg.cn/20190802161716223.png)



## 使用kubeadm安装Kubernetes集群

初始化控制平面节点Master

```shell
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.1.128 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```

成功安装会输出如下信息：

```shell
[init] Using Kubernetes version: vX.Y.Z
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kubeadm-cp localhost] and IPs [10.138.0.4 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kubeadm-cp localhost] and IPs [10.138.0.4 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubeadm-cp kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.138.0.4]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 31.501735 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-X.Y" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "kubeadm-cp" as an annotation
[mark-control-plane] Marking the node kubeadm-cp as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kubeadm-cp as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: <token>
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```



执行：

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 向集群中添加一个新的node

```shell
root@lxy-H61H2-CM:~# kubeadm token create --print-join-command
W1011 21:38:29.198995    6132 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
kubeadm join 192.168.1.128:6443 --token jngwo6.no3js51qjz1ru5vl     --discovery-token-ca-cert-hash sha256:47a092d53484973ed7880891c4d6332f00d7d581201de1aa0bdde0315bff6591
```





## 重置kubernetes

### 重置命令

```shell
sudo kubeadm reset
```

``` shell
[reset] Deleting contents of stateful directories: [/var/lib/etcd /var/lib/kubelet /var/lib/dockershim /var/run/kubernetes /var/lib/cni]

The reset process does not clean CNI configuration. To do so, you must remove /etc/cni/net.d

The reset process does not reset or clean up iptables rules or IPVS tables.
If you wish to reset iptables, you must do so manually by using the "iptables" command.

If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
to reset your system's IPVS tables.

The reset process does not clean your kubeconfig files and you must remove them manually.
Please, check the contents of the $HOME/.kube/config file.
```

### 手工清除：

#### 删除net.d

```shell
rm -rf /etc/cni/net.d
```

#### 重置iptables

```shell
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
sysctl net.bridge.bridge-nf-call-iptables=1
```

#### 删除 $HOME/.kube/config

```shell
rm -rf $HOME/.kube/config
```



#### 重装kubernetes

```shell
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.1.128 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 接上：

网络配置：

```shell
wget -c https://docs.projectcalico.org/v3.8/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
```

```shell
//安装的是v3.8.9
root@lxy-H61H2-CM:~# sed -i '/CALICO_IPV4POOL_IPIP/{n;s/Always/off/g}' calico.yaml
root@lxy-H61H2-CM:~# sed -i '/CALICO_IPV4POOL_CIDR/{n;s/192.168.0.0/10.244.0.0/g}' calico.yaml
root@lxy-H61H2-CM:~# kubectl apply -f calico.yaml
```


```shell
kubectl get pods -n kube-system
```

/etc/hosts文件修改 raw.githubusercontent.com对应的ip

``` shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta5/aio/deploy/recommended.yaml
```

```shell
kubectl delete ns kubernetes-dashboard        #删除旧版的dashboard
```

```shell
kubectl create -f recommended.yaml
```

```shell
kubectl get pod,svc -n kubernetes-dashboard  #查看信息
```

```shell
kubectl apply -f auth.yaml 
```

```shell
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

```shell
kubectl proxy
```





访问地址：http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/overview?namespace=kubernetes-dashboard



修改token过期时间：

### [modify token-ttl](https://www.cnblogs.com/xiaochina/p/12973241.html#2901301848)

默认900s/15分钟后认证token回话失效，需要重新登录认证，修改12h,方便使用
**在线修改kubernetes-dashboard （命名空间）的deployment**

```yaml
spec:
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
            defaultMode: 420
        - name: tmp-volume
          emptyDir: {}
      containers:
        - name: kubernetes-dashboard
          image: 'kubernetesui/dashboard:v2.0.0'
          args:
            - '--auto-generate-certificates'
            - '--namespace=kubernetes-dashboard'
            - '--token-ttl=43200'  #添加
          ports:
            - containerPort: 8443
              protocol: TCP
```



**或者：kubectl edit**
kubectl edit deployment kubernetes-dashboard -n kubernetes-dashboard //修改新增 --token-ttl=43200

