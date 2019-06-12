---
layout:     post
title:      "EOS钱包和账户"
subtitle:   "EOS笔记"
date:       2018-09-20
author:     "Liu Ke"
header-img: "img/in-post/eos.jpg"
tags:
    - EOS
---

> eos v1.2.4

>参考资料：[EOS官方教程](https://developers.eos.io/eosio-nodeos/v1.2.0/docs/learn-about-wallets-keys-and-accounts-with-cleos "EOS官方教程")

## Contents

1. [EOS账户体系](#EOS账户体系)
	1. [原生权限](#原生权限)
	2. [权重和阈值](#权重和阈值)
2. [钱包](#钱包)
	1. [创建钱包](#创建钱包)
	2. [钱包列表](#钱包列表)
	3. [钱包锁定和解锁](#钱包锁定和解锁)
	4. [打开钱包](#打开钱包)
	5. [生成密钥对](#生成密钥对)
	6. [密钥导入钱包](#密钥导入钱包)
3. [创建账户](#创建账户)

## EOS账户体系

eos账户体系与别的项目有很大的不同，**eos实现了基于角色的权限管理和账户恢复功能**，使用户能够更加灵活地管理账户。实现了“**公钥≠地址**”。

eos中，公私钥对并不代表账户，而只是代表**账户中的权限**，也就是“**密钥对=权限**”。可以给一个密钥对自定义一些只有特定功能的权限，这个密钥对就只能有权限对应的功能操作，无法进行权限之外的操作。例如，给一个密钥对只分配买卖RAM的权限，这个密钥对就只能买卖RAM，而没有转账等别的操作，买卖RAM时，只需要这个私钥对操作进行签名即可完成买卖操作。

EOS账户体系的巨大优势：**底层天然支持多签名**。对EOS的账户可以通过权限配置，扩展成组织账户，由多对公私钥对共同控制。同时可以为部分个体配置部分权限，实现非常灵活和安全的账户管理方式。

#### 原生权限

每个账户都有两个原生的标准权限，**owner**和**active**，这是账户权限的默认配置。

- owner可认为是“根权限”，代表账户的所有权。**可以进行账户的所有操作**，包括更改owner权限。可由**一对或多对**EOS公私钥或另一账户的某权限实现权限控制。建议对owner权限的密钥对进行冷存储。
- active是活跃权限，active的权限比owner小一点，除了**不支持修改owner**，其他操作基本都支持。也可由**一对或多对**EOS公私钥或另一账户的某权限实现权限控制。

#### 权重和阈值

由于权限配置机制的存在，有时候某个权限会由多个主体共同控制，这时EOS就会根据**权重和阈值**来判定对权限的控制权。

账户可给每个主体（如每对EOS公私钥）分配不同的权重，以及拥有该权限的阈值，只有当某些人拥有的公私钥数量所对应的权重之和不低于该权限的阈值，才能拥有该权限，进行相应操作。

这样的话，某个权限如果需要两个人达成一致，才能行使，那么可以设定每个人的权重都小于该权限的阈值，但是两个人的权重之和大于阈值，这两个人在意见一致的情况下就能行使某个权限，但是任何一个人都无法单独行使该权限（多方签名）。

## 钱包

钱包相当于是公钥-私钥对的加密存储库。钱包及其内容由keosd管理。

#### 创建钱包

在cleos下，使用`cleos wallet create`命令可以创建一个默认的钱包`default`。

```sh
$ cleos wallet create
ERROR: Either indicate a file using "--file" or pass "--to-console"
```

但是需要注意的是，如果直接使用`cleos wallet create`命令，会有错误提示。这里提示我们需要指明，输出到文件或者控制台。

```sh
$ cleos wallet create --to-console
Creating wallet: default
Save password to use in the future to unlock this wallet.
Without password imported keys will not be retrievable.
"PW5JsENKhzCKpo59dj9bDRdcLvJ3xC19m75uFDy73o6H8XsEcxk51"
```
创建成功，并返回钱包的密码，每隔900秒钱包就会自动锁定，需要这个密码来解锁。默认情况下，keosd会将钱包存储在`~/eosio-wallet`文件夹中，生成对应的`钱包名.wallet`文件。

在此`~/eosio-wallet`文件夹中除了钱包文件外，还有**keosd配置文件config.ini**。

使用`-n`来创建自定义名称的钱包：

```sh
$ cleos wallet create -n ke --to-console
Creating wallet: ke
Save password to use in the future to unlock this wallet.
Without password imported keys will not be retrievable.
"PW5HufPPfYCWE5nM6BFKwqUq6eW28sRXisrm4XZYpheBbCtXnkByY"
```

**使用wallet命令与'default'默认钱包交互不需要-n参数**，自定义名称的钱包与wallet命令交互需要使用`-n`参数。

#### 钱包列表

使用`cleos wallet list`命令查看现有的钱包列表：

```sh
$ cleos wallet list
Wallets:
[
  "default *",
  "ke *"
]
```

#### 钱包锁定和解锁

查看钱包列表时，如果钱包名后有`*`号，说明该钱包是解锁状态，如果没有 `*`号，则该钱包为锁定状态。

锁定指定钱包：

```sh
$ cleos wallet lock  -n  ke
Locked: ke
```

解锁指定钱包需要使用钱包的**密码**：

```sh
$ cleos wallet unlock -n ke --password PW5HufPPfYCWE5nM6BFKwqUq6eW28sRXisrm4XZYpheBbCtXnkByY
Unlocked: ke
```

#### 打开钱包

当我们将keosd停止后，查看钱包列表会发现看不到之前创建的钱包了。

```sh
$ pgrep keosd
5557
$ pkill keosd
$ cleos wallet list
"/usr/local/eosio/bin/keosd" launched
Wallets:
[]
```

钱包在操作之前需要打开。keosd关闭重新启动后，钱包未被打开。需要使用如下命令打开钱包：

```sh
$ cleos wallet open
Opened: default
$ cleos wallet open -n ke
Opened: ke
```

打开后的钱包是锁定状态，需要`unlock`解锁。

#### 生成密钥对

使用`cleos create key`命令可以很简单地生成**没有任何权限**的公私钥对。

```sh
$ cleos create key --to-console
Private key: 5HxFYWMSvZ2md4YPeTQPaiCUHw5qoED2L7npuS8WsqVpYfimbTs
Public key: EOS7BgwFWyX9Lkz9KGm9sAJFVdDyuXW6VnxG8fYdvtZitbcFaN4CA
$ cleos create key --to-console
Private key: 5Jk5TMs9rKnFaobMzJZuS6w7H9HE2Cvd85hR9qWWwuRpkjGV77h
Public key: EOS6Mvfw8ZDki97or1wBc9TiZjGZ8ubtYsRU5CaGaAZJSByAnLysp
```

#### 密钥导入钱包

在钱包解锁状态下，使用`cleos wallet import`将私钥导入钱包。导入成功后，会提示私钥对应的公钥。

```sh
$ cleos wallet import -n ke --private-key 5HxFYWMSvZ2md4YPeTQPaiCUHw5qoED2L7npuS8WsqVpYfimbTs
imported private key for: EOS7BgwFWyX9Lkz9KGm9sAJFVdDyuXW6VnxG8fYdvtZitbcFaN4CA
$ cleos wallet import -n ke --private-key 5Jk5TMs9rKnFaobMzJZuS6w7H9HE2Cvd85hR9qWWwuRpkjGV77h
imported private key for: EOS6Mvfw8ZDki97or1wBc9TiZjGZ8ubtYsRU5CaGaAZJSByAnLysp
```

使用`cleos wallet keys`可以查看所有解锁钱包里的所有公钥。

```sh
$ cleos wallet keys
[
  "EOS6Mvfw8ZDki97or1wBc9TiZjGZ8ubtYsRU5CaGaAZJSByAnLysp",
  "EOS7BgwFWyX9Lkz9KGm9sAJFVdDyuXW6VnxG8fYdvtZitbcFaN4CA"
]

```

使用`cleos wallet private_keys`命令可以查看指定钱包导入了哪些密钥，需要配合钱包密码使用，会完整显示导入的所有公钥和私钥对。

```sh
$ cleos wallet private_keys -n ke --password PW5HufPPfYCWE5nM6BFKwqUq6eW28sRXisrm4XZYpheBbCtXnkByY
[[
    "EOS6Mvfw8ZDki97or1wBc9TiZjGZ8ubtYsRU5CaGaAZJSByAnLysp",
    "5Jk5TMs9rKnFaobMzJZuS6w7H9HE2Cvd85hR9qWWwuRpkjGV77h"
  ],[
    "EOS7BgwFWyX9Lkz9KGm9sAJFVdDyuXW6VnxG8fYdvtZitbcFaN4CA",
    "5HxFYWMSvZ2md4YPeTQPaiCUHw5qoED2L7npuS8WsqVpYfimbTs"
  ]
]
```


## 创建账户

账户名规则：

1. 为12个字符
2. 只能包含以下符号：.12345abcdefghijklmnopqrstuvwxyz

对于自己搭建的eos私链，只有系统账户eosio才可以创建小于12个字符的账户，其他账户只能创建12位的账户。如需短账户名，择需`bid`竞标。

当系统合约没有部署的时候，创建EOS账户主要有以下步骤：

1. 创建钱包
2. 创建密钥对
3. 注册账户
4. 钱包导入私钥

当部署了系统合约之后，直接使用系统合约来创建，`system newaccount`命令形式如下：

```sh
cleos system newaccount --transfer useruseruser newnewnewnew  EOS5aNCeePuH2hQnaXrkNABex2qZeVWg3j8ikckbQm9EnWuFE13YD --stake-net "1.5000 EOS" --stake-cpu "4.0000 EOS" --buy-ram "4.5000 EOS"
```

创建账户时，需要给新账户配置系统资源，有两种方式。一种是直接确定EOS的数量，系统会根据当前资源的行情来自动换算成对应数量的资源。另一种是指明资源的具体数量，如以`bytes`为单位的值。



