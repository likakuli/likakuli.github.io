# Java服务突现毛刺


### 背景

容器原生设计为单进程模型，但公司线上运行的服务以多进程的方式运行，而且里面包含了很多的agent，例如日志采集、监控采集、数据配送等，耦合在了一个Container中，经过对线上资源使用率分析发现很大一部分资源消耗是在agent部分，而且与业务进程同时争抢业务容器申请的资源，彼此影响。虽然增量的容器部分agent迁移到了sidecar里面，解决了这些问题，但存量问题也需要解决，为此专门搞了一个项目用来优化这些问题。思想就是把agent进程从业务进程所在的cgroup中迁移出去，以不同cgroup层级存在，就可以避免相互影响，也可以限制各自资源大小，但是在灰度过程中发现部分Java容器服务开始出现毛刺。

### 排查过程

java服务毛刺问题在最早上云的时候就出现过，当时是因为jdk版本太低，在容器内运行时无法正确获取容器申请的cpu大小，导致创建过多的线程，从而导致容器内的进程内部争抢过高，业务开始出现毛刺。在某个版本（jdk8u191）之后，开发者完全不需要关注程序是运行在物理机还是容器环境下。

对比有问题的容器内的业务进程使用的jdk版本，结果（202 > 191）居然是没问题的版本，也就是已经可以自动识别cpu核数了，但为什么还是出现问题了呢？

首先需要确定下容器内java服务获取到的真正的cpu核数，可以执行如下命令

```shell
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintContainerInfo -version
```

输出结果中可以看到获取到的核数确实不是我们申请的核数，也就是说虽然采用了没问题的jdk版本，但还是获取到了错误的核数。

接下来就是看下为什么会获取到错误的核数信息，可以使用strace来分析java服务启动过程中的函数调用信息，其中在获取cpu核数的时候比较奇怪，正常是从cpu子系统获取，但是结果却显示从cpu_mirror获取，而这个目录下没有保存cpu核数的文件。后经过确定，这个目录就是前面提到的做agent隔离的时候自定义的挂在目录，容器内位于/sys/fs/cgroup下面。到此基本可以猜测是因为我们自己mount了一个目录到cgroup目录下，导致jdk获取到了错误目录，参数也获取错误。

最后，为什么jdk获取到了错误的路径呢？参考 [Container Support doesn't work for some Join Controllers combinations](https://hg.openjdk.java.net/jdk-updates/jdk11u/rev/426fae9c5382)，代码比较好理解，其实就是有问题的版本在获取路径时采用的是字符串截取，而不是精确匹配，导致cpu_mirror这种目录会被误认为是正确目录。

老版处理/prof/self/mountinfo直接遍历每行，用strstr判断是否包含cpu、memory等，包含就是找到了，处理/proc/self/cgroup的时候根据 **冒号** 分隔，去掉第一列序号，然后剩下的再取第一列，然后用strstr比较是不是包含cpu、memory等

新版针对/proc/self/mountinfo读一条数据，最后一列根据 **逗号** 分隔，然后用strcmp比较是否和cpu、memory相等，处理 /proc/self/cgroup的时候根据 **冒号** 分隔，去掉第一列序号，然后剩下的再取第一列 ，然后根据**逗号**分隔，遍历结果用strcmp比较是否是cpu、memory等

总结就是老版本 直接if contains(str, "cpu") then set cpu subsystme 新版本 for subStr in split(str, ",") if subStr equals "cpu" then set cpu subsystem

### 解决方案

让业务升级jdk版本无法快速解决此问题，最终决定把挂载的cpu_mirror等问题目录umount掉，容器内看不到的话也就不会出问题了，经过测试发现可行，最终又在宿主上批量执行了umount的操作（为后面一个线上问题埋下了伏笔，详见[这篇](../docker-cgroup-unknown)），长期方案是负责隔离的agent把问题目录改名，避免字符串匹配时匹配到。

