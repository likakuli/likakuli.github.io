# Pod terminating2


### 1. 背景

承接[上文](../docker-pod-terminating)，近期我们排查弹性云线上几起故障时，故障由多个因素共同引起，列举如下：

- 弹性云在逐步灰度升级docker版本至 `18.06.3-ce`
- 由于历史原因，弹性云启用了docker服务的systemd配置选项 `MountFlags=slave`
- 为了避免dockerd重启引起业务容器重建，弹性云启用了 `live-restore=true` 配置，docker服务发生重启，dockerd与shim进程mnt ns不一致

在以上三个因素合力作用下，线上容器在重建与漂移场景下，出现删除失败的事件。

同样，文章最后也给出了两种解决方案：

- 长痛：修改代码，忽略错误
- 短痛：修改配置，一劳永逸

作为优秀的社会主义接班人，我们当然选择短痛了！依据官方提示 `MountFlags=slave` 与 `live-restore=true` 不能协同工作，那么我们只需关闭二者之一就能解决问题。

与我们而言，docker提供的 `live-restore` 能力是一个很关键的特性。docker重启的原因多种多样，既可能是人为调试因素，也可能是机器的非预期行为，当docker重启后，我们并不希望用户的容器也发生重建。似乎关闭 `MountFlags=slave` 成了我们唯一的选择。

等等，回想一下[docker device busy问题解决方案](https://blog.terminus.io/docker-device-is-busy/)，别人正是为了避免docker挂载泄漏而引起删除容器失败才开启的这个特性。

但是，这个17年的结论真的还具有普适性吗？是与不是，我们亲自验证即可。

### 2. 对比实验

为了验证在关闭 `MountFlags=slave` 选项后，docker是否存在挂载点泄漏的问题，我们分别挑选了一台 `1.13.1` 与 `18.06.3-ce` 的宿主进行实验。实验步骤正如[docker device busy问题解决方案](https://blog.terminus.io/docker-device-is-busy/)所提示，在验证之前，环境准备如下：

- 删除docker服务的systemd配置项 `MountFlags=slave`
- 挑选启用systemd配置项 `PrivateTmp=true` 的任意服务，本文以 `httpd` 为例

下面开始验证：

```shell
////// docker 1.13.1 验证步骤及结果
// 1. 重新加载配置
[stupig@hostname2 ~]$ sudo systemctl daemon-reload
 
 
// 2. 重启docker
[stupig@hostname2 ~]$ sudo systemctl restart docker
 
 
// 3. 创建容器
[stupig@hostname2 ~]$ sudo docker run -d nginx
c89c2aeff6e3e6414dfc7f448b4a560b4aac96d69a82ba021b78ee576bf6771c
 
 
// 4. 重启httpd
[stupig@hostname2 ~]$ sudo systemctl restart httpd
 
 
// 5. 停止容器
[stupig@hostname2 ~]$ sudo docker stop c89c2aeff6e3e6414dfc7f448b4a560b4aac96d69a82ba021b78ee576bf6771c
c89c2aeff6e3e6414dfc7f448b4a560b4aac96d69a82ba021b78ee576bf6771c
 
 
// 6. 清理容器
[stupig@hostname2 ~]$ sudo docker rm c89c2aeff6e3e6414dfc7f448b4a560b4aac96d69a82ba021b78ee576bf6771c
Error response from daemon: Driver overlay2 failed to remove root filesystem c89c2aeff6e3e6414dfc7f448b4a560b4aac96d69a82ba021b78ee576bf6771c: remove /home/docker_rt/overlay2/6c77cfb6c0c4b1e809c47af3c5ff6a4732a783cc14ff53270a7709c837c96346/merged: device or resource busy
 
 
// 7. 定位挂载点
[stupig@hostname2 ~]$ grep -rwn /home/docker_rt/overlay2/6c77cfb6c0c4b1e809c47af3c5ff6a4732a783cc14ff53270a7709c837c96346/merged /proc/*/mountinfo
/proc/19973/mountinfo:40:231 227 0:40 / /home/docker_rt/overlay2/6c77cfb6c0c4b1e809c47af3c5ff6a4732a783cc14ff53270a7709c837c96346/merged rw,relatime shared:119 - overlay overlay rw,lowerdir=XXX,upperdir=XXX,workdir=XXX
/proc/19974/mountinfo:40:231 227 0:40 / /home/docker_rt/overlay2/6c77cfb6c0c4b1e809c47af3c5ff6a4732a783cc14ff53270a7709c837c96346/merged rw,relatime shared:119 - overlay overlay rw,lowerdir=XXX,upperdir=XXX,workdir=XXX
/proc/19975/mountinfo:40:231 227 0:40 / /home/docker_rt/overlay2/6c77cfb6c0c4b1e809c47af3c5ff6a4732a783cc14ff53270a7709c837c96346/merged rw,relatime shared:119 - overlay overlay rw,lowerdir=XXX,upperdir=XXX,workdir=XXX
/proc/19976/mountinfo:40:231 227 0:40 / /home/docker_rt/overlay2/6c77cfb6c0c4b1e809c47af3c5ff6a4732a783cc14ff53270a7709c837c96346/merged rw,relatime shared:119 - overlay overlay rw,lowerdir=XXX,upperdir=XXX,workdir=XXX
/proc/19977/mountinfo:40:231 227 0:40 / /home/docker_rt/overlay2/6c77cfb6c0c4b1e809c47af3c5ff6a4732a783cc14ff53270a7709c837c96346/merged rw,relatime shared:119 - overlay overlay rw,lowerdir=XXX,upperdir=XXX,workdir=XXX
/proc/19978/mountinfo:40:231 227 0:40 / /home/docker_rt/overlay2/6c77cfb6c0c4b1e809c47af3c5ff6a4732a783cc14ff53270a7709c837c96346/merged rw,relatime shared:119 - overlay overlay rw,lowerdir=XXX,upperdir=XXX,workdir=XXX
 
 
// 8. 定位目标进程
[stupig@hostname2 ~]$ ps -ef|egrep '19973|19974|19975|19976|19977|19978'
root     19973     1  0 15:13 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache   19974 19973  0 15:13 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache   19975 19973  0 15:13 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache   19976 19973  0 15:13 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache   19977 19973  0 15:13 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache   19978 19973  0 15:13 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
```

docker `1.13.1` 版本的实验结果正如网文所料，容器读写层挂载点出现了泄漏，并且 `docker rm` 无法清理该容器（注意 `docker rm -f` 仍然可以清理，原因参考上文）。

弹性云启用docker配置 `MountFlags=slave` 也是为了避免该问题发生。

那么现在压力转移到 docker `18.06.3-ce` 这边来了，新版本是否仍然存在这个问题呢？

```shell
////// docker 18.06.3-ce 验证步骤及结果
[stupig@hostname ~]$ sudo systemctl daemon-reload
 
 
[stupig@hostname ~]$ sudo systemctl restart docker
 
 
[stupig@hostname ~]$ sudo docker run -d nginx
718114321d67a817c1498e530b943c2514ed4200f2d0d138880f8c345df7048f
 
 
[stupig@hostname ~]$ sudo systemctl restart httpd
 
 
[stupig@hostname ~]$ sudo docker stop 718114321d67a817c1498e530b943c2514ed4200f2d0d138880f8c345df7048f
718114321d67a817c1498e530b943c2514ed4200f2d0d138880f8c345df7048f
 
 
[stupig@hostname ~]$ sudo docker rm 718114321d67a817c1498e530b943c2514ed4200f2d0d138880f8c345df7048f
718114321d67a817c1498e530b943c2514ed4200f2d0d138880f8c345df7048f
```

针对docker `18.06.3-ce` 的实验非常丝滑顺畅，不存在任何问题。回顾上文知识点，当容器读写层挂载点出现泄漏后，docker `18.06.3-ce` 清理容器必定失败，而现在的结果却成功了，说明容器读写层挂载点没有泄漏。

这简直就是黎明的曙光。

### 3. 蛛丝马迹

上一节对比实验的结果给了我们莫大的鼓励，本节我们探索两个版本的docker的表现差异，以期定位症结所在。

既然核心问题在于挂载点是否被泄漏，那么我们就以挂载点为切入点，深入分析两个版本docker的差异性。我们对比在两个环境下执行完 `步骤4` 后，不同进程内的挂载详情，结果如下：

```shell
// docker 1.13.1
[stupig@hostname2 ~]$ sudo docker run -d nginx
0fe8d412f99a53229ea0df3ec44c93496e150a39f724ea304adb7f924910d61b
 
[stupig@hostname2 ~]$ sudo docker inspect -f {{.GraphDriver.Data.MergedDir}} 0fe8d412f99a53229ea0df3ec44c93496e150a39f724ea304adb7f924910d61b
/home/docker_rt/overlay2/4e09fa6803feab9d96fe72a44fb83d757c1788812ff60071ac2e62a5cf14cd97/merged
 
// 共享命名空间
[stupig@hostname2 ~]$ grep -rw /home/docker_rt/overlay2/4e09fa6803feab9d96fe72a44fb83d757c1788812ff60071ac2e62a5cf14cd97/merged /proc/$$/mountinfo
223 1143 0:40 / /home/docker_rt/overlay2/4e09fa6803feab9d96fe72a44fb83d757c1788812ff60071ac2e62a5cf14cd97/merged rw,relatime - overlay overlay rw,lowerdir=XXX,upperdir=XXX,workdir=XXX
 
 
[stupig@hostname2 ~]$ sudo systemctl restart httpd
 
[stupig@hostname2 ps -ef|grep httpd|head -n 1
root     16715     1  2 16:09 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
 
// httpd进程命名空间
[stupig@hostname2 ~]$ grep -rw /home/docker_rt/overlay2/4e09fa6803feab9d96fe72a44fb83d757c1788812ff60071ac2e62a5cf14cd97/merged /proc/16715/mountinfo
257 235 0:40 / /home/docker_rt/overlay2/4e09fa6803feab9d96fe72a44fb83d757c1788812ff60071ac2e62a5cf14cd97/merged rw,relatime shared:123 - overlay overlay rw,lowerdir=XXX,upperdir=XXX,workdir=XXX
// docker 18.06.3-ce
[stupig@hostname ~]$ sudo docker run -d nginx
ce75d4fdb6df6d13a7bf4270f71b3752ee2d3849df1f64d5d5d19a478ac7db8d
 
[stupig@hostname ~]$ sudo docker inspect -f {{.GraphDriver.Data.MergedDir}} ce75d4fdb6df6d13a7bf4270f71b3752ee2d3849df1f64d5d5d19a478ac7db8d
/home/docker_rt/overlay2/a9823ed6b3c5a752eaa92072ff9d91dbe1467ceece3eedf613bf6ffaa5183b76/merged
 
// 共享命名空间
[stupig@hostname ~]$ grep -rw /home/docker_rt/overlay2/a9823ed6b3c5a752eaa92072ff9d91dbe1467ceece3eedf613bf6ffaa5183b76/merged /proc/$$/mountinfo
218 43 0:105 / /home/docker_rt/overlay2/a9823ed6b3c5a752eaa92072ff9d91dbe1467ceece3eedf613bf6ffaa5183b76/merged rw,relatime shared:109 - overlay overlay rw,lowerdir=XXX,upperdir=XXX,workdir=XXX
 
[stupig@hostname ~]$ sudo systemctl restart httpd
 
[stupig@hostname ~]$ ps -ef|grep httpd|head -n 1
root      63694      1  0 16:14 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
 
// httpd进程命名空间
[stupig@hostname ~]$ grep -rw /home/docker_rt/overlay2/a9823ed6b3c5a752eaa92072ff9d91dbe1467ceece3eedf613bf6ffaa5183b76/merged /proc/63694/mountinfo
435 376 0:105 / /home/docker_rt/overlay2/a9823ed6b3c5a752eaa92072ff9d91dbe1467ceece3eedf613bf6ffaa5183b76/merged rw,relatime shared:122 master:109 - overlay overlay rw,lowerdir=XXX,upperdir=XXX,workdir=XXX
```

咋一看，好像没啥区别啊！睁大你们的火眼金睛，是否发现差异所在了？

如果细心对比，还是很容易分辨出差异所在的：

- 共享命名空间中
  - docker `18.06.3-ce` 版本创建的挂载点是shared的
  - 而docker `1.13.1` 版本创建的挂载点是private的
- httpd进程命名空间中
  - docker `18.06.3-ce` 创建的挂载点仍然是共享的，并且接收共享组109传递的挂载与卸载事件，注意：共享组109正好就是共享命名空间中对应的挂载点
  - 而docker `1.13.1` 版本创建的挂载点虽然也是共享的，但是却与共享命名空间中对应的挂载点没有关联关系

可能会有用户不禁要问：怎么分辨挂载点是什么类型？以及不同类型挂载点的传递属性呢？请参阅：[mount命名空间说明文档](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html)。

问题已然明了，由于两个版本docker所创建的容器读写层挂载点具备不同的属性，导致它们之间的行为差异。

### 4. 刨根问底

相信大家如果理解了上一节的内容，就已经了解了问题的本质。本节我们继续探索问题的根因。

为什么两个版本的docker行为表现不一致？不外乎两个主要原因：

1. docker处理逻辑发生变动
2. 宿主环境不一致，主要指内核

第二个因素很好排除，我们对比了两个测试环境的宿主内核版本，结果是一致的。所以，基本还是因docker代码升级而产生的行为不一致。理论上，我们只需逐个分析docker `1.13.1` 与 docker `18.06.3-ce` 两个版本间的所有提交记录，就一定能够定位到关键提交信息，大力总是会出现奇迹。

但是，我们还是希望能够从现场中发现有用信息，缩小检索范围。

仍然从挂载点切入，既然两个版本的docker所创建的挂载点在共享命名空间中就已经出现差异，我们顺藤摸瓜，找找容器读写层挂载点链路上是否存在差异：

```shell
// docker 1.13.1
// 本挂载点
[stupig@hostname2 ~]$ grep -rw /home/docker_rt/overlay2/4e09fa6803feab9d96fe72a44fb83d757c1788812ff60071ac2e62a5cf14cd97/merged /proc/$$/mountinfo
223 1143 0:40 / /home/docker_rt/overlay2/4e09fa6803feab9d96fe72a44fb83d757c1788812ff60071ac2e62a5cf14cd97/merged rw,relatime - overlay overlay rw,lowerdir=XXX,upperdir=XXX,workdir=XXX
 
 
// 定位本挂载点的父挂载点
[stupig@hostname2 ~]$ grep -rw 1143 /proc/$$/mountinfo
1143 44 8:4 /docker_rt/overlay2 /home/docker_rt/overlay2 rw,relatime - xfs /dev/sda4 rw,attr2,inode64,logbsize=256k,sunit=512,swidth=512,prjquota
 
 
// 继续定位祖父挂载点
[stupig@hostname2 ~]$ grep -rw 44 /proc/$$/mountinfo
44 39 8:4 / /home rw,relatime shared:28 - xfs /dev/sda4 rw,attr2,inode64,logbsize=256k,sunit=512,swidth=512,prjquota
 
 
// 继续往上
[stupig@hostname2 ~]$ grep -rw 39 /proc/$$/mountinfo
39 1 8:3 / / rw,relatime shared:1 - ext4 /dev/sda3 rw,stripe=64,data=ordered
 
 
// docker 18.06.3-ce
// 本挂载点
[stupig@hostname ~]$ grep -rw /home/docker_rt/overlay2/a9823ed6b3c5a752eaa92072ff9d91dbe1467ceece3eedf613bf6ffaa5183b76/merged /proc/$$/mountinfo
218 43 0:105 / /home/docker_rt/overlay2/a9823ed6b3c5a752eaa92072ff9d91dbe1467ceece3eedf613bf6ffaa5183b76/merged rw,relatime shared:109 - overlay overlay rw,lowerdir=XXX,upperdir=XXX,workdir=XXX
 
 
// 定位本挂在点的父挂载点
[stupig@hostname ~]$ grep -rw 43 /proc/$$/mountinfo
43 61 8:17 / /home rw,noatime shared:29 - xfs /dev/sdb1 rw,attr2,nobarrier,inode64,prjquota
 
 
// 继续定位祖父挂载点
[stupig@hostname ~]$ grep -rw 61 /proc/$$/mountinfo
61 1 8:3 / / rw,relatime shared:1 - ext4 /dev/sda3 rw,data=ordered
```

两个版本的docker所创建的容器读写层挂载点链路上差异还是非常明显的：

- 容器读写层挂载点的父级挂载点不同
  - docker `18.06.3-ce` 创建的容器读写层挂载点的父级挂载点是 `/home/` ，并且是共享的
  - docker `1.13.1` 创建的容器读写层挂载点的父级挂载点是 `/home/docker_rt/overlay2` ，并且是私有的

这里补充一个背景，弹性云机器在初始化阶段，会将 `/home` 初始化为xfs文件系统类型，因此所有宿主上 `/home` 挂载点都具备相同属性。

那么，问题基本就是由 docker `1.13.1` 中多出的一层挂载层 `/home/docker_rt/overlay2` 引起。

如何验证这个猜想呢？现在，其实我们已经具备了检索代码的关键目标，docker `1.13.1` 会设置容器镜像层根目录的传递属性。拿着这个先验知识，我们直接查代码，检索过程基本没费什么功夫，直接展示相关代码：

```go
// filepath: daemon/graphdriver/overlay2/overlay.go
func init() {
   graphdriver.Register(driverName, Init)
}
 
func Init(home string, options []string, uidMaps, gidMaps []idtools.IDMap) (graphdriver.Driver, error) {
   if err := mount.MakePrivate(home); err != nil {
      return nil, err
   }
 
   supportsDType, err := fsutils.SupportsDType(home)
   if err != nil {
      return nil, err
   }
   if !supportsDType {
      // not a fatal error until v1.16 (#27443)
      logrus.Warn(overlayutils.ErrDTypeNotSupported("overlay2", backingFs))
   }
 
   d := &Driver{
      home:          home,
      uidMaps:       uidMaps,
      gidMaps:       gidMaps,
      ctr:           graphdriver.NewRefCounter(graphdriver.NewFsChecker(graphdriver.FsMagicOverlay)),
      supportsDType: supportsDType,
   }
 
   d.naiveDiff = graphdriver.NewNaiveDiffDriver(d, uidMaps, gidMaps)
 
   if backingFs == "xfs" {
      // Try to enable project quota support over xfs.
      if d.quotaCtl, err = quota.NewControl(home); err == nil {
         projectQuotaSupported = true
      }
   }
 
   return d, nil
}
```

很明显，问题就出在 `mount.MakePrivate` 函数调用上。

官方将 `GraphDriver` 根目录设置为 `Private`，本意是为了避免容器读写层挂载点泄漏。那为什么在高版本中去掉了这个逻辑呢？显然官方也意识到这么做并不能实现期望的目的，官方也在[修复](https://github.com/moby/moby/pull/36047)中给出了详细说明。

实际上，不设置 `GraphDriver` 根目录的传播属性，反而能避免绝大多数挂载点泄漏的问题。。。

### 5. 结语

现在，我们已经了解了问题的来龙去脉，我们总结下问题的解决方案：

- 针对 `1.13.1` 版本docker，存量宿主较多，我们可以忽略 `device or resource busy` 问题，基本也不会给线上服务带来什么影响
- 针对 `18.06.3-ce` 版本docker，存量宿主较少，我们删除docker服务的systemd配置项 `MountFlags`，通过故障自愈解决docker卡在问题
  - 在容器创建后，卸载容器读写层挂载，如果不影响容器内文件访问。那么可以直接卸载所有挂载点，修改docker配置，并重启docker服务【本方案尚未验证】
- 针对增量宿主，全部删除docker服务的systemd配置项 `MountFlags`



更多精彩内容可关注微信订阅号：幼儿园小班工程师
