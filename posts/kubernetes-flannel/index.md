# Flannel key not found


### 问题描述

> etcd 3.3.1
>
> flannel 0.11.0

flannel启动时报错，启动参数如下

```shell
./flannel -etcd-keyfile=/etc/kubernetes/ssl/etcd-client-key.pem -etcd-cafile=/etc/kubernetes/ssl/ca.pem  -etcd-endpoints=https://ip:port -etcd-certfile=/etc/kubernetes/ssl/etcd-client.pem -etcd-prefix=/coreos.com/network
```

错误信息如下：

```shell
E0908 20:27:17.671602    2331 main.go:382] Couldn't fetch network config: 100: Key not found (/coreos.com) [22]
timed out
E0908 20:27:18.680096    2331 main.go:382] Couldn't fetch network config: 100: Key not found (/coreos.com) [22]
timed out
E0908 20:27:19.688339    2331 main.go:382] Couldn't fetch network config: 100: Key not found (/coreos.com) [22]
```

其中coreos.com是启动flannel时-etcd-prefix参数的默认值（/coreos.com/network）

### 解决办法

报错提示很明显，没有对应的key，于是执行etcdctl的命令插入对应的key并设置其值

```shell
ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/ssl/ca.pem --cert=/etc/kubernetes/ssl/etcd-client.pem --key=/etc/kubernetes/ssl/etcd-client-key.pem  --endpoints=https://ip:port put /coreos.com/network/config '{"Network":"192.168.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}'

OK
```

重新启动flannel，依旧报错，执行etcdctl get获取key的信息也可以正常拿到之前的设置，一脸懵逼。网上搜了下说是etcd api版本的问题，不是很明白，然后去看代码，发现flannel在使用etcd时只支持etcd v2版本的api，因为上线添加key-value时使用的是v3版本的api，所以导致虽然添加成功了，但是用v2获取的时候还是失败，解决办法就是用v2版本的api添加一遍即可

```shell
etcdctl --cacert=/etc/kubernetes/ssl/ca.pem --cert=/etc/kubernetes/ssl/etcd-client.pem --key=/etc/kubernetes/ssl/etcd-client-key.pem  --endpoints=https://ip:port set /coreos.com/network/config '{"Network":"192.168.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}'
```

区别就是去掉设置v3的环境变量，put改为set，需要注意一下，在master最新代码中，不设置ETCDCTL_API就默认用v3版本的api，后续使用时还需要具体版本具体对待。

### 收获

etcd不同版本的api对应的url path的prefix不同，v2前缀为/v2/keys，v3前缀为/v3[alpha|beta]/kv，用法也不同，具体可以参考官网API说明。平时直接使用client包时这些信息都会忽略掉，封装的太好了会使使用者变傻，还是有必要看看源码是怎么实现的。
