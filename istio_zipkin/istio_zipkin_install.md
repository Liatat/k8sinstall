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

2、复制安装包到`/home/lxy/istio-1.6.14`，cd到该目录解压安装包，`root@lxy-H61H2-CM:/home/lxy/istio-1.6.14# tar -zxvf istio-1.6.14-linux.tar.gz`

3、将 `istioctl` 客户端路径增加到 path 环境变量中，`vim /etc/environment`添加

```
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/home/cnu101/istio-1.6.14/bin"
```

然后`source /etc/environment`使它生效。

istio 卸载命令： istioctl manifest generate --set profile=demo | kubectl delete -f -

4、先安装istio自带的zipkin服务，在目录root@cnu101-System-Product-Name:/home/cnu101/istio-1.6.14/samples/addons/extras#下执行：

kubectl apply -f zipkin.yaml

```
root@cnu101-System-Product-Name:/home/cnu101/istio-1.6.14/samples/addons/extras# kubectl get svc -n istio-system
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                    AGE
istio-pilot   ClusterIP   10.97.254.130   <none>        15010/TCP,15011/TCP,15012/TCP,8080/TCP,15014/TCP,443/TCP   277d
tracing       NodePort    10.101.101.33   <none>        80:31810/TCP                                               72s
```

5、快速部署istio:

istioctl install --set profile=default --set values.gateways.istio-ingressgateway.type=NodePort --set values.tracing.enabled=true --set values.tracing.provider=zipkin --set values.pilot.traceSampling=100

```
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