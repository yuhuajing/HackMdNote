借贷市场
1. 用户借贷需要提供抵押物，但是在去中心化的借贷平台中，用户难以直接评估抵押物的价值，因此需要预言机获取抵押物价值
2. 预言机：用户输入Token地址，返回当前代币锚定参照物的价格

预言机分类
1. 链下中心化预言机合约，价格更新的数据源在链下。由一个组织在链下获取代币价格，通过一个EOA地址定时上链更新预言机合约数据
2. 链下去中心化预言机合约，价格更新的数据源在链下。由多个组织获取价格，通过多签进行定期更新预言机合约数据，价格由平均值或中位数决策
3. 链上中心化预言机合约，价格更新的数据源在链上Dex。由一个组织在链上Dex获取代币价格，通过一个EOA地址定时上链更新预言机合约数据
4. 链上去中心化预言机合约，价格更新的数据源在链上Dex。此时，预言机合约数据的更新不在局限在某一特定组织，任何人都可以更新预言机合约
5. 常量预言机，例如USDT/USDC等稳定币

使用链上预言机必须进行的校验，因为用户可已通过大量代币的买卖操纵当前池子中的代币兑换利率，因此造成基于这些池子的借贷平台的利率巨幅波动，无法正确计算抵押物的价值，从而造成借出代币的价值远超抵押物价值的攻击问题，[基于Uniswap早期借贷的价格操纵](https://samczsun.com/taking-undercollateralized-loans-for-fun-and-for-profit/)

关键的解决方案：
1. FlashSwap 闪电贷等需要在一个区块交易中完成借出和归还，因此价格操纵的攻击需要在同一个区块中进行大量借出代币的买入，操纵交易对价格。因此，使用预言机获取数据价格的时候需要添加大量的校验/价格上下调整的幅度
2. 调用第三方预言机合约的函数，必须进行函数的校验审核。池子只关心配对代币的交易，但是借贷关心的是等价物的价值（不应该突然存在大幅波动）