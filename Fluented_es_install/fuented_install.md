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




