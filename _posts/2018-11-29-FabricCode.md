---
layout:     post
title:      "源码编译Fabric"
subtitle:   "Fabric学习笔记"
date:       2018-11-29
author:     "Liu Ke"
header-img: "img/BC.jpg"
tags:
    - Hyperledger Fabric
---

> fabric realse-1.3版本（v1.3）

## Contents

1. [安装配置go](#安装配置go)
2. [安装docker](#安装docker)
3. [安装docker-compose](#安装docker-compose)
4. [安装更新pip](#安装更新pip)
5. [安装go相关的工具](#安装go相关的工具)
6. [安装fabric的部分依赖库](#安装fabric的部分依赖库)
8. [对go包做特殊处理](#对go包做特殊处理)
9. [编译fabic](#编译fabic)
	1. [编译orderer节点](#编译orderer节点)
	2. [编译peer节点](#编译peer节点)
	3. [fabric工具编译](#fabric工具编译)
10. [生成docker镜像](#生成docker镜像)


由于国内很多网址被墙，所以从源码编译需要一些步骤来处理。如果没有墙的存在，由源码编译非常简单，`git clone`了Fabric源码之后，配置好go的开发环境，然后直接使用一个`make all`命令即可完成所有的编译步骤。

### 安装配置go  
  
第一步，自然是安装和配置go的开发环境。对于ubuntu系统，如果直接使用`apt-get`方式安装go，版本会非常老。
go官方的下载地址为[https://golang.org/dl/](https://golang.org/dl/ "https://golang.org/dl/")，但是这个地址被墙了，可以从go语言中文网下载[https://studygolang.com/dl](https://studygolang.com/dl)。

```sh
wget https://studygolang.com/dl/golang/go1.11.linux-amd64.tar.gz

#解压
tar -C /usr/local -xzf go1.11.linux-amd64.tar.gz 
```

**配置系统环境变量**：

centos修改`~/.bash_profile`文件，ubuntu修改
`~/.profile`文件，在文件末尾添加如下内容，保存。

```sh
export PATH=$PATH:/usr/local/go/bin 
export GOROOT=/usr/local/go 
export GOPATH=$HOME/go 
export PATH=$PATH:$HOME/go/bin
```

`GOROOT` 为go的安装目录，`GOPATH` 为go的工作目录。使用命令`source ~/.bash_profile`使修改立即生效。我设置`~/go`为go的工作目录，所以需要在创建该文件夹。

```sh
cd ~
mkdir go
mkdir go/src
mkdir go/src/github.com
cd go/src/github.com
mkdir hyperledger
cd hyperledger
git clone -b release-1.3 https://github.com/hyperledger/fabric.git
```

### 安装docker

可以按照docker官方的文档来安装。

CentOS：[https://docs.docker.com/install/linux/docker-ce/centos/](https://docs.docker.com/install/linux/docker-ce/centos/)

Ubuntu：[https://docs.docker.com/install/linux/docker-ce/ubuntu/](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

也可以使用阿里云镜像来安装，阿里云的相关文档为：[https://yq.aliyun.com/articles/110806](https://yq.aliyun.com/articles/110806)

对于阿里云的方式，可以使用如下方式自动安装：

```sh
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

安装完成后需要修改当前用户（我使用的用户叫ke）权限，将当前用户添加到docker用户组里：

```sh
sudo usermod -aG docker ke
```

重新启动系统，添加阿里云docker镜像加速器，可以提升获取Docker官方镜像的速度。
阿里云的官方文档为：[https://cr.console.aliyun.com/cn-beijing/mirrors](https://cr.console.aliyun.com/cn-beijing/mirrors)

```sh
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://nan4hd4m.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 安装docker-compose

参考docker-compose的官网：[https://github.com/docker/compose/releases](https://github.com/docker/compose/releases)，使用如下命令进行安装：

```sh
curl -L https://github.com/docker/compose/releases/download/1.23.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

`sudo chmod +x /usr/local/bin/docker-compose`命令是用于增加权限。

### 安装更新pip

```sh
sudo apt-get update
sudo apt-get install python-pip
sudo pip install --upgrade pip
```

**安装nodejs和npm**。Hyperledger-Fabric目前貌似仅支持**-version 6.x**。可查看官方的详细说明：[https://nodejs.org/en/download/package-manager](https://nodejs.org/en/download/package-manager)。或者nodesource提供的安装教程[https://github.com/nodesource/distributions/blob/master/README.md#debinstall](https://github.com/nodesource/distributions/blob/master/README.md#debinstall)

### 安装go相关的工具

由于`golang.org`这个网址被墙，我们需要从`github`获取到go相关的tool包，步骤如下：

```sh
mkdir –p $GOPATH/src/golang.org
mkdir  $GOPATH/src/golang.org/x
cd $GOPATH/src/golang.org/x
git clone https://github.com/golang/tools.git
git clone https://github.com/golang/lint.git
```

通过如下命令安装fabric用到的一些go的工具：

```sh
go get github.com/kardianos/govendor
go get github.com/golang/lint/golint
go get golang.org/x/tools/cmd/goimports
go get github.com/onsi/ginkgo/ginkgo
go get github.com/axw/gocov/...
go get github.com/client9/misspell/cmd/misspell
go get github.com/AlekSi/gocov-xml
go get github.com/golang/protobuf/protoc-gen-go
go get -u github.com/golang/dep/cmd/dep
```

上面的`go get github.com/golang/lint/golint`可能会由于墙的原因无法安装成功，可以在[https://www.golangtc.com/download/package](https://www.golangtc.com/download/package)地址下载这个第三方包，并安装，链接内有详细安装教程。

或者采用如下方法，先从github上clone对应的库，然后用`go install`进行安装。具体如下，打开`$GOPATH/src/golang.org/x`目录，然后执行`git clone https://github.com/golang/lint.git`来clone这个`lint`包。clone完成之后，`cd lint/golint`，然后用命令安装golint包`go install`。或者在clone完成后直接执行`go install golang.org/x/lint/golint`来进行安装。

Fabric用到了go官方依赖管理工具dep，使用如下命令来安装：

```sh
go get -u github.com/golang/dep/cmd/dep
```

### 安装fabric的部分依赖库

ubuntu:

```
sudo apt-get install libltdl-dev
sudo apt-get install jq
```

CentOS:

```
yum install -y git bzip2 gcc gcc-c++ libtool libltdl-dev libtool-ltdl-devel openssl
yum install jq
``` 

### 对go包做特殊处理

先将之前下载的go相关的工具，拷贝到如下目录下：

```sh
mkdir -p .build/docker/gotools/bin
cp ~/go/bin/* ~/go/src/github.com/hyperledger/fabric/.build/docker/gotools/bin
```

### 编译fabic

官方已经写好了所有的Makefile文件，所以可以在clone的fabric目录下，通过`make`命令来进行编译。

因为墙的存在，我们通过make命令来分别编译每个组件。实际上还有一种方法，就是将go tools等相关的工具包下载下来之后，通过修改Makefile文件来实现正常的编译。

##### 编译orderer节点

```sh
make orderer
```

结果如下：

```sh
.build/bin/orderer
CGO_CFLAGS=" " GOBIN=/root/go/src/github.com/hyperledger/fabric/.build/bin go install -tags "" -ldflags "-X github.com/hyperledger/fabric/common/metadata.Version=1.3.1 -X github.com/hyperledger/fabric/common/metadata.CommitSHA=d200615 -X github.com/hyperledger/fabric/common/metadata.BaseVersion=0.4.13 -X github.com/hyperledger/fabric/common/metadata.BaseDockerLabel=org.hyperledger.fabric -X github.com/hyperledger/fabric/common/metadata.DockerNamespace=hyperledger -X github.com/hyperledger/fabric/common/metadata.BaseDockerNamespace=hyperledger -X github.com/hyperledger/fabric/common/metadata.Experimental=false" github.com/hyperledger/fabric/orderer
Binary available as .build/bin/orderer
```

##### 编译peer节点

这步需要**确保**对之前下载的go tool工具进行处理，来应对墙的存在。同时需要保证**没有ccenv和javaenv两个镜像**，如果有的话，需要使用`docker rmi`删除掉。


```sh
mkdir -p .build/docker/gotools/bin
cp ~/go/bin/* ~/go/src/github.com/hyperledger/fabric/.build/docker/gotools/bin
```

然后使用如下命令进行编译：

```sh
make peer 
```

结果如下：

```sh
.build/bin/peer
CGO_CFLAGS=" " GOBIN=/root/go/src/github.com/hyperledger/fabric/.build/bin go install -tags "" -ldflags "-X github.com/hyperledger/fabric/common/metadata.Version=1.3.1 -X github.com/hyperledger/fabric/common/metadata.CommitSHA=d200615 -X github.com/hyperledger/fabric/common/metadata.BaseVersion=0.4.13 -X github.com/hyperledger/fabric/common/metadata.BaseDockerLabel=org.hyperledger.fabric -X github.com/hyperledger/fabric/common/metadata.DockerNamespace=hyperledger -X github.com/hyperledger/fabric/common/metadata.BaseDockerNamespace=hyperledger -X github.com/hyperledger/fabric/common/metadata.Experimental=false" github.com/hyperledger/fabric/peer
Binary available as .build/bin/peer
```

##### fabric工具编译

```sh
make configtxgen 
make cryptogen 
make configtxlator 
```

结果分别为：

```sh
.build/bin/configtxgen
CGO_CFLAGS=" " GOBIN=/root/go/src/github.com/hyperledger/fabric/.build/bin go install -tags "" -ldflags "-X github.com/hyperledger/fabric/common/tools/configtxgen/metadata.CommitSHA=d200615" github.com/hyperledger/fabric/common/tools/configtxgen
Binary available as .build/bin/configtxgen

.build/bin/cryptogen
CGO_CFLAGS=" " GOBIN=/root/go/src/github.com/hyperledger/fabric/.build/bin go install -tags "" -ldflags "-X github.com/hyperledger/fabric/common/tools/cryptogen/metadata.CommitSHA=d200615" github.com/hyperledger/fabric/common/tools/cryptogen
Binary available as .build/bin/cryptogen

.build/bin/configtxlator
CGO_CFLAGS=" " GOBIN=/root/go/src/github.com/hyperledger/fabric/.build/bin go install -tags "" -ldflags "-X github.com/hyperledger/fabric/common/tools/configtxlator/metadata.CommitSHA=d200615" github.com/hyperledger/fabric/common/tools/configtxlator
Binary available as .build/bin/configtxlator

```


### 生成docker镜像

观察上面的make结果会发现，编译完成后只生成了对应的二进制文件，需要进一步生成docker image镜像才能使用。

```sh
make orderer-docker

make peer-docker

make tools-docker
```

另外一些CouchDB、Kafka以及Zookeeper等工具，使用如下命令生成docker镜像：

```sh
make docker
```

最后可以使用`docker images`查看所有的镜像。













