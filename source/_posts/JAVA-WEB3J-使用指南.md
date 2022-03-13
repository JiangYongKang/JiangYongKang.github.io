---
title: Java Web3J 使用指南
date: 2022-03-12 23:03:02
tags: Web3J
categories: 区块链
---

**Web3J** 是一个轻量级、高度模块化、反应式、类型安全的 **Java** 和 **Android** 库，用于处理智能合约并与以太坊网络上的客户端（节点）集成。这使您可以使用以太坊区块链，而无需为平台编写自己的集成代码的额外开销。

<!-- more -->

### 提供的功能

- 基于 HTTP 和 IPC 的以太坊 JSON-RPC 客户端 API 的完整实现
- 以太坊钱包支持
- 自动生成 Java 智能合约包装器，以从本机 Java 代码创建、部署、交易和调用智能合约（支持 Solidity 和 Truffle 定义格式）
- 以太坊名称服务 (ENS) 支持
- 支持 Alchemy 和 Infura，因此您不必自己运行以太坊客户端
- 安卓兼容
- 命令行工具

### 如何使用

```XML
<dependency>
  <groupId>org.web3j</groupId>
  <artifactId>core</artifactId>
  <version>4.8.7</version>
</dependency>
```

```Java
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Web3j client = Web3j.build(new HttpService("https://ropsten.infura.io/v3/You Infura Project Id"));
        Web3ClientVersion clientVersion = client.web3ClientVersion().sendAsync().get();
        System.out.println(clientVersion.getWeb3ClientVersion());
        // => Geth/v1.10.15-omnibus-hotfix-f4decf48/linux-amd64/go1.17.6
    }
}
```

### 使用 Web3J 调用 JSON RPC

##### Example-1: 获取以太坊账户余额

```Java
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Web3j client = Web3j.build(new HttpService("https://ropsten.infura.io/v3/You Infura Project Id"));
        EthGetBalance ethGetBalance = client.ethGetBalance(
                "0x64f44b31ad0ed4537f94a5c084cfba8945463345",
                DefaultBlockParameterName.fromString(DefaultBlockParameterName.LATEST.name())
        ).sendAsync().get();
        System.out.println(ethGetBalance.getBalance());
        // => 741270235881990866
    }
}
```

##### Example-2: 获取当前的 Gas 价格

```Java
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Web3j client = Web3j.build(new HttpService("https://ropsten.infura.io/v3/You Infura Project Id"));
        EthGasPrice ethGasPrice = client.ethGasPrice().sendAsync().get();
        System.out.println(ethGasPrice.getGasPrice());
        // => 41493167936
    }
}
```

##### Example-3: 通过交易哈希获取交易详情

```Java
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Web3j client = Web3j.build(new HttpService("https://ropsten.infura.io/v3/You Infura Project Id"));
        String transactionHash = "0x9030edd43f8ae6c4ed49bcbc11dd7d6f6ce2798e8bb1c5ea4f1e130780fec74a";
        EthGetTransactionReceipt ethGetTransactionReceipt = client.ethGetTransactionReceipt(transactionHash).sendAsync().get();
        TransactionReceipt transactionReceipt = ethGetTransactionReceipt.getTransactionReceipt().orElseThrow(RuntimeException::new);
        System.out.println(transactionReceipt);
        // => TransactionReceipt{transactionHash='0x9030edd43f8ae6c4ed49bcbc11dd7d6f6ce2798e8bb1c5ea4f1e130780fec74a', transactionIndex='0x27', blockHash='0xf0f28e6040d3440abb4abb1da59bf144fc2e210c4d321061d58da3fa3b011b0f', blockNumber='0xb84dee', cumulativeGasUsed='0x79d27a', gasUsed='0xc704', contractAddress='null', root='null', status='0x1', from='0x5ba191f4b0d70b778d9877e217dfb2c5daa353be', to='0xf80a32a835f79d7787e8a8ee5721d0feafd78108', logs=[Log{removed=false, logIndex='0x21', transactionIndex='0x27', transactionHash='0x9030edd43f8ae6c4ed49bcbc11dd7d6f6ce2798e8bb1c5ea4f1e130780fec74a', blockHash='0xf0f28e6040d3440abb4abb1da59bf144fc2e210c4d321061d58da3fa3b011b0f', blockNumber='0xb84dee', address='0xf80a32a835f79d7787e8a8ee5721d0feafd78108', data='0x00000000000000000000000000000000000000000000152d02c7e14af6800000', type='null', topics=[0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef, 0x0000000000000000000000000000000000000000000000000000000000000000, 0x0000000000000000000000005ba191f4b0d70b778d9877e217dfb2c5daa353be]}], logsBloom='0x80000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001000000008000000000000000000000000000000000000000000000000020000000000000000000800000000000000000000000010000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000002000000000000000200000000000000000000000000000000000020000000000200000000000000000000000000000000000000000000000000000000', revertReason='null', type='0x2', effectiveGasPrice='0x15fb4c0517'}
    }
}
```

### 使用 Web3J 订阅链上的事件

**Web3J** 的函数式编程的特性让我们设置观察者很容易，我们可以设置订阅者，监听链上发生的一些事情，如出块，交易，日志等。

##### Example-1: 订阅新的区块

```Java
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException, ConnectException {
        WebSocketService webSocketService = new WebSocketService("wss://ropsten.infura.io/ws/v3/You Infura Project Id", true);
        webSocketService.connect();
        Web3j client = Web3j.build(webSocketService);
        Disposable subscribe = client.replayPastBlocksFlowable(DefaultBlockParameterName.fromString("earliest"), true).subscribe(ethBlock -> {
            System.out.println(ethBlock.getBlock());
            // => org.web3j.protocol.core.methods.response.EthBlock$Block@39fcf950
        });
    }
}
```

##### Example-2: 订阅新的交易

```Java
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException, ConnectException {
        WebSocketService webSocketService = new WebSocketService("wss://ropsten.infura.io/ws/v3/You Infura Project Id", true);
        webSocketService.connect();
        Web3j client = Web3j.build(webSocketService);
        Disposable subscribe = client.replayPastTransactionsFlowable(DefaultBlockParameterName.fromString("earliest")).subscribe(transaction -> {
            System.out.println(transaction);
            // => org.web3j.protocol.core.methods.response.EthBlock$TransactionObject@47b0a71a
        });
    }
}
```

##### Example-2: 订阅新的合约事件

```Java
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException, ConnectException {
        WebSocketService webSocketService = new WebSocketService("wss://ropsten.infura.io/ws/v3/You Infura Project Id", true);
        webSocketService.connect();
        Web3j client = Web3j.build(webSocketService);
        Disposable subscribe = client.ethLogFlowable(
                new EthFilter(
                        DefaultBlockParameterName.EARLIEST,
                        DefaultBlockParameterName.LATEST,
                        // 合约地址
                        "0x7b52aae43e962ce6a9a1b7e79f549582ae8bcff9"
                )
        ).subscribe(event -> {
            System.out.println(event);
            // => Log{removed=false, logIndex='0x39', transactionIndex='0x13', transactionHash='0x9c3653c27946cb39c3a256229634cfedbca54f146a740f46b661b986590f8358', blockHash='0x1e674b9d111601519f977b75f9a1865ba359b474550ff5d0d87daec1f543d64d', blockNumber='0xb83ccd', address='0x7b52aae43e962ce6a9a1b7e79f549582ae8bcff9', data='0x', type='null', topics=[0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef, 0x0000000000000000000000000000000000000000000000000000000000000000, 0x0000000000000000000000006ab3d0bd875f8667026d2a9bb3060ec44380bcc2, 0x0000000000000000000000000000000000000000000000000000000000000017]}
        }, Throwable::printStackTrace);
    }
}
```

### 使用 Web3J 签名并发送交易

```Java
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException, ConnectException {
        Web3j client = Web3j.build(new HttpService("https://ropsten.infura.io/v3/You Infura Project Id"));

        // 获取 nonce 值
        EthGetTransactionCount ethGetTransactionCount = client
                .ethGetTransactionCount("0x64f44b31ad0ed4537f94a5c084cfba8945463345", DefaultBlockParameterName.PENDING)
                .sendAsync().get();
        BigInteger nonce = ethGetTransactionCount.getTransactionCount();
        System.out.println(nonce);

        // 构建交易
        RawTransaction etherTransaction = RawTransaction.createEtherTransaction(
                nonce,
                client.ethGasPrice().sendAsync().get().getGasPrice(),
                DefaultGasProvider.GAS_LIMIT,
                "0x64f44b31ad0ed4537f94a5c084cfba8945463345",
                Convert.toWei("0.001", Convert.Unit.ETHER).toBigInteger()
        );
        System.out.println(etherTransaction);

        // 加载私钥
        Credentials credentials = Credentials.create("You Private Key");

        // 使用私钥签名交易并发送
        byte[] signature = TransactionEncoder.signMessage(etherTransaction, credentials);
        String signatureHexValue = Numeric.toHexString(signature);
        EthSendTransaction ethSendTransaction = client.ethSendRawTransaction(signatureHexValue).sendAsync().get();
        System.out.println(ethSendTransaction.getResult());
    }
}
```

### 使用 Web3J 生成合约包装类

Web3J 可以生成智能合约的包装类，方便使用纯 Java 代码与合约进行交互。

```Shell
$ web3j generate solidity -a ./contract.abi -o ./ -p com.contract.proxy
              _      _____ _
             | |    |____ (_)
__      _____| |__      / /_
\ \ /\ / / _ \ '_ \     \ \ |
 \ V  V /  __/ |_) |.___/ / |
  \_/\_/ \___|_.__/ \____/| |
                         _/ |
                        |__/
by Web3Labs
Generating com.contract.proxy.Contract ...
Warning: Duplicate field(s) found: [FUNC_SAFETRANSFERFROM]. Please don't use names which will be the same in uppercase.
File written to .

$ tree com
com
└── contract
    └── proxy
        └── Contract.java # 包装类

2 directories, 1 file
```

### 使用 Web3J 与合约进行交互

```Java
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Web3j client = Web3j.build(new HttpService("https://ropsten.infura.io/v3/You Infura Project Id"));
        Credentials credentials = Credentials.create("You Private Key");
        BigInteger currentGasPrice = client.ethGasPrice().sendAsync().get().getGasPrice();
        Contract foundersKeyContract = Contract.load("0x7b52aae43e962ce6a9a1b7e79f549582ae8bcff9", client, credentials, new DefaultGasProvider() {
            /**
             * 使用动态获取的 Gas Price
             * @return
             */
            @Override
            public BigInteger getGasPrice() {
                return currentGasPrice;
            }
        });
        BigInteger balance = foundersKeyContract.balanceOf("0x64f44b31ad0ed4537f94a5c084cfba8945463345").send();
        System.out.println(balance);
        // => 41493167936
    }
}
```