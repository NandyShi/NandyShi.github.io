---
layout:     post
title:      "eos多节点测试主网搭建"
subtitle:   "EOS笔记"
date:       2018-11-15
author:     "Liu Ke"
header-img: "img/in-post/eos.jpg"
tags:
    - EOS
---

> eos v1.2.4


## Contents

1. [创建钱包和密钥](#创建钱包和密钥)
2. [导入私钥](#导入私钥)
3. [修改创世节点和keosd配置文件](#修改创世节点和keosd配置文件)
4. [启动nodeos节点服务](#启动nodeos节点服务)
5. [创建系统账户](#创建系统账户)
6. [部署系统合约并发行token](#部署系统合约并发行token)
7. [创建节点账户和普通账户](#创建节点账户和普通账户)
8. [注册出块节点](#注册出块节点)
9. [其他节点配置和启动](#其他节点配置和启动)
10. [投票质押赎回](#投票质押赎回)
11. [解除eosio账户特权](#解除eosio账户特权)


本文梳理一下由一个创世节点和三个出块节点启动eos测试主网，发行token，并实现出块节点轮流出块的过程。

### 创建钱包和密钥

首先在四台机器中确定一台，为创世节点。clone了eos代码成功编译安装之后，创建钱包。

```sh
$ cleos create wallet --to-console

$ cleos create key --to-console

```
然后创建密钥对，这里我创建6对密钥对，分别对应4个节点账户和2个普通账户。

### 导入私钥

将生成的6对密钥的私钥对全部导入到钱包中。

```sh
$ cleos wallet import 
```

### 修改创世节点和keosd配置文件

创世节点的`~/.local/share/eosio/nodeos/config`目录下，修改`config.ini`节点配置。通过`nodeos`命令启动节点服务之后，会在该目录下自动生成该配置文件。

node配置需要修改的参数有如下几个：

```ini
#改为本机自己的IP或0.0.0.0
bnet-endpoint = 0.0.0.0:4321

#本地节点的rpc访问地址，节点自己的ip或者0.0.0.0，默认端口为8888
http-server-address = 0.0.0.0:8888

#bp节点间的访问地址，改成节点机器自己的IP，默认端口为9876
p2p-listen-endpoint = 0.0.0.0:9876

#链状态数据库的大小，如果设置过小，可能会因链状态数据库处于不安全水平导致节点程序挂掉，与机器内存和硬盘空间有关，最好设置的与机器内存大小相近，默认的数值较小
chain-state-db-size-mb = 7168

# 链数据库可用空间低于这个数值，就会关闭节点程序
chain-state-db-guard-size-mb = 128

#节点的名字
agent-name = "EOS test Agent"
#出块的节点的账户名，创世节点必须为eosio
producer-name = eosio

#节点的私钥，网络启动之后，创世节点不能修改私钥
signature-provider = EOS6cDratMo8CQuEF4Xv1j4woaDaaiW7PagxVj3U8hckQTivq5toF=KEY:5KbHF8P67fJ4uqti6V85nzbufVLm5D2ogFretauDyAgfQ67t95V

#创世节点必须为true
enable-stale-production = true

#添加另外三个bp节点的访问地址
p2p-peer-address =

# 可以改成一个较大的值，如3000,防止手动启动主网时报交易超时的错误
max-transaction-time = 30

#将这个参数设置为*可以保证通过get action操作获取到交易的信息
filter-on = *

#默认为1，改成false。如果设置为false，那么任何传入的“Host”报头都被认为是有效的。通过postman、getman等测试rpc接口就会正常调用。
http-validate-host = false

#添加启动节点服务时加载的插件
plugin = eosio::chain_api_plugin
plugin = eosio::history_plugin
plugin = eosio::history_api_plugin
plugin = eosio::producer_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::history_plugin
plugin = eosio::history_api_plugin
plugin = eosio::producer_plugin
```

对于keosd配置文件，为`~/eos-wallet/config.ini`。keosd配置文件需要修改的：
```ini
#本地节点的钱包服务keosd地址，改成创世节点机器自己的IP或者0.0.0.0，默认端口为8888，需更改端口号使之与node节点端口号不同，避免冲突。我这里改为8889
http-server-address = 0.0.0.0:8889

#默认为1，改成false。如果设置为false，那么任何传入的“Host”报头都被认为是有效的。通过postman、getman等测试钱包rpc接口就会正常调用。
http-validate-host = false

#钱包自动锁定的时间，单位为秒，可根据需要进行修改
unlock-timeout = 900
```

### 启动nodeos节点服务

使用如下命令启动nodeos，会将创世节点初始化信息保存在genesis.json文件中，其他三个节点使用这个文件来启动nodeos，能够保证网络状态信息的一致。
```sh
nodeos --extract-genesis-json genesis.json
```

### 创建系统账户


```sh
cleos --url http://172.17.0.9:8888 create account eosio eosio.bpay EOS6cDratMo8CQuEF4Xv1j4woaDaaiW7PagxVj3U8hckQTivq5toF
cleos --url http://172.17.0.9:8888 create account eosio eosio.msig EOS6cDratMo8CQuEF4Xv1j4woaDaaiW7PagxVj3U8hckQTivq5toF
cleos --url http://172.17.0.9:8888 create account eosio eosio.names EOS6cDratMo8CQuEF4Xv1j4woaDaaiW7PagxVj3U8hckQTivq5toF
cleos --url http://172.17.0.9:8888 create account eosio eosio.ram EOS6cDratMo8CQuEF4Xv1j4woaDaaiW7PagxVj3U8hckQTivq5toF
cleos --url http://172.18.0.1:8888 create account eosio eosio.ramfee EOS6cDratMo8CQuEF4Xv1j4woaDaaiW7PagxVj3U8hckQTivq5toF
cleos --url http://172.18.0.1:8888 create account eosio eosio.saving EOS6cDratMo8CQuEF4Xv1j4woaDaaiW7PagxVj3U8hckQTivq5toF
cleos --url http://172.18.0.1:8888 create account eosio eosio.stake EOS6cDratMo8CQuEF4Xv1j4woaDaaiW7PagxVj3U8hckQTivq5toF
cleos --url http://172.18.0.1:8888 create account eosio eosio.token EOS6cDratMo8CQuEF4Xv1j4woaDaaiW7PagxVj3U8hckQTivq5toF
cleos --url http://172.18.0.1:8888 create account eosio eosio.vpay EOS6cDratMo8CQuEF4Xv1j4woaDaaiW7PagxVj3U8hckQTivq5toF
```

### 部署系统合约并发行token

```sh
cleos -u http://172.18.0.1:8888 set contract eosio.token ~/eosio/build/contracts/eosio.toke/
cleos -u http://172.18.0.1:8888 set contract eosio.msig  ~/eosio/build/contracts/eosio.msig/
cleos -u http://172.18.0.1:8888 push action eosio.token create '["eosio", "1000000000.0000 SYS",0,0,0]' -p eosio.token
cleos -u http://172.18.0.1:8888 push action eosio.token issue '["eosio", "1000000000.0000 SYS", "issue"]' -p eosio


cleos -u http://172.18.0.1:8888 set contract eosio ~/eosio/build/contracts/eosio.system/ 
cleos -u http://172.18.0.1:8888 push action eosio setpriv '["eosio.msig", 1]' -p eosio@active
```

### 创建节点账户和普通账户

我的三个节点分别命名为aaa，bbb，ccc，两个普通账户命名为user1，user2。

```sh
cleos -u http://172.18.0.1:8888 system newaccount --transfer eosio aaa EOS7V5hcYss6e89yK7W5Kq4x9iXWHM64F4VWHQdhP4TTakAJYjLgS --stake-net "1000 SYS" --stake-cpu "1000 SYS" --buy-ram "1000 SYS"   

cleos -u http://172.18.0.1:8888 system newaccount --transfer eosio bbb EOS7YQP2MxbRbSqJJ89VZJBqs44biQkzqDQj3Cxb8hVaXWZdUYpcm --stake-net "1000 SYS" --stake-cpu "1000 SYS" --buy-ram "1000 SYS"   

cleos -u http://172.18.0.1:8888 system newaccount --transfer eosio ccc EOS8b1DYUPTH6BE65jDxzqkP95PxwStoPqfV4VWHiZX5z2sR2rVJw --stake-net "1000 SYS" --stake-cpu "1000 SYS" --buy-ram "1000 SYS"   

cleos -u http://172.18.0.1:8888 system newaccount --transfer eosio user1 EOS5Vz3XzoBEjbk99yZU31cijzdLcYdADvjxWMFAQu6beUBTwdjWV --stake-net "1000 SYS" --stake-cpu "1000 SYS" --buy-ram "1000 SYS"   
   
```

投票的时候会有一个问题：如果是eosio账户直接转账出来的token进行抵押投票不会改变total_activated_stake的值，但是会影响投票比率。total_activated_stake除10000及为已经投票的token数量。所以这里eosio将准备投票的token先转给user1，然后**user2直接由user1创建，创建的时候就给user2配置好足够的net和cpu**，这样的话，创建完成之后，user2直接进行投票操作即可。

```sh
cleos -u http://172.18.0.1:8888 transfer eosio user1 "900000000.0000 SYS"

cleos -u http://172.18.0.1:8888 system newaccount --transfer eosio user2 EOS6you4AMUo4K7qoLGMkTrp7qLyMe5FtpBmsSUNsVKspQhKCNqzX --stake-net "400000000 SYS" --stake-cpu "400000000 SYS" --buy-ram "1000 SYS" 

```

查看账户资金：

```sh
cleos -u http://172.18.0.1:8888 get currency balance eosio.token user2
```

获取账户和投票信息：
```sh
cleos -u http://172.18.0.1:8888 get account user2
```


### 注册出块节点

```sh
cleos -u http://172.18.0.1:8888  system regproducer aaa EOS7V5hcYss6e89yK7W5Kq4x9iXWHM64F4VWHQdhP4TTakAJYjLgS

cleos -u http://172.18.0.1:8888  system regproducer bbb EOS7YQP2MxbRbSqJJ89VZJBqs44biQkzqDQj3Cxb8hVaXWZdUYpcm

cleos -u http://172.18.0.1:8888  system regproducer ccc EOS8b1DYUPTH6BE65jDxzqkP95PxwStoPqfV4VWHiZX5z2sR2rVJw

```

查看出块节点列表：

```sh
cleos -u http://172.18.0.1:8888 system listproducers
```

### 其他节点配置和启动

另外三个节点的`config.ini`配置文件主要修改的有：

```ini
#改为本机自己的IP
bnet-endpoint = 

#本地节点的rpc访问地址，改成创世节点机器自己的IP，默认端口为8888
http-server-address = 

#bp节点间的访问地址，改成节点机器自己的IP，默认端口为9876
p2p-listen-endpoint = 

#节点的名称
agent-name = 
#节点的账户名
producer-name = 
#节点的公私钥（需要妥善保存，不能上传到github等地方）
signature-provider =
 
#除创世节点外，其他节点该配置项都设置为false
enable-stale-production = false
#添加除自己之外的所有节点的访问地址
p2p-peer-address = 

#添加另外三个bp节点的访问地址
p2p-peer-address =

# 可以改成一个较大的值，如3000,防止手动启动主网时报交易超时的错误
max-transaction-time = 30

# 数据库相关的设置，如果设置过小，在网络运行过程中会报数据库错误而断掉
chain-state-db-size-mb = 65535
reversible-blocks-db-size-mb = 2048

#将这个参数设置为*可以保证通过get action操作获取到交易的信息
filter-on = *
access-control-allow-origin = *
access-control-allow-headers = *
access-control-allow-credentials = false

#默认为1，改成false。如果设置为false，那么任何传入的“Host”报头都被认为是有效的。通过postman、getman等测试rpc接口就会正常调用。
http-validate-host = false

#最大交易时间，可以根据实际情况设置大一点，防止手动启动主网时因为超时而报错
max-transaction-time = 3000


#添加启动节点服务时加载的插件
plugin = eosio::chain_api_plugin
plugin = eosio::history_plugin
plugin = eosio::history_api_plugin
plugin = eosio::producer_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::history_plugin
plugin = eosio::history_api_plugin
plugin = eosio::producer_plugin
```

所有设置自己ip都可以改为0.0.0.0。

配置好之后，通过创世节点的`genesis.json`文件来分别启动这三个节点。需要注意的是，使用`genesis.json`文件启动仅限**第一次**启动，之后链正常运行时，启动节点直接用`nodeos`命令启动即可。`$GENESIS_DIR`为从创世节点拷贝过来的`genesis.json`文件所在的目录。

```sh
nodeos --genesis-json $GENESIS_DIR/genesis.json
```

**注意参数：**
```sh
# 用于存储链状态的数据库尺寸,以兆为单位
chain-state-db-size-mb = 10240

# 用于存储链状态的数据库尺寸低于多少兆时安全的退出节点
chain-state-db-guard-size-mb = 100

# 用于存储不可逆的区块数据 的 数据库尺寸,以兆为单位
reversible-blocks-db-size-mb = 10240

# 用于存储不可逆区块的数据库尺寸低于多少兆时安全的退出节点
reversible-blocks-db-guard-size-mb = 100
```

这几个参数也要注意，如果设置太小，后续节点可能会自关闭。网络运行一段时间后，可能会报数据库水平不安全错误：

```sh
Database has reached an unsafe level of usage, 
shutting down to avoid corrupting the database.

Please increase the value set for "chain-state-db-size-mb" and restart the process!
Details: 3060101 database_guard_exception: Database usage is at unsafe levels
database free: 134215280, guard size: 134217728
    {"f":134215280,"g":134217728}
    thread-0  controller.cpp:1627 validate_db_available_size

shutdown..
```

### 投票质押赎回

**投票**
```sh
cleos -u http://172.18.0.1:8888  system voteproducer prods user2 aaa

cleos -u http://172.18.0.1:8888  system voteproducer prods user2 bbb

cleos -u http://172.18.0.1:8888  system voteproducer prods user2 ccc

```

投票的token数量为所有质押的token数量，包括CPU、NET资源。用户投票时，账户自身`staked`的所有token的，都算作票数。
**查询抵押信息**
```sh
cleos -u http://172.18.0.1:8888  system listbw user2
```


**查看已经投票的token总量**

```sh
cleos  -u http://172.18.0.1:8888 get table eosio eosio global

# 会显示如下内容
{
  "rows": [{
      "max_block_net_usage": 1048576,
      "target_block_net_usage_pct": 1000,
      "max_transaction_net_usage": 524288,
      "base_per_transaction_net_usage": 12,
      "net_usage_leeway": 500,
      "context_free_discount_net_usage_num": 20,
      "context_free_discount_net_usage_den": 100,
      "max_block_cpu_usage": 200000,
      "target_block_cpu_usage_pct": 1000,
      "max_transaction_cpu_usage": 150000,
      "min_transaction_cpu_usage": 100,
      "max_transaction_lifetime": 3600,
      "deferred_trx_expiration_window": 600,
      "max_transaction_delay": 3888000,
      "max_inline_action_size": 4096,
      "max_inline_action_depth": 4,
      "max_authority_depth": 6,
      "max_ram_size": "68719476736",
      "total_ram_bytes_reserved": 50948099,
      "total_ram_stake": 13711100,
      "last_producer_schedule_update": "2018-12-18T09:01:02.500",
      "last_pervote_bucket_fill": "1545061578000000",
      "pervote_bucket": 0,
      "perblock_bucket": 0,
      "total_unpaid_blocks": 101737,
      "total_activated_stake": "2800200110000",
      "thresh_activated_stake_time": "1545061577500000",
      "last_producer_schedule_size": 3,
      "total_producer_vote_weight": "14027788730150111232.00000000000000000",
      "last_name_close": "2000-01-01T00:00:00.000"
    }
  ],
  "more": false
}

```
其中`total_activated_stake`除10000及为已经投票的token数量，这个数量达到全网15%之后，就会启动主网。


**追加抵押增加票数**
```sh
cleos -u http://172.18.0.1:8888 system delegatebw user2 aaa '100000000 SYS ' '100000000 SYS'

cleos -u http://172.18.0.1:8888 system delegatebw user2 bbb '100000000 SYS ' '100000000 SYS'

cleos -u http://172.18.0.1:8888 system delegatebw user2 ccc '100000000 SYS ' '100000000 SYS'
```

当总票数超过15%时，创世节点就会停止出块。然后三个bp节点就会开始轮流出块，主网启动。

**查看出块节点列表**

```sh
cleos -u http://172.18.0.1:8888 system listproducers
```

**查看账户及投票信息**
```sh
cleos -u http://172.18.0.1:8888 get account user2
```

**赎回抵押**

赎回抵押同时撤销相应的票数，三天后才能领取返还的token。
```sh
cleos -u http://172.18.0.1:8888 system undelegatebw user2  user2 '0.02 SYS' '0.02 SYS'
```

**领取返还的token**
```sh
cleos push action eosio refund '["本人账户名"]' -p 本人账户名
```

**查询账户token数量**

```sh
cliseat -u http://172.17.14.9:8888 get currency balance cubetrain.tk samaritan
```

### 解除eosio账户特权

主网启动之后，`eosio`账户仍有许多特权。对于我们自己的测试网络，这点可以不必考虑。但是如果是个开放的区块链项目，需要解除`eosio`账户的特权，可使用如下命令：

```sh
$ cleos push action eosio updateauth '{"account": "eosio", "permission": "owner", "parent": "", "auth": {"threshold": 1, "keys": [], "waits": [], "accounts": [{"weight": 1, "permission": {"actor": "eosio.prods", "permission": "active"}}]}}' -p eosio@owner
$ cleos push action eosio updateauth '{"account": "eosio", "permission": "active", "parent": "owner", "auth": {"threshold": 1, "keys": [], "waits": [], "accounts": [{"weight": 1, "permission": {"actor": "eosio.prods", "permission": "active"}}]}}' -p eosio@active

```