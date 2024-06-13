![image](https://hackmd.io/_uploads/H1xJPiGal0.png)

1. 全部的功能都在[Router合约](https://etherscan.io/address/0x7a250d5630b4cf539739df2c5dacb4c659f2488d),Router合约在部署时需要提供 [WETH](https://etherscan.io/address/0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2)、[Factory](https://etherscan.io/address/0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f)参数，在兑换、增加/删除流动性的时候，通过内部参数直接计算 代币池 的合约，调用 池子合约计算汇率。
```soli=
    function pairFor(address factory, address tokenA, address tokenB) internal pure returns (address pair) {
        (address token0, address token1) = sortTokens(tokenA, tokenB);
        pair = address(uint(keccak256(abi.encodePacked(
                hex'ff',
                factory,
                keccak256(abi.encodePacked(token0, token1)),
                hex'96e8ac4277198ff8b6f785478aa9a39f403cb768dd02cbee326c3e7da348845f' // init code hash
            ))));
    }
```
2. Pool需要持有资产、资产可以充值|提现、Pool可以根据市场状况自动调整代币的兑换比例、流动性提供者能够从trades中获取利润
3. Pool创建交易对，根据交易对的代币余额计算流动性
4. 直接支持 ERC20/ERC20 的配对
5. 支持非标准的ERC20函数，标准的transfer、transferFrom返回 bool 表明转账交易成功与否，但是有些项目没有采用标准的返回值（什么都不返回），如果没有返回值的话，V1会认为交易失败，直接revert全部交易。但是V2采用 true 的默认值，非标准的转账不会 revert。同时，V2 增加 Lock Modifier 防止函数的重入问题
6. 提供流动性：用户通过存入代币对增加流动性获得流动性代币，流动性代币标识当前合约中代币对的份例数量。
7. Swap，每笔Swap都是需要（0.25+0.05）%的手续费，用户实际的代币输入是扣除手续费后的代币数量。手续费加在合约余额，由全部的流动性提供者在退款时按照份额划分，因此，每次swap都会增加流动性代币的实际份例，并且造成 X*Y=K（持续增长）
![image](https://hackmd.io/_uploads/ByWAFG6gR.png)
8. 支持perimit链下签名，链上校验转账。
```solidity=
    function permit(address owner, address spender, uint value, uint deadline, uint8 v, bytes32 r, bytes32 s) external {
        require(deadline >= block.timestamp, 'UniswapV2: EXPIRED');
        bytes32 digest = keccak256(
            abi.encodePacked(
                '\x19\x01',
                DOMAIN_SEPARATOR,
                keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, nonces[owner]++, deadline))
            )
        );
        address recoveredAddress = ecrecover(digest, v, r, s);
        require(recoveredAddress != address(0) && recoveredAddress == owner, 'UniswapV2: INVALID_SIGNATURE');
        _approve(owner, spender, value);
    }
```

## UniswapV2Factory

1. 创建新的代币对池子，配置池子是否单独收取手续费（收益地址）
2. 工厂合约存储全部的池子合约 `address[] public allPairs;`
3. [function createPair(address tokenA, address tokenB) external returns (address pair) ](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Factory.sol#L23C5-L23C90),创建新的代币池子
4. [转入代币对，添加流动性](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router01.sol#L58)
- 第一次添加流动性的话，需要燃烧一部分流动性mint到 0 地址`liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);_mint(address(0), MINIMUM_LIQUIDITY); `; 因为流动性的计算采用的 `sqrt`，因此初始提供流动性的时候会限制池子创始人提供的代币对的数量，从而后续提供流动性不需要提供大量的代币成本

- 流动性代币取决于注入的最小的代币对的代币数量，多余的代币不会退还。`liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);`

- 添加流动性会更新池子合约中`reserve=balance`，但是只有在每个区块的所有Uniswap的第一个交易中才会更新价格累加器，累加器中累计 A/B，B/A代币的兑换比例，累加器的累计区块跨度越大，oracle获取的实时价格的波动受当前大宗交易的影响越小。
```solidity=
    if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
            // * never overflows, and + overflow is desired
            price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
            price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
        }
```
[mint流动性代币](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L110)
```solidity=
    function mint(address to) external lock returns (uint liquidity) {
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        uint balance0 = IERC20(token0).balanceOf(address(this));
        uint balance1 = IERC20(token1).balanceOf(address(this));
        uint amount0 = balance0.sub(_reserve0);
        uint amount1 = balance1.sub(_reserve1);

        bool feeOn = _mintFee(_reserve0, _reserve1);
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        if (_totalSupply == 0) {
            liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
           _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
        } else {
            liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
        }
        require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
        _mint(to, liquidity);

        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
        emit Mint(msg.sender, amount0, amount1);
    }
```
5. [burn流动性代币，退还代币](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L134)
- 根据流动性计算代币对的对应数量，退还代币
- Swap转账时，需要扣除手续费,进行swap计算的是扣除手续费的代币数量。但是扣除的代币数量加在了整个池子的余额中，为流动性提供者产生收益。
- 流动性提供者在退款时能够根据流动性的占比退款存入的代币和池子swap产生的手续费
```solidity=
    function _safeTransfer(address token, address to, uint value) private {
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(SELECTOR, to, value));
        require(success && (data.length == 0 || abi.decode(data, (bool))), 'UniswapV2: TRANSFER_FAILED');
    }
```
6. [swap 代币交换](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L159)
- 代币交换后执行公式判断 `require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');`

## UniswapV2Router
1. [代币数量计算](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router01.sol#L30C5-L37C6)
- 根据预期输入的A代币占据总体代币的比例计算预期输入的B代币，要求B代币满足区间`[Desired,Min]`
- 返回 预期输入的 A，B 代币数量
```solidity=
    function _addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin
    )
```


2. [Token/Token增加流动性](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router01.sol#L58C1-L67C6)
- 有效期
- 根据木桶效应计算添加流动性实际的代币对输入数量
- 流动性代币mint到 to 地址
``` solidity=
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    )()
```

3. [增加ETH/Token流动性](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router01.sol#L74)
- ETH 转为WETH Token，池子中只有Token/Token的代币对
- 多余的ETH`msg.value`会退还，代币只是走的Approve,只扣除对应的部分
```solidity=
    function addLiquidityETH(
        address token,
        uint amountTokenDesired,
        uint amountTokenMin,
        uint amountETHMin,
        address to,
        uint deadline
    )
```

4. [移除Token/Token流动性，输入流动性代币的数量](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router01.sol#L99)
- 授权/转移，从`msg.sender` 转移流动性代币到 Pair 合约
- 根据输入的流动性代币的数量计算应该退还的代币对数量
- 代币数量 = 流动性代币的比例 * 代币总额
- `amount0 = liquidity.mul(balance0) / _totalSupply`
- 退还的代币对数量必须大于 Min 值
```solidity=
    function removeLiquidity(
        address tokenA,
        address tokenB,
        uint liquidity,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    )
```

5. [移除ETH/Token流动性，输入流动性代币的数量](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router01.sol#L116)
- 授权/转移，从`msg.sender` 转移流动性代币到 Pair 合约
- 根据输入的流动性代币的数量计算应该退还的代币对数量
- 代币数量 = 流动性代币的比例 * 代币总额
- `amount0 = liquidity.mul(balance0) / _totalSupply`
- 退还的代币对数量必须大于 Min 值
```solidity=
    function removeLiquidityETH(
        address token,
        uint liquidity,
        uint amountTokenMin,
        uint amountETHMin,
        address to,
        uint deadline
    )
```

6. [Permit移除Token/Token流动性，输入流动性代币的数量](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router01.sol#L116)
- 通过Permit直接校验签名，免去Approve操作，直接从`msg.sender`转移流动性代币到Pair合约
- 根据输入的流动性代币的数量计算应该退还的代币对数量
- 代币数量 = 流动性代币的比例 * 代币总额
- `amount0 = liquidity.mul(balance0) / _totalSupply`
- 退还的代币对数量必须大于 Min 值
```solidity=
    function removeLiquidityWithPermit(
        address tokenA,
        address tokenB,
        uint liquidity,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline,
        bool approveMax, uint8 v, bytes32 r, bytes32 s
    )
```

7. [Permit移除ETH/Token流动性，输入流动性代币的数量](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router01.sol#L152)
- 通过Permit直接校验签名，免去Approve操作，直接从`msg.sender`转移流动性代币到Pair合约
- 根据输入的流动性代币的数量计算应该退还的代币对数量
- 代币数量 = 流动性代币的比例 * 代币总额
- `amount0 = liquidity.mul(balance0) / _totalSupply`
- 退还的代币对数量必须大于 Min 值
```solidity=
    function removeLiquidityETHWithPermit(
        address token,
        uint liquidity,
        uint amountTokenMin,
        uint amountETHMin,
        address to,
        uint deadline,
        bool approveMax, uint8 v, bytes32 r, bytes32 s
    ) 
```

8. [计算Chains（一个输入，链式结构下每种代币的预期输出）](https://github.com/Uniswap/v2-periphery/blob/master/contracts/libraries/UniswapV2Library.sol#L62)
- 输入TokenA，经过 TokenB->TokenC->TokenD的链式计算，计算 TokenD的预期输出
- 输出结果保留每次兑换后的结果
- Index       0           1              2             3
- Path      TokenA     TokenB          TokenC         TokenD
- Amounts   TokenA  TokenA->TokenB  TokenB->TokenC  TokenC->TokenD
- 通过输入TokenA的数量，链式计算 Path 路径中每个代币对的 预期输出数量
```solidity=
    function getAmountsOut(address factory, uint amountIn, address[] memory path) internal view returns (uint[] memory amounts) {
        require(path.length >= 2, 'UniswapV2Library: INVALID_PATH');
        amounts = new uint[](path.length);
        amounts[0] = amountIn;
        for (uint i; i < path.length - 1; i++) {
            (uint reserveIn, uint reserveOut) = getReserves(factory, path[i], path[i + 1]);
            amounts[i + 1] = getAmountOut(amounts[i], reserveIn, reserveOut);
        }
    }
```

9. [计算Chains（一个输出，链式结构下每种代币的预期输入）](https://github.com/Uniswap/v2-periphery/blob/master/contracts/libraries/UniswapV2Library.sol#L62)
- 输入TokenD，经过 TokenC->TokenB->TokenA的链式计算，计算 TokenA 的预期输入
- 输出结果保留每次兑换后的结果
- Index            0               1              2           3
- Path          TokenA          TokenB         TokenC       TokenD
- Amounts   TokenB->TOkenA  TokenC->TokenB  TokenD->TokenC  TokenD
- 通过输出的TokenD的数量，链式计算 Path 路径中每个代币对的 预期输入数量
```solidity=
    function getAmountsIn(address factory, uint amountOut, address[] memory path) internal view returns (uint[] memory amounts) {
        require(path.length >= 2, 'UniswapV2Library: INVALID_PATH');
        amounts = new uint[](path.length);
        amounts[amounts.length - 1] = amountOut;
        for (uint i = path.length - 1; i > 0; i--) {
            (uint reserveIn, uint reserveOut) = getReserves(factory, path[i - 1], path[i]);
            amounts[i - 1] = getAmountIn(amounts[i], reserveIn, reserveOut);
        }
    }
```

10. [通过输入一定数量的Token和Path路径，获取链式兑换下的Token](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router01.sol#L179)
- 最终输出的代币数量 大于等于 最小约束
```solidity=
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) 
```

10. [Token和Path路径，为获取最终Token，计算链式兑换中应该输入的预期Token数量](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router01.sol#L191)
- 最终输入的代币数量 小于等于 最大约束
```solidity=
    function swapTokensForExactTokens(
        uint amountOut,
        uint amountInMax,
        address[] calldata path,
        address to,
        uint deadline
    )
```

11. [msg.value 和Path路径，输入ETH，计算链式兑换中最终输出的预期Token数量](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router01.sol#L203C1-L208C40)
- 最终输出的代币数量 大于等于 最小约束
```solidity=
    function swapExactETHForTokens(uint amountOutMin, address[] calldata path, address to, uint deadline)
        external
        override
        payable
        ensure(deadline)
        returns (uint[] memory amounts)
```

12. [msg.value 和Path路径，为获取预期Token，计算链式兑换中应该输入的ETH数量](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router01.sol#L217)
- 最终输入的ETH数量 小于等于 最大约束
```solidity=
    function swapTokensForExactETH(uint amountOut, uint amountInMax, address[] calldata path, address to, uint deadline)
        external
        override
        ensure(deadline)
        returns (uint[] memory amounts)
```

## 无常损失
无常损失表示在池子币价下跌时造成的损失，因为在AMM等去中心化的交易所中，流动性提供者自动成为代币对的买方和卖方，一旦有人发起交易，就会改变池子币价。

无常损失的公式为：
（Value_pool - Value_hold）/Value_hold * 100%

因此再有人砸盘时，会加大全部流动性提供者的损失，引起恐慌撤盘。此时，池子的深度变差，滑点变大，将会引起更多人恐慌撤资砸盘。

避免方式： 1. 套期保值： 在存入流动性的同时，将代币1：1做空，保证对冲。
