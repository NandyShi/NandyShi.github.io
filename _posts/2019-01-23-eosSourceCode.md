---
layout:     post
title:      "EOS代码整体梳理"
subtitle:   "EOS代码分析学习笔记"
date:       2019-01-21
author:     "Liu Ke"
header-img: "img/in-post/eos.jpg"
tags:
    - EOS
---

> eos v1.4


## Contents

1. [脚本相关](#脚本相关)
2. [插件](#插件)
3. [合约](#合约)
4. [主程序](#主程序)
5. [依赖库](#依赖库)
6. [测试](#测试)
7. [docker工具](#docker工具)
8. [说明文档](#说明文档)


EOS项目的整体代码框架如下图所示：

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/190123/framwork.png)

### 脚本相关

EOS项目的整体编译、依赖库的下载以及代码编译等，都是通过一整套脚本体系来实现的。主要包括三部分，分别是`eos/eosio_build.sh`，`eos/scripts`和`eos/CMakeModules`。

##### eosio_build.sh

`eosio_build.sh`文件是EOS的主编译脚本。EOS项目clone到本地之后，可以通过运行`eosio_build.sh`这个脚本，一键编译好EOS项目，安装好所有的依赖项，配置好EOS的开发环境。

`eosio_build.sh`文件成功执行完毕之后，可以再执行`eosio_install.sh`文件，会将所有EOS相关的命令配置好系统环境变量，便于执行EOS相关命令进行开发。

##### scripts

`eos/scripts`这个目录下包含了编译项目的其他一些脚本文件。

`abi_to_rc`、`abigen.sh`和`abi_is_json.py`负责将C++编写的智能合约编译成`.abi`文件，然后将`.abi`文件编译成可执行文件。

以`eosio_build_`为前缀的`sh`脚本文件是针对不同系统的编译子脚本，一般在执行`eosio_build.sh`文件过程中报错，会根据对应的系统环境，在对应的`eosio_build_`文件中修改。如Ubuntu系统就对应的就是`eosio_build_ubuntu.sh`文件。

`eosio-tn_`前缀的脚本负责自动化运行、关闭节点等。

如图所示：

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/190123/scripts.png)

##### CMakeModules

`eos/CMakeModules`目录下主要是CMake编译所需的一些配置信息。EOS就是基于CMake工具编译的。

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/190123/CMakeModules.png)

### 插件

EOS的节点程序是通过各种不同的插件组合，来实现一系列区块链上的服务。插件相关的代码主要在`eos/plugins`目录下。

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/190123/plugins.png)

##### 插件模板

`eos/plugins/template_plugin`定义了EOS项目中所有插件的模板，`template_plugin.cpp`为插件模板源码，CMake的所有的语句都写在`CMakeLists.txt`文件中用于编译。

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/190123/template_plugin.png)

##### 基类插件

EOS项目中有`chain_plugin`、`net_plugin`、`http_plugin`、`history_plugin`、`wallet_plugin`这些基类插件。

- **chain_plugin**：定义了**链处理**插件。这个插件用于nodeos节点程序与区块链之间的交互功能的实现，包括读取本地不可逆区块链的基本信息、设置本地链检查点、设置本地链参数、设置可逆区块数据库参数、设置账户黑（白）名单、设置智能合约黑（白）名单、重载区块链初始状态文件，以及删除、重写、替换本地区块链数据等。

- **net_plugin**：定义了**P2P网络**插件。这个插件用于EOS系统的P2P网络中TCP/IP层相关功能的实现，包括建立节点之间握手并互联；监听、发送、接收新交易请求；监听、发送、接收新区块请求；验证接收数据合法性等。

- **http_plugin**：定义了**网络HTTP**插件。这个插件用于EOS系统的P2P网络中HTTP层相关功能的实现，包括监听、发送、接收新交易请求；监听、发送、接收新区块请求；验证接收数据合法性等。

- **history_plugin**：定义了**历史记录查询**插件。这个插件用于节点程序对本地链发起的查询相关功能的实现，包括指定区块查询、指定账户状态查询、指定交易查询等。

- **wallet_plugin**：定义了**钱包**插件。这个 插件用于nodeos节点程序与钱包交互的相关功能的实现，包括创建、读取钱包文件；设置unlock timeout时间；将密钥导入钱包。`wallet.cpp`文件实现了钱包文件的基本功能，包括创建新钱包、导入密钥等；`wallet_manager.cpp`文件实现了对钱包的管理功能，包括设置unlock超时时间、lock指定钱包等；`wallet_plugin.cpp`文件对上述功能进行插件化，包括定义插件参数等，实现了nodeos节点程序通过调用插件处理钱包文件的功能。

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/190123/wallet_plugin.png)

##### 派生类插件

派生类插件分别继承几个基类插件，实现指定的功能。

- **bnet_plugin**：定义了EOS的P2P网络中不同节点之间同步各自本地链数据的算法。主要有：查找本地链上的最后一个区块ID；如果本地产生新区块，则将该区块发送给其他节点；如果本地不产生新区块，则将接收的未确认交易发送给其他节点。
- **faucet_testnet_plugin**：定义了在测试网上快速建立测试账号的插件，便于测试。
- **http_client_plugin**：定义了EOS网络HTTP层响应请求，并做相应的安全验证的客户端插件。
- **mongo_db_plugin**：定义了本地MongoDB数据库基本配置的插件，用于保存并管理本地不可逆转区块链数据。
- **producer_plugin**：定义了区块生产节点功能的插件，包括生产、打包新区块数据；对新区块签名；对接收的区块进行验证，包括区块头合法性、签名合法性和交易合法性。
- **txn_test_gen_plugin**：定义了一个每秒自动产生指定数量的交易信息的插件，用于EOS网络的吞吐量TPS的测试。

##### 封装类API插件

主要用于外界和EOS链的交互，并对特定的插件进行封装，只暴露API提供相应的接口服务。

- **chain_api_plugin**：依赖于`chain_plugin`，提供与外部调用链相关操作的接口服务。
- **db_size_api_plugin**：提供与外部调用数据有关的接口服务。
- **history_api_plugin**：依赖于`history_plugin`，提供外部调用相应历史记录的接口服务。
- **net_api_plugin**：依赖于`net_plugin`，提供与外部调用网络相关的操作的接口服务。
- **producer_api_plugin**：依赖于`producer_plugin`，提供与外部调用区块生产节点相关功能的接口服务。
- **wallet_api_plugin**：依赖于`wallet_plugin`，提供与外部调用钱包交互的接口服务。

### 合约

EOS项目的基本功能是通过系统合约来实现的，用户可以调用已经部署在链上的智能合约来实现特定的功能。也可以使用C++自己编写`.cpp`的智能合约，然后通过EOS系统自带的`eosiocpp`编译器将`.cpp`、`.hpp`文件编译成`.wasm`和`.abi`文件，然后部署上链。EOS项目智能合约相关的代码位于`eos/contracts`目录下。

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/190123/contracts.png)

##### 系统合约

EOS目前有五个系统合约，分别是`eosio.bios`、`eosio.msig`、`eosio.sudo`、`eosio.system`和`eosio.token`，相应的代码在对应的目录下。

- **eosio_bios**：该合约用于启动EOS的P2P网络。这个合约可以直接控制其他账户的资源分配并访问其他特权API调用。启动P2P网络时，过程如下（1）初始启动节点部署该合约，并设置所需的参数；（2）待连接节点通过初始启动节点的地址连接；（3）初始启动节点调用`bios`合约，为待连接节点设置权限。
- **eosio.msig**：该合约定义了多重签名系统，实现多重签名功能。EOS要求系统的每一次更新都需要出块节点完成一次多签，当签名书达到比超级节点的2/3多一个时更新才生效，所以出块节点可以调用这个合约实现多签功能。
- **eosio.sudo**：该合约实现了创建EOS系统中root账户的功能，用于修改系统代码与更新合约。
- **eosio.system**：该合约实现了EOS系统中**所有的基本功能**。例如：创建账户、部署智能合约、交易RAM、抵押获取NET和CPU资源、投票以及领取节点奖励等。
- **eosio.token**：该合约实现了发行Token的功能。EOS系统自己的系统token和基于EOS的token都是通过该合约发行的，主体函数包括发行新token、初始分发、转账和查询余额等。

##### 合约相关依赖

系统合约的实现依赖于很多类库，包括account和asset等数据的定义以及权限管理和序列化等常用函数，相应依赖库对应的目录如下：

- `eos/contracts/asserter`：定义了assert的相关结构体，并完成对智能合约事件的分发。
- `eos/contracts/bancor`：定义了bancor结构体，包含于凯恩斯国际货币单位相关的内容，主要用于货币单位之间的转换。
- `eos/contracts/eosiolib`：包含EOS运行所依赖的库的头文件。
- `eos/contracts/musl`：Linux操作系统下的一个标准库。
- `eos/contracts/noop`：实现一个空的智能合约。
- `eos/contracts/proxy`：实现代理相关的内容。

##### 合约相关测试与示例

EOS给了一些示例合约和测试文件，用于用户理解原理和测试功能。

- `eos/contracts/dice`：掷骰子对赌合约.
- `eos/contracts/bancor`：bancor算法调用入口文件.
- `eos/contracts/exchange`：去中心化交易所合约.
- `eos/contracts/hello`：helloworld合约.
- `eos/contracts/social`：只包含基本功能类Steem社交平台合约.
- `eos/contracts/test_`：测试文件.

### 主程序
	
EOS项目的主程序源码位于`eos/program`目录下，包含6个基本的功能组件。

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/190123/programs.png)

- **nodeos**：节点程序源码，可以配置不同插件来运行不同类型的节点。该进程主要负责提供区块生产，封装API接口和本地开发的功能。
- **cleos**：通过终端与nodeos之间交互的命令行工具cleos源码。编译后与nodeos公开的REST API进行交互。
- **keosd**：钱包程序源码。配合钱包相关插件，可以通过HTTP接口或RPC API完成钱包相关功能。
- **eosio-abigen**：智能合约编译器源码。用于生成智能合约的`.abi`文件。
- **eosio-launcher**：P2P网络组成启动器源码，简化了nodeos节点组网的流程。

### 依赖库

`eos/libraries`目录主要是EOS项目运行相关的依赖库。

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/190123/libraries.png)

- **abi-generator**：智能合约编译器所需的依赖文件，编译器的主要源码就在这个目录。
- **appbase**：提供了一个用于从一组插件构建应用程序的基本框架。这个模块负责管理插件的生命周期，并确保所有插件按正确的顺序进行配置、初始化、启动和关闭。该依赖库主要特征：动态指定要加载的插件、自动加载依赖插件、插件可以指定命令行参数和配置文件选项、程序正常退出SIGINT和SIGTERM、最小依赖（Boost1.60和C++14等）。
- **builtins**：包含了EOS项目编译过程中所需要的`compiler-RT`编译器（libgcc的替换库）的源码，包括编译器本身以及相关功能函数的代码描述。
- **chain**：EOS项目的核心内容，包括区块、区块链、Merkle树等数据结构，以及初始区块、控制器等关键算法。
- **chainbase**：定义了保存EOS区块链数据的数据库结构。
- **fc**：EOS项目的细胞级模块，定义了EOS项目中的基本变量数据结构。包括String、Time、Base系列编码。
- **softfloat**：包含了一个`Berkeley SoftFloat`，即符合IEEE浮点运算标准的二进制浮点软件实现。
- **testing**：包含了几个测试实例，包括对区块链数据库的链接测试、P2P网络的链接测试等。
- **utilities**：包含了一些通用的标准函数。
- **wasn-jit**：包含了一个`WebAssembly`的独立VM。它可以加载标准的二进制格式，也可以加载WebAssembly参考解释器定义的文本格式。对于文本格式，它可以加载标准堆栈机器语法和参考解释器使用的老式AST语法，以及所有测试命令。

### 测试

EOS提供了一些测试文件，主要位于`eos/tests`目录下，供用户测试节点是否正常运行，主要有对链功能的测试和对网络层的测试：

- 对链功能的测试：与区块链之间的数据交互、transaction分发等。
- 对网络层的测试：包括P2P网络传输功能、cleos与nodeos之间的通信等。

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/190123/tests.png)

### docker工具

EOS允许用户通过Docker运行节点或钱包，`eos/Docker`目录下包含了通过Docker启动并运行程序的必要文件。

![](https://github.com/dugu0808/dugu0808.github.io/raw/master/img/in-post/190123/docker.png)

### 说明文档

EOS项目还包含了一些说明文档：

- eos/README：节点部署等说明.
- eos/LICENSE.txt：版本号与许可文件.
- eos/tutorials：主网启动与exchange合约的使用教程.

> 参考资料：《深入理解EOS 原理解析与开发实战》 李万才等 著；
