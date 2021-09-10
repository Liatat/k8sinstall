## nfs 配置
1、先进行nfs_install，即 主节点和从节点配置nfs[参考：Ubuntu18.04下安装NFS详细步骤](https://blog.csdn.net/gys_20153235/article/details/80516560)
2、集群内配置nfs服务[参考:Kubernetes 配置 StorageClass（NFS）](https://cloud.tencent.com/developer/article/1754329)
  - rbac.yaml
  - deployment.yaml
  - class.yaml 注意：class.yaml中的StorageClass的名字
  
 ## ES 安装
参考：[Kubernetes Helm3 部署 ElasticSearch & Kibana 7 集群] (https://cloud.tencent.com/developer/article/1754675)


