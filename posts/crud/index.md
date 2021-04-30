# CRUD工程师的福音


## CRUD还能这么玩？

作为一名有多年经验的CRUD工程师，在做某项目时，需要在搭建完apiserver框架以及dao部分，三下五除二的就把apiserver部分搞定了，然后到了dao部分。用屁股想了想网上肯定有现成的工具可以自动生成go struct，于是在网上找了找，果不其然，很多，但基本都是根据table来自动生成go struct的，于是搞定了创建table的sql，创建完table之后顺利的用工具生成了go struct。你以为这就完了吗。当然没有，因为在此过程中我发现了一个**BUG级**的项目：https://github.com/smallnest/gen，点开作者主页一看，原来又是一位自己关注已久的博主：鸟窝，肃然起敬。

## 项目介绍

这个项目能干什么呢？简单来说就是他可以根据你的数据库自动生成go struct及dao层、api层，直接生成一套可运行的restful api项目，而且还搭配了swagger文档。抛开最终生成的代码不说，单就这个思想就已经够我们学习了。

接下来简单介绍下这个项目，为啥简单介绍呢，因为使用起来确实很简单，而且其文档说明比较完善。

### 安装

项目采用golang编写，生成的也是golang代码，安装方式自然也是golang项目的安装方式

`go get -u github.com/smallnest/gen`

### 使用

在项目代码的example文件夹下存在一个sample.db文件，可以直接根据这个文件来生成完成的项目，当然也支持自定义配置来生成自己需要的部分。

```shell
## 根据sample.db生成项目代码
$ gen --sqltype=sqlite3 \
   	--connstr "./sample.db" \
   	--database main  \
   	--json \
   	--gorm \
   	--guregu \
   	--rest \
   	--out ./example \
   	--module example.com/rest/example \
   	--mod \
   	--server \
   	--makefile \
   	--json-fmt=snake \
   	--generate-dao \
   	--generate-proj \
   	--overwrite

## 编译 (使用make命令进行编译，packr2会被自动安装)
$ cd ./example
$ make example

## 二级制位置./bin/example
$ cp ../../sample.db  .
$ ./example


## 打开浏览器访问这里 http://127.0.0.1:8080/swagger/index.html

## 同样可以使用命令行工具进行访问
curl http://localhost:8080/artists
```

> 可能会报错，提示找不到某些文件或者包，安装即可，部分文件判断方式是直接检查GOPATH下是否存在，所以会出现使用go mod模式时get下来之后仍然报错的情况，可以直接到GOPATH下下载对应的项目即可。

是不是so easy，只需要提供数据库即可生成完整项目，但是他的功能不止于此。因为每个人有自己的习惯的编码风格，自动生成的代码不可能满足所有人的需求，那怎么办呢？

### 高级功能

gen支持自定义模板，可以把原模板导出，基于其修改，或者干脆按照自己的风格制作一套模板，通过`--templateDir=`参数指定自己的模板的路径即可

如果只想进行简单改动，可以尝试`--exec=`参数，允许开发者自定义代码生成规则，在项目自带的custom文件夹下有sample.gen文件，可以直接指定`--exec=sample.gen`看执行效果，如果使用的是最新的master，则执行会报错，看代码可以发现在2020/8/4的一次修改，改变了某些方法的形参，而sample.gen没有进行对应修改，按需修改即可。

## 原理

项目用到的主要的技术就是golang的template功能，包括template的常用能力及定制函数等功能。另外包括GIN框架、GORM框架等crud工程师们常用的包。里面还有一个平时不怎么用的包packr，其主要功能就是把静态文件嵌入到最终生成的二进制文件中，程序就可以在任何位置直接运行而不需要受制于额外静态文件的位置等因素。

## 结束语

被CRUD折磨的人啊，是否有考虑过将其自动化呢，虽然业务逻辑是在变化的，但项目框架部分是基本保持不变的，搞一套自己习惯的开箱即用框架岂不是更美，毕竟从头写的话也需要一点时间，懒果然是人类进步的阶梯，赶紧去下载项目试一试吧。
