---
layout:     post
title:      "EOS配置MongoDB"
subtitle:   "EOS笔记"
date:       2019-02-14
author:     "Liu Ke"
header-img: "img/in-post/eos.jpg"
tags:
    - EOS
---

> v1.4.0

## 启动MongoDB

在启动MongoDB之前，当然还是先配置好eos节点的开发环境，具体请看[EOS开发环境搭建](http://keliu.me/2018/09/13/eosEnv/)。

使用官方脚本编译安装了节点程序之后，eos会自动将所需的依赖全部安装好，包括MongoDB。MongoDB默认安装在`~/opt/mongodb`目录下。

打开`~/opt/mongodb`目录下的`mongod.conf`文件，修改好配置信息：

```conf
systemLog:
 destination: file
 path: /home/ke/opt/mongodb/log/mongodb.log
 logAppend: true
 logRotate: reopen
net:
 bindIp: 0.0.0.0,::27017
 ipv6: true
storage:
 dbPath: /home/ke/opt/mongodb/data
```

这里主要是`bindIP`和`dbPath`。MongoDB默认端口一般为27017。如果数据库比较大，home挂载的分区空间不足，可以通过修改`dbPath`保存在别的路径中。


配置好MongoDB之后，通过如下命令启动MongoDB，如果需要后台运行，可以配合`nohup`和`&`一起使用：

```sh
sudo ~/opt/mongodb/bin/mongod -f ~/opt/mongodb/mongod.conf
```

## 配置nodeos

修改`~/.local/share/eosio/nodeos/config`目录下的`config.ini`配置文件，增加如下两条配置内容，启动mongodb插件和配置mongodb_uri：

```ini
# MongoDB URI connection string, see: https://docs.mongodb.com/master/reference/connection-string/. If not specified then plugin is disabled. Default database 'EOS' is used if not specified in URI. Example: mongodb://127.0.0.1:27017/SEAT (eosio::mongo_db_plugin)
mongodb-uri = mongodb://localhost:27017/eosmainnet

plugin = cubetrain::mongo_db_plugin
```

这里的`eosmainnet`为要写入MongoDB数据库的名字，eos会自动创建该数据库。同时可以根据需要，修改`config.ini`配置文件中与MongoDB数据库有关的配置。

## 启动nodeos

如果区块链是一条新链，则通过指定`genesis.json`文件文件来启动。

如果之前存在区块信息，则通过如下命令启动：

```sh
nodeos -–replay-blockchain
```

如果链非法停止造成了数据错误，则使用：

```sh
nodeos --hard-replay-blockchain
```

如果之前的命令都不行，那谨慎使用`--delete-all-blocks`参数来删除所有区块，重新同步区块信息。

以上命令可以配合`--mongodb-wipe`这个插件来清空MongoDB数据库中的旧数据。

成功启动后，eos终端log中会显示加载`mongo_db_plugin`保存数据。此时可以通过MongoDB相关的命令，查看MongoDB数据库中的表和数据。

## MongoDB可视化工具

可视化工具[Robo 3T](https://robomongo.org/download)，可以非常方便的查询和管理MongoDB数据库。

打开上文链接，点击`Download Robo 3T`按钮，下载程序完成后解压到`~/top`目录下，打开`robo3t-1.2.1-linux-x86_64-3e50a65/bin/robo3t`启动程序。

点击Create创建一个连接，在弹出的配置选项卡上填写配置信息，Name填写MongoDb@localhost，Address填写localhost，点击save保存。在管理窗口点击连接，进入后数据库管理界面。就可以很方便的查看和管理我们之前创建的eosmainnet数据库了。


