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
