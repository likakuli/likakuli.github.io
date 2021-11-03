# docker for mac - 1


## 本篇由来

分享在使用 `Docker for Mac` 过程中遇到的问题，希望可以帮助到遇到问题的人。

## 网络不通

`Docker for Mac` 没有提供从宿主的macOS通过容器IP访问容器的方式。参考 [Known limitations, use cases, and workarounds](https://docs.docker.com/docker-for-mac/networking/#i-cannot-ping-my-containers)。遇到此问题可以参照此文档操作：[mac docker connector](https://github.com/wenjunxiao/mac-docker-connector/blob/master/README-ZH.md)。

## 查看容器文件

`Docker for Mac` 容器是运行在虚拟机中的，在 `MacOS` 上运行了一个虚机。所以通过 `docker inspect` 看到的命令，尤其和文件系统相关的信息，直接在宿主机上是无法找到的，需要找到目录对应的在 `MacOS` 上的目录，网上搜索可以得到两种方案，都是通过 `screen` 命令，后接具体文件，版本不同，文件不同，但最新版都已经不可用。

可以通过一个万能的指令来实现，原理还是利用 linux 的各种 namespace 机制，命令如下

```bash
docker run -it --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh
```

此命令是以特权模式启动一个新容器，共享虚机的 pid namespace，启动之后通过 nsenter 命令进去到进程1 的 mount、uts、net namespaec 中。因为启动的新容器利用了虚机的 pid namespace，所以看到的1号进程就是虚机的1号进程。

多么巧妙的实现~

## Exec Permission denied

`docker run`报错，报如上的错。出现这种情况大概率是在 linux 上打了镜像，在 mac 上运行时出的错。利用上一步中提到的方法进入容器找到对应的文件， 可以参考[这里](https://stackoverflow.com/questions/54336677/error-starting-container-process-caused-exec-docker-entrypoint-sh-permi?rq=1)。

```bash
chmod 755 filename
```




