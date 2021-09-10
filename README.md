# k8sinstall
记录使用k8s搭建集群的过程，以及后续在此基础上做的一些事。。。。。

# 一、k8s安装记录

# 二、Sock-shop部署

1、下载github上的包到本地新建文件夹sock-shop

2、解压文件 `unzip 包名  `

3、选择kubernetes平台安装方式，进入文件夹`/home/lxy/sock_shop/microservices-demo-master/deploy/kubernetes`,修改该目录下的部署文件为[complete-demo.yaml](https://github.com/Liatat/liatatk8sinstall.github.io/edit/main/complete-demo.yaml)

4、创建命名空间`kubectl create namespace sock-shop`

5、在该文件所在目录下运行`kubectl create -f complete-demo.yaml`,后续对该文件有修改的话运行`kubectl apply -f 文件名`进行更新

6、`kubectl get pod -n sock-shop `查看pod状态是否都running，`kubectl get svc -n sock-shop`查看services，以及端口号，如下图：30003端口作为sock-shop的服务入口。浏览器 {NodeIp}:30003-->192.168.1.111:30003进行访问。



# 三、Istio安装

## 1.5.10版本安装

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

4、快速部署istio，cd 到install/kubernetes目录里面执行`root@lxy-H61H2-CM:/home/lxy/istio/istio-1.5.10/install/kubernetes# kubectl apply -f istio-demo.yaml`

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

## Istio可视化

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

## 1.6.14版本安装
参考官网
### 安装istio及其自带zipkin
```shell
#安装istio，并进行自定义配置，此时采用的自带的zipkin
istioctl manifest apply --set profile=demo --set values.gateways.istio-ingressgateway.type=NodePort --set values.grafana.enabled=true --set values.grafana.service.type=NodePort --set values.tracing.enabled=true --set values.tracing.provider=zipkin --set values.pilot.traceSampling=100.0
```
安装完成后可以参考排坑记录上的方式修改zipkin服务类型为NodePort。

### 安装istio绑定已安装zipkin（未检验）

```shell
#安装istio， 并采用自己定义的zipkin（将zipkin已经部署到了zipkin-ns命名空间）
istioctl manifest apply --set profile=demo --set values.gateways.istio-ingressgateway.type=NodePort --set values.grafana.enabled=true --set values.grafana.service.type=NodePort --set values.global.tracer.zipkin.address=zipkin.zipkin-ns:9411 --set values.pilot.traceSampling=100.0
```

# 四、Elasticresearch

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

同级目录创建文件夹path/to/data

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
path.data: /home/cnu101/path/to/data
#
# Path to log files:
#
path.logs: /home/cnu101/path/to/logs
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
passwd esroot

将es安装目录属主权限改为esroot用户(注意包含path文件夹)
chown -R ESroot /home/cnu101/elasticsearch-6.0.1

切换用户到esroot
su esroot
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

