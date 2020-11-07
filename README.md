# liatatk8sinstall
记录使用k8s搭建集群的过程，以及后续在此基础上做的一些事。。。。。

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
**1.添加公钥**

```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```

如果Linux网络无法访问，此时会报错

点击此链接 https://packages.cloud.google.com/apt/doc/apt-key.gpg （非国内资源）获取pgp文件，然后

通过 `apt-key add apt-key.gpg`来加载。

无法下载的自行在网盘中提取。

网盘地址：链接：https://pan.baidu.com/s/1aHtwOveSt0-QLPw9SYS8xw 
         提取码：uqjf 
         
         
**2.更新源并下载工具**

```
apt-get update && apt-get install -y apt-transport-https curl
```

![img](https://img-blog.csdnimg.cn/20190802155849879.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L004Ml9BMQ==,size_16,color_FFFFFF,t_70)



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

```shell
kubeadm join 192.168.1.128:6443 --token iut7km.88nu0r6fwsevb0sm --discovery-token-ca-cert-hash sha256:939a4a1fb089e545fef0c788ee510e32ce26d3376a4e2db8bc5e45a017b0b455
```
### 配置网络
```shell
wget -c https://docs.projectcalico.org/v3.8/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
```
```shell
//安装的是v3.8.9
sed -i '/CALICO_IPV4POOL_IPIP/{n;s/Always/off/g}' calico.yaml
sed -i '/CALICO_IPV4POOL_CIDR/{n;s/192.168.0.0/10.244.0.0/g}' calico.yaml
kubectl apply -f calico.yaml
```

### 向集群中添加新的node：（在master结点上运行）
自动生成命令：

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
### 网络配置
```shell
wget -c https://docs.projectcalico.org/v3.8/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
```
```shell
//安装的是v3.8.9
sed -i '/CALICO_IPV4POOL_IPIP/{n;s/Always/off/g}' calico.yaml
sed -i '/CALICO_IPV4POOL_CIDR/{n;s/192.168.0.0/10.244.0.0/g}' calico.yaml
kubectl apply -f calico.yaml
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



# 二、Sock-shop部署

1、下载github上的包到本地新建文件夹sock-shop

2、解压文件 `unzip 包名  `

3、选择kubernetes平台安装方式，进入文件夹`/home/lxy/sock_shop/microservices-demo-master/deploy/kubernetes`,修改该目录下的部署文件`complete-demo.yaml`为：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: carts-db
  labels:
    app: carts-db
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: carts-db
  template:
    metadata:
      labels:
        app: carts-db
    spec:
      containers:
      - name: carts-db
        image: mongo
        ports:
        - name: mongo
          containerPort: 27017
        securityContext:
          capabilities:
            drop:
              - all
            add:
              - CHOWN
              - SETGID
              - SETUID
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /tmp
          name: tmp-volume
      volumes:
        - name: tmp-volume
          emptyDir:
            medium: Memory
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: carts-db
  labels:
    app: carts-db
  namespace: sock-shop
spec:
  ports:
    # the port that this service should serve on
  - port: 27017
    targetPort: 27017
  selector:
    app: carts-db
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: carts
  labels:
    app: carts
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: carts
  template:
    metadata:
      labels:
        app: carts
    spec:
      containers:
      - name: carts
        image: weaveworksdemos/carts:0.4.8
        ports:
         - containerPort: 80
        env:
         - name: ZIPKIN
           value: zipkin.jaeger.svc.cluster.local
         - name: JAVA_OPTS
           value: -Xms64m -Xmx128m -XX:PermSize=32m -XX:MaxPermSize=64m -XX:+UseG1GC -Djava.security.egd=file:/dev/urandom
        securityContext:
          runAsNonRoot: true
          runAsUser: 10001
          capabilities:
            drop:
              - all
            add:
              - NET_BIND_SERVICE
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /tmp
          name: tmp-volume
      volumes:
        - name: tmp-volume
          emptyDir:
            medium: Memory
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: carts
  labels:
    app: carts
  namespace: sock-shop
spec:
  ports:
    # the port that this service should serve on
  - port: 80
    targetPort: 80
  selector:
    app: carts
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalogue-db
  labels:
    app: catalogue-db
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalogue-db
  template:
    metadata:
      labels:
        app: catalogue-db
    spec:
      containers:
      - name: catalogue-db
        image: weaveworksdemos/catalogue-db:0.3.0
        env:
          - name: MYSQL_ROOT_PASSWORD
            value: fake_password
          - name: MYSQL_DATABASE
            value: socksdb
        ports:
        - name: mysql
          containerPort: 3306
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: catalogue-db
  labels:
    app: catalogue-db
  namespace: sock-shop
spec:
  ports:
    # the port that this service should serve on
  - port: 3306
    targetPort: 3306
  selector:
    app: catalogue-db
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalogue
  labels:
    app: catalogue
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalogue
  template:
    metadata:
      labels:
        app: catalogue
    spec:
      containers:
      - name: catalogue
        image: weaveworksdemos/catalogue:0.3.5
        ports:
        - containerPort: 80
        securityContext:
          runAsNonRoot: true
          runAsUser: 10001
          capabilities:
            drop:
              - all
            add:
              - NET_BIND_SERVICE
          readOnlyRootFilesystem: true
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: catalogue
  labels:
    app: catalogue
  namespace: sock-shop
spec:
  ports:
    # the port that this service should serve on
  - port: 80
    targetPort: 80
  selector:
    app: catalogue
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: front-end
  labels:
    app: front-end
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: front-end
  template:
    metadata:
      labels:
        app: front-end
    spec:
      containers:
      - name: front-end
        image: weaveworksdemos/front-end:0.3.12
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 8079
        securityContext:
          runAsNonRoot: true
          runAsUser: 10001
          capabilities:
            drop:
              - all
          readOnlyRootFilesystem: true
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: front-end
  labels:
    app: front-end
  namespace: sock-shop
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8079
    nodePort: 30003
  selector:
    app: front-end
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-db
  labels:
    app: orders-db
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: orders-db
  template:
    metadata:
      labels:
        app: orders-db
    spec:
      containers:
      - name: orders-db
        image: mongo
        ports:
        - name: mongo
          containerPort: 27017
        securityContext:
          capabilities:
            drop:
              - all
            add:
              - CHOWN
              - SETGID
              - SETUID
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /tmp
          name: tmp-volume
      volumes:
        - name: tmp-volume
          emptyDir:
            medium: Memory
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: orders-db
  labels:
    app: orders-db
  namespace: sock-shop
spec:
  ports:
    # the port that this service should serve on
  - port: 27017
    targetPort: 27017
  selector:
    app: orders-db
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
  labels:
    app: orders
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: orders
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
      - name: orders
        image: weaveworksdemos/orders:0.4.7
        env:
         - name: ZIPKIN
           value: zipkin.jaeger.svc.cluster.local
         - name: JAVA_OPTS
           value: -Xms64m -Xmx128m -XX:PermSize=32m -XX:MaxPermSize=64m -XX:+UseG1GC -Djava.security.egd=file:/dev/urandom
        ports:
        - containerPort: 80
        securityContext:
          runAsNonRoot: true
          runAsUser: 10001
          capabilities:
            drop:
              - all
            add:
              - NET_BIND_SERVICE
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /tmp
          name: tmp-volume
      volumes:
        - name: tmp-volume
          emptyDir:
            medium: Memory
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: orders
  labels:
    app: orders
  namespace: sock-shop
spec:
  ports:
    # the port that this service should serve on
  - port: 80
    targetPort: 80
  selector:
    app: orders
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment
  labels:
    app: payment
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment
    spec:
      containers:
      - name: payment
        image: weaveworksdemos/payment:0.4.3
        ports:
        - containerPort: 80
        securityContext:
          runAsNonRoot: true
          runAsUser: 10001
          capabilities:
            drop:
              - all
            add:
              - NET_BIND_SERVICE
          readOnlyRootFilesystem: true
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: payment
  labels:
    app: payment
  namespace: sock-shop
spec:
  ports:
    # the port that this service should serve on
  - port: 80
    targetPort: 80
  selector:
    app: payment
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue-master
  labels:
    app: queue-master
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: queue-master
  template:
    metadata:
      labels:
        app: queue-master
    spec:
      containers:
      - name: queue-master
        image: weaveworksdemos/queue-master:0.3.1
        ports:
        - containerPort: 80
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: queue-master
  labels:
    app: queue-master
  annotations:
    prometheus.io/path: "/prometheus"
  namespace: sock-shop
spec:
  ports:
    # the port that this service should serve on
  - port: 80
    targetPort: 80
  selector:
    app: queue-master
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rabbitmq
  labels:
    app: rabbitmq
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
      - name: rabbitmq
        image: rabbitmq:3.6.8
        ports:
        - containerPort: 5672
        securityContext:
          capabilities:
            drop:
              - all
            add:
              - CHOWN
              - SETGID
              - SETUID
              - DAC_OVERRIDE
          readOnlyRootFilesystem: true
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
  labels:
    app: rabbitmq
  namespace: sock-shop
spec:
  ports:
    # the port that this service should serve on
  - port: 5672
    targetPort: 5672
  selector:
    app: rabbitmq
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shipping
  labels:
    app: shipping
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shipping
  template:
    metadata:
      labels:
        app: shipping
    spec:
      containers:
      - name: shipping
        image: weaveworksdemos/shipping:0.4.8
        env:
         - name: ZIPKIN
           value: zipkin.jaeger.svc.cluster.local
         - name: JAVA_OPTS
           value: -Xms64m -Xmx128m -XX:PermSize=32m -XX:MaxPermSize=64m -XX:+UseG1GC -Djava.security.egd=file:/dev/urandom
        ports:
        - containerPort: 80
        securityContext:
          runAsNonRoot: true
          runAsUser: 10001
          capabilities:
            drop:
              - all
            add:
              - NET_BIND_SERVICE
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /tmp
          name: tmp-volume
      volumes:
        - name: tmp-volume
          emptyDir:
            medium: Memory
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: shipping
  labels:
    app: shipping
  namespace: sock-shop
spec:
  ports:
    # the port that this service should serve on
  - port: 80
    targetPort: 80
  selector:
    app: shipping
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-db
  labels:
    app: user-db
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: user-db
  template:
    metadata:
      labels:
        app: user-db
    spec:
      containers:
      - name: user-db
        image: weaveworksdemos/user-db:0.4.0
        ports:
        - name: mongo
          containerPort: 27017
        securityContext:
          capabilities:
            drop:
              - all
            add:
              - CHOWN
              - SETGID
              - SETUID
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /tmp
          name: tmp-volume
      volumes:
        - name: tmp-volume
          emptyDir:
            medium: Memory
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: user-db
  labels:
    app: user-db
  namespace: sock-shop
spec:
  ports:
    # the port that this service should serve on
  - port: 27017
    targetPort: 27017
  selector:
    app: user-db
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user
  labels:
    app: user
  namespace: sock-shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: user
  template:
    metadata:
      labels:
        app: user
    spec:
      containers:
      - name: user
        image: weaveworksdemos/user:0.4.7
        ports:
        - containerPort: 80
        env:
        - name: MONGO_HOST
          value: user-db:27017
        securityContext:
          runAsNonRoot: true
          runAsUser: 10001
          capabilities:
            drop:
              - all
            add:
              - NET_BIND_SERVICE
          readOnlyRootFilesystem: true
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: user
  labels:
    app: user
  namespace: sock-shop
spec:
  ports:
    # the port that this service should serve on
  - port: 80
    targetPort: 80
  selector:
    app: user
```

4、创建命名空间`kubectl create namespace sock-shop`

5、在该文件所在目录下运行`kubectl create -f complete-demo.yaml`,后续对该文件有修改的话运行`kubectl apply -f 文件名`进行更新

6、`kubectl get pod -n sock-shop `查看pod状态是否都running，`kubectl get svc -n sock-shop`查看services，以及端口号，如下图：30003端口作为sock-shop的服务入口。浏览器 {NodeIp}:30003-->192.168.1.111:30003进行访问。

![image-20201006152146156](/home/lxy/.config/Typora/typora-user-images/image-20201006152146156.png)

# 三、Istio安装

1、官网上（github）下载对应版本安装包，采用istioctl方式安装。安装1.5.10版本。

> 官方https://istio.io/latest/zh/docs/setup/getting-started/上对istio1.7安装包的说明是：
>
> 安装目录包含如下内容：
>
> - `install/kubernetes` 目录下，有 Kubernetes 相关的 YAML 安装文件
> - `samples/` 目录下，有示例应用程序
> - `bin/` 目录下，包含 [`istioctl`](https://istio.io/latest/zh/docs/reference/commands/istioctl) 的客户端文件。`istioctl` 工具用于手动注入 Envoy sidecar 代理。
>
> 但是！！！实际上我看了1.7和1.6.6的版本上根本没有install文件。。。我在1.5.10版本安装包上找全了这三个目录，所以我最后下载安装的是1.5.10版本的istio

2、复制安装包到`/home/lxy/istio`，cd到该目录解压安装包，`root@lxy-H61H2-CM:/home/lxy/istio# tar -zxvf istio-1.5.10-linux.tar.gz `

3、将 `istioctl` 客户端路径增加到 path 环境变量中，`vim /etc/environment`添加`:/home/lxy/istio/istio-1.5.10/bin`

```shell
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/home/lxy/istio/istio-1.5.10/bin"

```

然后`source /etc/environment`使它生效。

4、快速部署istio，cd到install/kubernetes目录里面执行`root@lxy-H61H2-CM:/home/lxy/istio/istio-1.5.10/install/kubernetes# kubectl apply -f istio-demo.yaml`

5、` kubectl get svc -n istio-system`查看发现istio-ingressgateway是pending，修改文件

```shell
root@lxy-H61H2-CM:/home/lxy/istio/istio-1.5.10/install/kubernetes# kubectl get svc -n istio-system
NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                                      AGE
grafana                     ClusterIP      10.107.68.227    <none>        3000/TCP                                                                                                                                     97m
istio-citadel               ClusterIP      10.110.237.70    <none>        8060/TCP,15014/TCP                                                                                                                           97m
istio-egressgateway         ClusterIP      10.103.111.176   <none>        80/TCP,443/TCP,15443/TCP                                                                                                                     97m
istio-galley                ClusterIP      10.103.142.129   <none>        443/TCP,15014/TCP,9901/TCP                                                                                                                   97m
istio-ingressgateway        LoadBalancer   10.101.52.185    <pending>     15020:32122/TCP,80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:30592/TCP,15030:31932/TCP,15031:30442/TCP,15032:31570/TCP,15443:30995/TCP   97m
istio-pilot                 ClusterIP      10.103.174.97    <none>        15010/TCP,15011/TCP,8080/TCP,15014/TCP                                                                                                       97m
istio-policy                ClusterIP      10.108.163.128   <none>        9091/TCP,15004/TCP,15014/TCP                                                                                                                 97m
istio-sidecar-injector      ClusterIP      10.104.15.69     <none>        443/TCP,15014/TCP                                                                                                                            97m
istio-telemetry             ClusterIP      10.101.73.175    <none>        9091/TCP,15004/TCP,15014/TCP,42422/TCP                                                                                                       97m
jaeger-agent                ClusterIP      None             <none>        5775/UDP,6831/UDP,6832/UDP                                                                                                                   96m
jaeger-collector            ClusterIP      10.104.155.177   <none>        14267/TCP,14268/TCP,14250/TCP                                                                                                                96m
jaeger-collector-headless   ClusterIP      None             <none>        14250/TCP                                                                                                                                    96m
jaeger-query                ClusterIP      10.101.50.102    <none>        16686/TCP                                                                                                                                    96m
kiali                       ClusterIP      10.96.65.247     <none>        20001/TCP                                                                                                                                    97m
prometheus                  ClusterIP      10.109.136.108   <none>        9090/TCP                                                                                                                                     97m
tracing                     ClusterIP      10.106.114.110   <none>        80/TCP                                                                                                                                       96m
zipkin                      ClusterIP      10.99.229.224    <none>        9411/TCP   
```

`vim istio-demo.yaml`搜索service下的istio-ingressgateway，将spec.type从LoadBalancer修改为NodePort。

修改完以后执行`kubectl apply -f istio-demo.yaml`

6、将sidecar注入到已经部署好的应用中，`root@lxy-H61H2-CM:/home/lxy/sock_shop/microservices-demo-master/deploy/kubernetes# istioctl kube-inject -f complete-demo.yaml |kubectl apply -f -`

查看状态`kubectl get pod -n sock-shop`

![image-20201009211938705](/home/lxy/文档/k8s安装记录.assets/image-20201009211938705.png)
