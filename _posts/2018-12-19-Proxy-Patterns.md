---
layout:     post
title:      Proxy Patterns
subtitle:   
date:       2018-12-19
author:     NandyShi
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - zos
---

##Proxy Patterns
One of the biggest advantages of Ethereum is that every transaction of moving funds, every contract deployed, and every transaction made to a contract is immutable on a public ledger we call the blockchain. There is no way to hide or amend any transactions ever made. The huge benefit is that any node on the Ethereum network can verify the validity and state of every transaction making Ethereum a very robust decentralized system.

But the biggest disadvantage is that you cannot change the source code of your smart contract after it’s been deployed. Developers working on centralized applications (like Facebook, or Airbnb) are used to frequent updates in order to fix bugs or introduce new features. This is impossible to do on Ethereum with traditional patterns.

Remember the infamous Parity Wallet Multisig hack where 150,000 ETH were stolen? During the attack, a bug was being exploited in the Parity multisig wallet contract and high profile wallets were being drained of their funds. The only reconciliation that could be done was to try and be faster than the hacker and exploit the same vulnerability to hack the remaining wallets to redistribute the ETH to their rightful owners after the attack.

If only there were a way to update the source code after the smart contract has been deployed…

##Introducing Proxy Patterns
Although it is not possible to upgrade the code of your already deployed smart contract, it is possible to set-up a proxy contract architecture that will allow you to use new deployed contracts as if your main logic had been upgraded.

A proxy architecture pattern is such that all message calls go through a Proxy contract that will redirect them to the latest deployed contract logic. To upgrade, a new version of your contract is deployed, and the Proxy is updated to reference the new contract address.

![](https://i.loli.net/2018/12/19/5c19abac5e38d.jpg)

Zeppelin has been working on several proxy patterns as part of their effort to implement zeppelin_os. The three options explored are:

1. Inherited Storage
2. Eternal Storage
3. Unstructured Storage

All three patterns rely on low-level delegatecalls. Although Solidity provides a delegatecall function, it only returns true/false whether the call succeeded and doesn’t allow you to manage the returned data.

Before we dive-in, there are two key concepts that are important to understand:

When a function call to a contract is made that it does not support, the fallback function will be called. You can write a custom fallback function to handle such scenarios. The proxy contract uses a custom fallback function to redirect calls to other contract implementations.
Whenever a contract A delegates a call to another contract B, it executes the code of contract B in the context of contract A. This means that msg.value and msg.sender values will be kept and every storage modification will impact the storage of contract A.
Zeppelin’s Proxy contract, shared by all proxy patterns, implements its own delegatecall function for this particular reason which returns the value that resulted in calling the logic contract. If you’re planning on using Zeppelin’s Proxy contract code, you should understand in full detail the code you’ll be using. Let’s explore exactly how it works and understand the assembly opcodes it uses to achieve this. (Feel free to reference Solidity’s Assembly docs to get more information)

```
 assembly {
    let ptr := mload(0x40)
    calldatacopy(ptr, 0, calldatasize)
    let result := delegatecall(gas, _impl, ptr, calldatasize, 0, 0)
    let size := returndatasize
    returndatacopy(ptr, 0, size)

    switch result
    case 0 { revert(ptr, size) }
    default { return(ptr, size) }
 }
```
 
In order to delegate a call to another solidity contract function we have to pass it the msg.data the proxy received. Since msg.data is of type bytes, a dynamic data structure, it has a varying size that is stored in the first word size (of 32 bytes) in msg.data. If we wanted to extract just the actual data, we would need to step over the first word size, and start at 0x20 (32 bytes) of msg.data. However, there are two opcodes we’ll leverage to do this instead. We’ll use calldatasize to get the size of msg.data and calldatacopy to copy it over to our ptr variable.

Notice how we initialize our ptr variable. In Solidity, the memory slot at position 0x40 is special as it contains the value for the next available free memory pointer. Every time you save a variable directly to memory, you should consult where you should save it to by checking the value at 0x40. Now that we know where we’re allowed to save a variable, we can use calldatacopy to copy calldata of size calldatasize  starting from 0 of call data to location at ptr.
```
   let ptr := mload(0x40)
   calldatacopy(ptr, 0, calldatasize)
```
   
Let’s look at the following line in the assembly block that uses the delegatecall opcode:

```
   let result := delegatecall(gas, _impl, ptr, calldatasize, 0, 0)
```
*Parameters*

- gas  we pass in the gas needed to execute the function
- _impl the address of the logic contract we’re calling
- ptr the memory pointer for where data starts
- calldatasize the size of the data we’re passing.
- 0 for data out representing the returned value from calling the logic contract. This is unused because we do not yet know the size of data out and therefore cannot assign it to a variable. We can still access this information using returndata opcode later
- 0 for size out. This is unused because we didn’t get a chance to create a temp variable to store data out, since we didn’t know the size of it prior to calling the other contract. We can get this value using an alternative way by calling the returndatasize opcode later
The next line grabs the size of the returned data using the returndatasize opcode
```
   let size := returndatasize
```
And we use the size of the returned data to copy over the contents of returned data to our ptr variable with a helper opcode function returndatacopy
```
   returndatacopy(ptr, 0, size)
```
Lastly, the switch statement returns either the returned data or throws an exception if something went wrong.
![](https://i.loli.net/2018/12/19/5c19abac5e38d.jpg)


Great, we now have a way to retrieve the appropriate resulting value from the logic contract.

Now that we understand how the Proxy contract works, let’s look at Zeppelin’s three proposed patterns: Upgradeability using Inherited Storage, Unstructured Storage, and Eternal Storage.

The three approaches have different ways to tackle the same technical difficulty: how to ensure that the logic contract does not overwrite state variables that are used in the proxy for upgradeability.

The main concern with any proxy architecture pattern is how to handle storage allocation. Bear in mind that since we are using one contract for the storage and another for the logic, any one of them could potentially overwrite an already used storage slot. This means that if the Proxy contract has a state variable to keep track of the latest logic contract address at some storage slot and the logic contract doesn’t know about it, then the logic contract could store some other data in the same slot thus overwriting the proxy’s critical information. Zeppelin’s three approaches present different ways to architect your system to make your contracts upgradeable via a proxy pattern.

##Upgradeability using Inherited Storage
The Inherited Storage approach relies on making the logic contract incorporate the storage structure required by the proxy. Both the proxy and the logic contract inherit the same storage structure to ensure that both adhere to storing the necessary proxy state variables.

While exploring this approach we tried the idea of having a Registry contract to keep track of the different versions of your logic contract. In order to upgrade to a new logic contract you will need to register it as a new version in the Registry and ask the proxy to upgrade to it. Note that having a Registry does not affect the storage mechanism; in fact it could be implemented in any of the storage patterns showed in this post.

![](https://i.loli.net/2018/12/19/5c19f510773a6.jpg)

###How to Initialize

1. Deploy a Registry contract
2. Deploy an initial version of your contract (v1). Make sure it inherits the Upgradeable contract
3. Register the address of your initial version to the Registry
4. Ask the Registry contract to create an UpgradeabilityProxy instance
5. Call your UpgradeabilityProxy to upgrade to the your initial version of the contract

###How to Upgrade

1. Deploy a new version of your contract (v2) that inherits from your initial version to make sure it keeps the storage structure of the proxy and the one in the initial version of your contract.
2. Register the new version of your contract to the Registry
3. Call your UpgradeabilityProxy instance to upgrade to the new registered version.

###Key Takeaways

We can introduce upgraded functions as well as new functions and new state variables in future deployed logic contracts by still calling the same UpgradeabilityProxy contract.

##Upgradeability using Eternal Storage
In the Eternal Storage pattern, the storage schemas are defined in a separate contract that both the proxy and logic contract inherit from. The storage contract holds all the state variables the logic contract will need, and since the proxy is aware of them too, it can define its own state variables necessary for upgradeability without the concern of them being overwritten. Note that all future versions of the logic contract should not define any other state variable. All versions of the logic contract must always use the eternal storage structure defined in the beginning.

This implementation provided in the Zeppelin’s labs repository also introduces the concept of proxy ownership. A proxy owner is the only address that can upgrade a proxy to point to a new logic contract, and the only address that can transfer ownership.

![](https://i.loli.net/2018/12/19/5c19f569ec9e2.jpg)

###How to Initialize

1. Deploy an EternalStorageProxy instance
2. Deploy an initial version of your contract (v1)
3. Call your EternalStorageProxy instance to upgrade to the address of your initial version
4. If your logic contract relies on its constructor to set up some initial state, that would have to be redone after its linked to the proxy since the proxy’s storage doesn’t know about those values. EternalStorageProxy has a function upgradeToAndCall specifically to call some function on your logic contract to redo the setup after the proxy upgrades to it.

###How to Upgrade

1. Deploy a new version of your contract (v2) making sure it holds the eternal storage structure.  
2. Call your EternalStorageProxy instance to upgrade to the new version.

###Key Takeaways

Intuitive approach without significant overhead for the token logic contract. Future logic contacts can upgrade existing methods and introduce new ones, but should not introduce new state variables.

##Upgradeability using Unstructured Storage
The Unstructured Storage pattern is similar to Inherited Storage but doesn’t require the logic contract to inherit any state variables associated with upgradeability. This pattern uses an unstructured storage slot defined in the proxy contract to save the data required for upgradeability.

In the proxy contract we define a constant variable that, when hashed, should give a random enough storage position to store the address of the logic contract that the proxy should call to.
```
bytes32 private constant implementationPosition = 
                     keccak256("org.zeppelinos.proxy.implementation");
```
Since constant state variables do not occupy storage slots, there’s no concern of the implementationPosition being accidentally overwritten by the logic contract. Due to how Solidity lays out its state variables in storage there is extremely little chance of collision of this storage slot being used by something else defined in the logic contract.

By using this pattern, none of the logic contract versions have to know about the storage structure of the proxy, however all future logic contracts must inherit the storage variables declared by their ancestor versions. Just like in Inherited Storage pattern, future upgraded token logic contracts can upgrade existing functions as well as introduce new functions and new storage variables.

This implementation provided in the Zeppelin’s labs repository also uses the concept of proxy ownership. A proxy owner is the only address that can upgrade a proxy to point to a new logic contract, and the only address that can transfer ownership.

![](https://i.loli.net/2018/12/19/5c19f5e196540.jpg)

###How to Initialize

1. Deploy an OwnedUpgradeabilityProxy instance
2. Deploy an initial version of your contract (v1)
3. Call your OwnedUpgradeabilityProxy instance to upgrade to the address of your initial version
4. If your logic contract relies on its constructor to set up some initial state, that would have to be redone after its linked to the proxy since the proxy’s storage doesn’t know about those values. OwnedUpgradeabilityProxy has a function upgradeToAndCall specifically to call some function on your logic contract to redo the setup after the proxy upgrades to it.

###How to Upgrade

1. Deploy a new version of your contract (v2) making sure it inherits the state variable structures used in previous versions.
2. Call your OwnedUpgradeabilityProxy instance to upgrade to the address of your new contract version.  

###Key Takeaways

This approach is great as it does not require the Token logic contract to be aware whatsoever that it is part of a Proxy contract system.

##On Upgradeability
Important: if your logic contract relies on its constructor to set up some initial state, this has to be redone after the proxy upgrades to your logic contract. For example, it’s common for logic contracts to inherit from Zeppelin’s Ownable contract implementation. When your logic contract inherits from Ownable, it also inherits Ownable’s constructor that sets who the owner is upon contract creation. When you link the proxy contract to use your logic contract, the value of who the owner is from the proxy’s perspective is lost.

A common pattern for upgrading the proxy contract is that the proxy to immediately call an initialize method on the logic contract. The initialize method should mimic everything you would traditionally put in a constructor. You would also want to include a flag so that you can’t initialize a logic contract more than once.

Your logic contract should then look something like this:
```
contract Token is Ownable {
   ...
   bool internal _initialized;
   
   function initialize(address owner) public {
      require(!_initialized);
      setOwner(owner);
      _initialized = true;
   }
   ...
}
```
Depending on your deployment strategy, you could either have a helper deployer contract, or you could deploy the proxy and logic contract separately. If you deploy separately, you would link the proxy to your logic contract using upgradeToAndCall that might look something like this:
```
const initializeData = encodeCall('initialize', ['address'], [tokenOwner])
await proxy.upgradeToAndCall(logicContract.address, initializeData, { from: proxyOwner })
```
##Conclusion
The concept of proxy patterns has been around for awhile, but hasn’t received wide adoption due to its complexity, fear of introducing security vulnerabilities, and the controversy of bypassing blockchain’s immutability. Past solutions are also fairly inflexible requiring future logic contracts to be severely limited in what they can amend and add. However, it’s clear from a developer standpoint that there is a great need for the ability to upgrade contracts. Zeppelin has provided the code and tests for the three proxy patterns they explored to help developers architect their projects to incorporate upgradeability.

Even though the concept of proxy patterns isn’t new, its adoption is still early, and it’s exciting to see more advanced DApp architectures being achieved with this paradigm. If you built something with proxy patterns, let me and Zeppelin know on Twitter, and then join the Zeppelin slack channel to show it off there too :)

##Further Reading
As part of Zeppelin’s effort to implement zeppelin_os, a distributed platform of tools and services on top of the EVM, the Zeppelin team is currently moving forward with the Unstructured Storage approach. The Unstructured Storage approach has huge advantages of requiring the bare minimum implementation from logic contracts by introducing a novel way to maintain the proxy’s needed storage. Feel free to read more about Zeppelin’s choice of using Unstructured Storage for their upcoming release of zeppelin_os Kernel.

Zeppelin also posted a detailed [technical blog on Eternal Storage](https://blog.zeppelinos.org/smart-contract-upgradeability-using-eternal-storage/ "technical blog on Eternal Storage").

Over a year ago, Aragon and Zeppelin have teamed up to write two blog posts on Proxy Libraries that can be found [here](https://blog.zeppelin.solutions/proxy-libraries-in-solidity-79fbe4b970fd?gi=1f8d084a621e "here") and [here](https://blog.aragon.one/advanced-solidity-code-deployment-techniques-dc032665f434 "here").

Arachnid, or Nick Johnson, core developer on go-ethereum, lead developer of ENS, has written about his take on Upgradeable & Dispatcher contracts in this gist that he published over 2 years ago. 

If you’re looking to build something simple and won’t foresee huge changes in your future contracts, you might consider looking at this very simple example.  

Solidity docs are always helpful and I recommend looking at the doc’s for Solidity’s delegate call function and assembly opcodes.
### 参考

- [精通比特币第二版](https://github.com/bitcoinbook/bitcoinbook)
- [比特币开发指南](https://bitcoin.org/en/developer-guide)


