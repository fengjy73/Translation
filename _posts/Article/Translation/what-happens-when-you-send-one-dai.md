---
title: What happens when you send 1 DAI
authorURL: ""
originalURL: https://www.notonlyowner.com/learn/what-happens-when-you-send-one-dai
translator: "fengjy73"
reviewer: ""
---

# 当你发送1DAI的时候会发生什么

<!-- more -->

---

![article cover](https://notonlyowner.com/images/what-happens-dai-cover-intro.png)

你拥有1个 [DAI][2].

使用钱包界面 (例如 [Metamask][3]), 你点击足够多的按钮并输入足够多的文本来确认您要向`0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` (即vitalik.eth)这个地址发送 1 DAI。

然后点击发送

一段时间后，钱包显示交易已确认。突然间，Vitalik 多了 1 个 DAI。到底发生了什么？

让我们倒带。然后以慢动作重播这个过程。

准备好了吗?

---

## 目录

1.  [构建交易][4]
    -   [交易的数据字段][5]
    -   [Gas的魔法][6]
    -   [访问控制列表和交易类型][7]
    -   [对交易签名][8]
    -   [序列化][9]
    -   [提交交易][10]
2.  [接收交易][11]
    -   [检查内存池][12]
3.  [传播][13]
4.  [准备工作和交易打包][14]
5.  [执行][15]
    -   [准备工作(第1部分)][16]
    -   [准备工作(第2部分)][17]
    -   [调用][18]
    -   [解释器(第1部分)][19]
    -   [Solidity执行][20]
    -   [EVM执行][21]
        -   [释放内存指针并调用值][22]
        -   [验证calldata (第1部分)][23]
        -   [函数调度器][24]
        -   [验证calldata (第2部分)][25]
        -   [读取参数][26]
        -   [transfer函数][27]
        -   [transferFrom函数][28]
        -   [日志记录][29]
        -   [返回][30]
    -   [解释器(第2部分)][31]
    -   [Gas退还和收取][32]
    -   [构建交易收据][33]
6.  [打包区块][34]
7.  [广播区块][35]
8.  [验证区块][36]
9.  [检索交易][37]
10.  [后记][38]

---

## 构建交易

[钱包][39] 是一种可以方便地向以太坊网络发送 _交易_ 的软件.

一笔交易是你作为用户告诉以太坊网络你想执行一个操作的方式。在这个案例中，你先执行的操作是向Vitalik发送1个DAI。钱包（例如Metamask）以相对新手友好的方式帮助构建这样的交易。

让我们首先速览一下钱包将构建的交易。它可以表示为一个具有字段及其相应数值的对象。

在我们的案例中，最开始看起来是这样的:

`{     "to": "0x6b175474e89094c44da98b954eedeac495271d0f",     // [...] }`

其中 字段 _to_ 表示目标地址. 在本案例中, `0x6b175474e89094c44da98b954eedeac495271d0f` 是DAI智能合约的地址.

等等, 发生了什么?

我们不是应该向Vitalik发送1个DAI吗？ `to` 难道不是应该是 Vitalik 的地址吗?


发送 DAI 时，基本上必须构建一个交易，用来执行存储在区块链（也就是以太坊数据库的一个花哨名称）中的一段代码，从而 _更新_ DAI 余额记录。执行此类更新所需的逻辑和相关存储都保存在以太坊数据库中的一个不可篡改的、公共的计算机程序中。即 DAI 智能合约。

因此，你想构建一笔交易来告诉合约“嘿，朋友，更新你的内部余额吧，从我的余额中取出1个DAI，并将1个DAI添加到Vitalik的余额中”。在以太坊行话中，“嘿，朋友”这个短语被翻译为在交易的 `to` 字段中设置DAI的地址。

然而， 光有`to` 字段还不足够。根据你在你最喜欢的钱包界面提供的信息，钱包会填写其他几个字段，从而构建格式完整的交易。

`{     "to": "0x6b175474e89094c44da98b954eedeac495271d0f",     "amount": 0,     "chainId": 31337,     "nonce": 0,     // [...] }`

它用 `0` 填充了 `amount` 字段。所以你要向 Vitalik 发送 `1` DAI，既不使用 Vitalik 的地址，也不在 `amount` 字段中放置 `1` 。这就是生活的艰辛（我们只是在热身）。 `amount` 字段也需要在交易中包含是因为要用于指定你在交易中发送了多少以太币（以太坊的原生货币）。由于你现在不想发送以太币，那么钱包会正确地将该字段设置为 `0`。

至于 `chainId` , 它是一个用于指定交易在哪条链上被执行的字段. 对于以太坊的主网, `chainId` 的值是1. 然而,  **由于我们要在主网的本地副本上进行这个实验**, 所以我将chain ID指定为了: 31337. [其他的链具有其他的ID号][40].

关于 `nonce` 字段呢？这是一个数字，每次向网络发送交易时都应该增加它。它是一种用于避免重放问题的防御机制。钱包通常会为您设置它。为此，它们查询网络，查询您的账户最新使用的`nonce`值，然后相应地设置当前交易的`nonce`值。在上面的示例中，它设置为0，但实际上这将取决于您的账户已经执行的交易数量。

我刚刚说钱包“查询网络”。我的意思是钱包对以太坊节点执行只读调用，然后节点会用请求的数据进行响应。有多种方式可以从以太坊节点读取数据，这取决于节点的位置以及它所公开的API类型。

让我们想象一下，钱包已经直连到以太坊节点的网络。不过更常见的情况是，钱包是与第三方提供商（如Infura、Alchemy、QuickNode等）进行交互。与节点交互的请求遵循特殊的协议，以执行远程调用。这个协议被称为 [JSON-RPC][41]。

钱包发出的尝试获取账户`nonce`的请求将类似于以下内容：

`POST / HTTP/1.1 connection: keep-alive Content-Type: application/json content-length: 124  {     "jsonrpc":"2.0",     "method":"eth_getTransactionCount",     "params":["0x6fC27A75d76d8563840691DDE7a947d7f3F179ba","latest"],     "id":6 } --- HTTP/1.1 200 OK Content-Type: application/json Content-Length: 42  {"jsonrpc":"2.0","id":6,"result":"0x0"}`

其中，发送方的账户应该是 `0x6fC27A75d76d8563840691DDE7a947d7f3F179ba`。 从响应的数据中可以看到它的nonce值是0

钱包通过网络请求获取数据（在本例中，通过HTTP），以访问节点公开的JSON-RPC端点。上面我只写了一个请求，但实际上，钱包可以查询它们需要构建交易的任何数据。如果在现实案例中注意到更多的网络请求来查找其他内容，不要感到惊讶。例如，以下是几分钟内Metamask流量访问本地测试节点的片段：

![在本地网络中Metamask流量的Wireshark截图快照](https://notonlyowner.com/images/metamask-traffic.png)

### 交易的数据字段

DAI是一个智能合约. 它的核心逻辑在以太坊主网`0x6b175474e89094c44da98b954eedeac495271d0f`这个地址上实现.

更具体地说，DAI是符合ERC20标准的同质化代币——一种非常特殊的合约类型。这意味着至少DAI应该实现[ERC20 标准][42]中详细描述的接口。在（有些牵强的）web2行话中，DAI是在以太坊上运行的不可篡改的开源网络服务。鉴于它遵循ERC20规范，可以预先知道（而不一定要查看源代码）与之交互的具体公开的端点。

短小的说明：并非所有的ERC20代币都是相同的。实现特定接口（用于促进交互和集成）并不一定保证具体的行为。尽管如此，对于这个练习，我们可以安全地假设DAI在行为上是一个相当标准的ERC20代币。

DAI智能合约中有许多功能（源代码可在[此处][43]找到），其中许多直接取自ERC20规范。特别引人关注的是[只能被合约外部调用的 `transfer` 函数][44。

`contract Dai is LibNote {     ...     function transfer(address dst, uint wad) external returns (bool) {         ...     } }`

该函数允许持有 DAI 代币的任何人将其中一些代币转移到另一个以太坊账户。其声明是 `transfer(address,uint256)` 。第一个参数是接收账户的地址，第二个参数是表示要转移的代币数量的无符号整数。

暂时不要专注于函数行为的具体细节。相信我，当函数按照预期运行时，它会减少发送者的余额，并相应地增加接收者的余额。

这很重要，因为在构建与智能合约交互的交易时，应该知道要执行合约的哪个函数，以及要传递哪些参数。就像在web2中，如果你想向web API发送POST请求一样。您很可能需要在请求中指定带有参数的准确的URL。这是一样的。我们想要转移1个DAI，所以我们必须知道如何在交易中指定要去执行DAI智能合约上的 `transfer` 函数。

幸运的是，这是非常直接和直观的。

开玩笑的。不是的。这是你必须包括在发送1个DAI给Vitalik这个交易中内容（记住，地址是 `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` ）。

`{     // [...]     "data": "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000" }`

让我解释一下。

为了简化集成过程并制定一种标准化的方式与智能合约进行交互，以太坊生态系统已经（一定程度上）开始采用所谓的“合约ABI规范”（ABI代表应用二进制接口）。在常见的使用情况下，我强调，是在常见的使用情况下，为了执行智能合约函数，您必须首先按照[合约ABI规范][45]对函数调用进行编码。更高阶的使用情况可能不遵循这个规范，但我们 _绝对_ 不会深入探讨这个问题。可以说，通常使用[Solidity][46]编程的智能合约，比如DAI，通常遵循合约ABI规范。


你在上面看到的是对将1DAI转账到地址 `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` 并使用DAI的 `transfer(address,uint256)` 函数的调用进行ABI编码后得到的字节 。

有许多工具可以对ABI进行编码，大多数钱包都在一定程度上实现了ABI编码以与合约进行交互。为了举例说明，我们可以使用一个名为[cast][47]的命令行工具来验证上述字节序列是否正确，该程序能够使用指定参数对调用进行ABI编码：

`$ cast calldata "transfer(address,uint256)" 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 1000000000000000000  0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000`

有什么困扰你的吗？怎么了？

抱歉，是的。那个1000000000000000000。老实说，我 _真的_ 很想为你提供更有力的论据。解释是：许多ERC20代币使用18位小数进行表示，比如DAI。然而，我们只能使用无符号整数。因此，1个DAI实际上存储为1 \* 10^18 - 也就是1000000000000000000。就这样吧。

我们有一个美丽的ABI编码的字节序列，它需要被包含在交易的 `data` 字段中。到目前为止我们构建的交易看起来是：

`{     "to": "0x6b175474e89094c44da98b954eedeac495271d0f",     "amount": 0,     "chainId": 31337,     "nonce": 0,     "data": "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000" }`

一旦我们开始实际执行交易，我们将重新审视这个 data 字段的内容。

### Gas的魔法

下一个步骤是，要确定为交易支付多少费用。请记住，所有交易都必须向网络中的节点支付费用，因为这些节点需要时间和资源来执行和验证它们。

执行交易的成本是以以太币支付的。最终的以太币数量将取决于你的交易消耗了多少[gas][48]（即计算成本有多高）、你愿意为每个gas单位花销支付多少以及网络愿意接受的最低金额是多少。

从用户的角度来看，底线 _通常_ 是支付的金额越多，交易被打包在区块里的速度就越快。因此，如果你想在下一个区块中支付 Vitalik 1 个 DAI，你可能需要设置比愿意等待几分钟（或更长时间，有时 _甚至_ 更长）直到 gas 价格更便宜时更高的费用。

不同的钱包可能会采取不同的方式来决定支付多少gas费。我还不知道一个确保所有人使用时都的万无一失的机制。确定正确费用的策略可能涉及从节点查询与gas相关的信息（例如网络接受的最低基本费用）。

例如，在以下请求中，你可以看到Metamask浏览器扩展在构建交易时向本地测试节点发送请求以获取gas费数据：

![向节点查询gas有关数据时的Metamask网络流量](https://notonlyowner.com/images/metamask-gas-traffic.png)

简化后的请求——相应如下：

`POST / HTTP/1.1 Content-Type: application/json Content-Length: 99  {     "id":3951089899794639,     "jsonrpc":"2.0",     "method":"eth_feeHistory",     "params":["0x1","0x1",[10,20,30]] } --- HTTP/1.1 200 OK Content-Type: application/json Content-Length: 190  {     "jsonrpc":"2.0",     "id":3951089899794639,     "result":{         "oldestBlock":"0x1",         "baseFeePerGas":["0x342770c0","0x2da4d8cd"],         "gasUsedRatio":[0.0007],         "reward":[["0x59682f00","0x59682f00","0x59682f00"]]     } }`

其中，`eth_feeHistory` 端点由一些允许查询交易费数据节点公开。如果你感兴趣，可以在[这里][49]阅读或尝试操作[这个][50]，或者在[这里][51]查看规范。

热门钱包还使用更复杂的链下服务来获取gas价格估算值从而向用户建议合理的数值。以下是一个钱包访问网络服务的公共端点，并接收大量有用的与gas相关的数据的示例：

![Wireshark中包括以太坊历史gas费查询请求的网络流量](https://notonlyowner.com/images/gas-data-requests.png)

让我们看一下响应的片段：

![Wireshark中包括以太坊历史gas费查询响应的网络流量](https://notonlyowner.com/images/gas-data-response.png)

这很酷，对吧？

希望你已经开始逐渐了解设定gas价格并不是一件简单的事情，并且这是构建成功交易的基本步骤。即使你只是想发送1个DAI。[这][52]是一个有趣的入门指南，可以深入了解一些涉及更准确地设置交易费用的机制。

在了解基本背景之后，现在让我们回到实际的交易。有三个与gas相关的字段需要设置：

`{     "maxPriorityFeePerGas": ...,     "maxFeePerGas": ...,     "gasLimit": ..., }`

钱包将使用前面提到的机制来为你填充好前两个字段。有趣的是，每当钱包用户界面让您在“慢速”、“常规”或“快速”交易版本之间进行选择时，实际上是在尝试决定哪些数值最适合这些参数。现在你可以更好地理解我之前展示给你的钱包收到的JSON格式响应的内容。

要确定第三个字段的值，即gas限制，钱包用一个方便的机制来模拟交易在实际提交之前会消耗多少gas。这使它们能够近似估计交易将消耗多少gas，从而设置合理的燃气限制，在除了为您提供交易的最终美元成本估算之外。

为什么不设置一个无限大的gas限制呢？当然是为了保护你的资金。智能合约可能具有任意逻辑，而你是为其执行付费的人。通过在交易开始时选择合理的gas限制，您可以防止不良情况发生，这些不良情况可能会导致所有您账户中的以太币资金被gas费耗尽。

Gas 估算可以使用节点的 `eth_estimateGas` 端点进行。在发送1 DAI之前，钱包可以利用这种机制来模拟你的交易，并确定 DAI 转账的正确 gas 限制。这是钱包的请求——响应看起来的样子。

`POST / HTTP/1.1 Content-Type: application/json  {     "id":2697097754525,     "jsonrpc":"2.0",     "method":"eth_estimateGas",     "params":[         {             "from":"0x6fC27A75d76d8563840691DDE7a947d7f3F179ba",             "value":"0x0",             "data":"0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000",             "to":"0x6b175474e89094c44da98b954eedeac495271d0f"         }     ] } --- HTTP/1.1 200 OK Content-Type: application/json  {"jsonrpc":"2.0","id":2697097754525,"result":"0x8792"}`

在响应中，您可以看到转账大约需要34706个单元的gas。

让我们将这些信息合并到交易有效负载中：

`{     "to": "0x6b175474e89094c44da98b954eedeac495271d0f",     "amount": 0,     "chainId": 31337,     "nonce": 0,     "data": "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000",     "maxPriorityFeePerGas": 2000000000,     "maxFeePerGas": 120000000000,     "gasLimit": 40000 }`

请记住， `maxPriorityFeePerGas` 和 `maxFeePerGas` 最终取决于发送交易时的网络条件。上面我只是为了举例而设置了一些相对随意的值。至于gas限制的设定值，我只是在估算值上稍微增加了一些以确保安全。

### 访问控制列表和交易类型

让我们简要讨论一下你的交易中需要设置的另外两个字段。

首先是 `accessList` 字段。高级使用场景或边界情况可能需要事先指定交易要访问的账户地址和合约存储槽，从而使其成本略微降低。

然而，事先建立这样的清单可能并不那么简单，目前的gas节省可能并不那么显著。特别是对于发送1 DAI这样的简单交易。因此，我们可以将其设置为空列表。但要记住，它确实是有其[存在的理由][53]，而且在未来可能变得更加重要。

第二，[交易的类型][54]。它在 `type` 字段中指定。类型用来指示是交易内部的内容的。我们的例子属于类型2交易，因为它遵循[这里][55]指定的格式。


`{     "to": "0x6b175474e89094c44da98b954eedeac495271d0f",     "amount": 0,     "chainId": 31337,     "nonce": 0,     "data": "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000",     "maxPriorityFeePerGas": 2000000000,     "maxFeePerGas": 120000000000,     "gasLimit": 40000,     "accessList": [],     "type": 2 }`

### 对交易签名

节点如何知道是 _您的_ 账户而不是别人的账户发送的交易？

我们来到了构建有效交易的关键步骤：对交易签名。

一旦钱包收集到足够的信息来构建交易，并且你点击了发送，它将对你的交易进行数字签名。具体过程？使用你账户的私钥（你的钱包可以访问）和一个涉及椭圆曲线的加密算法，称为 [ECDSA][56]。

满足一下你的好奇心，实际上被签名的是将交易类型和交易的 [RLP 编码][57] 的拼接起来后取`keccak256` 哈希

`keccak256(0x02 || rlp([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas, gasLimit, to, amount, data, accessList]))`

你不必对密码学有深入了解才能理解这这一部分。简单来说，这个过程封装了交易。它通过在交易上加上一个只有你的私钥才能产生的智能印章，使其变得不可篡改。从现在开始，任何可以访问该签名交易的人（例如，以太坊节点）都可用密码学验证这个交易是你的账户产生的。

顺便说一下：签名 _并不等同于_ 加密。你的交易始终以明文形式存在。一旦公开，任何人都可以看到其内容。

对交易签名的过程毫无意外地产生了一个签名。实际上，这是一堆看起来奇怪且难以阅读的数值。这些数值随交易一起被传播，通常被称为 `v`、`r` 和 `s`。如果你想更深入地了解这些实际代表什么，以及它们在恢复你的账户地址方面的重要性，互联网是你的好帮手。

你可以通过查看 [@ethereumjs/tx][58] 这个软件包来更好地了解签名的实现方式。还可以使用 [ethers][59] 这样的工具进行一些实践操作。作为一个极其简化的例子，发送 1 DAI 的交易签名可能如下所示：

`const { FeeMarketEIP1559Transaction } = require("@ethereumjs/tx");  const txData = {     to: "0x6b175474e89094c44da98b954eedeac495271d0f",     amount: 0,     chainId: 31337,     nonce: 0,     data: "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000",     maxPriorityFeePerGas: ethers.utils.parseUnits('2', 'gwei').toNumber(),     maxFeePerGas: ethers.utils.parseUnits('120', 'gwei').toNumber(),     gasLimit: 40000,     accessList: [],     type: 2, };  const tx = FeeMarketEIP1559Transaction.fromTxData(txData); const signedTx = tx.sign(Buffer.from(process.env.PRIVATE_KEY, 'hex'));  console.log(signedTx.v.toString('hex')); // 1  console.log(signedTx.r.toString('hex')); // 57d733933b12238a2aeb0069b67c6bc58ca8eb6827547274b3bcf4efdad620a  console.log(signedTx.s.toString('hex')); // e49937ec81db89ce70ebec5e51b839c0949234d8aad8f8b55a877bd78cc293`

结果对象将类似于：

`{     "to": "0x6b175474e89094c44da98b954eedeac495271d0f",     "amount": 0,     "chainId": 31337,     "nonce": 0,     "data": "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000",     "maxPriorityFeePerGas": 2000000000,     "maxFeePerGas": 120000000000,     "gasLimit": 40000,     "accessList": [],     "type": 2,     "v": 1,     "r": "57d733933b12238a2aeb0069b67c6bc58ca8eb6827547274b3bcf4efdad620a",     "s": "e49937ec81db89ce70ebec5e51b839c0949234d8aad8f8b55a877bd78cc293", }`

### 序列化

下一步是对已经签名的交易进行 _序列化_ 。也就是要将上面的美观易读的对象编码成原始字节序列，以便它可以发送到以太坊网络并被接收节点使用。

以太坊选择的编码方法称为 [RLP][60]。交易的编码方式如下：

`0x02 || rlp([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas, gasLimit, to, value, data, accessList, v, r, s])`

其中，第一个字节是交易类型。

在前面的代码片段基础上，你实际上可以看到对序列化后的交易如下：

`console.log(signedTx.serialize().toString('hex')); // 02f8b1827a69808477359400851bf08eb000829c40946b175474e89094c44da98b954eedeac495271d0f80b844a9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000c001a0057d733933b12238a2aeb0069b67c6bc58ca8eb6827547274b3bcf4efdad620a9fe49937ec81db89ce70ebec5e51b839c0949234d8aad8f8b55a877bd78cc293`

这就是发送 1 DAI 给我本地的以太坊主网副本上的 Vitalik 的实际有效载荷。

### 提交交易

交易被构建好、签名完毕并序列化之后，必须被发送到一个以太坊节点。

节点可能公开了一个便于接收这类请求的 JSON-RPC 端点。这个请求是 `eth_sendRawTransaction`。这是一个钱包在提交交易时的网络流量：

![使用 eth_sendRawTransaction 方法发送原始交易的 Wireshark 流量](https://notonlyowner.com/images/send-raw-transaction.png)

请求-响应的总结如下：

`POST / HTTP/1.1 Content-Type: application/json Content-Length: 446  {     "id":4264244517200,     "jsonrpc":"2.0",     "method":"eth_sendRawTransaction",     "params":["0x02f8b1827a69808477359400851bf08eb000829c40946b175474e89094c44da98b954eedeac495271d0f80b844a9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000c001a0057d733933b12238a2aeb0069b67c6bc58ca8eb6827547274b3bcf4efdad620a9fe49937ec81db89ce70ebec5e51b839c0949234d8aad8f8b55a877bd78cc293"] } --- HTTP/1.1 200 OK Content-Type: application/json Content-Length: 114  {     "jsonrpc":"2.0",     "id":4264244517200,     "result":"0xbf77c4a9590389b0189494aeb2b2d68dc5926a5e20430fb5bc3c610b59db3fb5" }`

响应结果包含交易的哈希：`bf77c4a9590389b0189494aeb2b2d68dc5926a5e20430fb5bc3c610b59db3fb5`。这个由32个十六进制字符组成的序列是提交的交易的唯一标识符。

## 接收交易

我们应该如何去弄清楚以太坊节点在接收序列化的已签名交易时会发生什么？

有些人可能会在 Twitter 上询问，其他人可能会阅读一些 Medium 文章。还有人可能会阅读文档。[真可惜！][61]

只有一个地方可以找到真相：就在源头。让我们使用 [go-ethereum v1.10.18][62]（即 Geth），这是一个热门的以太坊节点实现（以太坊转向PoS后称为“执行客户端”）。从现在开始，我会在合适位置插入指向 Geth 源代码的链接，供你跟进。

在[端点][63] 收到对`eth_sendRawTransaction` 的JSON-RPC 调用后，节点需要弄清楚请求正文中包含的序列化交易。因此 [它从反序列化交易开始][64]。从现在开始，节点将更容易访问到交易的字段。

此时，节点已经开始验证交易。[首先][65]，确保交易的手续费（即价格 * gas限制）不超过节点愿意接受的最大值（显然，[默认情况下这是 1 以太币][66]）。[接着][67]，确保交易具有重放保护（遵循 [EIP 155][68] - 记得我们在交易中设置的 `chainID` 字段？），或者节点愿意接受未受保护的交易。

接下来的步骤包括 [发送][69] [交][70] [易][71] [到][72] [交易池][73]（即内存池）。简单来说，这个池代表了节点在特定时刻所知道的交易集合。就节点所知，这些交易都还没上链。

在 _真正_ 将交易加入到交易池之前，节点会[检查][74]自己是否已经知道这个交易，以及这个交易的 ECDSA 签名是否 [有效][75]。否则将丢弃交易。

然后开始[处理繁重的内存池工作][76]。正如你可能注意到的，有很多意义非凡的逻辑以确保交易池保持快乐和健康。

这里进行了相当多的重要 [验证][77]。例如gas限制 [低于区块gas限制][78]，或交易的大小 [不超过][79] [允许的最大值][80]，或 nonce 是 [预期的][81]，或发送者有 [足够的资金][82] 来支付潜在成本（即价值 + gas限制 * 价格），等等。

虽然我们可以继续，但我们不在这里成为内存池专家。即使我们想成为，我们也需要考虑到，只要他们遵循网络共识规则，每个节点操作者可能会采取不同的内存池管理方法，他们可能会执行特殊验证或遵循自定义交易优先级规则。在本案例中，我们只是要发送 1 DAI，我们可以将内存池视为一组急切等待被挑选并包含在区块中的交易。

成功将交易入池 (并进行内部 [日志][83] 记录)之后,，节点会 [返回交易哈希值][84]. 这正是我们之前在JSON-RPC的 请求——响应 中看到的返回结果 😎

### 检查内存池

如果你通过 Metamask 或任何默认情况下会连接到传统节点的钱包发送交易，交易最终会进入公共节点的内存池。你可以自己检查内存池来确认这一点。

有些节点会公开一个方便接入的端点，叫做 `eth_newPendingTransactionFilter`。可能是[抢先交易][85][机器人][86]的老朋友了。定期查询这个端点应该可以让我们观察到，在交易被打包进区块之前，要发送 1 DAI 的交易进入了本地测试节点的内存池。

在 Javascript 代码中，可以这样实现:

```js
const hre = require("hardhat");
hre.ethers.provider.on('pending', async function (tx)
{
    // 对交易做一些处理
});
```

要查看实际的 `eth_newPendingTransactionFilter` 调用，我们只需检查网络流量:

![查看 pending 交易的 JSON-RPC 调用的 Wireshark 流量]()

从现在开始，这个脚本会(自动地)查询内存池的变化。以下是随后许多个定期查询更改的请求中的首次请求：

![请求内存池变化的 JSON-RPC 调用的 Wireshark 流量]()

在接收到交易后，节点最终会以交易哈希作出回应：

![使用查询到的交易哈希作为应答的 JSON-RPC 调用的 Wireshark 流量]()

总结后请求——响应如下:

```http
POST / HTTP/1.1
Content-Type: application/json
content-length: 74
{
    "jsonrpc":"2.0",
    "method":"eth_getFilterChanges",
    "params":["0x1"],
    "id":58
}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 105

{
    "jsonrpc":"2.0",
    "id":58,
    "result":["0xbf77c4a9590389b0189494aeb2b2d68dc5926a5e20430fb5bc3c610b59db3fb5"]
}
```

之前我提到了“传统节点”，但没有多加解释。我的意思是，还有[更专业的节点][87]，它们有私有内存池的功能。它们允许用户在交易被打包进区块之前"隐藏"交易。

无论具体细节如何，这种机制通常包括在交易发送者和打包区块的人之间建立私有通道。[Flashbots Protect 服务][88]就是一个著名的例子。实际结果就是，即使你用上面展示的方法监控内存池，也无法获取通过私有通道发送给区块打包者的交易。

我们在这里假设发送 1 DAI 的交易是通过普通渠道提交到网络的，没有利用这类服务。

## 传播

要让交易被打包进区块，它需要以某种方式到达能够构建它并提交它的节点哪里。在[工作量证明PoW][89]以太坊中，这些节点叫矿工。在[权益证明PoS][90]以太坊中，这些节点叫验证者。不过现实往往更复杂一些。要知道[区块的构建可能会被外包][91]给专门的服务。

作为普通用户，你不需要知道这些区块的生产者是谁，也不需要知道他们在哪里。相反，你可以简单地把一笔有效交易发送给网络中的任意一个普通节点，让它把这个交易加入交易池，然后让P2P协议完成剩下的工作。

有[一系列P2P协议][92]让以太坊的节点之间互相连接。除此之外，它们还可以频繁地[交换交易][93]。

从一开始，所有节点就在[监听并向它们的对等节点广播][94]交易(默认最多[50 个对等节点][95])。

一旦一笔交易到达内存池，它就会[被发送给所有还不知道这笔交易的已连接对等节点][96]。

为了提高效率，只有一个已连接节点的随机子集([平方根数量级][97] 🤓)会[收到完整的交易][98]。其余的节点[只会收到交易哈希][99]。这些节点可以在需要时请求完整的交易信息。

一笔交易不能永远停留在节点的内存池中。如果它没有因为其他原因先被丢弃(例如，交易池满了、交易的 gas 定价太低，或者它被一个 nonce/price 更高的新交易替换)，它可能会在一段时间后自动[被移除][100](默认值为[3 小时][101])。

内存池中被区块生产者认为已经准备好被提取和处理的有效交易会被[跟踪][102]在一个[待处理交易][103]列表中。区块生产者可以[查询][104]这个数据结构，获取允许被打包上链的可处理交易。

## 准备工作和交易打包

交易在遍历内存池后应该会到达一个挖矿节点(至少在撰写本文时是这样)。这类节点在多任务处理方面表现得尤其出色。对于熟悉 Golang 的人来说，这意味着在挖矿相关的逻辑中会有相当多的 go routine 和 channel。对于不熟悉 Golang 的人来说，这意味着矿工的常规操作无法像我希望的那样简单的解释。

这一节有两个目标。首先，理解我们的交易如何以及何时从内存池中被矿工提取。其次，找出交易的执行从哪个点开始。

当节点的挖矿组件被初始化时，至少会发生两件相关的事情。一是它[开始监听][105]新交易进入内存池的情况；二是触发[一些基本循环][106]。

用 Geth 的“行话”来说，构建一个包含交易的区块并封装它的行为叫做"提交工作"。因此我们想理解在什么情况下会触发提交。

让我们关注一下["new work"循环][107]。那是一个独立的协程，当节点收到[不同类型的通知][108]时，它会触发工作的提交。触发实质上意味着[将一些工作处理需求发送][109]到节点另一个活跃的监听器上(在矿工的["main"循环][110]中运行)。一旦受到这样的处理需求，[工作的提交就开始了][112]。

节点从一些[初始准备][113]开始。主要包括构建区块头。这包括一些任务，比如[找到父区块][114]，确保[正在构建的区块的时间戳][115]正确，[设置块高][116]，[gas 限制][117]，[coinbase 地址][118]和[基础费用][119]。

之后，共识引擎被调用来为区块头做["共识准备"][120]。这会[计算正确的区块难度][121]([取决于][122]网络的当前版本)。如果你听说过以太坊的"难度炸弹"，就在那里。

接下来[创建][123]区块打包所需的上下文。除此之外的其他操作，这包括[获取最新已知的状态][124]。这是正在构建的区块中的第一笔交易将在其上执行的状态。那可能就是我们发送 1 DAI 的交易。

在准备好区块后，现在用交易来[填充它][125]。

我们终于到了这样一个点，我们的待处理交易，迄今为止只是舒适地停留在节点的内存池中，现在被与其他交易一起[取出来处理][126]。

默认情况下，区块内的[交易是按 gas 价格 和 nonce 排序的][127]。在本例中，交易在区块内的位置实际上无关紧要。

现在这些交易的[顺序执行][128]开始了。每一笔交易都在前一笔的结果状态之上[被执行][129]。

## Execution

An Ethereum transaction can be thought of as a state transition.

State 0: you have 100 DAI, and Vitalik has 100 as well.

Transaction: you send 1 DAI to Vitalik.

State 1: you have 99 DAI, and Vitalik has 101.

Hence, executing a transaction entails _applying_ a sequence of operations to the current state of the blockchain. Producing a new (different) state as a result. This will be considered the new current state until a another transaction comes in.

In reality this is _far_ more interesting (and complex). Let's see.

### Preparation (part 1)

In Geth's jargon, miners [_commit_ transactions][130] in the block. The act of committing a transaction is done in an [_environment_][131]. Such environment contains, among other things, a given [_state_][132].

So in short, committing a transaction is essentially: [(1)][133] remembering the current state, [(2)][134] modifying it by applying the transaction, [(3)][135] depending on the transaction's success, either accepting the new state or rolling back to the original one.

The juicy stuff happens in (2): [applying the transaction][136].

First thing to notice is that [the transaction is turned into a "message"][137]. If you're familiar with Solidity, where you are usually writing things like `msg.data` or `msg.sender`, finally reading "message" in Geth's code is THE sign on the road welcoming you into friendly lands.

Inspecting [what a message looks like][138] quickly leads to notice at least one difference with the transaction. A message has [a `from` field][139]! This field is the signer's Ethereum address, which is [derived][140] from [the public signature][141] included in the transaction (remember the weird `v`, `r` and `s` fields?).

Now the environment for execution [is further prepared][142]. First, the [block-related context is created][143], which includes stuff like block number, timestamp, the coinbase address and the block gas limit. And then...

[the beast walks in][144].

The Ethereum Virtual Machine (EVM), the [stack-based][145] [256-bit][146] computing engine in charge of executing the transaction, shows up all chill like everything is cool man, cool, and [starts putting some clothes on][147]. Yeap, it was naked. It's the EVM, what were you expecting?

The EVM is a machine. And as a machine, it has a set of instructions (a.k.a. opcodes) it can execute. The instruction set has changed over the years. So _there has to be_ some piece of code telling the EVM which opcodes it should use today. And behold, there is. When the EVM [instances its interpreter][148], it [chooses the correct set of opcodes][149], depending on the version being used.

Lastly, [two final steps][150] prior to [real execution][151]. The EVM's transaction context is created (ever used `tx.origin` or `tx.gasPrice` in your Solidity smart contracts?), and the EVM is given access to the current state.

### Preparation (part 2)

It's turn for the EVM to perform the [state transition][152]. Given a message, an environment and the original state, it will use a limited set of instructions to move to a new state. One in which Vitalik has 1 additional DAI 💰.

Before applying the state transition, the EVM must make sure that it abides to [specific consensus rules][153]. Let's see how that's done in detail.

Validation begins in what Geth calls [the "pre-check"][154]. It consists of:

1.  Validating the message's nonce. It [must match][155] the nonce of the message's `from` address. Also, it [must not be the maximum possible nonce][156] (by checking whether incrementing the nonce by one causes an overflow).
2.  Making sure that the account corresponding to the message's `from` address [does not have code][157]. That is, that the transaction origin is an externally-owned account (EOA). Thus abiding by the [EIP 3607][158] spec.
3.  Validating that the `maxFeePerGas` (the `gasFeeCap` in Geth) and `maxPriorityFeePerGas` (the `gasTipCap` in Geth) fields set in the transaction are [within expected bounds][159]. Moreover, that the priority fee [is not greater][160] than the max fee. And that the `maxFeePerGas` [is greater][161] than the current block's base fee.
4.  [Buying gas][162]. In turn checking that the account [can pay][163] for all the gas it intends to consume. And that there's [enough gas left][164] in the block to process the transaction. Finally making the account [pay in advance][165] for the gas (don't worry, there're refund mechanisms later).

Next, the EVM [accounts for the "intrinsic gas"][166] that the transaction consumes. There are a few factors to consider when calculating intrinsic gas. For starters, [whether the transaction is a contract creation][167]. Ours is not, so the gas starts at [21000 units][168]. Afterwards, the [amount of non-zero bytes][169] in the message's `data` field is also taken into consideration. [16 units][170] are charged per non-zero byte (following [this specification][171]). Only [4 units][172] are charged for each zero byte. Finally, some more gas would be accounted in advance [if we provided access lists][173].

We set the `value` field of the transaction to zero. Had we specified a positive value, now would be the moment for the EVM to check [whether the sender account actually has enough balance][174] to execute the ETH transfer. Furthermore, had we set access lists, [now they would be initialized in state][175].

The transaction being executed is not creating a contract. The EVM knows it [because the `to` field is not zero][176]. Therefore, it will already [increment][177] the sender's account nonce by one, and [execute a call][178].

The call will go from the `from` to the `to` message's addresses, passing along the `data`, no value, and whatever remaining gas is left after consuming the intrinsic gas.

### The call

(not [this call][179])

The DAI smart contract is stored at address `0x6b175474e89094c44da98b954eedeac495271d0f`. That's the address we set in the `to` field of the transaction. This initial call is meant for the EVM to execute whatever code is stored at it. Opcode by opcode.

Opcodes are EVM instructions represented with hex numbers ranging from 00 to FF. Though they're usually referred to with their names. For example, `00` is `STOP` and `FF` is `SELFDESTRUCT`. A handy list of opcodes is available at [evm.codes][180].

So what are DAI's opcodes anyway ? Glad you asked:

![EVM opcodes of the DAI smart contract](https://notonlyowner.com/images/DAI-opcodes.png)

Don't panic. It's still early to make sense out of all of that.

Let's start slowly, tearing [the initial call][181] apart. Its [brief docs][182] provide a good summary:

`// Call executes the contract associated with the addr with the given input as // parameters. It also handles any necessary value transfer required and takes // the necessary steps to create accounts and reverses the state in case of an // execution error or failed value transfer. func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {     ... }`

To begin with, the logic [checks][183] that the call depth limit hasn't been reached. This limit is [set to 1024][184], which means there can only be a maximum of 1024 nested calls in a single transaction. [Here][185] is an interesting article to read about some of the reasoning and subtleties behind this behavior of the EVM. Later we'll explore how the call depth is increased/decreased.

Relevant side note: the call depth limit is _not_ the EVM's stack size limit - which (coincidentally?) [is 1024 elements][186] as well.

The next step is to [make sure][187] that if a positive value was specified in the call, the sender has enough balance to execute the transfer ([performed][188] a few steps later). We can ignore this because our call has zero value. Additionally, a [snapshot of the current state][189] is taken. This allows [easily reverting][190] any state changes upon failure.

We know that DAI's address refers to an account that has code. Thus, it [must already exist][191] in Ethereum's state.

However, let's imagine for a moment this was not a transaction to send 1 DAI. Say it was a trivial transaction with no value targetting a new address. The corresponding account would need to be [_added_ to the state][192]. However, what if said account would end up being just empty ? There doesn't seem to be a reason to keep track of it - other than wasting nodes' disk space. [EIP 158][193] introduced some changes to the Ethereum protocol to help avoid such scenarios. That's why you're seeing [this `if` clause][194] when calling any account.

Another thing we know is that DAI [is _not_ a precompile contract][195]. What's a precompiled contract ? Here's what the [Ethereum yellow paper][196] has to offer:

> \[...\] preliminary piece of architecture that may later become native extensions. The contracts in addresses 1 to 9 execute the elliptic curve public key recovery function, the SHA2 256-bit hash scheme, the RIPEMD 160-bit hash scheme, the identity function, arbitrary precision modular exponentiation, elliptic curve addition, elliptic curve scalar multiplication, an elliptic curve pairing check, and the BLAKE2 compression function F respectively.

In short, there're (so far) 9 different special contracts in Ethereum's state. These accounts (ranging from [0x0000000000000000000000000000000000000001][197] to [0x0000000000000000000000000000000000000009][198]) out-of-the-box include the necessary code to execute the operations mentioned in the yellow paper. Of course, you can [check this by yourself in Geth's code][199].

To add some color to the story of precompiled contracts, note that in Ethereum mainnet all these accounts have _at least_ 1 wei in balance. This was done [intentionally][200] (at least before users started sending Ether by mistake). Look, [here's an almost 5-year-old transaction][201] sending 1 wei to the `0x0000000000000000000000000000000000000009` precompile.

Anyway. Having realized the call's target address does not correspond to a precompiled contract, the node [reads the account's code][202] from the state. Then [ensures it's non-empty][203]. At last, [orders the EVM][204] to use its interpreter to run the code with the given input (the contents of the transaction's `data` field).

### The interpreter (part 1)

It's time for the EVM to actually execute DAI's code. To accomplish this, the EVM has a couple of elements at hand. It has [a stack][205] that can hold [up to 1024 elements][206] (though only the first 16 are directly accessible with the available opcodes). It has a volatile [read/write memory space][207]. It has a [program counter][208]. It has a special read-only memory space called calldata [where the call's input data is kept][209]. Among other stuff.

As usual, there's some necessary setup and validations before jumping into the juicy stuff. First, the [call depth is incremented][210] by one. Second, the [read-only mode is set][211] if necessary. Ours is not a read-only call (see the `false` argument passed [here][212]). Otherwise some EVM operations wouldn't be allowed. These include state-changing EVM instructions [`SSTORE`][213], [`CREATE`][214], [`CREATE2`][215], [`SELFDESTRUCT`][216], [`CALL`][217] with positive value, and [`LOG`][218].

The interpreter now enters [the execution loop][219]. It consists of sequentially [executing][220] the opcodes in DAI's code as [indicated by the program counter][221] and the current EVM instruction set. For the time being we're using the [London instruction set][222] - which was [configured in the jump table][223] when the interpreter was first instantiated.

The loop also takes care of [keeping a healthy stack][224] (avoiding under/overflows). And spending each operation's [fixed gas costs][225], as well as [dynamic gas costs][226] when appropriate. These dynamic costs [include][227], for example, the expansion of EVM memory (read more about memory expansion costs are calculated [here][228]). Note that gas is consumed _before_ execution of an opcode - not after.

The actual behavior of each possible instruction can be found implemented in [this Geth file][229]. By just skimming through it one can begin to see how these instructions work with the stack, the memory, the calldata and the state.

At this point we'd need to jump straight into DAI's opcodes and follow their execution for our transaction. Yet I don't think that's the best way to approach this. I'd rather first walk away from the EVM and Geth, and move into Solidity lands. This should give us a more valuable overview of the high-level behavior of an ERC20 transfer operation.

### Solidity execution

The DAI smart contract was coded in [Solidity][230]. It is an object-oriented, high-level language that when compiled, outputs EVM bytecode able to deploy smart contracts on an EVM-compatible chain (Ethereum in our case).

DAI's source code can be found [verified in block explorers][231], or [in GitHub][232]. For ease of reference, I'll be pointing to the first.

Before we begin, let's always keep in mind that the EVM knows nothing about Solidity. It knows nothing about its variables, functions, the layout of contracts, ABI-encoding, etc. The Ethereum blockchain stores plain hard EVM bytecode, not fancy Solidity code.

You might wonder then how come when you go to any block explorer, they show you Solidity code at Ethereum addresses. Well, it's just a façade. In most block explorers people can upload Solidity source code, and the explorer takes care of compiling the source with specific compiler settings. If the compiler's output produced by the explorer matches what's stored at the specified address on the blockchain, then the contract's source code is said to be "verified". From then on, anyone that navigates to that address sees the Solidity code of that address, instead of only the EVM bytecode stored at it.

A non-trivial consequence of the above is that to some extent we're trusting block explorers to show us the legitimate code (which might not necessarily be true, even if [accidentally][233]). There are might be [alternatives][234] to this though - unless every time you want to read a contract you verify source code against your own node.

Anyway, back to DAI's Solidity code now.

On the DAI smart contract (compiled with Solidity [v0.5.12][235]), let's focus on [the function][236] to execute: `transfer`.

`function transfer(address dst, uint wad) external returns (bool) {     return transferFrom(msg.sender, dst, wad); }`

When `transfer` is run, it will [call another function][237] named `transferFrom`, then returning whatever boolean flag the latter returns. The first and second argument of `transfer` (here named `dst` and `wad`) are passed directly to `transferFrom`. This function additionally reads the sender's address (available as a [Solidity global variable][238] in `msg.sender`).

For our case, these would be the values passed to `transferFrom`:

`return transferFrom(     msg.sender, // 0x6fC27A75d76d8563840691DDE7a947d7f3F179ba (my address on the local testing node)     dst,        // 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 (Vitalik's address)     wad         // 1000000000000000000 (1 DAI in wei units) );`

Let's see the [`transferFrom` function][239] then.

`function transferFrom(address src, address dst, uint wad) public returns (bool) {    ... }`

First, the sender's [balance is checked][240] against the amount being transferred.

`require(balanceOf[src] >= wad, "Dai/insufficient-balance");`

It's simple: you cannot transfer more DAI than what you have in balance. If I didn't have 1 DAI, execution would halt at this point, returning an error with a message. Note that each address' balance is tracked on the smart contract storage. In a [map-like data structure named `balanceOf`][241]. If you have at least 1 DAI, I can assure you your account's address has a record somewhere in there.

Second, token [allowances are validated][242].

`// don't bother too much about this :) if (src != msg.sender && allowance[src][msg.sender] != uint(-1)) {     require(allowance[src][msg.sender] >= wad, "Dai/insufficient-allowance");     allowance[src][msg.sender] = sub(allowance[src][msg.sender], wad); }`

This does not concern us right now. Because we're not executing the transfer on behalf of another account. Though do note that's a mechanism all ERC20 tokens should implement - DAI not being the exception. In essence, you can approve other accounts to transfer DAI tokens from your account.

Third, the actual [balance swap][243] happens.

`balanceOf[src] = sub(balanceOf[src], wad); balanceOf[dst] = add(balanceOf[dst], wad);`

When sending 1 DAI, the sender's balance is decreased by 1000000000000000000, and the receiver's balance is incremented by 1000000000000000000. These operations are done reading and writing on the `balanceOf` data structure. It's worth noting the use of two special functions `add` and `sub` to do the math.

Why not simply use the `+` and `-` operators ?

Remember: this contract was compiled with Solidity 0.5.12. At that point in time the compiler did not include over/underflow checks as it does today. Thus developers had to remember (or be reminded 😛) to implement them by themselves where appropriate. Thus the use of `add` and `sub` in the DAI contract. They are just custom internal functions to perform addition and subtraction with bound checks to avoid arithmetic issues.

`function add(uint x, uint y) internal pure returns (uint z) {     require((z = x + y) >= x); }  function sub(uint x, uint y) internal pure returns (uint z) {     require((z = x - y) <= x); }`

The `add` function sums `x` and `y`, halting execution if the result of the operation is lower than `x` (thus preventing integer overflow).

The `sub` function subtracts `y` from `x`, halting execution if the result of the operation is greater than `x` (thus preventing integer underflow).

Fourth, a `Transfer` event is [emitted][244] (as [suggested by the ERC20 spec][245]).

`emit Transfer(src, dst, wad);`

An event is a logging operation. Data emitted in an event can later be retrieved from off-chain services reading the blockchain, though never by other contracts.

In our transfer operation the emitted event appears to log three elements. The sender's address (`0x6fC27A75d76d8563840691DDE7a947d7f3F179ba`), the receiver's address (`0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`), and the amount sent (`1000000000000000000`).

The first two correspond to the parameters labeled as `indexed` in the event's declaration. Indexed parameters facilitate data retrieval, allowing filtering for any of the corresponding logged values. Unless the event is labeled as `anonymous`, the event identifier is also included as a topic.

Hence, being more specific, the `Transfer` event we're dealing with actually logs 3 topics (the event's identifier, the sender's address and the receiver's address) and 1 value (the amount of DAI transferred). We'll cover more details about this event once we get to lower-level EVM stuff.

At the end of the function, the boolean value `true` is [returned][246] (as [suggested by the ERC20 spec][247]).

`return true;`

That's a way of signaling that the transfer was successfully executed. This boolean flag is passed up to the outer `transfer` function that initiated the call (which simply returns it as well).

And that's it! If you've ever sent DAI, be certain that's the logic you've executed. That's the job you've paid to be done for you by a global decentralized network of nodes.

Hold on. I may have gone too far. That's kind of a lie. Because as I told you earlier, the EVM knows nothing about Solidity. Nodes execute no Solidity. They execute EVM bytecode.

It's time for the real deal.

### EVM execution

I'm turning quite technical in this section. I'll assume you're somewhat familiar with looking at some EVM bytecode. If you're not, I _highly_ recommend reading [this series][248] or [this newer one][249]. There you will find lots of the concepts in this section explained individually and in more depth.

DAI's raw bytecode is tough to look at - we already witnessed it in a previous section. A prettier way to study it is using a disassembled version. You can find Dai's disassembled bytecode [here][250] (I've extracted it to [this gist][251] for ease of reference).

#### Free memory pointer and call's value

The first three instructions shouldn't come as a surprise if you're already familiar with the Solidity compiler. It's simply initializing the free memory pointer.

`0x0: PUSH1     0x80 0x2: PUSH1     0x40 0x4: MSTORE`    

The Solidity compiler reserves memory slots from `0x00` to `0x80` for internal stuff. So the "free memory pointer" is a pointer to the first slot of EVM memory that can be freely used. It is stored at `0x40`, and at initialization points to `0x80`.

Keep in mind that all EVM opcodes have a counterpart implementation in Geth. For example, you can [really see][252] how the implementation of `MSTORE` pops two stack elements and writes to the EVM memory a word of 32 bytes:

`func opMstore(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {     // pop value of the stack     mStart, val := scope.Stack.pop(), scope.Stack.pop()     scope.Memory.Set32(mStart.Uint64(), &val)     return nil, nil }`

The next EVM instructions in DAI's bytecode ensure that the call doesn't hold any value. If it had, execution would halt at the `REVERT` instruction. Note the use of the `CALLVALUE` instruction (implemented [here][253]) to read the current call's value.

`0x5: CALLVALUE  0x6: DUP1       0x7: ISZERO     0x8: PUSH2     0x10 0xb: JUMPI      0xc: PUSH1     0x0 0xe: DUP1       0xf: REVERT`

Our call doesn't hold any value (the `value` field of the transaction was set to zero) - so we're good to continue.

#### Validating calldata (part 1)

Next: another check introduced by the compiler. This time it's figuring out whether the calldata's size (obtained with the `CALLDATASIZE` instruction - implemented [here][254]) is lower than 4 bytes (see the `0x4` and the `LT` instruction below ?). In such case it would jump to position `0x142`. Halting execution at the `REVERT` instruction in position `0x146`.

`0x10: JUMPDEST 0x11: POP        0x12: PUSH1     0x4 0x14: CALLDATASIZE 0x15: LT         0x16: PUSH2     0x142 0x19: JUMPI  ...  0x142: JUMPDEST   0x143: PUSH1     0x0 0x145: DUP1       0x146: REVERT` 

That means that in the DAI smart contract calldata's size is enforced to be _at least_ 4 bytes. That's because the ABI-encoding mechanism used by Solidity identifies functions with the first four bytes of the keccak256 hash of their signature (usually called "function selector" - [see the spec][255]).

If calldata didn't have at least 4 bytes, it wouldn't be possible to identify the function. So the compiler introduces the necessary EVM instructions to fail early in that scenario. That's what you witnessed above.

In order to call the `transfer(address,uint256)` function, the first four bytes of the calldata must match the function's selector. These are:

`$ cast sig "transfer(address,uint256)" 0xa9059cbb`

That's right. Exactly the same first 4 bytes of the `data` field of the transaction we built earlier:

`0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000`

Now that the length of the calldata has been validated, it's time to use it. See below how the first four bytes calldata are placed at the top of the stack (main EVM instruction to notice here is `CALLDATALOAD`, implemented [here][256]).

`0x1a: PUSH1     0x0 0x1c: CALLDATALOAD 0x1d: PUSH1     0xe0 0x1f: SHR`       

In reality `CALLDATALOAD` pushes [32 bytes of calldata][257] to the stack. It needs to be chopped with [the `SHR` instruction][258] to keep the first 4 bytes.

#### Function dispatcher

Don't try to understand the following line by line. Instead, pay attention to the high-level pattern that stands out. I'll add some dividing lines to make it clearer:

`0x20: DUP1 0x21: PUSH4     0x7ecebe00 0x26: GT         0x27: PUSH2     0xb8 0x2a: JUMPI`

`0x2b: DUP1       0x2c: PUSH4     0xa9059cbb 0x31: GT         0x32: PUSH2     0x7c 0x35: JUMPI`

`0x36: DUP1       0x37: PUSH4     0xa9059cbb 0x3c: EQ         0x3d: PUSH2     0x6b4 0x40: JUMPI`

`0x41: DUP1       0x42: PUSH4     0xb753a98c 0x47: EQ         0x48: PUSH2     0x71a 0x4b: JUMPI`

It's no coincidence that some of the hex values being pushed to the stack are 4 bytes long. Those are, indeed, function selectors.

The set of instructions above is a common structure of the bytecode that the Solidity compiler produces. It's usually referred to as "function dispatcher". It resembles an if-else or switch flow. It's simply trying to match the first four bytes of the calldata against the set of known selectors of the contract's functions. Once it finds a coincidence, execution will jump to another section of the bytecode. Where the instructions for that particular function are placed.

Following the above logic, the EVM matches the first four bytes of calldata against the selector of the ERC20 `transfer` function: `0xa9059cbb`. And jumps to bytecode position `0x6b4`. That's how the EVM is told to start executing the transfer of DAI.

#### Validating calldata (part 2)

Having matched the selector and jumped, now the EVM is about to start running specific code related to the function. But before jumping into its details, it needs to somehow remember where to continue executing once all function-related logic has been executed.

The way to do that is simply keeping the appropriate bytecode position in the stack. See the `0x700` value being pushed below. It will linger in stack until at some point (later down the road) it will be retrieved and be used to jump back to wrap up execution.

`0x6b4: JUMPDEST   0x6b5: PUSH2     0x700`

Now let's get more specific to the `transfer` function.

The compiler embeds some logic to ensure the calldata's size is correct for a function with two parameters of `address` and `uint256` type. For the `transfer` function, that is at least 68 bytes (4 bytes for the selector + 64 bytes for the two ABI-encoded parameters).

`0x6b8: PUSH1     0x4 0x6ba: DUP1       0x6bb: CALLDATASIZE 0x6bc: SUB        0x6bd: PUSH1     0x40 0x6bf: DUP2       0x6c0: LT         0x6c1: ISZERO     0x6c2: PUSH2     0x6ca 0x6c5: JUMPI      0x6c6: PUSH1     0x0 0x6c8: DUP1       0x6c9: REVERT`

If the calldata's size was lower, execution would halt at the `REVERT` in position `0x6c9`. Since our transaction's calldata has been correctly ABI-encoded and therefore has the appropriate length, execution jumps to position `0x6ca`.

#### Reading parameters

Next step is for the EVM to read the two parameters provided in the calldata. Those would be the 20-bytes-long address `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` and the number `1000000000000000000` (`0x0de0b6b3a7640000` in hex). Both were ABI-encoded in chunks of 32 bytes. Thus there needs to be some basic manipulation to read the right values and place them at the top of the stack.

`0x6ca: JUMPDEST   0x6cb: DUP2       0x6cc: ADD        0x6cd: SWAP1      0x6ce: DUP1       0x6cf: DUP1       0x6d0: CALLDATALOAD 0x6d1: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0x6e6: AND        0x6e7: SWAP1      0x6e8: PUSH1     0x20 0x6ea: ADD        0x6eb: SWAP1      0x6ec: SWAP3      0x6ed: SWAP2      0x6ee: SWAP1      0x6ef: DUP1       0x6f0: CALLDATALOAD 0x6f1: SWAP1      0x6f2: PUSH1     0x20 0x6f4: ADD        0x6f5: SWAP1      0x6f6: SWAP3      0x6f7: SWAP2      0x6f8: SWAP1      0x6f9: POP        0x6fa: POP        0x6fb: POP        0x6fc: PUSH2     0x1df4 0x6ff: JUMP`

Just to make it more visual, after sequentially applying the above set of instructions (up to `0x6fb`), the top of stack looks like this:

`0x0000000000000000000000000000000000000000000000000de0b6b3a7640000 0x000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa96045`

And that's how the EVM swiftly extracts both arguments from calldata, placing them on the stack for future use.

The final two instructions above (bytecode positions `0x6fc` and `0x6ff`) simply make execution jump to position `0x1df4`. Let's continue there.

#### The `transfer` function

During the brief Solidity analysis, we saw that the `transfer(address,uint256)` function was a thin wrapper that called the more complex `transferFrom(address,address,uint256)` function. The compiler translates such internal call to [these EVM instructions][259]:

`0x1df4: JUMPDEST   0x1df5: PUSH1     0x0 0x1df7: PUSH2     0x1e01 0x1dfa: CALLER     0x1dfb: DUP5       0x1dfc: DUP5       0x1dfd: PUSH2     0xa25 0x1e00: JUMP`

First notice the instruction pushing the value `0x1e01`. That's how the EVM is instructed to "remember" the exact position where it should jump back to continue execution after the upcoming internal call.

Then, pay attention to the use of `CALLER` (because in Solidity the internal call uses `msg.sender`). As well as to the two `DUP5` instructions. Together, these are putting at the top of the stack the three necessary arguments for `transferFrom`: the caller's address, the receiver's address, and the amount to be transferred. The last two were already somewhere in the stack, thus the use of `DUP5`. The top of the stack now holds all necessary arguments:

`0x0000000000000000000000000000000000000000000000000de0b6b3a7640000 0x000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa96045 0x0000000000000000000000006fc27a75d76d8563840691dde7a947d7f3f179ba`

Finally, following instructions `0x1dfd` and `0x1e00`, execution jumps to position `0xa25`. Where the EVM will start executing the instructions corresponding to the `transferFrom` function.

#### The `transferFrom` function

First thing that needs to be checked is whether the sender has enough DAI in balance - otherwise reverting. The sender's balance is kept in the contract storage. The fundamental EVM instruction needed is then `SLOAD`. However, `SLOAD` needs to know what storage _slot_ needs to be read. For mappings (the type of Solidity data structure that is holding account balances in the DAI smart contract), that's not so straightforward to tell.

I won't dive here into the internal layout of Solidity state variables in contract storage. You may read about it [here for v0.5.15][260]. Suffice to say that given the key address `k` for the mapping `balanceOf`, its corresponding `uint256` value will be kept at storage slot `keccak256(k . p)`, where `p` is the slot position of the mapping itself and `.` is concatenation. You can do the math yourself.

For simplicity, let's just highlight a couple of operations that need to happen. The EVM must i) calculate the storage slot for the mapping, ii) read the value, iii) compare it against the amount to be transferred (a value already in stack). Therefore we should see instructions like `SHA3` for the hashing, `SLOAD` for reading storage, and `LT` for the comparison.

`0xa25: JUMPDEST   0xa26: PUSH1     0x0 0xa28: DUP2       0xa29: PUSH1     0x2 0xa2b: PUSH1     0x0 0xa2d: DUP7       0xa2e: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xa43: AND        0xa44: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xa59: AND        0xa5a: DUP2       0xa5b: MSTORE     0xa5c: PUSH1     0x20 0xa5e: ADD        0xa5f: SWAP1      0xa60: DUP2       0xa61: MSTORE     0xa62: PUSH1     0x20 0xa64: ADD        0xa65: PUSH1     0x0 0xa67: SHA3      --> calculating storage slot 0xa68: SLOAD     --> reading storage 0xa69: LT        --> comparing balance against amount 0xa6a: ISZERO     0xa6b: PUSH2     0xadc 0xa6e: JUMPI`    

If the sender didn't have enough DAI, execution would [follow at `0xa6f` and finally hit the `REVERT` at `0xadb`][261]. Since I did not forget to load 1 DAI in my sender account's balance, let's then proceed to position `0xadc`.

The following set of instructions correspond to the EVM validating whether the caller matches the sender's address (remember the `if (src != msg.sender ...) { ... }` code segment in the contract).

`0xadc: JUMPDEST   0xadd: CALLER     0xade: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xaf3: AND        0xaf4: DUP5       0xaf5: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xb0a: AND        0xb0b: EQ         0xb0c: ISZERO     0xb0d: DUP1       0xb0e: ISZERO     0xb0f: PUSH2     0xbb4 0xb12: JUMPI ... 0xbb4: JUMPDEST   0xbb5: ISZERO     0xbb6: PUSH2     0xdb2 0xbb9: JUMPI`

Since they don't match, continue executing at position `0xdb2`.

Doesn't this code below remind you of something ? Check out the instructions being used. Again, don't focus on them individually. Use your intuition to spot high-level patterns and the most relevant instructions.

`0xdb2: JUMPDEST   0xdb3: PUSH2     0xdfb 0xdb6: PUSH1     0x2 0xdb8: PUSH1     0x0 0xdba: DUP7       0xdbb: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xdd0: AND        0xdd1: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xde6: AND        0xde7: DUP2       0xde8: MSTORE     0xde9: PUSH1     0x20 0xdeb: ADD        0xdec: SWAP1      0xded: DUP2       0xdee: MSTORE     0xdef: PUSH1     0x20 0xdf1: ADD        0xdf2: PUSH1     0x0 0xdf4: SHA3       0xdf5: SLOAD      0xdf6: DUP4       0xdf7: PUSH2     0x1e77 0xdfa: JUMP`

If it resembles reading a mapping from storage, it's because it is! The above is the EVM reading the sender's balance from the `balanceOf` mapping.

Execution then jumps to position `0x1e77`, where the body of the `sub` function is placed.

The `sub` function subtracts two numbers, reverting upon integer underflow. I'm not including the bytecode, though you can follow it [here][262]. The result of the arithmetic operation is kept on the stack.

Back to the instructions corresponding to `transferFrom` function's body, now the result of the subtraction is to be written to storage - updating the `balanceOf` mapping. Try to notice below the calculation performed to obtain the appropriate storage slot of the mapping entry, which leads to the execution of the `SSTORE` instruction. This instruction is the one that effectively writes data to state - that is, that updates the contract's storage.

`0xdfb: JUMPDEST   0xdfc: PUSH1     0x2 0xdfe: PUSH1     0x0 0xe00: DUP7       0xe01: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xe16: AND        0xe17: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xe2c: AND        0xe2d: DUP2       0xe2e: MSTORE     0xe2f: PUSH1     0x20 0xe31: ADD        0xe32: SWAP1      0xe33: DUP2       0xe34: MSTORE     0xe35: PUSH1     0x20 0xe37: ADD        0xe38: PUSH1     0x0 0xe3a: SHA3       0xe3b: DUP2       0xe3c: SWAP1      0xe3d: SSTORE` 

A pretty similar set of opcodes is run to update the receiver's account balance. First [is read from the `balanceOf` mapping in storage][263]. Then the balance is added to the amount being transferred [using the `add` function][264]. At last the result is [written to the appropriate storage slot][265].

#### Logging

In the contract's code the `Transfer` event was emitted after updating balances. So there has to be a set of instructions in the bytecode under analysis that take care of emitting such event with the appropriate data.

However, events are yet another thing that belong to Solidity's fantasy world. In EVM world, events correspond to logging operations.

Logging is performed with the available set of `LOG` instructions. There are a couple of variants, depending on how many topics are to be logged. In DAI's case, we already noted that the emitted `Transfer` event has 3 topics.

Then it's no surprise to find a set of instructions that lead to running the `LOG3` instruction.

`0xeca: POP        0xecb: DUP3       0xecc: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xee1: AND  0xee2: DUP5 0xee3: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xef8: AND        0xef9: PUSH32    0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 0xf1a: DUP5       0xf1b: PUSH1     0x40 0xf1d: MLOAD      0xf1e: DUP1       0xf1f: DUP3       0xf20: DUP2       0xf21: MSTORE     0xf22: PUSH1     0x20 0xf24: ADD        0xf25: SWAP2      0xf26: POP        0xf27: POP        0xf28: PUSH1     0x40 0xf2a: MLOAD      0xf2b: DUP1       0xf2c: SWAP2      0xf2d: SUB        0xf2e: SWAP1      0xf2f: LOG3`      

There's at least one value that stands out in those instructions: `0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef`. That's the event's main identifier. Also called topic 0. It is a static value calculated by the compiler at compiling time (embedded in the contract's runtime bytecode). As noted previously, no more than the hash of the event's signature:

`$ cast keccak "Transfer(address,address,uint256)" 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef`

Right before reaching the `LOG3` instruction, the stack looks like this:

`0x0000000000000000000000000000000000000000000000000000000000000080 0x0000000000000000000000000000000000000000000000000000000000000020 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef -- topic 0 (event identifier) 0x0000000000000000000000006fc27a75d76d8563840691dde7a947d7f3F179ba -- topic 1 (sender's address) 0x000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa96045 -- topic 2 (receiver's address)`

Where's the amount of the transfer then ? In memory! Before reaching `LOG3`, the EVM was first instructed to store the amount in memory. So that it can later be consumed by the logging instruction. If you look at position `0xf21`, you'll see the `MSTORE` instruction in charge of doing so.

So once `LOG3` is reached, it is safe for the EVM to grab the actual value logged from memory, starting at offset `0x80` and reading `0x20` bytes (first two stack elements above).

Another way of understanding logging is looking at its [implementation in Geth][266]. In there you'll find a single function in charge of handling all logging instructions. You can see how i) an empty array of topics [is initialized][267], ii) the memory offset and data size are [read from the stack][268], iii) the topics are [read from stack][269] and [inserted in the array][270], iv) the value is [read from memory][271], v) the log, containing address where it was emitted, topics and value, [is appended][272].

How those logs are later recovered, we'll find out soon enough.

#### Returning

Last thing for the `transferFrom` function is to return the boolean value `true`. That's why the first instruction after `LOG3` is simply pushing the `0x1` value to the stack.

`0xf30: PUSH1    0x1`

The next instructions prepare the stack to exit the `transferFrom` function, going back to its wrapper `transfer` function. Remember that the position for this next jump had been already stored in the stack - that's why you don't see it in the opcodes below.

`0xf32: SWAP1      0xf33: POP        0xf34: SWAP4      0xf35: SWAP3      0xf36: POP        0xf37: POP        0xf38: POP        0xf39: JUMP`

Back in the `transfer` function, all there's to do is to prepare the stack for the final jump. To a position where execution will be wrapped up. The position for this upcoming jump had also been stored in the stack previously (remember the `0x700` value being pushed?).

`0x1e01: JUMPDEST   0x1e02: SWAP1      0x1e03: POP        0x1e04: SWAP3      0x1e05: SWAP2      0x1e06: POP        0x1e07: POP        0x1e08: JUMP` 

All that's left is to prepare the stack for the final instruction: `RETURN`. This instruction is in charge of reading some data from memory, and passing it back to the original caller.

For the DAI transfer, the returned data would simply include the `true` boolean flag returned by the `transfer` function. Remember that the value had been already placed at the stack.

The EVM begins with grabbing the first available position of free memory. This is done by reading the free memory pointer:

`0x700: JUMPDEST   0x701: PUSH1     0x40 0x703: MLOAD`

Next, the value must be stored in memory with `MSTORE`. Although not so straightforward to tell, the instructions below are just the ones the compiler found most appropriate to prepare the stack for the `MSTORE` operation.

`0x704: DUP1       0x705: DUP3       0x706: ISZERO     0x707: ISZERO     0x708: ISZERO     0x709: ISZERO     0x70a: DUP2       0x70b: MSTORE`

The `RETURN` instruction copies the returned data from memory. So it needs to be told how much memory to read, and where to start. The instructions below simply tell the EVM to read and return `0x20` bytes from memory starting at the free memory pointer.

`0x70c: PUSH1     0x20 0x70e: ADD        0x70f: SWAP2      0x710: POP        0x711: POP        0x712: PUSH1     0x40 0x714: MLOAD      0x715: DUP1       0x716: SWAP2      0x717: SUB        0x718: SWAP1      0x719: RETURN` 

The value `0x0000000000000000000000000000000000000000000000000000000000000001` (corresponding to the boolean `true`) is returned.

Execution halts.

### The interpreter (part 2)

Bytecode execution has finished. The interpreter must stop [iterating][273]. In Geth, that's done like this:

`// interpreter's execution loop for {      ...     // execute the operation     res, err = operation.execute(&pc, in, callContext)     if err != nil {         break     }     ... }`

That means that the implementation of the `RETURN` opcode should somehow return an error. Even for successful executions such as ours. Indeed, [it does][274]. Though it acts as a flag - the error is actually [deleted][275] when it matches the flag returned by the successful execution of the `RETURN` opcode.

### Gas refunds and payments

With the interpreter run finished, we're [back in the call][276] that originally triggered it. The run [was completed successfully][277]. Thus the returned data and any gas remaining is simply [returned][278].

The call's finished as well. Execution follows wrapping up the state transition.

First [providing gas refunds][279]. These are added to any gas leftovers in the transaction. The refunded amount is capped to 1/5 of the gas used (due to [EIP 3529][280]). All gas available now (remaining plus refunded) is [paid back in ETH][281] to the sender's account, at the rate originally set by the sender in the transaction. All gas left is [re-added][282] to the available gas in the block - so that subsequent transactions can consume it.

Then [paying the coinbase address][283] (the miner's address in PoW, the validator's address in PoS) what was originally promised: the tip. Interestingly, payment is done for all gas used during execution. Even if some of it was later refunded. Moreover, note [here][284] how the _effective_ tip is calculated. Not only noticing that it is capped by the `maxPriorityFeePerGas` transaction field. But more importantly, realizing that it _does not_ include the base fee! That's no mistake - Ethereum enjoys [watching the ETH burn][285].

At last [the execution result is wrapped in a prettier structure][286]. Including the used gas, any EVM error that could have aborted execution (none in our case), along with the returned data from the EVM.

### Building the transaction receipt

The structure representing the execution result is now passed [back][287] [up][288]. At this point Geth does [some internal cleanup][289] of the execution state. Once done, it [accumulates the gas used][290] in the transaction (including refunds).

Most importantly, now is the time the transaction receipt is created. The receipt's an object summarizing data related to the transaction's execution. It includes information such as [execution status][291] (success/failure), the [transaction's hash][292], [gas units used][293], [address of created contract][294] (none in our case), [logs emitted][295], the [transaction's bloom filter][296], and [more][297].

We'll retrieve the full contents of our transaction's receipt soon.

If you'd like to dig deeper into the transaction's logs and the role of the bloom filter, check out [noxx's article][298].

## Sealing the block

Execution of subsequent transactions continue happening until the block runs out of space.

That's when the node invokes the consensus engine to [finalize][299] the block. In PoW that entails [accumulating mining rewards][300] (issuing [full rewards][301] in ETH to the coinbase address, along with [partial rewards][302] for [ommer blocks][303]) and [updating the final state root][304] of the block accordingly.

Next, the actual block [is assembled][305], putting [all data in its right place][306]. Including information such as the header's [transaction hash][307], or the [receipts hash][308].

All ready for the real PoW mining now. [A new "task"][309] is created and [pushed to the right listener][310]. The [sealing task][311], delegated to the consensus engine, starts.

I won't explain in detail how the actual mining is done for PoW. Lots about it on the Internet already. Just note that in Geth this involves a [multithreaded try-and-error process][312] to [find a number][313] that [satisfies a necessary condition][314]. Needless to say, once Ethereum switches to Proof of Stake the sealing process will be handled quite differently.

The mined block is [pushed to the appropriate channel][315] and [received at the results loop][316]. Where [receipts and logs are updated accordingly][317] with the latest block data after its been effectively mined.

The block is finally [written to the chain][318], placing it at its head.

## Broadcasting the block

Next step is to [announce to the whole network][319] that a new block has been mined. Meanwhile the block is internally [stored into the set of pending ones][320]. Patiently awaiting for confirmations from other nodes.

The announcement is done [posting a specific event][321], picked up by the [mined broadcast loop][322]. In there the block is fully [propagated to a subset of peers][323] and [made available][324] to the rest in a lighter fashion.

More specifically, propagation entails [sending block data to the square root of connected peers][325]. Internally this is implemented [pushing the data to the queued blocks channel][326], until its [sent][327] via the [p2p layer][328]. The p2p message is identified as `NewBlockMsg`. The rest receives a [lightweight announcement including the block hash][329].

Note that this is only valid for PoW. [Block propagation will happen on consensus engines][330] in Proof of Stake.

## Verifying the block

Peers are continuously [listening for messages][331]. Each type of [possible message has an associated handler][332] that is [invoked][333] as soon as the corresponding message is received.

Consequently, upon getting the `NewBlockMsg` message with the block's data, [its corresponding handler][334] is executed. The handler [decodes the message][335] and runs some [early validations][336] on the propagated block. These include preliminary [sanity checks][337] on the header's data, mostly ensuring they are filled and bounded. As well as validations for the block's [uncle][338] and [transaction][339] hashes.

Then [the peer that sent the message is marked][340] as owning the block. Thus avoiding to later propagate the block back to it.

At last, the packet is [passed down][341] to [a second handler][342], where the block is going to be [enqueued for import][343] into the local copy of the chain. The enqueuing is done by [sending a direct import request to the corresponding channel][344]. When the request is [picked up][345], it triggers the [actual enqueuing operation][346]. Finally [pushing][347] the block data to the queue.

The block is now in the local queue ready to be processed. This queue is [read from periodically][348] in the node's block fetcher main loop. When the block makes it to the front, the node will pick it up and [attempt to import it][349].

There are a at least two validations worth highlighting prior to the actual insertion of the candidate block.

First, the local chain must already [include the parent of the propagated block][350].

Second, the block's header [must be valid][351]. These validations are _the real ones_. Meaning, the ones that actually matter for consensus, and are specified in Ethereum's [yellow paper][352]. Hence, they are [handled by the consensus engine][353].

Just as examples, the engine checks that the [block's proof of work is valid][354], or that the block's timestamp is [not in the past][355] nor [not too far ahead into the future][356], or that the block's number [has been increased correctly][357]. Among others.

Having verified that its header follows consensus rules, the whole block is further [propagated to a subset of peers][358]. Only then [the actual import is run][359].

There's _a lot_ happening during an import. So I'll cut right to the chase.

After [several additional validations][360], the [parent's block state is retrieved][361]. This is the state on top of which the first transaction of the new block is going to be executed. Using it as a reference point, [the entire block is processed][362]. If you've ever heard that all Ethereum nodes are expected to execute and validate every single transaction, now you can be certain of it. Afterwards, the [post-state is validated][363] (see how [here][364]). Finally, [the block is written][365] to the local chain.

The successful import leads to [announcing][366] (not fully broadcasting) the block to the rest of the node's peers.

The whole verification process is replicated across all nodes that receive the block. A good portion will accept it into their local chains, and later newer blocks will arrive to insert on top of it.

## Retrieving the transaction

After a few blocks have been mined on top of the one where the transaction was included, one can start to safely assume that the transaction has been indeed confirmed.

Retrieving the transaction from the chain is quite simple. All we need is its hash. Conveniently, it was obtained as soon as we first submitted the transaction.

The data of the transaction itself, plus the block's hash and number, can always be retrieved at the node's `eth_getTransactionByHash` endpoint. Unsurprisingly, it now returns:

`{     "hash": "0xbf77c4a9590389b0189494aeb2b2d68dc5926a5e20430fb5bc3c610b59db3fb5",     "type": 2,     "accessList": [],     "blockHash": "0xe880ba015faa9aeead0c41e26c6a62ba4363822ddebde6dd77a759a753ad2db2",     "blockNumber": 15166167,     "transactionIndex": 0,     "confirmations": 6,     "from": "0x6fC27A75d76d8563840691DDE7a947d7f3F179ba",     "maxPriorityFeePerGas": 2000000000,     "maxFeePerGas": 120000000000,     "gasLimit": 40000,     "to": "0x6B175474E89094C44Da98b954EedeAC495271d0F",     "value": 0,     "nonce": 0,     "data": "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000",     "r": "0x057d733933b12238a2aeb0069b67c6bc58ca8eb6827547274b3bcf4efdad620a",     "s": "0x00e49937ec81db89ce70ebec5e51b839c0949234d8aad8f8b55a877bd78cc293",     "v": 1,     "creates": null,     "chainId": 31337 }`

The transaction's receipt can be requested at the `eth_getTransactionReceipt` endpoint. Depending on the node you're running this query, it might be the case you also get additional information on top of the expected transaction receipt data. This is the transaction receipt I got from my local fork of mainnet:

`{     "to": "0x6B175474E89094C44Da98b954EedeAC495271d0F",     "from": "0x6fC27A75d76d8563840691DDE7a947d7f3F179ba",     "contractAddress": null,     "transactionIndex": 0,     "gasUsed": 34706,     "logsBloom": "0x00000000000000000800000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000008000000000000008000000000000000000000000000000000000000011000000000000000000000000000000000000000000000000000010000000000000004000000000000000000000000080000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000002000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000",     "blockHash": "0x8b6d44d6cf39d01181b90677f8a77a2605d6e70c40d649eda659499063a19c77",     "transactionHash": "0xbf77c4a9590389b0189494aeb2b2d68dc5926a5e20430fb5bc3c610b59db3fb5",     "logs": [         {             "transactionIndex": 0,             "blockNumber": 15166167,             "transactionHash": "0xbf77c4a9590389b0189494aeb2b2d68dc5926a5e20430fb5bc3c610b59db3fb5",             "address": "0x6B175474E89094C44Da98b954EedeAC495271d0F",             "topics": [                 "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef",                 "0x0000000000000000000000006fc27a75d76d8563840691dde7a947d7f3f179ba",                 "0x000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa96045"             ],             "data": "0x0000000000000000000000000000000000000000000000000de0b6b3a7640000",             "logIndex": 0,             "blockHash": "0x8b6d44d6cf39d01181b90677f8a77a2605d6e70c40d649eda659499063a19c77"         }     ],     "blockNumber": 15166167,     "confirmations": 6, // number of blocks I waited before fetching the receipt     "cumulativeGasUsed": 34706,     "effectiveGasPrice": 9661560402,     "type": 2,     "byzantium": true,     "status": 1 }`

You saw that ? It says `"status": 1`.

That can only mean one thing: success!

## Afterword

There's definitely _far more_ to this story.

In some sense it's never-ending. There's always one more caveat. One more side note. An alternative execution path. Another node implementation. Another EVM instruction I might have skipped. Another wallet that handles things differently. All things that would take us one step closer to finding out "the real truth" behind what happens when you send 1 DAI.

Luckily, that's not what I intended to do. I hope the last +10k words did not make you think that 😛. Allow me to shed some light here.

In hindsight, this article is the byproduct of mixing curiosity and frustration.

Curiosity because I've been doing Ethereum smart contract security for more than 4 years, yet I hadn't took as much as time as I would've liked to manually explore, in depth, the complexities of the base layer. I really wanted to gain first-hand experience looking at the actual implementation of Ethereum itself. But smart contracts always got in the middle. Now that I've finally managed to found more peaceful times, it seemed the right time to go back to roots and embark on this adventure.

Curiosity wasn't enough though. I needed an excuse. A trigger. I knew what I had in mind was gonna be tough. So I needed a strong enough reason not only to get started. But also, more importantly, to get re-started whenever I would feel I was tired of trying to make sense out of Ethereum's code.

I found it where I wasn't looking. I found it in frustration.

Frustration at the absolutely mind-blowing lack of transparency we've grown so used to when sending money. If you've ever needed to do it in a developing country under increasingly strict capital controls, no doubt you feel me. So I wanted to remind myself that we can do better. I decided to channel my frustration in writing.

This article's also served me as a reminder. That if you can escape the fuzz, the price, the monkeys in JPEGs, the ponzis, the rugpulls, the thefts, there's still value in here. This is no "magic" internet money. There's real math, cryptography and computer science going. Being open source, you can see every single piece moving. You can almost touch them. No matter the day nor the time. No matter who you are. No matter where you come from.

So, I'm sorry for the clickbait-y title. This was never about what happens when you send 1 DAI.

It was about having the possibility to understand it.

[1]: https://notonlyowner.com/learn/que-pasa-cuando-envias-un-dai
[2]: https://makerdao.world/en/learn/Dai/
[3]: https://metamask.io
[4]: #building-the-transaction
[5]: #the-transactions-data-field
[6]: #gas-wizardry
[7]: #access-list-and-transaction-type
[8]: #signing-the-transaction
[9]: #serialization
[10]: #submitting-the-transaction
[11]: #reception
[12]: #inspecting-the-mempool
[13]: #propagation
[14]: #work-preparation-and-transaction-inclusion
[15]: #execution
[16]: #preparation-part-1
[17]: #preparation-part-2
[18]: #the-call
[19]: #the-interpreter-part-1
[20]: #solidity-execution
[21]: #evm-execution
[22]: #free-memory-pointer-and-calls-value
[23]: #validating-calldata-part-1
[24]: #function-dispatcher
[25]: #validating-calldata-part-2
[26]: #reading-parameters
[27]: #the-transfer-function
[28]: #the-transferFrom-function
[29]: #logging
[30]: #returning
[31]: #the-interpreter-part-2
[32]: #gas-refunds-and-payments
[33]: #building-the-transaction-receipt
[34]: #sealing-the-block
[35]: #broadcasting-the-block
[36]: #verifying-the-block
[37]: #retrieving-the-transaction
[38]: #afterword
[39]: https://github.com/ethereumbook/ethereumbook/blob/develop/05wallets.asciidoc
[40]: https://chainlist.org/
[41]: https://www.jsonrpc.org/
[42]: https://eips.ethereum.org/EIPS/eip-20
[43]: https://library.dedaub.com/contracts/Ethereum/6b175474e89094c44da98b954eedeac495271d0f/source
[44]: https://library.dedaub.com/contracts/Ethereum/6b175474e89094c44da98b954eedeac495271d0f/source?line=122
[45]: https://docs.soliditylang.org/en/latest/abi-spec.html
[46]: https://soliditylang.org/
[47]: https://book.getfoundry.sh/reference/cast/cast-calldata.html
[48]: https://ethereum.org/en/developers/docs/gas/#what-is-gas
[49]: https://docs.infura.io/infura/networks/ethereum/json-rpc-methods/eth_feehistory
[50]: https://composer.alchemyapi.io?share=eJwdyEEKgCAQBdC7.LULCyTwBK1atomIoSaKUkMnIqK7J_0e78G40OphtYJnuULcfjuWJUwNOYZF9jAz12uSEG8oHBTJtbSfnGA7FDrfTsJJMrrSKKNVZXr07wcDJR59
[51]: https://github.com/ethereum/execution-apis/blob/main/src/eth/fee_market.json#L14-L94
[52]: https://www.blocknative.com/blog/eip-1559-fees
[53]: https://eips.ethereum.org/EIPS/eip-2930
[54]: https://eips.ethereum.org/EIPS/eip-2718
[55]: https://eips.ethereum.org/EIPS/eip-1559#abstract
[56]: https://en.wikipedia.org/wiki/ECDSA
[57]: https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/
[58]: https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/tx
[59]: https://docs.ethers.io/v5/single-page/
[60]: https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/
[61]: https://www.youtube.com/watch?v=uZ7vkmUNTPA
[62]: https://github.com/ethereum/go-ethereum/tree/v1.10.18
[63]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1698-L1706
[64]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1702
[65]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1623-L1625
[66]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/ethconfig/config.go#L95
[67]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1626-L1629
[68]: https://eips.ethereum.org/EIPS/eip-155
[69]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1630
[70]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/api_backend.go#L245-L247
[71]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L852-L857
[72]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L843-L850
[73]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L226-L271
[74]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L896-L901
[75]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L902-L910
[76]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L648-L753
[77]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L586-L646
[78]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L604-L607
[79]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L595-L598
[80]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L49-L53
[81]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L628-L631
[82]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L632-L636
[83]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1633-L1645
[84]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1646
[85]: https://www.paradigm.xyz/2020/08/ethereum-is-a-dark-forest
[86]: https://arxiv.org/pdf/1904.05234.pdf
[87]: https://github.com/flashbots/mev-geth
[88]: https://docs.flashbots.net/flashbots-protect/rpc/quick-start
[89]: https://ethereum.org/en/developers/docs/consensus-mechanisms/pow/
[90]: https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/
[91]: https://github.com/ethereum/builder-specs
[92]: https://github.com/ethereum/devp2p/
[93]: https://github.com/ethereum/devp2p/blob/master/caps/eth.md#transaction-exchange
[94]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L524-L528
[95]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/node/defaults.go#L64
[96]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L603-L607
[97]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L621-L622
[98]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/peer.go#L183-L198
[99]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/peer.go#L213-L223
[100]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L390-L396
[101]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L184
[102]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L1338-L1345
[103]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L254
[104]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L536
[105]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L280
[106]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L293-L296
[107]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L531-L627
[108]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L456-L477
[109]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L436
[110]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L456-L459
[111]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L533
[112]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L534
[113]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1114-L1117
[114]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L982-L989
[115]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L990-L998
[116]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1003
[117]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1004
[118]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1006
[119]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1017
[120]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1024-L1027
[121]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L587
[122]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L337-L351
[123]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1031-L1035
[124]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L758-L768
[125]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1128
[126]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1063
[127]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1071-L1082
[128]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L849
[129]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L901-L904
[130]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L835
[131]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L87
[132]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L90
[133]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L836
[134]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L838
[135]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L839-L842
[136]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L144
[137]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L145
[138]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/transaction.go#L563
[139]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/transaction.go#L565
[140]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/transaction.go#L612
[141]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/transaction_signing.go#L195
[142]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L149-L151
[143]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/evm.go#L58-L69
[144]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L151
[145]: https://en.wikipedia.org/wiki/Stack_machine
[146]: https://en.wikipedia.org/wiki/256-bit_computing
[147]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L128
[148]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L137
[149]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L72-L101
[150]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L97-L98
[151]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L101
[152]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L181
[153]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L279-L284
[154]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L287
[155]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L218-L224
[156]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L225-L228
[157]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L229-L233
[158]: https://eips.ethereum.org/EIPS/eip-3607
[159]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L239-L246
[160]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L247-L250
[161]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L253
[162]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L259
[163]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L201-L203
[164]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L204-L206
[165]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L210
[166]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L305-L313
[167]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L124
[168]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/params/protocol_params.go#L32
[169]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L124
[170]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/params/protocol_params.go#L88
[171]: https://eips.ethereum.org/EIPS/eip-2028
[172]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/params/protocol_params.go#L34
[173]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L124
[174]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L316-L318
[175]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L320-L323
[176]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L302
[177]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L332
[178]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L333
[179]: https://www.youtube.com/watch?v=wMOkm57vu0k
[180]: https://www.evm.codes
[181]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L168
[182]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L164-L167
[183]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L169-L172
[184]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/params/protocol_params.go#L75
[185]: https://medium.com/arbitrary-execution/testing-the-limits-of-evm-stack-depth-c40ba55ca78e
[186]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/params/protocol_params.go#L79
[187]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L173-L176
[188]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L196
[189]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L177
[190]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L235-L236
[191]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L180
[192]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L194
[193]: https://eips.ethereum.org/EIPS/eip-158
[194]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L181
[195]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L214-L231
[196]: https://ethereum.github.io/yellowpaper/paper.pdf#section.8
[197]: https://etherscan.io/address/0x0000000000000000000000000000000000000001
[198]: https://etherscan.io/address/0x0000000000000000000000000000000000000009
[199]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/contracts.go#L45-L93
[200]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L407-L408
[201]: https://etherscan.io/tx/0xbdba0ec52ae2c9785468a453745b9f7024af6e165887a0e9038a5c2d2fea3909
[202]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L219
[203]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L220-L221
[204]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L228
[205]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L141
[206]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/params/protocol_params.go#L79
[207]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L140
[208]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L150
[209]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L164
[210]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L118-L120
[211]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L122-L127
[212]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L228
[213]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L524-L527
[214]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L580-L583
[215]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L626-L629
[216]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L826-L829
[217]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L679-L681
[218]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L846-L848
[219]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L181-L244
[220]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L239
[221]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L188-L189
[222]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/jump_table.go#L94-L99
[223]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L74-L75
[224]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L191-L196
[225]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L197-L199
[226]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L221-L225
[227]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L201-L217
[228]: https://www.evm.codes/about#memoryexpansion
[229]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go
[230]: https://soliditylang.org/
[231]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code
[232]: https://github.com/makerdao/dss/blob/master/src/dai.sol
[233]: https://samczsun.com/hiding-in-plain-sight/
[234]: https://sourcify.dev
[235]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L6
[236]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L122
[237]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L123
[238]: https://docs.soliditylang.org/en/v0.5.12/units-and-global-variables.html#block-and-transaction-properties
[239]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L125
[240]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L128
[241]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L90
[242]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L129
[243]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L133
[244]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L135
[245]: https://eips.ethereum.org/EIPS/eip-20#events
[246]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L136
[247]: https://eips.ethereum.org/EIPS/eip-20#methods
[248]: https://blog.openzeppelin.com/deconstructing-a-solidity-contract-part-i-introduction-832efd2d7737/
[249]: https://noxx.substack.com/p/evm-deep-dives-the-path-to-shadowy
[250]: https://library.dedaub.com/contracts/Ethereum/6b175474e89094c44da98b954eedeac495271d0f/disassembled
[251]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c
[252]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L505-L506
[253]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L277-L281
[254]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L294-L297
[255]: https://docs.soliditylang.org/en/v0.5.15/abi-spec.html#function-selector
[256]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L283-L292
[257]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L286
[258]: https://www.evm.codes/#1c
[259]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c#file-dai-asm-L3346-L3353
[260]: https://docs.soliditylang.org/en/v0.5.15/miscellaneous.html#mappings-and-dynamic-arrays
[261]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c#file-dai-asm-L1583-L1620
[262]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c#file-dai-asm-L3444-L3465
[263]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c#file-dai-asm-L1884-L1907
[264]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c#file-dai-asm-L3467-L3487
[265]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c#file-dai-asm-L1908-L1929
[266]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L844-L869
[267]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L849
[268]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L850-L851
[269]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L853
[270]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L854
[271]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L857
[272]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L858-L865
[273]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L181
[274]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L807
[275]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L246-L248
[276]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L228
[277]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L235-L243
[278]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L244
[279]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L339-L342
[280]: https://eips.ethereum.org/EIPS/eip-3529
[281]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L364-L366
[282]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L370
[283]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L343-L347
[284]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L345
[285]: https://watchtheburn.com/
[286]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L349-L353
[287]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L181
[288]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L101
[289]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L109
[290]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L113
[291]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L118-L122
[292]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L123
[293]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L124
[294]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L128
[295]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L132
[296]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L133
[297]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L134-L136
[298]: https://noxx.substack.com/i/55077458/bloom-filters
[299]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1155
[300]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L595
[301]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L645
[302]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L645
[303]: https://ethereum.org/en/glossary/#ommer
[304]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L596
[305]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L606
[306]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/block.go#L200-L228
[307]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/block.go#L206
[308]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/block.go#L214
[309]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1162
[310]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L648
[311]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L668
[312]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/sealer.go#L99
[313]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/sealer.go#L182
[314]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/sealer.go#L182
[315]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/sealer.go#L112
[316]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L687
[317]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L712-L732
[318]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L733-L734
[319]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L742-L743
[320]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L745-L746
[321]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/event/event.go#L97
[322]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L647
[323]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L652
[324]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L653
[325]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L586-L590
[326]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/peer.go#L291
[327]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/broadcast.go#L46
[328]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/peer.go#L281
[329]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L597
[330]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L562-L572
[331]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handler.go#L186
[332]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handler.go#L167-L182
[333]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handler.go#L215
[334]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L328
[335]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L329-L333
[336]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L334-L344
[337]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/block.go#L128-L143
[338]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L337-L340
[339]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L341-L344
[340]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L349
[341]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L351
[342]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler_eth.go#L67-L68
[343]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler_eth.go#L123-L124
[344]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L268
[345]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L420
[346]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L429
[347]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L796
[348]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L354
[349]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L376
[350]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L848-L853
[351]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L855
[352]: https://ethereum.github.io/yellowpaper/paper.pdf#subsection.4.3
[353]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L263
[354]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L310
[355]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L274
[356]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L270
[357]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L305
[358]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L859
[359]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L871
[360]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/blockchain.go#L1414-L1595
[361]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/blockchain.go#L1603
[362]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/blockchain.go#L1632
[363]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/blockchain.go#L1654
[364]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/block_validator.go#L81
[365]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/blockchain.go#L1669-L1674
[366]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L877
