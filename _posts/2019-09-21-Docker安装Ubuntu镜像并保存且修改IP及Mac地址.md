---
layout:     post
title:      Docker安装Ubuntu镜像并保存且修改IP及Mac地址
subtitle:   入坑易，出坑难。
date:       2019-09-21
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Docker
    - Linux
    - Ubuntu
---

> 本文仅作为一个Docker入坑笔记。
> 主要介绍：
> 1、Docker在Linux上的安装配置；
> 2、在Docker容器中安装一个Ubuntu镜像并保存更改；
> 3、以任意IP地址和Mac地址启动刚才安装的Ubuntu镜像。

### 运行环境
简介一下我的环境，方便参考：**Linux 4.15.0-64-generic #73~16.04.1-Ubuntu SMP Fri Sep 13 09:56:18 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux**

### Docker安装配置
无论何时，都要先找官方文档，对于入门来说，这比任何东西都管用。这里以Ubuntu为例，其他系统类似。
起手update源：
```bash
sudo apt-get update
```
装一些必要的工具（一般来说不是刚装系统的话，都可以略过此步）：
```bash
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```
添加官方key：
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
验证一下搞对没得：
```bash
sudo apt-key fingerprint 0EBFCD88
```
如果输出：
> pub   rsa4096 2017-02-22 [SCEA]
>    9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
> uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
> sub   rsa4096 2017-02-22 [S]

就没问题。
接下来配置稳定版仓库（这里仅示例x86_64 / amd64架构的处理器）：
```bash
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
# 再更新一下源
sudo apt-get update
```
妈耶，终于可以安装Docker了，走起：
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
我安装的时候要下载390MB，看来Docker还是挺大的一个项目。
最后测试一下OK不，跑个HelloWorld：
```bash
sudo docker run hello-world
```

### 安装Ubuntu镜像
#### 安装
对，我要在Ubuntu系统上的Docker里面再装一个Ubuntu，可以理解成Ubuntu套娃！
先搜索一下：
```bash
sudo docker search ubuntu
```
就可以看到各种Ubuntu镜像，然后我们当然是拉取第一个官方的：
```bash
sudo docker pull ubuntu
```
然后查看一下你已有的镜像：
```bash
sudo docker images
```
会看到刚才安装的hello-world和ubuntu。
#### 运行
现在我们来运行这个套娃Ubuntu：
```bash
sudo docker run -ti ubuntu
```
进入之后你的终端就会切换成：
```bash
root@7c529c6e5b94:/#
```
这个**7c529c6e5b94**每次都不一样，可以理解成git中的commit id，等会儿保存镜像时需要用到。
#### 保存
然后我们在这个Ubuntu镜像中可以瞎搞一些事情，比如安装wget啊，net-tools啊等等。退出套娃Ubuntu：
```bash
root@7c529c6e5b94:/# exit
```
搞完了就可以提交保存了，第一个参数是刚才的id，第二个参数是给镜像设置一个自定义名称，并加上latest的tag：
```bash
sudo docker commit 7c529c6e5b94 my-ubuntu-img:latest
```
下一次再进入自己保存的镜像时直接：
```bash
sudo docker run -ti my-ubuntu-img
```
即可。

### 自定义Mac和IP地址
#### Mac地址
如果想给刚才的套娃Ubuntu设置一个Mac地址，很简单，直接带参数run就行啦：
```bash
sudo docker run -ti --mac-address xx:xx:xx:xx:xx:xx my-ubuntu-img
```
进入系统之后可以看看是否设置成功：
```bash
ifconfig -a
# 如果提示没有ifconfig命令，需要先安装net-tools
sudo apt-get install net-tools
```
#### IP地址
我们需要先在Docker中创建一个自定义网络类型，同时指定网段（这里示例命名为my-net）：
```bash
sudo docker network create --subnet=192.168.0.0/16 my-net
```
然后可以通过network命令查看：
```bash
sudo docker network ls
```
使用自定义的IP启动容器：
```bash
sudo docker run -it --network my-net --ip 192.168.0.2 my-ubuntu-img
```
结合上述的Mac地址参数，两者同时修改就是：
```bash
sudo docker run -it --mac-address xx:xx:xx:xx:xx:xx --network my-net --ip 192.168.0.2 my-ubuntu-img
```

### 参考
[https://docs.docker.com/install/linux/docker-ce/ubuntu/](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
[https://blog.csdn.net/mtgege/article/details/78462290](https://blog.csdn.net/mtgege/article/details/78462290)
[https://blog.csdn.net/wanghao_0206/article/details/79583325](https://blog.csdn.net/wanghao_0206/article/details/79583325)
