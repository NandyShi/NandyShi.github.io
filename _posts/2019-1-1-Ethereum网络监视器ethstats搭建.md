---
layout:     post
title:      Ethereum 网络监视器 ethstats 搭建
subtitle:   
date:       2019-1-1
author:     NandyShi
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - ethstats
---


# Ethereum 网络监视器 ethstats 搭建


ethstats 提供了一个 Web-based 的状态监视器，可以通过节点端获取ETH网络节点的状态，來查看整个区块链的状态。本文将说明如何在linux中安装ethstats。

首先可以访问官方的 https://ethstats.net/ 查看主网节点的状态

## 安裝步骤

本部分说明如何正确安装 eth-netstats服务，包含以下两个部分：

- Monitoring site
- Client side


### Monitoring site

#### 安裝使用到的相关工具：

以ubuntu为例：（centos 或其他系统自行安装）
```
$ sudo apt-get install -y make g++ git
$ sudo apt-get install nodejs
```

#### 启动eth-netstats
```
$ git clone https://github.com/cubedro/eth-netstats
$ cd eth-netstats
$ sudo npm install
$ sudo npm install -g grunt-cli
$ grunt
$ PORT="3000" WS_SECRET="admin" npm start
```
在沒有任何 Clinet节点连上情況下，访问http://yourIP:3000 会看到空白网页。

也可以通过脚本eth-netstats.sh放置到后台执行：
```
#!/bin/bash
# History:
# nandyshi
#2019-01-14
export PORT="3000"
export WS_SECRET="admin"

echo "Starting private eth-netstats ..."
screen -dmS netstats /usr/bin/npm start

$ chmod u+x eth-netstats.sh
$ ./eth-netstats.sh
```
>Starting private eth-netstats ...


### Client side

#### 安裝要使用到的工具：

```
$ sudo apt-get install -y make g++ git
$ curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
$ sudo apt-get install nodejs
```
#### 安装eth-net-intelligence-api

```
$ git clone https://github.com/cubedro/eth-net-intelligence-api
$ cd eth-net-intelligence-api
$ sudo npm install && sudo npm install -g pm2
```

#### 编辑app.json，粘贴以下內容：

```
[
  {
    "name"        : "mynode",
    "cwd"         : ".",
    "script"      : "app.js",
    "log_date_format"   : "YYYY-MM-DD HH:mm Z",
    "merge_logs"    : false,
    "watch"       : false,
    "exec_interpreter"  : "node",
    "exec_mode"     : "fork_mode",
    "env":
    {
      "NODE_ENV"    : "test",
      "RPC_HOST"    : "localhost",
      "RPC_PORT"    : "8545",
      "INSTANCE_NAME"   : "mynode-1",
      "WS_SERVER"     : "http://localhost:3000",
      "WS_SECRET"     : "admin",
    }
  },
   {
    "name"        : "mynode2",
    "cwd"         : ".",
    "script"      : "app.js",
    "log_date_format"   : "YYYY-MM-DD HH:mm Z",
    "merge_logs"    : false,
    "watch"       : false,
    "exec_interpreter"  : "node",
    "exec_mode"     : "fork_mode",
    "env":
    {
      "NODE_ENV"    : "test",
      "RPC_HOST"    : "localhost",
      "RPC_PORT"    : "8545",
      "INSTANCE_NAME"   : "mynode-2",
      "WS_SERVER"     : "http://localhost:3000",
      "WS_SECRET"     : "admin",
    }
  }
]
```
- RPC_HOST： ethereum 的 rpc ip address。
- RPC_PORT： ethereum 的 rpc port。
- INSTANCE_NAME： ethereum 实例名称。
- WS_SERVER： eth-netstats 的 URL。
- WS_SECRET： eth-netstats 的 secret。

#### 启动服务：

```
$ pm2 start app.json
$ sudo tail -f $HOME/.pm2/logs/mynode-out-0.log
```
