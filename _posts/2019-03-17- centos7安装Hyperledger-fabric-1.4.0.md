
---
layout:     post
title:      centos7 安装 Hyperledger fabric 1.4.0
subtitle:   
date:       2019-03-17
author:     NandyShi
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 超级账本
---

# centos7 安装 Hyperledger fabric 1.4.0
## 软件环境
CentOS-7-x86_64

## 工具准备
### go安装
1.11.5，安装步骤[参考](https://blog.csdn.net/ASN_forever/article/details/87533459)

![](https://i.loli.net/2019/03/17/5c8dc715512f0.png)

### docker安装
1.13.1，安装步骤[参考](https://blog.csdn.net/ASN_forever/article/details/87536535)
![](https://i.loli.net/2019/03/17/5c8dc73480faa.png)

### docker-compose安装
 1.18.0，安装步骤参考上个小节
![](https://i.loli.net/2019/03/17/5c8dc81659b11.png)
### nodejs
需要安装nodejs（要求版本8.9.x）和npm（要求版本5.6.x）
![](https://i.loli.net/2019/03/17/5c8dc83a59153.png)
注意： 在 nvm install v8之后，再 install n，然后 n stable。
如果在node *.js中发现有permission错误，需关注相关软件是否安装成功。
 

### 安装git 
因为要用到git，所以需要先安装git
 ```
yum install git
```
### 下载相关镜像文件
在想要安装fabric的目录下运行以下命令来下载fabric （时间可能会有点久）
```
cd /opt/
git clone https://github.com/hyperledger/fabric.git
```
下载完成后会得到一个fabric文件夹，进入fabric/scripts目录可以看到一个bootstrap.sh脚本
> 注意刚开始是没有fabric-samples这个文件夹的，是执行脚本后生成的

![](https://i.loli.net/2019/03/17/5c8dc94d86152.png)
直接执行bootstrap.sh脚本，就会自动进行fabric相关镜像的下载.
```
./bootstrap.sh
```
当相关镜像全部下载完成后，会自动罗列出下载的内容
![](https://i.loli.net/2019/03/17/5c8dc9b559996.png)


## 构建网络
仍然以root 用户，进入 /opt/fabric/scripts/fabric-samples/fabcar/javascript 目录下，安装相关依赖：
```
cd /opt/fabric/scripts/fabric-samples/fabcar/javascript

npm i node-pre-gyp
npm i node-gyp
yum install gcc-c++
npm install fabric-client 
npm install fabric-ca-client
npm i
```
安装成功之后，剩余步骤可[参考](http://nandy.top/2019/03/08/%E5%9F%BA%E4%BA%8E%E8%B6%85%E7%BA%A7%E8%B4%A6%E6%9C%AC%E7%BC%96%E5%86%99%E7%AC%AC%E4%B8%80%E4%B8%AA%E5%BA%94%E7%94%A8/)
