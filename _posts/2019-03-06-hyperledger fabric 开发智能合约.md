---
layout:     post
title:      hyperledger fabric 开发智能合约
subtitle:   
date:       2019-03-06
author:     NandyShi
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 超级账本
---

## hyperledger fabric 开发智能合约
### 一、编写智能合约代码HelloWorld.go
代码很简单，每个合约包含两个方法，Init、Invoke。

```
package main
 
import (
    "fmt"
    "github.com/hyperledger/fabric/core/chaincode/shim"
    "github.com/hyperledger/fabric/protos/peer"
)
 
type Helloworld struct {
 
}
 
func (t * Helloworld) Init(stub shim.ChaincodeStubInterface) peer.Response{
 
    args:= stub.GetStringArgs()
 
    err := stub.PutState(args[0],[]byte(args[1]))
 
    if err != nil {
        shim.Error(err.Error())
    }
 
    return shim.Success(nil)
}
 
func (t *Helloworld) Invoke (stub shim.ChaincodeStubInterface) peer.Response{
 
    fn, args := stub.GetFunctionAndParameters()
 
    if fn =="set" {
        return t.set(stub, args)
    }else if fn == "get"{
        return t.get(stub , args)
    }
    return shim.Error("Invoke fn error")
}
 
func (t *Helloworld) set(stub shim.ChaincodeStubInterface , args []string) peer.Response{
    err := stub.PutState(args[0],[]byte(args[1]))
    if err != nil {
        return shim.Error(err.Error())
    }
    return shim.Success(nil)
}
 
func (t *Helloworld) get (stub shim.ChaincodeStubInterface, args [] string) peer.Response{
 
    value, err := stub.GetState(args[0])
 
    if err != nil {
        return shim.Error(err.Error())
    }
 
    return shim.Success(value)
}
 
func main(){
    err := shim.Start(new(Helloworld))
    if err != nil {
        fmt.Println("start error")
    }
}
```
> 注意go文件格式合法

### 二、将代码文件夹拷贝到fabric-samples下面的chaincode文件夹
```
cd ../fabric-samples/chaincode/
mkdir hello
cd hello
vim HelloWorld.go
```
将第一步编写好的go文件拷贝到此处。



### 三、启动网络
```
cd ../fabric-samples/chaincode-docker-devmode
docker-compose -f docker-compose-simple.yaml up
```

### 四、编译链码,并启动
新开一个会话
```
docker exec -it chaincode bash
cd hello/
go build
CORE_PEER_ADDRESS=peer:7052 CORE_CHAINCODE_ID_NAME=mycc:0 ./hello
```
### 五、链码交互
进入docker容器
```
docker exec -it cli bash
```
安装链码
```
peer chaincode install -p chaincodedev/chaincode/hello -n mycc -v 0
```
实例化链码
```
peer chaincode instantiate -n mycc -v 0 -c '{"Args":["str","HelloWorld"]}' -C myc
```

查询链码
```
peer chaincode query -n mycc -c '{"Args":["get","str"]}' -C myc
```

修改链码
```
peer chaincode invoke -n mycc -c '{"Args":["set","str","newHelloWorld"]}' -C myc
```
再次查询
```
peer chaincode query -n mycc -c '{"Args":["get","str"]}' -C myc
```
