---
layout:     post
title:      "密码学"
subtitle:   "区块链学习笔记"
date:       2018-03-14
author:     "Liu Ke"
header-img: "img/BC.jpg"
tags:
    - Blockchain区块链
    - 密码学
---



## Hash算法

目前常见的Hash算法包括MD5和SHA系列算法。

- MD4(RFC120)
- MD5(RFC1321)
- SHA:SHA-1,SHA-224,SHA-256,SHA-384,SHA-512

Hash算法不是加密算法，不能用于信息的保护。常用于对口令的保存。口令强度不够，容易被字典攻击（搜集常用口令计算对应哈希值制成字典）和彩虹表攻击（只保存一条Hash链的首尾值，相对字典攻击可以节省存储空间），它们都是空间换时间的攻击方法。为了防止这类攻击，一般采用加盐（salt）的方法。保存的不是口令明文的Hash值，而是口令明文再加上一段随机字符串（即“盐”）之后的哈希值。Hash结果和“盐”分别存放在不同的地方，两者不同时泄露，很难破解。

## 数字摘要

对数字内容进行Hash运算，获取唯一的摘要值来指代原始完整的数字内容。

## 加解密算法

### 对称加密算法

又称公共密钥加密。加解密的密钥相同，参与方需提前共享密钥，不能用于签名场景。

实现原理：

- 分组密码：DES(Data Encryption Standard),3DES,AES(Advanced Encryption Standard),IDEA(International Data Encryption Algorithm).
- 序列密码：又称流密码，每次通过伪随机数生成器来生成伪随机密钥串。代表算法RC4。

### 非对称加密算法 

又称公钥加密。加密密钥和解密密钥不同，分公钥和私钥。安全性一般需要基于数学问题来保障，目前主要有基于大数质因子分解、离散对数、椭圆曲线等经典数学难题。

- RSA：利用对大数进行质因子分解困难的特性。
- Diffie-Hellman密钥交换：基于离散对数无法快速求解，可在不安全的通道上，双方协商一个公共密钥。
- ElGamal：利用模运算下求离散对数困难的特性。
- 椭圆曲线算法（Elliptic Curve Cryptography, ECC）：基于对椭圆曲线上特定点进行特殊乘法逆运算难以计算特性。
- SM2(ShangMi 2):国家商用密码算法，基于椭圆曲线算法，加密强度优于RSA系列算法。

### 混合加密机制

## 消息认证码和数字签名

### 消息认证码

基于Hash的消息认证码（HMAC）

### 数字签名

- 盲签名
- 多重签名
- 群签名
- 环签名

## 数字证书

### X.509证书

## PKI体系

- CA（Certification Authority）：负责证书的颁发和作废，接收来自RA的请求，是最核心部分。
- RA（Registration Authority）:对用户身份进行验证，检验数据合法性，负责登记，审核过了就发给CA；
- 证书数据库：存放证书，多采用X.500系列标准格式。可以配合LDAP目录服务管理用户信息。

一般流程为，用户通过RA登记申请证书，提供身份和认证信息等；CA审核后完成证书的制造，颁发给用户。用户如果需要撤销证书，则需要再次向CA发出申请。

## Merkle树

默克尔树，哈希树，典型二叉树结构。最下面的叶节点包含存储数据或哈希值，非叶子节点（包括中间节点和根节点）都是它两个孩子节点内容哈希值。先计算叶子节点的哈希值，然后层层计算哈希。叶子节点数据的任何改变逐层传递直到根节点。Merkle树能够快速比较大量数据，两个Merkle树根节点相同，则两组数据必然相同。

## 布隆过滤器

布隆过滤器是一种基于Hash的高效查找结构，能够快速回答“某个元素是否在一个集合内”的问题。Hash有天然的“内容-索引关系”，由基于Hash的快速查找算法，设计出布隆过滤结构。采用了多个Hash函数来提高空间利用率。对同一个给定输入，多个Hash函数计算出多个地址，分别在位串的这些地址上标记1。进行查找时，进行同样的计算过程，并查看对应元素，如果都为1，则说明较大概率是存在该输入。

## 同态加密

对密文直接进行处理，跟对明文进行处理后再对处理结果加密，两者结果相同




> 参考资料：《区块链 原理、设计与应用》 杨保华, 陈昌编著


