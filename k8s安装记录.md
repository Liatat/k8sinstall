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



# 二、Sock-shop部署

1、下载github上的包到本地新建文件夹sock-shop

2、解压文件 `unzip 包名  `

3、选择kubernetes平台安装方式，进入文件夹`/home/lxy/sock_shop/microservices-demo-master/deploy/kubernetes`,修改该目录下的部署文件`complete-demo.yaml`为：

首先创建命名空间：`kubectl create ns sock-shop`

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

6、`kubectl get pod -n sock-shop `查看pod状态是否都running，`kubectl get svc -n sock-shop`查看services，以及端口号，如下图：30003端口作为sock-shop的服务入口。浏览器 {NodeIp}:30003-->192.168.1.127:30003进行访问。



### 排坑记录

事情的起因：在一台崭新的服务器上部署sock-shop，运行完部署文件以后查看svc和pod发现都是正常的状态，然后开心的在浏览器访问localhost：30003。。。。。漫长的等待啥子也没有；然后在节点上试验了一下，发现只能显示一部分（标头）。这个问题以前也遇见过，当时情景跟这个差不多，原因是配置文件有问题，没有修改对导致selctor找不到对应服务，拼接不起来一个完整的服务。因此此时怀疑是连接上的问题。由于肯定配置文件是正确的，所以肯定是网络连接出现了问题，而且曾经docker ps -a的时候就发现有一个calico的容器停止了。

解决：

参考：https://blog.csdn.net/nklinsirui/article/details/109424100

怀疑是网络原因，本例中的Kubernetes集群用的是Calico网络组件。



登录浏览器只显示一部分的还有另外一种情况就是数据库没有成功连接。。。

排查过程：

首先看一下是哪个server访问不通。直接cluster-ip+port浏览器输入。

```shell
root@cnu101-System-Product-Name:~# kubectl get svc -n sock-shop
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
carts          ClusterIP   10.104.13.127    <none>        80/TCP         297d
carts-db       ClusterIP   10.103.26.239    <none>        27017/TCP      297d
catalogue      ClusterIP   10.101.12.52     <none>        80/TCP         297d
catalogue-db   ClusterIP   10.104.37.102    <none>        3306/TCP       297d
front-end      NodePort    10.106.115.213   <none>        80:30003/TCP   297d
orders         ClusterIP   10.103.201.221   <none>        80/TCP         297d
orders-db      ClusterIP   10.101.130.7     <none>        27017/TCP      297d
payment        ClusterIP   10.109.151.80    <none>        80/TCP         297d
queue-master   ClusterIP   10.96.80.55      <none>        80/TCP         297d
rabbitmq       ClusterIP   10.109.113.59    <none>        5672/TCP       297d
shipping       ClusterIP   10.97.101.144    <none>        80/TCP         297d
user           ClusterIP   10.106.30.217    <none>        80/TCP         297d
user-db        ClusterIP   10.109.107.118   <none>        27017/TCP      297d
```

这次首先catalogue访问不通，那么就可以去看看对应的容器是谁，然后去看容器日至。

```shell
root@cnu101-System-Product-Name:~# docker ps -a | grep catalogue
e4a492223fcf        weaveworksdemos/catalogue-db:0.3.0                              "docker-entrypoint.s…"   7 months ago        Up 6 hours                  3306/tcp                                                   docker-compose_catalogue-db_1
285620a47139        weaveworksdemos/catalogue:0.3.5                                 "/app -port=80"          7 months ago        Up 6 hours                  80/tcp                                                     docker-compose_catalogue_1

```

发现日至上报错连接数据库失败

```shell
docker logs 285620a47139
Abs(images): "/images" (<nil>)
Getwd: "/" (<nil>)
ls: ["images/WAT.jpg" "images/WAT2.jpg" "images/bit_of_leg_1.jpeg" "images/bit_of_leg_2.jpeg" "images/catsocks.jpg" "images/catsocks2.jpg" "images/classic.jpg" "images/classic2.jpg" "images/colourful_socks.jpg" "images/cross_1.jpeg" "images/cross_2.jpeg" "images/holy_1.jpeg" "images/holy_2.jpeg" "images/holy_3.jpeg" "images/puma_1.jpeg" "images/puma_2.jpeg" "images/rugby_socks.jpg" "images/sock.jpeg" "images/youtube_1.jpeg" "images/youtube_2.jpeg"]
ts=2021-09-01T00:46:59Z caller=main.go:108 Error="Unable to connect to Database" DSN=catalogue_user:default_password@tcp(catalogue-db:3306)/socksdb
ts=2021-09-01T00:46:59Z caller=main.go:136 transport=HTTP port=80
```

因为以前连接时没有问题的，所以首先考虑重新启动服务。

```shell
root@cnu101-System-Product-Name:/home/cnu101/sock_shop/microservices-demo-master/deploy/kubernetes# kubectl replace --force -f complete-demo.yaml -n sock-shop
```

问题解决。。。



## 检查Calico

```bash
# Check daemonset 
kubectl get ds -n kube-system -l k8s-app=calico-node

# Check pod status and ready
kubectl get pods -n kube-system -l k8s-app=calico-node

# Check apiservice status
kubectl get apiservice v1.crd.projectcalico.org -o yaml
```

发现`calico-node`的2个Pod都不是READY状态。

查看`calico-node` Pod的详细信息：

```bash
kubectl describe -n kube-system pod -l k8s-app=calico-node
```

查看`calico-node` 某个Pod的日志：

```bash
kubectl logs -f -n kube-system calico-node-<hash>
```

发现错误日志为：“*calico/node is not ready: BIRD is not ready: BGP not established*“

在Calico的Manifest中的 “# Auto-detect the BGP IP address.” 前添加：

```yaml
# Specify interface
- name: IP_AUTODETECTION_METHOD
  value: "interface=eno2"
```

说明：

- `interface=eth0` 表示选择节点机器的`eth0`网卡，可以通过`ip addr show`来查看网卡。
- **这个服务器的网卡命名改了，是eno2**

一个完整的Calico Manifest的例子参见：

- [calico-eth-ens.yaml](https://github.com/cookcodeblog/k8s-deploy/blob/master/kubeadm_v1.19.3/calico/calico-eth-ens.yaml)

`interface`示例：

```bash
# 指定某个网卡
interface=eth0
interface=ens33

# 指定网卡的前缀
interface=eth.*
interface=ens.*

# 指定多个网卡的前缀
interface=eth.*,ens.*

```

> calico_ip_autodetection_method`: This parameter is used to select  the interface on the node, and for pod to pod data communication across  the nodes. Three methods are available to help select the data path for  pods.

参见：

- https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.1/manage_network/calico.html
- https://docs.projectcalico.org/networking/ip-autodetection

更新Calico，再重新检查Calico。

再在busybox中测试通过服务名访问成功。

# 三、Istio安装

卸载命令： istioctl manifest generate --set profile=demo | kubectl delete -f -

1、官网上（github）下载对应版本安装包，采用istioctl方式安装。安装1.6.14版本。

文件结构：

- bin：istioctl工具在的位置
- manifest
  - charts
  - deploy
  - examples
  - profiles: istioctl安装命令使用的文件都在这里
  - translataConfig
- samples：各种安装示例文件（可以看成安装模板库）
  - addons
    - extras:
      - zipkin.yaml:这个是istio自带的zipkin的安装配置文件位置
    - grafana.yaml
    - jaeger.yaml
    - kiali.yaml
    - promrtheus.yaml
  - 其他
- tools

2、复制安装包到`/home/lxy/istio-1.6.14`，cd到该目录解压安装包，`root@lxy-H61H2-CM:/home/lxy/istio-1.6.14# tar -zxvf istio-1.6.14-linux.tar.gz `

3、将 `istioctl` 客户端路径增加到 path 环境变量中，`vim /etc/environment`添加

```shell
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/home/cnu101/istio-1.6.14/bin"
```

然后`source /etc/environment`使它生效。

4、先安装istio自带的zipkin服务，在目录root@cnu101-System-Product-Name:/home/cnu101/istio-1.6.14/samples/addons/extras#下执行：

kubectl apply -f zipkin.yaml

```shell
root@cnu101-System-Product-Name:/home/cnu101/istio-1.6.14/samples/addons/extras# kubectl get svc -n istio-system
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                    AGE
istio-pilot   ClusterIP   10.97.254.130   <none>        15010/TCP,15011/TCP,15012/TCP,8080/TCP,15014/TCP,443/TCP   277d
tracing       NodePort    10.101.101.33   <none>        80:31810/TCP                                               72s
```



5、快速部署istio:

istioctl install --set profile=default --set values.gateways.istio-ingressgateway.type=NodePort --set values.tracing.enabled=true --set values.tracing.provider=zipkin --set values.pilot.traceSampling=100

```shell
root@cnu101-System-Product-Name:/home/cnu101# istioctl install --set profile=default --set values.gateways.istio-ingressgateway.type=NodePort --set values.tracing.enabled=true --set values.tracing.provider=zipkin --set values.pilot.traceSampling=100
Detected that your cluster does not support third party JWT authentication. Falling back to less secure first party JWT. See https://istio.io/docs/ops/best-practices/security/#configure-third-party-service-account-tokens for details.
✔ Istio core installed                                                                                                                                                         
✔ Istiod installed                                                                                                                                                             
✔ Ingress gateways installed                                                                                                                                                   
✔ Addons installed                                                                                                                                                             
✔ Installation complete     
root@cnu101-System-Product-Name:/home/cnu101# kubectl get svc -n istio-system
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                      AGE
istio-ingressgateway   NodePort    10.102.85.243   <none>        15021:31855/TCP,80:30022/TCP,443:32693/TCP,15443:32329/TCP   66s
istio-pilot            ClusterIP   10.97.254.130   <none>        15010/TCP,15011/TCP,15012/TCP,8080/TCP,15014/TCP,443/TCP     277d
istiod                 ClusterIP   10.110.114.6    <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP,853/TCP                104s
prometheus             ClusterIP   10.105.88.28    <none>        9090/TCP                                                     66s
tracing                ClusterIP   10.101.101.33   <none>        80/TCP                                                       7m20s
zipkin                 ClusterIP   10.96.7.223     <none>        9411/TCP                                                     66s
root@cnu101-System-Product-Name:/home/cnu101# kubectl get pod -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-86db79bd5b-mw9sk   1/1     Running   0          2m33s
istio-tracing-6648d8896f-lgfl9          1/1     Running   0          2m33s
istiod-7fc46c9479-zxdwv                 1/1     Running   0          3m10s
prometheus-b74f7ff6f-vlqr7              2/2     Running   0          2m33s
```

6、将sidecar注入到已经部署好的应用中，`root@lxy-H61H2-CM:/home/cnu101/sock_shop/microservices-demo-master/deploy/kubernetes# istioctl kube-inject -f complete-demo.yaml |kubectl apply -f -`

或者以前开启了注入，就重启一下应用服务

7、istioctl dashboard zipkin



其他的附件可以参考部署，一个思路。

## Istio可视化（旧）

默认采用Grafana

1、验证 `prometheus` 服务正在集群中运行。

在 Kubernetes 环境中，执行以下命令：

```bash
$ kubectl -n istio-system get svc prometheus
NAME         CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
prometheus   10.59.241.54   <none>        9090/TCP   2m
```

2、验证 Grafana 服务正在集群中运行。

在 Kubernetes 环境中，执行以下命令：

```bash
$ kubectl -n istio-system get svc grafana
NAME      CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
grafana   10.59.247.103   <none>        3000/TCP   2m
```

3、通过 Grafana UI 打开 Istio Dashboard。

在 Kubernetes 环境中，执行以下命令：

```bash
$ kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```

在浏览器中访问 http://localhost:3000/dashboard/db/istio-mesh-dashboard。





# 四、Fluented-elasticsearch

1、安装helm，二进制安装

https://github.com/helm/helm/releases

版本：helm3.6.3 Linux amd64

![](/home/cnu101/图片/helm.png)

2、artifacthub搜索fluented-elasticsearch

https://artifacthub.io/packages/helm/kokuwa/fluentd-elasticsearch

```shellbash
helm repo add kokuwa https://kokuwaio.github.io/helm-charts
helm install  my-efk kokuwa/fluentd-elasticsearch
```

卸载：helm uninstall my-efk

构建本地仓库：

root@cnu101-System-Product-Name:/opt# mkdir charts

docker run -d -p 8888:8080 -e DEBUG=1 -e STORAGE=local -e STORAGE_LOCAL_ROOTDIR=/charts -v /opt/charts:/charts chartmuseum/chartmuseum:latest
01ffdcc5a4d5304ffab41c17b13754b2d9449b1b28ed7fcb013ef23f5d979fa5
root@cnu101-System-Product-Name:/opt# curl localhost:8888/api/charts
{}



root@cnu101-System-Product-Name:/opt/charts# helm install my-efk kokuwa/fluentd-elasticsearch --set image.tag=v3.0.2 --set fluentdLogFormat=json --set elasticsearch.hosts="192.168.1.127:9200" --set elasticsearch.auth.enabled=true --set elasticsearch.auth.user=ESroot --set elasticsearch.auth.password=ESroot  --set elasticsearch.log400Reason=true

helm install my-efk kokuwa/fluentd-elasticsearch --set image.tag=v3.0.0 --set fluentdLogFormat=json --set elasticsearch.hosts="192.168.1.127:9200" --set elasticsearch.auth.enabled=true --set elasticsearch.auth.user=elastic --set elasticsearch.auth.password=admin@123  --set elasticsearch.log400Reason=true --set fluentdLogFormat=json

helm install my-efk kokuwa/fluentd-elasticsearch --set image.tag=v3.0.0 --set fluentdLogFormat=json --set elasticsearch.hosts="192.168.1.127:9200" --set elasticsearch.auth.enabled=true --set elasticsearch.auth.user=elastic --set elasticsearch.auth.password=admin@123  --set elasticsearch.typeName=myesdoc --set elasticsearch.log400Reason=true

--set elasticsearch.path="/home/cnu101/elasticsearch-6.0.1/path" 
NAME: my-efk
LAST DEPLOYED: Mon Sep  6 20:30:20 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:

1. To verify that Fluentd has started, run:

  kubectl --namespace=default get pods -l "app.kubernetes.io/name=fluentd-elasticsearch,app.kubernetes.io/instance=my-efk"

THIS APPLICATION CAPTURES ALL CONSOLE OUTPUT AND FORWARDS IT TO elasticsearch . Anything that might be identifying,
including things like IP addresses, container images, and object names will NOT be anonymized.
root@cnu101-System-Product-Name:/opt/charts# kubectl --namespace=default get pods -l "app.kubernetes.io/name=fluentd-elasticsearch,app.kubernetes.io/instance=my-efk"
NAME                                 READY   STATUS    RESTARTS   AGE
my-efk-fluentd-elasticsearch-4gsxr   1/1     Running   0          81s
my-efk-fluentd-elasticsearch-cnglt   1/1     Running   0          81s
my-efk-fluentd-elasticsearch-tbdbk   1/1     Running   0          81s

# 五、Elasticresearch

## 准备环节

检查是否安装了jdk

```shell
java -version
```

没有版本信息的话，首先安装一下jdk

去oracle官网下载安装包jdk8

解压到指定目录

```shell
mkdir /usr/lib/jvm
tar -zxvf jdk-8u271-linux-x64.tar.gz -C /usr/lib/jvm
```

修改环境变量`vim ~/.bashrc`

在文件末尾添加

```shell
#set oracle jdk environment
export JAVA_HOME=/usr/lib/jvm/jdk1.8.0_271
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

使修改立即生效`source ~/.bashrc`

## 安装一个node

单机部署elasticsearch

1、下载安装包

```shell
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.0.1.tar.gz
```

2、解压到对应的文件夹下

我的目录-->  /home/cnu101

进入目录创建文件夹path/to/data

​									path/to/logs

```
/home/cnu@101目录结构介绍：
-elasticsearch-6.0.1
	+bin：可执行文件，运行es的命令
	+config：配置文件目录
 	+config/elasticsearch.yml：ES启动基础配置
 	+config/jvm.options：ES启动时JVM配置
 	+config/log4j2.properties：ES日志输出配置文件
	+lib：依赖的jar
	+logs：日志文件夹
	+modules：es模块
	+plugins：可以自己开发的插件
	-path:
		-to:
			+data
			+logs
```

3、配置ES基础设置打开config目录下的elasticsearch.yaml

​	配置集群名

```yaml
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
cluster.name: es_zipkin
#
```

​	配置当前es节点名称（默认是被注释的，并且默认有一个节点名）

```yaml
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
node.name: node-1
#
```

​	配置存储数据的目录路径（用逗号分隔多个位置）和日志文件路径

```yaml
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#
path.data: /home/cnu101/elasticsearch-7.9.0-linux-x86_64/elasticsearch-7.9.0/path/to/data
#
# Path to log files:
#
path.logs: /home/cnu101/elasticsearch-7.9.0-linux-x86_64/elasticsearch-7.9.0/path/to/logs
#
```

​	绑定地址为特定IP地址（设置其它节点和该节点交互的ip地址，如果不设置它会自动判断）默认为0.0.0.0，绑定这台机器的任何一个ip

```yaml
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
network.host: 192.168.1.127
#
# Set a custom port for HTTP:
#
http.port: 9200
#
```

4、启动ES

> 注意：1.5版本之后不支持root启动 ,需要新建用户

1) 创建用户并赋予es安装目录权限

```shell
创建一个esroot用户并设置初始密码
useradd -c 'ES-user' -d /home/cnu101 ESroot
#  -c 注解资料位于/etc/passwd
# -d 设定用户的目录
#  用户名
passwd ESroot

将es安装目录属主权限改为esroot用户(注意包含path文件夹)
chown -R ESroot /home/cnu101/elasticsearch-6.0.1

进入/home/cnu101/elasticsearch-6.0.1切换用户到ESroot
su ESroot
```

进入/bin运行`./elasticsearch`

 如果报 问题：max virtual memory areas vm.max_map_count [65530] is too  low,increase to at least [262144]原因：elasticsearch用户拥有的内存权限太小，至少需要262144  解决：切换到root用户，在/etc/sysctl.conf文件最后添加一行 vm.max_map_count=655360  添加完毕之后，执行命令： sysctl -p

```shell
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
vm.max_map_count=655360
```

再次切换到esroot普通用户，重新启动es服务。

![image-20201110131046487](/home/cnu101/.config/Typora/typora-user-images/image-20201110131046487.png)

网页访问：localhost：9200可以查看

![image-20201110131154350](/home/cnu101/.config/Typora/typora-user-images/image-20201110131154350.png)



参考：

https://blog.csdn.net/ltgsoldier1/article/details/97393154

https://www.cnblogs.com/yidiandhappy/p/7714489.html



## ElasticHD

1、 通过https://github.com/360EntSecGroup-Skylar/ElasticHD/releases下载对应的zip包

2、解压授权

```shell
# 解压
unzip elasticHD_linux_amd64.zip
# 授权，这一步可以自行查看是否有可执行权限，根据实际情况选择执行或者不执行
chmod 777 ./ElasticHD
```

3、运行

```shell
./ElasticHD -p 127.0.0.1:9800 
```

![image-20201111191226727](/home/cnu101/.config/Typora/typora-user-images/image-20201111191226727.png)



## kibana

https://github.com/elastic/helm-charts/tree/master/kibana#install-released-version-using-helm-repository

```
helm repo add elastic https://helm.elastic.co
```

```
helm install kibana elastic/kibana --set imageTag=7.9.0 --set elasticsearchHosts=http://192.168.1.127:9200 
```

export POD_NAME=$(kubectl get pods --namespace default -l "app=kibana,release=kibana" -o jsonpath="{.items[0].metadata.name}")
    echo "Visit http://127.0.0.1:5601 to use Kibana"

```shell
helm install -f values.yaml kibana elastic/kibana
```

# ES

## nfs服务端master

https://blog.csdn.net/gys_20153235/article/details/80516560

下载：

apt-get install rpcbind

apt-get install nfs-kernel-server

准备：

在/home/cnu101/下创建文件夹storage

vim /etc/exports

```
/home/cnu101/storage  192.168.1.*(insecure,rw,sync,no_root_squash)
```

重启服务 /etc/init.d/nfs-kernel-server restart

启动服务：

service rpcbind start
service nfs-kernel-server start

## nfs客户端node01、02、03

sudo apt-get install nfs-common

sudo mkdir /mnt/nfs

mount -t nfs 192.168.1.127:/home/cnu101/storage /mnt/nfs

开机自动挂载

vim /etc/fstab

添加：192.168.1.127:/home/cnu101/storage /mnt/nfs nfs defaults 0 0



## PV PVC StroageClass

https://cloud.tencent.com/developer/article/1754329

报错 mount failed: exit status 32：https://cloud.tencent.com/developer/article/1861347

## ES-cluster

https://cloud.tencent.com/developer/article/1754675





