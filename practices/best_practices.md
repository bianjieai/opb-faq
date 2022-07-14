# 文昌链最佳实践建议

## 简介

感谢技术社区对于文昌链的信任，随着文昌链支持的应用越来越多，我们关注到技术社区中许多开发同学遇到的共性问题。为此我们决定写一篇最佳实践来指导解决大家的问题

### 概念介绍

节点分类：

1. 共识节点：区块链网络中负责出块的网络
2. 全节点：区块链网络中负责接收查询交易的节点（不参与共识）

## 实施建议

### 建议1: 加入文昌链技术社区，关注社区讨论及技术团队的咨询回复

文昌链现在的业务发展非常快，技术服务团队会第一时间更新链的服务发展，例如：

1. 链升级内容及对用户的影响
2. 产品升级内容及对用户的影响
3. 升级时间对用户造成的影响
4. SDK & AVATA 的更新内容及对产品的影响
5. 同时社区中还有其他热心社区成员提供帮助，有机器人帮助你找到不同资源的入口


### 建议2: 使用 Sync 模式而不是 Commit 模式或 Async 模式提交交易

#### 不同交易模式的解释：

1. Commit 模式：用户提交交易后会建立一个结果接收线程等待链通知交易执行的结果；
2. Sync 模式：用户提交交易后会链经过验证后会返回给用户一个交易 hash
3. Async 模式：用户提交交易后，立刻返回一个交易 hash

#### 为什么要使用 Sync 模式？

1. Commit 模式 **不推荐使用**：在链的并发较大时，可能会导致等待线程超时，无法把当前交易的执行结果返回给用户。如果链的压力比较大的时候，交易可能无法发送。（未来的计划是会删除此种交易模式）
2. Sync 模式 **推荐使用**：对于用户提交的交易，Sync 模式发送的交易，只需要经过了合法性验证就可以进入 mempool(交易池) 等待被广播到共识节点，用户如果想要拿到结果，仅需要查询一次提交交易返回的 hash 即可，读写分离，性能高
3. Async 模式 **不推荐使用**： Async 模式不做任何的交易校验执行，仅会根据当前的交易内容返回一个 hash，用户无法感知交易是否被发送到链上。

#### 示例
##### OPB-SDK-Go

对于使用 opb-sdk-go 的用户在配置的时候请做如下配置：

```go
    options := []types.Option{
        types.KeyDAOOption(store.NewMemory(nil)),
        types.FeeOption(types.NewDecCoinsFromCoins(fee)),
        types.ModeOption(coretypes.Sync), // 配置广播模式
    }
    //..... 其他掠过
    // 初始化 Tx 基础参数
    baseTx := types.BaseTx{
        From:     "test_key_name", // 对应上面导入的私钥名称
        Password: "test_password", // 对应上面导入的私钥密码
        Gas:      200000,          // 单 Tx 消耗的 Gas 上限
        Memo:     "",              // Tx 备注
        Mode:     types.Sync,    // Tx 广播模式
    }
```
##### OPB-SDK-JAVA

对于使用 opb-sdk-java 的用户在配置的时候请做如下配置：

```java
    //使用BroadcastMode.Sync 指定Sync模式
    BaseTx baseTx = new BaseTx(200000, new Fee("200000", "ugas"), BroadcastMode.Sync);
```
### 建议3: 离线维护 Sequence/Nonce 并配合 Sync 模式

推荐架构如下：

![推荐建构](./Recommended-Architecture.png)

#### 示例

##### 对于 OPB-SDK-Go

opb-sdk-go 默认做了维护 sequence （注意：不要完全信任，这个值是存在于内存中的，如果网络抖动，可能会导致 sequence 不对）可以用如下方式自己本地静态维护：

###### 签名和广播分开（推荐做法，和架构图一致）

```go

    // 伪代码
    options := []types.Option{
        types.KeyDAOOption(store.NewMemory(nil)),
        types.FeeOption(types.NewDecCoinsFromCoins(fee)),
        types.ModeOption(coretypes.Sync),
    }
    
    //  ..... 其他掠过
    // 创建客户端
    client := opb.NewClient(cfg, &authToken)
    
    // 初始化 Tx 基础参数
    baseTx := types.BaseTx{
        From:     "test_key_name", // 对应上面导入的私钥名称
        Password: "test_password", // 对应上面导入的私钥密码
        Gas:      200000,          // 单 Tx 消耗的 Gas 上限
        Memo:     "",              // Tx 备注
        Mode:     types.Sync,    // Tx 广播模式
    }
    accountAddr := "iaa1lxvmp9h0v0dhzetmhstrmw3ecpplp5tljnr35f"
    baseAccount, err := client.QueryAccount(accountAddr)
    if err != nil {
        return
    }
    
    // 初始的 sequence 可以从 baseAccount 拿到
    // 获取最新的离线的 sequence
    sequence = getCurSequence(accountAddr)
    
    // 创建写入的 msgs
    // msgs 是一个接口数组，任何文昌链的原生交易的结构都可以 append 到这个数组
    var msgs coretypes.Msgs
    tmpNFTID := "testnft001"
    tmpMsg := &nft.MsgMintNFT{
        Id:        tmpNFTID,
        DenomId:   "testclass",
        Name:      "testnftname",
        URI:       "http://example.com",
        Data:      "",
        Sender:    accountAddr,
        Recipient: accountAddr,
    }
    msgs = append(msgs, tmpMsg)
    
    // 签名
    signTx, err := client.BuildAndSignWithAccount(feeGraterAddr, baseAccount.AccountNumber, sequence, msgs, baseTx)

    if err != nil{
        return
    }
    //...
    // 把 signTx 发到队列中（先进先出的队列）
    // 广播
    // 注意签名和广播可以放到不同的程序中，参考架构图，
    result, err := tc.BroadcastTxSync(context.Background(), signTx)
    if err != nil{
        return
    }
 
    // 更新sequence(必须等结果返回后，这个时候代表交易已经进入链的交易池子，等待被广播)
    sequence += 1

    //.....

```


##### 签名和广播不分开

**注意：** opb-go-sdk v0.2 会自动在内存中维护账户的 sequence ，但不要过度信任，在大多数情况下是可用的，但在网络特别差的时候可能会出现意外。

```go

    // 伪代码
    options := []types.Option{
        types.KeyDAOOption(store.NewMemory(nil)),
        types.FeeOption(types.NewDecCoinsFromCoins(fee)),
        types.ModeOption(coretypes.Sync),
    }
    
    //  ..... 其他掠过
    // 创建客户端
    client := opb.NewClient(cfg, &authToken)
    
    // 初始化 Tx 基础参数
    baseTx := types.BaseTx{
        From:     "test_key_name", // 对应上面导入的私钥名称
        Password: "test_password", // 对应上面导入的私钥密码
        Gas:      200000,          // 单 Tx 消耗的 Gas 上限
        Memo:     "",              // Tx 备注
        Mode:     types.Sync,    // Tx 广播模式
    }
    accountAddr := "iaa1lxvmp9h0v0dhzetmhstrmw3ecpplp5tljnr35f"
    baseAccount, err := client.QueryAccount(accountAddr)
    if err != nil {
        return
    }
    
    // 初始的 sequence 可以从 baseAccount 拿到
    
    // 创建写入的 msgs
    // msgs 是一个接口数组，任何文昌链的原生交易的结构都可以 append 到这个数组
    var msgs coretypes.Msgs
    tmpNFTID := "testnft001"
    tmpMsg := &nft.MsgMintNFT{
        Id:        tmpNFTID,
        DenomId:   "testclass",
        Name:      "testnftname",
        URI:       "http://example.com",
        Data:      "",
        Sender:    accountAddr,
        Recipient: accountAddr,
    }
    msgs = append(msgs, tmpMsg)
    
    // 签名并广播
    result, err := client.BuildAndSend(msgs, baseTx)
	if err != nil {
		return
	}
    // 判断 result 没有错误在发送下一笔

    //.....

```
##### 对于 OPB-SDK-JAVA
```java
        //指定Sync模式
        BaseTx baseTx = new BaseTx(200000, new Fee("200000", "ugas"), BroadcastMode.Sync);
        Account account = baseClient.queryAccount(baseTx);
		
        //获取sequence
        // 初始的 sequence 可以从 baseAccount 拿到
        // 获取最新的离线的 sequence
        long sequence = account.getSequence();
        // 创建写入的 msgs	
        Tx.MsgMintMT.Builder mintMTBuilder = Tx.MsgMintMT
                .newBuilder()
                .setDenomId(denomId)
                .setAmount(20)
                .setData(ByteString.copyFrom(data.getBytes()))
                .setSender(account.getAddress());
        mintMTBuilder.setRecipient(recipient);
        Tx.MsgMintMT msgMintMT = mintMTBuilder.build();
        List<GeneratedMessageV3> msgs2 = Collections.singletonList(mintMT2);
        //签名 广播交易
        ResultTx resultTx2 =  baseClient.buildAndSend(msgs2, baseTx, account);
        // 更新sequence(必须等结果返回后，这个时候代表交易已经进入链的交易池子，等待被广播)
        sequence += 1
        //.....
```

### 建议4：对于 EVM 交易和 DDC 的并发交易必须离线维护 Nonce

**所有的交易是由全节点接收的。** 
当区块链网络的压力比较大时，全节点的交易可能无法被及时的 广播到共识节点，如果不离线维护 Nonce 值，可能会导致出现一些关于 Nonce 值的错误；
#### 使用建议

1. 如果你是使用 DDC-SDK-Java 的用户请使用 SDK 中 setNonce 的操作
2. 如果你是使用其他 web3 SDK 请根据他们文档中提供的方式设置 Nonce
3. 最佳实践架构和 建议3 类似

**注意：** 对于 EVM 交易，正确的做法是：**发送完一笔交易后查询一下交易的执行结果。** EVM 的目前的机制是调用失败 Nonce 值不会增加
