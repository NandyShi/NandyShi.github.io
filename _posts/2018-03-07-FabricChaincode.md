---
layout:     post
title:      "深入学习Fabric链码Chaincode"
subtitle:   "区块链学习笔记"
date:       2018-03-07
author:     "Liu Ke"
header-img: "img/BC.jpg"
tags:
    - Blockchain区块链
    - Hyperledger Fabric
---



链码（chaincode）或者链上代码，是Fabric中十分关键的一个概念。链码源自智能合约的思想，并进行了进一步扩展，支持多种高级编程语言。

目前Fabric项目中提供了用户链码和系统链码。前者运行在单独的容器中，提供对上层应用的支持，后者则嵌入在系统内，提供对系统进行配置、管理的支持。

一般所谈的链码为用户链码，通过提供可编程能力提供了对上层应用的支持。用户通过链码相关的API编写用户链码，即可对账本中的状态进行更新操作。

链码会对Fabric应用程序发送的交易做出响应，执行代码逻辑，与账本进行交互。区块链网络中的成员商定业务逻辑后，可将业务逻辑编程到链码中，然后大家遵循合约来执行。

链码会创建一些状态（state）并写入账本中。状态带有绑定到链码的命名空间，并且仅限于创建它的链码所使用，不能被其他链码直接访问。但是，在合适的许可范围内，一个链码也可以调用另一个链码，间接访问其状态。另外，在一些场景下，不仅需要访问状态的当前值，还需要能够查询状态所有历史值，这就存放账本状态的数据库提出了更多的要求。

链码最核心的机构为`ChaincodeSpec`，对链码的部署和调用都基于该结构进行进一步封装（`ChaincodeDeploymentSpec`和`ChaincodeInvocationSpec`）,链码信息至少需要指定名称、版本号和实例化策略。


![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/180307/%E9%93%BE%E7%A0%81%E7%9B%B8%E5%85%B3%E7%BB%93%E6%9E%84.png)

链码经过安装和实例化操作之后，即可被调用。在安装的时候，需要指定安装到哪个Peer节点(Endorser);实例化的时候还需要指定是在哪个通道内进行实例化。链码之间还可以通过互相调用，创建更为灵活的应用逻辑。

链码在Fabric节点上的隔离沙盒（目前为Docker容器）中执行，并通过gRPC协议来与节点进行交互。必要的交互包括调用链码、读写账本、返回响应结果等。Fabric目前主要支持Go语言的链码。

## gRPC消息协议

Fabric中大量采用了gRPC消息在不同组件之间进行通信交互，主要包括如下几种情况：

- 客户端访问Peer节点

- 客户端和Peer访问Orderer节点

- 链码容器跟Peer节点之间通信

- 多个Peer节点之间的通信

对于链码容器和Peer节点之间的操作，链码容器启动后，会向Peer节点进行注册，gRPC地址为`/protos.ChaincodeSupport/Register`。消息为ChaincodeMessage结构，如下图所示。`Type`为消息类型，`TxId`为关联交易的ID，`Payload`中存储消息内容。（定义在`protos/peer/chaincode_shim.proto`文件）。其中，Payload域中可以包括各种Chaincode操作消息，如`GetHistoryForKey`、`GetQueryResult`、`PutStateInfo`、`GetStateByRange`等。
注册完成后，双方建立起双工通道，通过更多消息类型来实现多种交互。

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/180307/ChaincodeMessage%E6%B6%88%E6%81%AF%E7%BB%93%E6%9E%84.png)

## 用户链码

用户链码相关的代码都在`core/chaincode`路径下。其中`core/chaincode/shim`包中的代码主要是供链码容器侧调用使用，其他代码主要是Peer侧使用。

### Chaincode接口
Fabric中为链码提供了很好的封装支持，编写链码还是相对比较简单。以Golang为例，每个链码都需要实现以下Chaincode接口：


    type Chaincode interface{
    
    	Init(stub ChaincodeStubInterface) pb.Response
    
    	Invoke(stub ChaincodeStubInterface) pb.Response
    
	}



其中:

- `Init`:当链码收到实例化（instantiate）或升级（upgrade）类型的交易时，Init方法会被调用。

- `Invoke`:当链码收到调用（invoke）或查询（query）类型的交易时，Invoke方法会被调用。

### 链码结构

一个链码的必要结构如下所示，在其中利用`shim.ChaincodeStubInterface`结构，实现跟账本的交互逻辑：


    package main
    
    //引入必要的包
    
    import(
    
    	"github.com/hyperledger/fabric/core/chaincode/shim"
    
    	pb "github.com/hyperledger/fabric/protos/peer"
    
    )
    
    //声明一个结构体
    
    type SimpleChaincode struct {}
    
    
    //为结构体添加Init方法
    
    func (t *SimpleChaincode) Init(stub shim.ChaincodeStubInterface) pb.Response{
    
    	//在该方法中实现链码运行中初始化或升级的处理逻辑
    
    	//编写时可灵活使用stub中的API
    
    }
    
    //为结构体添加Invoke方法
    
    func (t *SimpleChaincode) Invoke(stub shim.ChaincodeStubInterface) pb.Response{
    
    	//在该方法中实现链码运行中被调用或查询时的处理逻辑
    
    	//编写时可灵活使用stub中的API
    
    }
    
    //主函数，需要调用shim.Start()方法
    
    func main() {
    
    	err := shim.Start(new(SimpleChaincode))
    
    	if err != nil {
    
    		fmt.Printf("Error start Simple chaincode : %s", err)
    
    	}
    
    }

链码需要引入如下的依赖包：

- `"github.com/hyperledger/fabric/core/chaincode/shim"`：`shim`包提供了链码与账本交互的中间层。链码通过`shim.ChaincodeStub`提供的方法来读取和修改账本状态。

- `pb "github.com/hyperledger/fabric/protos/peer"`：`Init`和`Invoke`方法需要返回`pb.Response`类型

编写链码关键的就是`Init`和`Invoke`这两个方法。当部署或升级链码时，`Init`方法会被调用，用来完成一些初始化工作。当通过调用链码做一些实际工作时，`Invoke`方法被调用，响应调用或查询的业务逻辑都需要在`Invoke`方法中实现。`Init`或`Invoke`方法以`stub shim.ChaincodeStubInterface`作为传入参数，`pb.Response`作为返回类型。其中，`stub`包含丰富的API，包块对账本进行操作、读取交易参数、调用其他链码等。

### 链码与Peer的交互

用户链码目前运行在Docker容器中，跟Peer节点之间通过gRPC通道进行通信，双方通过ChaincodeMessage消息进行交互。消息为ChaincodeMessage结构, Type为消息类型，TxId为关联交易的ID，Payload中存储消息内容。

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/180307/ChaincodeMessage%E6%B6%88%E6%81%AF%E7%BB%93%E6%9E%84.png)

用户链码容器和所属Peer节点的主要交互过程如图：

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/in-post/180307/%E9%93%BE%E7%A0%81%E6%B6%88%E6%81%AF%E4%BA%A4%E4%BA%92.png)

消息类型有如上图所示的十几种消息类型，负责整个用户链码的完整生命周期。一般情况下，链码从注册到Peer开始，一直到被调用，主要步骤同样如图所示。

### 链码基本工作原理

首先，用户通过客户端（SDK或CLI），Fabric的背书节点（endorser）发出调用链码的交易提案（proposal）。节点对提案进行包括ACL权限检查在内的各种检验，通过后则创建模拟执行这一交易的环境。

之后，节点和链码容器之间通过gRPC消息来交互，模拟执行交易并给出背书结论。两者之间采用ChaincodeMessage消息。

链码容器的shim层则是节点与链码交互的中间层。当链码的代码逻辑需要读写账本时，链码会通过shim层发送相应操作类型的ChaincodeMessage给节点，节点本地操作账本后返回相应消息。

客户端收到足够的背书节点的支持后，便可以将这笔交易发送给排序节点（orderer）进行排序，并最终写入区块链。




> 参考资料：《区块链 原理、设计与应用》 杨保华, 陈昌编著


