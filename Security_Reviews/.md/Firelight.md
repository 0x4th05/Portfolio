<!--
---
title: Security Review Report
author: 4th05
date: July 31, 2024
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries 4th05\par}
    \vspace{3cm}
    {\Huge Zaros\par} 
    \vspace{0.5cm}
    {\Huge\itshape Security Review Report\par}
    \vfill
    {\Large \ 31 July 2024\par}
\end{titlepage}

\maketitle
CANCEL THIS LINE AND THE FIRST ONE TO TO ABOLISH NOTES AND GET PDF FILE -->




<!-- Your report starts here! -->

<!-- Prepared by: [4th05](https://x.com/0x4th05)
Lead Auditors:  
- xxxxxxx
\begin{flushright}...\end{flushright}
-->
    
# Table of Contents

- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Overview](#overview)
- [Scope](#scope)
- [Issues found](#issues-found)
- [Findings](#findings)

&nbsp;

&nbsp;

# Protocol Summary

Firelight transforms staked XRP into protection for DeFi. Builders buy transparent cover; stakers earn fees for backing it. Firelight’s staking mechanism for XRP lets you earn rewards on your XRP without sacrificing liquidity. When you stake XRP through Firelight, the protocol automatically issues you a liquid staking token (LST) that represents your staked XRP. This token enables you to not only capture Firelight rewards, but also explore DeFi opportunities while your principal remains staked.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended. 


# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |



# Overview 

|                  |                                          |
| ---------------- | :--------------------------------------: |
| Contest platform |                Immunefi                  |
| LOC              |                   528                    |
| Language         |                 Solidity                 |
| Commit           | cc29f140e5b56a69a1b22775666149bc609802ec |
| Previous audits  |               OpenZeppelin               |


# Scope
- https://github.com...FirelightVault.sol
- https://github.com...htVaultStorage.sol



# Issues found

| Severity | Number of issues found |
| :------- | :--------------------: |
| High     |           0            |
| Medium   |           0            |
| Low      |           1            |
| Info     |           0            |


# Findings

**Title**
Underflow issue leading to a `periodAtTimestamp` DOS


**Summary**
Using the `FirelightVault::periodAtTimestamp` with an input `Timestamp` corresponding to a configuration period `Config` so that `Config.epoch > block.timestamp` the function reverts instead of returning the period number that it should return for the given input value. 

&nbsp;

**Vulnerability Details**
The `FirelightVault::periodAtTimestamp` reverts when called using `timestamps` that correspond to a configuration period not yet started. 

```solidity
@>    function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
        PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        // solhint-disable-next-line max-line-length
@>        return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
    }
```
This, because of the function `_sinceEpoch` that takes the value provided in input by the users and without making any check, it uses it to compute its return value.
If the value provided is in the future the result will underflow thus, reverting 
the call to the `_sinceEpoch` which will in turn revert the `periodAtTimestamp` function.

```solidity
    function _sinceEpoch(uint48 epoch) private view returns (uint48) {
@>      return Time.timestamp() - epoch;
    }
```


&nbsp;

**Impact**
The function does not do what it should but there is no loss of money. 

So the severity is low. 

Although within the `FirelightVault` contract the `periodAtTimestamp` function is called just once by the `FirelightVault::currentPeriod` using always `Time.timestamp()` as argument, it is still public meaning that any user could call it.

This function should return the period number for a given `timestamp` input. However, because of the underflow issue above, if the `periodConfiguration.epoch > block.timestamp` it `revert` without returning the period number to the user.

&nbsp;

**POC**
Create a new foundry test file `FirelightVault.t.sol` like the one below and run `forge test --mt testPeriodAtTimestamp -vvvvv`. 

To pass the test the `FirelightVault::periodAtTimestamp` should revert because of an underflow.

```solidity
/* SPDX-License-Identifier: UNLICENSED */
pragma solidity 0.8.28;
import {ERC20Upgradeable} from "@openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {FirelightVaultStorage} from "../FirelightVaultStorage.sol";
import {FirelightVault} from "../FirelightVault.sol";
import {Test, console} from "lib/forge-std/src/Test.sol";


contract assetVaultMock is ERC20 {
    constructor() ERC20 ("AssetVault", "AV")  {
    }
}

contract FirelightVaultTest is Test {
    FirelightVault public vault;
    assetVaultMock public assetvaultmock;
    
    function setUp() public {
        vault = new FirelightVault();
        assetvaultmock = new assetVaultMock();
    }

function testPeriodAtTimestamp() public {
    FirelightVault.InitParams memory vaultparams = FirelightVault.InitParams({
                defaultAdmin: address(0x1),
                limitUpdater: address(0x2),
                blocklister: address(0x3),
                pauser: address(0x4),
                periodConfigurationUpdater: address(0x5),
                depositLimit: 1000,
                periodConfigurationDuration: 1 days
            });
            
    bytes memory vaultparamsencoded = abi.encode(vaultparams);
    vault.initialize(IERC20(assetvaultmock), "AssetVault", "AV", vaultparamsencoded);
    
    uint256 start = block.timestamp;

    vm.startPrank(vaultparams.periodConfigurationUpdater);
    vault.addPeriodConfiguration(uint48(start) + 2 days, 2 days);

    vm.warp(start + 3 days);
    
    vault.addPeriodConfiguration(uint48(start) + 6 days, 4 days);
    vm.stopPrank();

    uint256 periodnumber;
    vm.expectRevert();
    periodnumber = vault.periodAtTimestamp(uint48(start+ 10 days));   
}
}
```
The result of the test below:

```
[⠊] Compiling...
[⠒] Files to compile:
- test/FirelightVault.t.sol
[⠒] Compiling 1 files with Solc 0.8.28
[⠑] Solc 0.8.28 finished in 1.38s
Compiler run successful!

Ran 1 test for test/FirelightVault.t.sol:FirelightVaultTest
[PASS] testPeriodAtTimestamp() (gas: 466434)
Traces:
  [4071255] FirelightVaultTest::setUp()
    ├─ [3575813] → new FirelightVault@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← [Return] 17859 bytes of code
    ├─ [398320] → new assetVaultMock@0x2e234DAe75C793f67A35089C9d99245E1C58470b
    │   └─ ← [Return] 1764 bytes of code
    └─ ← [Stop] 

  [466434] FirelightVaultTest::testPeriodAtTimestamp()
    ├─ [338400] FirelightVault::initialize(assetVaultMock: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], "AssetVault", "AV", 0x0000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000500000000000000000000000000000000000000000000000000000000000003e80000000000000000000000000000000000000000000000000000000000015180)
    │   ├─ [176] assetVaultMock::decimals() [staticcall]
    │   │   └─ ← [Return] 18
    │   ├─ emit PeriodConfigurationAdded(periodConfiguration: PeriodConfiguration({ epoch: 1, duration: 86400 [8.64e4], startingPeriod: 0 }))
    │   ├─ emit RoleGranted(role: 0x0000000000000000000000000000000000000000000000000000000000000000, account: ECRecover: [0x0000000000000000000000000000000000000001], sender: FirelightVaultTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0x7a239212a00ca755b79509a034715154386895f9f9e24abe8401e5b1b9a1a5a0, account: SHA-256: [0x0000000000000000000000000000000000000002], sender: FirelightVaultTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0xdef752f6ad59be9880423e079755189539de01983d091b3c097e4742fd9916d1, account: RIPEMD-160: [0x0000000000000000000000000000000000000003], sender: FirelightVaultTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0x139c2898040ef16910dc9f44dc697df79363da767d8bc92f2e310312b816e46d, account: Identity: [0x0000000000000000000000000000000000000004], sender: FirelightVaultTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0x6c6a5de61c8667e917ea675d182a58870b6de23041e69efc8e0465f7ef75e7b9, account: ModExp: [0x0000000000000000000000000000000000000005], sender: FirelightVaultTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit Initialized(version: 1)
    │   ├─  storage changes:
    │   │   @ 0x52c63247e1f47db19d5ce0460030c497f067ca4cebf71ba98eeadabe20bace03: 0 → 0x41737365745661756c7400000000000000000000000000000000000000000014
    │   │   @ 0x52c63247e1f47db19d5ce0460030c497f067ca4cebf71ba98eeadabe20bace04: 0 → 0x4156000000000000000000000000000000000000000000000000000000000004
    │   │   @ 0xd97e9296c8127589b70b40d04fc511484a9919d4cba447c552aea0f159b53c7e: 0 → 1
    │   │   @ 0x8cf2bad4504154ea622c9df3408a6b24dc7c8c99afd0e5464f81cffd2d7932c0: 0 → 1
    │   │   @ 0xc2575a0e9e593c00f959f8c92f12db2869c3395a3b0502d05e2516446f71f85b: 0 → 0x0000000000000000000000000000000000000000000000015180000000000001
    │   │   @ 0xf0c57e16840df040f15088dc2f81fe391c3923bec73e23a9662efc9c229c6a00: 0 → 1
    │   │   @ 0x62b6521d3e2ea7e1b496d12660935a7ab083105635e8cf856873c0a5e8ec5e07: 0 → 1
    │   │   @ 0x6027e88424fe4b9777edef53462c97928118e55208c51c7d47e08eede0d4e650: 0 → 1
    │   │   @ 0x0773e532dfede91f04b12a73d3d2acd361424f41f76b4fb79f090161e36b4e00: 0 → 0x0000000000000000000000122e234dae75c793f67a35089c9d99245e1c58470b
    │   │   @ 1: 0 → 1
    │   │   @ 0xdff03bbac111df6b20306de44f1bfc9a12da2102242fcf4d7a52b3a427c3b02f: 0 → 1
    │   │   @ 0x9b779b17422d0df92223018b32b4d1fa46e071723d6817e2486d003becc55f00: 0 → 1
    │   │   @ 3: 0 → 1
    │   │   @ 0: 0 → 1000
    │   └─ ← [Stop] 
    ├─ [0] VM::startPrank(ModExp: [0x0000000000000000000000000000000000000005])
    │   └─ ← [Return] 
    ├─ [53729] FirelightVault::addPeriodConfiguration(172801 [1.728e5], 172800 [1.728e5])
    │   ├─ emit PeriodConfigurationAdded(periodConfiguration: PeriodConfiguration({ epoch: 172801 [1.728e5], duration: 172800 [1.728e5], startingPeriod: 2 }))
    │   ├─  storage changes:
    │   │   @ 0xc2575a0e9e593c00f959f8c92f12db2869c3395a3b0502d05e2516446f71f85e: 0 → 2
    │   │   @ 3: 1 → 2
    │   │   @ 0xc2575a0e9e593c00f959f8c92f12db2869c3395a3b0502d05e2516446f71f85d: 0 → 0x000000000000000000000000000000000000000000000002a30000000002a301
    │   └─ ← [Stop] 
    ├─ [0] VM::warp(259201 [2.592e5])
    │   └─ ← [Return] 
    ├─ [56482] FirelightVault::addPeriodConfiguration(518401 [5.184e5], 345600 [3.456e5])
    │   ├─ emit PeriodConfigurationAdded(periodConfiguration: PeriodConfiguration({ epoch: 518401 [5.184e5], duration: 345600 [3.456e5], startingPeriod: 4 }))
    │   ├─  storage changes:
    │   │   @ 0xc2575a0e9e593c00f959f8c92f12db2869c3395a3b0502d05e2516446f71f860: 0 → 4
    │   │   @ 0xc2575a0e9e593c00f959f8c92f12db2869c3395a3b0502d05e2516446f71f85f: 0 → 0x000000000000000000000000000000000000000000000005460000000007e901
    │   │   @ 3: 2 → 3
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return] 
    ├─ [3746] FirelightVault::periodAtTimestamp(864001 [8.64e5]) [staticcall]
    │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    ├─  storage changes:
    │   @ 0xc2575a0e9e593c00f959f8c92f12db2869c3395a3b0502d05e2516446f71f860: 0 → 4
    │   @ 0xc2575a0e9e593c00f959f8c92f12db2869c3395a3b0502d05e2516446f71f85e: 0 → 2
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 12.49ms (8.12ms CPU time)

Ran 1 test suite in 4.32s (12.49ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

&nbsp;

**References**
https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol#L246C4-L250

https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol#L264-L266

&nbsp;

**Tools Used**

Manual review

&nbsp;

**Recommendations**

```diff
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
+  if(periodConfiguration.epoch <= block.timestamp){        
   // solhint-disable-next-line max-line-length
   return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
+  } else {
+  return periodConfiguration.startingPeriod + (timestamp - periodConfiguration.epoch) / periodConfiguration.duration;
+        }
    }
```