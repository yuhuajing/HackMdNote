1. ERC20只能和ETH配对
2. 在Factory中，为每个ERC20合约通过工程合约创建一个 Exchange 合约，将该Token代币和Exchange代币配对后存储和 Factory合约
3. Uniswap的factory合约存储全部的代币和Exchange的配对信息
4. Exchange合约是代币-ETh的配对，任何人通过提供 ETH/ERC20代币提供流动性，流动性代币在提供流动性时mint,在退款时burn
5. 通过ETH进行 ERC20/ERC20的交易，需要执行两笔交易和付出两笔手续费。
6. 每笔交易会抽取0.3%的手续费，作为提供流动性的奖励（计算恒定乘积的时候扣除手续费，但是Exchange整体的余额增加，导致整体的乘积上涨。）每笔交易都会让恒定乘积上涨。
7. 用户可以设置滑点控制价格波动
8. public: [[addLiquidity(min_liquidity: uint256, max_tokens: uint256, deadline: timestamp)](https://github.com/Uniswap/docs/blob/main/docs/contracts/v1/reference/02-exchange.md#addliquidity)]
用户增加流动性
A. 函数中需要执行预期最小的流动性滑点，初始添加流动性的时候参数不起实际作用。
B. 根据实际存入的msg.value计算流动性参数，根据转账值计算出来的新增流动性值应该>= min_liquidity, 需要转入的配对的ERC20的数量应该 <= max_tokens
C. 将计算所需的代币存入合约
D. 时间未过期
9. public: [[removeLiquidity(amount: uint256, min_eth: uint256(wei), min_tokens: uint256, deadline: timestamp)](https://github.com/Uniswap/v1-contracts/blob/master/contracts/uniswap_exchange.vy#L83C5-L83C102)]
用户去除流动性，根据流动性代币的数量分别计算需要退还的交易对的数量
A.amout:表明需要退款的流动性代币。
B.根据实际退款的流动性代币数量 计算 配对代币的退款数量，根据 amout计算出来的实际退款ETH 应该 >= min_eth, 需要退款的配对的ERC20的数量应该 >= min_tokens
C.时间未过期
10. private: [[getInputPrice(input_amount: uint256, input_reserve: uint256, output_reserve: uint256)](https://github.com/Uniswap/v1-contracts/blob/master/contracts/uniswap_exchange.vy#L106C5-L106C90)]
对于配对的交易对，根据某一方的输入计算另一方的输出
A. (X+dX)(Y-dY)=XY==> dY=(dX*Y) / (X+dX) ,dX标识输入的代币数量，X标识现在合约中X代币的总量，Y表明现在合约中Y代币的总量
B.根据存入的代币数量计算能够得到的代币数量
C.需要考虑0.3%的手续费
D.新增的dX的有效值 仅为dx =（dX * 997）/ 1000, X = X * 1000
E.dY=(dX*Y) / (X+dX)
11. private: [[getOutputPrice(output_amount: uint256, input_reserve: uint256, output_reserve: uint256)](https://github.com/Uniswap/v1-contracts/blob/master/contracts/uniswap_exchange.vy#L120C5-L120C92)]
对于配对的交易对，根据某一方的预期输出计算另一方的输入
A. (X+dX)(Y-dY)=XY==> dx=(dY * X) / (Y-dY) ,dY标识预期输出的代币数量，X标识现在合约中X代币的总量，Y表明现在合约中Y代币的总量
B.根据预期输出的代币数量计算需要输入的代币数量
C.需要考虑0.3%的手续费
D.新增的dX的有效值 仅为dx' =（dX * 997）/ 1000, dx = (dx' * 1000)/997
E.dx=(dY * X) * 1000 / (Y-dY) * 997
12. private: [[ethToTokenInput(eth_sold: uint256(wei), min_tokens: uint256, deadline: timestamp, buyer: address, recipient: address)](https://github.com/Uniswap/v1-contracts/blob/master/contracts/uniswap_exchange.vy#L127)]
用户买Token，根据Eth的输入计算Token的预期输出
A. eth_sold: msg.value, 用户实际输入的代币数量
B. buyer: msg.sender
C. 根据购买的Eth计算应该输出的代币数量, 数量应该 >= min_tokens
D. 配对的代币转账到 recipient 地址
E.时间未过期
13. private: [[ethToTokenOutput(tokens_bought: uint256, max_eth: uint256(wei), deadline: timestamp, buyer: address, recipient: address)](https://github.com/Uniswap/v1-contracts/blob/master/contracts/uniswap_exchange.vy#L167)]
用户买Token，根据预期购买的Token数量扣除实际的Eth，退还剩余的Eth
A. tokens_bought, 用户预期获取的代币数量
B. buyer: msg.sender
C. 根据预期获取的代币数量计算应该转账的ETH数量, 数量应该 <= max_eth
D. 配对的代币转账到 recipient 地址,多余的ETH退还到 buyer 地址
E.时间未过期
14. private: [[tokenToEthInput(tokens_sold: uint256, min_eth: uint256(wei), deadline: timestamp, buyer: address, recipient: address)](https://github.com/Uniswap/v1-contracts/blob/master/contracts/uniswap_exchange.vy#L202C5-L202C122)]
用户卖出Token，根据卖出输入的Token数量计算得到的Eth数量
A. tokens_sold, 用户预期转入卖掉的的代币数量
B. buyer: msg.sender
C. 根据用户预期转入卖掉的的代币数量计算应该得到的ETH数量, 数量应该 >= min_eth
D. 获得的Eth转账到 recipient 地址
E. 代币从 buyer转入到合约地址
F.时间未过期
15. private: [[tokenToEthOutput(eth_bought: uint256(wei), max_tokens: uint256, deadline: timestamp, buyer: address, recipient: address)](https://github.com/Uniswap/v1-contracts/blob/master/contracts/uniswap_exchange.vy#L237)]
用户卖出Token，根据预期的Eth计算需要卖出的Token数量
A. eth_bought, 用户预期得到的Eth数量
B. buyer: msg.sender
C. 根据用户预期得到的Eth代币数量计算应该卖出的Token数量, 数量应该 <= max_tokens
D. 获得的Eth转账到 recipient 地址
E. 代币从 buyer 转入到合约地址
F. 时间未过期
16. private: [[tokenToTokenInput(tokens_sold: uint256, min_tokens_bought: uint256, min_eth_bought: uint256(wei), deadline: timestamp, buyer: address, recipient: address, exchange_addr: address)](https://github.com/Uniswap/v1-contracts/blob/master/contracts/uniswap_exchange.vy#L271C5-L271C183)]
用户进行Token-Token的Swap，根据TokenA的输入计算TokenB的输出
A. 在TokenA的Exchange合约中 根据 tokens_sold，调用 getInputPrice 计算出预期得到的Eth,Eth >= min_eth_bought
B. 将TokenA转入TokenA的Exchange合约
C. 根据用户预期得到的min_tokens_bought TokenB数量 和 TokenA卖出的Eth， 发送到TokenB的 ethToTokenTransferInput 函数处理
D. 在Token的Exchange合约中，将 Eth购入的TokenB转到 Receipt地址
E. 实际进行了两笔交易，在TokenA Exchange中调用 TokenB Exchange 的 ethToTokenTransferInput。
F. 实际付出了两笔手续费
G. 时间未过期
17. private: [[tokenToTokenOutput(tokens_bought: uint256, max_tokens_sold: uint256, max_eth_sold: uint256(wei), deadline: timestamp, buyer: address, recipient: address, exchange_addr: address)](https://github.com/Uniswap/v1-contracts/blob/master/contracts/uniswap_exchange.vy#L312C5-L312C182)]
用户进行Token-Token的Swap，根据TokenB的预期输出计算TokenA的输入
A. 在TokenB的Exchange合约中 根据 TokenB的预期输出tokens_bought，getEthToTokenOutputPrice 计算出需要的Eth,Eth <= max_eth_sold
B. 在TokenA的Exchange合约中 根据 Eth数量，调用 getOutputPrice 计算出预期输入的TokenA的数量,数量 <= max_tokens_sold
C. 将TokenA的预期卖出数量 转入TokenA的Exchange合约 得到Eth， 发送到TokenB的 ethToTokenTransferInput 函数处理
D. 在Token的Exchange合约中，将 Eth购入的TokenB转到 Receipt地址
E. 实际进行了两笔交易，在TokenA Exchange 中调用 TokenB Exchange 的 ethToTokenTransferInput。
F. 实际付出了两笔手续费
G. 时间未过期
18. 流动性代币实际就是ERC20代币，支持转账
19. 参开资料
A. https://hackmd.io/C-DvwDSfSxuh-Gd4WKE_ig
B. https://github.com/Uniswap/v1-contracts/blob/master/contracts/uniswap_exchange.vy#L312C5-L312C182
C. https://github.com/Uniswap/docs/tree/main/docs/contracts/v1