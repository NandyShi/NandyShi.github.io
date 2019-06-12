---
layout:     post
title:      "eos创建账户"
subtitle:   "EOS笔记"
date:       2018-10-31
author:     "Liu Ke"
header-img: "img/in-post/eos.jpg"
tags:
    - EOS
---

> eos v1.2.4

## Contents

1. [本地测试网创建](#本地测试网创建)
2. [加载系统合约后创建](#加载系统合约后创建)
3. [常见问题](#常见问题)

## 本地测试网创建

在本地启动单节点测试网络后，可以通过命令行来创建账号。主要有以下步骤：


1. 创建钱包
2. 创建密钥对
3. 钱包导入私钥
4. 配置config文件
5. 启动keosd服务
6. 注册账户

帐户名称必须符合以下**规则**：

- 必须少于13个字符
- 只能包含以下符号：**.12345abcdefghijklmnopqrstuvwxyz**，需要注意的是‘.’点号也是支持的。


前面的文章[EOS钱包和账户](http://keliu.me/2018/09/20/eosWallet/)讲了钱包相关的一些命令行操作。

创建了钱包之后，生成一对公私钥对，并将其中的私钥导入钱包中。然后配置nodeos的`config.ini`文件，使这对密钥称为节点的密钥。将`signature-provider`参数按如下形式使用之前生成的密钥对,用来对区块链上的操作进行签名。

```sh
$ cleos create key
```
---

```sh
$ cleos wallet import --private-key PRIVATE_KEY
```
---

```ini
# ID of producer controlled by this node (e.g. inita; may specify multiple times) (eosio::producer_plugin)
producer-name = eosio

# Key=Value pairs in the form <public-key>=<provider-spec>
# Where:
#    <public-key>       is a string form of a vaild EOSIO public key
# 
#    <provider-spec>    is a string in the form <provider-type>:<data>
# 
#    <provider-type>    is KEY, or KEOSD
# 
#    KEY:<data>         is a string form of a valid EOSIO private key which maps to the provided public key
# 
#    KEOSD:<data>       is the URL where keosd is available and the approptiate wallet(s) are unlocked (eosio::producer_plugin)

signature-provider = EOS9TRyAjQq8ud7hVNYcfnVPJqcVpscN0So8BhtHuGYqET5GDW5CV=KEY:5KQwrPbwdL6PhXujxW47FSSQZ1JiwsST5cqQzDeyXtP79zkvFD3
```

配置好config文件之后，启动keosd服务。

然后使用如下命令注册账户：

```sh
$ cleos create account eosio NEW_ACCOUNT_NAME OWNER_KEY ACTIVE_KEY
```

一般在config中配置了`http-server-address`之后，可使用如下形式：

```sh
$ cleos -u http://127.0.0.1:8888 create account eosio NEW_ACCOUNT_NAME OWNER_KEY ACTIVE_KEY
```

NEW_ACCOUNT_NAME为想要创建的账户名称，规则如前文所讲，少于13个指定字符范围的字符串。

OWNER_KEY为owner权限的公钥。

ACTIVE_KEY为active权限的公钥。

> OWNER_KEY和ACTIVE_KEY可以相同。

由于是本地测试网，会提示如下形式的警告，“交易在本地执行，可能未被网络确认”：

```sh
executed transaction: 472b05a94752cd13bd36d4ca8f33106a4cc0d2886685e1bd0031810c2211d4e9  200 bytes  1040 us
#     eosio <= eosio::newaccount        {"creator":"eosio","name":"dlfjal","owner":{"threshold":1,"keys":[{"key":"EOS6cjq6iQXTMBcdwQSPJ...
warning: transaction executed locally, but may not be confirmed by the network yet    ]
```

这样就在本地测试网络创建了账户。

## 加载系统合约后创建

当启动了多节点主网，部署了`eosio.system`系统合约之后，就不能通过上面的方法创建账号。需要使用系统合约来创建账户，同时需要购买RAM、CPU和NET资源放在一个事务中。

```sh
cleos system newaccount
``` 

## 常见问题

##### Error 3120006

```sh
Error 3120006: No available wallet
Ensure that you have created a wallet and have it open
```

没有创建钱包。使用`cleos wallet create --to-console`创建钱包。

##### Error 3090003

```sh
Error 3090003: Provided keys, permissions, and delays do not satisfy declared authorizations
Ensure that you have the related private keys inside your wallet and your wallet is unlocked.
Error Details:
transaction declares authority '{"actor":"cubetrain","permission":"active"}', but does not have signatures for it.
```

可能的情况：

1. 配置文件中节点签名相关的私钥没有导入钱包
2. 钱包未处于解锁状态

