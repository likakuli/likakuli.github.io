# Knative组件镜像制作


### 背景

> knative 0.14.0

最近在搭建公司级的serverless平台，遇到某些问题，看了源码发现无法通过其扩展机制来解决，遂决定修改源码来解决

### 过程

源码很快修改完了，本地编译通过，knative的组件是容器化运行的，这就需要我们再制作镜像，但是浏览完官方github项目，并未发现有Dockerfile文件，于是决定使用逆向方法通过image反推出来Dockerfile，于是利用之前保存的的shell脚本进行反向解析，如下

```shell
docker history --no-trunc 8b13dd01e81b  | tac | tr -s ' ' | cut -d " " -f 5- | sed 's,^/bin/sh -c #(nop) ,,g' | sed 's,^/bin/sh -c,RUN,g' | sed 's, && ,\n  & ,g' | sed 's,\s*[0-9]*[\.]*[0-9]*\s*[kMG]*B\s*$,,g' | head -n -1

bazel build ...
ko publish knative.dev/net-istio/cmd/webhook 463kB kodata contents, at $KO_DATA_PATH
ko publish knative.dev/net-istio/cmd/webhook 52.9MB go build output, at /ko-app/webhook
```

WTF?

这和我认知里的Dockerfile完全不是一回事啊，赶紧google，首先google搜索了bazel，然后区项目中查看，并没有发现有啥相关的文件，倒是有个.ko.yaml的文件，里面有一条语句，是个镜像名称，然后google搜索了ko，果然，大公司就是不一样，一个ko解决了从diamante编译，打镜像，上传镜像，部署到k8s集群中的所有步骤（心中暗自感叹google是真的牛），当然也支持只把镜像load到本地，而不进行push，也不在k8s中创建，加个--local就好了。

### 总结

其实整个过程还是花了较长时间的，主要有两个原因

1. 欠缺某些知识：这种情况下我们往往无法直接找到正确答案，只能通过踩坑之后逐步排除掉错误答案，才能一步步的找到正确的答案
2. knative比较新（0.14.0），网上很难找到需要的答案

整个过程虽然花费较多时间，但是收获颇丰。之所以写这一篇内容，也是希望为后来人解决一下此类烦恼，在比较紧急时，为大家节省时间，希望可以帮助到一部分人。



更多精彩内容可关注微信订阅号：幼儿园小班工程师
