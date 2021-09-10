1、下载github上的包到本地新建文件夹sock-shop

2、解压文件 `unzip 包名  `

3、选择kubernetes平台安装方式，进入文件夹`/home/lxy/sock_shop/microservices-demo-master/deploy/kubernetes`,修改该目录下的部署文件`complete-demo.yaml`为：complete-demo.yaml

首先创建命名空间：`kubectl create ns sock-shop`

4、创建命名空间`kubectl create namespace sock-shop`

5、在该文件所在目录下运行`kubectl create -f complete-demo.yaml`,后续对该文件有修改的话运行`kubectl apply -f 文件名`进行更新

6、`kubectl get pod -n sock-shop `查看pod状态是否都running，`kubectl get svc -n sock-shop`查看services，以及端口号，如下图：30003端口作为sock-shop的服务入口。浏览器 {NodeIp}:30003-->192.168.1.127:30003进行访问。

