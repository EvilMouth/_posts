---
layout: post
title: 智能合约真好玩
date: 2018-12-29 14:51:16
updated: 2018-12-29 14:51:16
tags:
  - ethereum
  - contract
categories: Blockchain
---

> 吐槽一下 Mist 客户端 mac 版，网络连接异常+是不是崩溃，不过智能合约开发起来真好玩
> 这几天学习了智能合约开发语言 solidity，实践起来部署了一份合约并在以太坊主网验证发布，拿着开发的币在测试账号转来转去超好玩。这几天开发遇到了各种各样的小问题和智能合约开发需要注意的一些问题，总结记录一下

<!-- More -->

### rinkeby 测试节点

在 mist 客户端可以切换节点到 rinkeby，就可以测试开发，还可以在 rinkeby 上获取点 eth 来部署合约
[rinkeby](https://www.rinkeby.io/#stats)

### remix 在线开发

使用[remix](https://remix.ethereum.org/#optimize=false)在线测试部署你的合约

### sol 文件

智能合约是用 solidity 语言开发，也就是 sol 后缀，我是用 vscode+sol 插件开发的

### event 事件兼容

定义一个 event 事件如下

```solidity
event Something(unit256 value);
```

触发 event 事件需要使用 emit

```solidity
emit Something(value);
```

### 所有者访问限制 - modifier

某些方法如果需要加上身份验证，可以使用 modifier，首先定义一个 modifier

```solidity
modifier onlyOwner() {
    require(msg.sender == ethFundDeposit, "auth fail");
    _;
}
```

之后直接在需要验证身份的方法后面加上 onlyOwner，例如

```solidity
function action() external onlyOwner {
    ...
}
```

- external 必须在最前

### throw 弃用 - require assert revert

以前使用 throw 来抛出异常现在有三个代替语法

- require(condition, string) 一般放在方法最前面，会退回剩余 gas
- assert(condition) 会消耗所有 gas
- revert(string) 会撤销修改状态，会退回剩余 gas

### constant view pure

constant 被拆分成 view 和 pure

- view 与 constant 效果一致，只能读状态变量不能改
- pure 不能改甚至不能读状态变量

### decimals

一开始看到计算总量的时候以为看错，原来只是精度

### constructor

构造函数需要使用 constructor 声明，并且是 public 修饰

- 注意 constructor 不需要 function 声明，否则会有安全问题

### No data is deployed on the contract address!

在部署合约的时候可能会遇到这种 gas 不足的问题，可以通过手动加 gas 解决，虽然会花费多一些 eth
