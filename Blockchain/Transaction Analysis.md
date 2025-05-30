# Transaction Analysis

```text
https://app.blocksec.com/explorer/tx/bsc/0xff77c9d0530fe6bbf6a5f24c5ddff466e0eaaa7630ecdd8cc6015c2eabf57881

1. What is the money flow. Who made money?
2. How this attacker made money, step by step, with detailed reasoning.
3. Implement the attack in foundry (https://book.getfoundry.sh/forge/).
```

首先先打开给的 transaction 地址进行查看,主要看看 `Fund Flow` 和 `Invocation Flow` (主要是后者),然后在右上角选择 All 和 Debug 选项然后可以简略的看一下一些常见的函数,这里看到了 flashloan 那大抵思路就差不多确定了

![image.png](https://img.z3n1th1.com/img/2025/05/56539438018529c6536bb1d5f88ddc6c.png)

然后就是逐步跟进流程,在 flashloan 然后能看到源码部分然后也看到了名字为 DPP

![image.png](https://img.z3n1th1.com/img/2025/05/b561a3362f92dc74fc659118ed4acabe.png)

这里我根据 DPPflashloan 和 address(0x6098a5638d8d7e9ed2f952d35b2b67c34ec6b476) 进行搜索后发现了这个 [github链接](https://github.com/SunWeb3Sec/DeFiHackLabs/blob/main/src/test/2025-01/Mosca2_exp.sol)

![image.png](https://img.z3n1th1.com/img/2025/05/a9e9c12c5e17f084c0051fac14f55196.png)

然后又跟进查看了 Mosca.sol 和 Mosca2_exp.sol,然后根据上面的题目回答一下问题

## Q1

主要是猜测因为并无完整代码,第一步是获取闪电贷(从攻击合约调用 DPP.flashloan 把资金获取到攻击合约),然后与目标合约进行交互,然后利用漏洞从目标合约提取资金(核心是提取了超出其应得份额的资金)和利润然后再偿还闪电贷最后实现利润

盈利的就是攻击者 EOA (应该是 exploiter 的地址)

## Q2

1.攻击者 EOA 发起交易调用攻击合约的 test 函数
2.从 DPP 请求闪电贷,攻击合约调用 DPP 合约的 `flashLoan(baseAmount, quoteAmount, address(this), callbackData)` 函数
3.DPP 闪电贷回调执行合约的 `DPPFlashLoanCall` 函数
4.在回调函数内利用攻击合约
5.偿还闪电贷
6.然后用多个代币循环套利(registration->transfer->approval)

## Q3

这里我简单写的 flashloan 来进行 implement

```bash
forge init flashloan
```

exp.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

import "./MockFlashLender.sol";
import "./IFlashLoanReceiver.sol";
import "./BaseTestWithBalanceLog.sol";

contract Exploit is BaseTestWithBalanceLog, IFlashLoanReceiver {
    MockFlashLender public lender;

    constructor(address _lender) {
        lender = MockFlashLender(payable(_lender));
    }

    function run() public balanceLog {
        lender.flashLoan(address(this), 1 ether);
    }

    function receiveLoan(uint256 amount) external {
    	emit log_named_uint("Got loan", amount);

    	lender.donateToAttacker{value: 0.1 ether}();

    	payable(msg.sender).transfer(amount);
    }

    receive() external payable {}
}

```

exp.t.sol

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.15;

import "forge-std/Test.sol";
import "../src/Exploit.sol";
import "../src/MockFlashLender.sol";

contract ExploitTest is Test {
    Exploit public exploit;
    MockFlashLender public lender;

    function setUp() public {
        lender = new MockFlashLender();
        vm.deal(address(lender), 10 ether);  
        exploit = new Exploit(address(lender));
        vm.deal(address(exploit), 1 ether);  
    }

    function testFlashLoan() public {
        exploit.run();
    }
}

```

![image.png](https://img.z3n1th1.com/img/2025/05/101e614a668fffdab6ce41607d03ebbc.png)
