<!--
---
title: Security Review Report
author: 4th05
date: December 7, 2024
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
    {\Huge InterPol\par} 
    \vspace{0.5cm}
    {\Huge\itshape Security Review Report\par}
    \vfill
    {\Large \ 7 December 2024\par}
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
  - [\[M1\] Wrong amount of distributed fees when there is `_overridingReferrer`](#m1-wrong-amount-of-distributed-fees-when-there-is-_overridingreferrer)

# Protocol Summary
InterPol is a protocol allowing users to lock their liquidity, no matter the duration, without having to renounce to the rewards possible.

InterPoL is a protocol specially designed for the Berachain ecosystem which allows token creators, launchpads, protocols, and more, to deploy protocol-owned liquidity while retaining the ability to stake it and participate in Proof of Liquidity flywheels.


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
| Contest platform |                 Cantina                  |
| LOC              |                   630                    |
| Language         |                 Solidity                 |
| Commit           | d14616dad7b8d4b9b5d89f26b6843c2972e441d5 |
| Previous audits  |          Pashov audit, LightChaser       |


# Scope

 - src/Beekeeper.sol
 - src/HoneyLocker.sol
 - src/HoneyQueen.sol



# Issues found

| Severity | Number of issues found |
| :------- | :--------------------: |
| High     |           0            |
| Medium   |           1            |
| Low      |           0            |
| Info     |           0            |


# Findings


## [M1] Wrong amount of distributed fees when there is `_overridingReferrer`

**Summary**

In case of `referrerOverrides[_referrer]!=0` the amount of fees distributed to the `_overridingReferrer` is not correct because it does not take into account the `_referrerFeeShare[_referrer]` value which has been set for the OG `referrer`. 

**Finding Description**

In the `Beekeeper::distributeFees` the `referrer` is updated considering any `_overridingReferrer`. However, getting the `FeeShare` value it passes as argument the updated `referrer` that is not linked with the `_referrerFeeShare[_referrer]` of the OG `referrer`.

```solidity
function distributeFees(address _referrer, address _token, uint256 _amount) external payable {
        bool isBera = _token == address(0);
        if (!isBera && _token.code.length == 0) revert NoCodeForToken();
        // if not an authorized referrer, send everything to treasury
        if (!isReferrer[_referrer]) {
            isBera ? STL.safeTransferETH(treasury, _amount) : STL.safeTransfer(_token, treasury, _amount);
            emit FeesDistributed(treasury, _token, _amount);
            return;
        }
        // use the referrer fee override if it exists, otherwise use the original referrer
@>      address referrer = referrerOverrides[_referrer] != address(0) ? referrerOverrides[_referrer] : _referrer;
@>      uint256 referrerFeeShareInBps = referrerFeeShare(referrer);
        uint256 referrerFee = (_amount * referrerFeeShareInBps) / 10000;

        if (isBera) {
            STL.safeTransferETH(referrer, referrerFee);
            STL.safeTransferETH(treasury, _amount - referrerFee);
        } else {
            STL.safeTransfer(_token, referrer, referrerFee);
            STL.safeTransfer(_token, treasury, _amount - referrerFee);
        }

        emit FeesDistributed(referrer, _token, referrerFee);
        emit FeesDistributed(treasury, _token, _amount - referrerFee);
    }
```

Therefore, the `Beekeeper::referrerFeeShare` function returns the `standardReferrerFeeShare` instead of the `_referrerFeeShare[_referrer]` of the OG `referrer`.

```solidity
 function setReferrerOverride(address _referrer, address _overridingReferrer) external onlyOwner {
        if (!isReferrer[_referrer]) revert NotAReferrer();
        referrerOverrides[_referrer] = _overridingReferrer;
    }
    function setReferrerFeeShare(address _referrer, uint256 _shareOfFeeInBps) external onlyOwner {
        if (!isReferrer[_referrer]) revert NotAReferrer();
        _referrerFeeShare[_referrer] = _shareOfFeeInBps;
    }
    /*###############################################################
                            VIEW ONLY
    ###############################################################*/
    /// @notice Returns the fee share for a given referrer
    /// @dev If a custom fee share is set for the referrer, it returns that value.
    ///      Otherwise, it returns the standard referrer fee share.
    /// @param _referrer The address of the referrer
    /// @return The fee share for the referrer in basis points (bps)
    function referrerFeeShare(address _referrer) public view returns (uint256) {
@>  return _referrerFeeShare[_referrer] != 0 ? _referrerFeeShare[_referrer] : standardReferrerFeeShare;
    }
```

**Impact Explanation**

The `_overridingReferrer` gets as `FeeShare` the `standardReferrerFeeShare` which could be greater or lower than the `_referrerFeeShare[_referrer]` would have been taken from the OG `referrer`.

&nbsp;

**Likelihood Explanation**

The likelihood is high as this happens every time that `FeeShare` has to be distributed to any of the `_overridingReferrer`.

**Proof of Concept**

```solidity
pragma solidity ^0.8.23;

import {Test, console} from "forge-std/Test.sol";
import {StdCheats} from "forge-std/StdCheats.sol";

import {ERC20} from "solady/tokens/ERC20.sol";
import {LibString} from "solady/utils/LibString.sol";
import {Solarray as SLA} from "solarray/Solarray.sol";

import {HoneyLocker} from "../src/HoneyLocker.sol";
import {HoneyQueen} from "../src/HoneyQueen.sol";
import {Beekeeper} from "../src/Beekeeper.sol";
import {LockerFactory} from "../src/LockerFactory.sol";
import {HoneyLockerV2} from "./mocks/HoneyLockerV2.sol";
import {GaugeAsNFT} from "./mocks/GaugeAsNFT.sol";
import {IStakingContract} from "../src/utils/IStakingContract.sol";
import {IBGT} from "./interfaces/IBGT.sol";

// prettier-ignore
contract POCTest is Test {
    using LibString for uint256;

    LockerFactory public factory;
    HoneyLocker public honeyLocker;
    HoneyLocker public newhoneyLocker;


    HoneyQueen public honeyQueen;
    Beekeeper public beekeeper;
    
    address public constant THJ = 0x4A8c9a29b23c4eAC0D235729d5e0D035258CDFA7;
    address public constant referral = address(0x5efe5a11);
    address public constant treasury = address(0x80085);
    address public constant operator = address(0xaaaa);

    string public constant PROTOCOL = "BGTSTATION";

    // These addresses are for the BARTIO network
    ERC20 public constant BGT = ERC20(0xbDa130737BDd9618301681329bF2e46A016ff9Ad);
    ERC20 public constant weHONEY_LP = ERC20(0x556b758AcCe5c4F2E1B57821E2dd797711E790F4);
    IStakingContract public weHONEY_GAUGE = IStakingContract(0x86DA232f6A4d146151755Ccf3e4555eadCc24cCF);

    function setUp() public {
        vm.createSelectFork("https://bartio.rpc.berachain.com/");

        vm.startPrank(THJ);

        beekeeper = new Beekeeper(THJ, treasury);
        beekeeper.setReferrer(referral, true);

        honeyQueen = new HoneyQueen(treasury, address(BGT), address(beekeeper));
        // prettier-ignore
        honeyQueen.setProtocolOfTarget(address(weHONEY_GAUGE), PROTOCOL);
        honeyQueen.setIsSelectorAllowedForProtocol(bytes4(keccak256("stake(uint256)")), "stake", PROTOCOL, true);
        honeyQueen.setIsSelectorAllowedForProtocol(bytes4(keccak256("withdraw(uint256)")), "unstake", PROTOCOL, true);
        honeyQueen.setIsSelectorAllowedForProtocol(bytes4(keccak256("getReward(address)")), "rewards", PROTOCOL, true);
        honeyQueen.setValidator(THJ);

        factory = new LockerFactory(address(honeyQueen));
        
        honeyLocker = HoneyLocker(payable(factory.clone(THJ, referral)));
        honeyLocker.setOperator(operator);

        vm.stopPrank();

        vm.label(address(honeyLocker), "HoneyLocker");
        vm.label(address(honeyQueen), "HoneyQueen");
        vm.label(address(weHONEY_LP), "weHONEY_LP");
        vm.label(address(weHONEY_GAUGE), "weHONEY_GAUGE");
        vm.label(address(this), "Tests");
        vm.label(THJ, "THJ");


        // ---> EDIT IF NEEDED <---
        // Deal yourself LP tokens
        StdCheats.deal(address(weHONEY_LP), THJ, 1);

    }

    function test_FeesDistributedForOverrideReferrersETH() public {        
        // ETH case
        address overridereferral = makeAddr("overridereferral");
        vm.deal(address(beekeeper),1 ether);
        vm.startPrank(THJ);
        beekeeper.setStandardReferrerFeeShare(100);
        beekeeper.setReferrerFeeShare(referral, 200);
        beekeeper.setReferrerOverride(referral, overridereferral);
        beekeeper.distributeFees(referral, address(0), 0.01 ether);
        console.log(overridereferral.balance);
        assertTrue(overridereferral.balance == 0.0002 ether, "Fee distributed with standardfeeshare instead of the referrerFeeShare");
    }

    # -- [78540] Beekeeper::distributeFees(0x000000000000000000000000000000005EfE5a11, 0x0000000000000000000000000000000000000000, 10000000000000000 [1e16])
    #   # -- [0] overridereferral::fallback{value: 100000000000000}()
    #   #   # -- [Stop] 
    #   # -- [0] 0x0000000000000000000000000000000000080085::fallback{value: 9900000000000000}()
    #   #   # -- [Stop] 
    #   #-- emit FeesDistributed(recipient: overridereferral: [0xAc3d52AFdDe3A152E4E585560d797D16bddaFD13], token: 0x0000000000000000000000000000000000000000, amount: 100000000000000 [1e14])
    #   # -- emit FeesDistributed(recipient: 0x0000000000000000000000000000000000080085, token: 0x0000000000000000000000000000000000000000, amount: 9900000000000000 [9.9e15])
    #   # -- [Stop] 
    # -- [0] console::log(100000000000000 [1e14]) [staticcall]
    #   # -- [Stop] 
    # -- [0] VM::assertTrue(false, "Fee distributed with standardfeeshare instead of the referrerFeeShare") [staticcall]
    #   # -- [Revert] Fee distributed with standardfeeshare instead of the referrerFeeShare
    # -- [Revert] Fee distributed with standardfeeshare instead of the referrerFeeShare
    

function test_FeesDistributedForOverrideReferrersERC20() public {      
        // ERC20 case
        address overridereferral = makeAddr("overridereferral");
        ERC20 USDCToken = ERC20(0xd6D83aF58a19Cd14eF3CF6fe848C9A4d21e5727c);
        deal(address(USDCToken),address(beekeeper), 1 ether);
        vm.startPrank(THJ);
        beekeeper.setStandardReferrerFeeShare(100);
        beekeeper.setReferrerFeeShare(referral, 200);
        beekeeper.setReferrerOverride(referral, overridereferral);
        beekeeper.distributeFees(referral, address(USDCToken), 0.01 ether);
        console.log(USDCToken.balanceOf(overridereferral));
        assertTrue(overridereferral.balance == 0.0002 ether, "Fee distributed with standardfeeshare instead of the referrerFeeShare");
    }

    # -- [59960] Beekeeper::distributeFees(0x000000000000000000000000000000005EfE5a11, 0xd6D83aF58a19Cd14eF3CF6fe848C9A4d21e5727c, 10000000000000000 [1e16])
    #   # -- [24782] 0xd6D83aF58a19Cd14eF3CF6fe848C9A4d21e5727c::transfer(overridereferral: [0xAc3d52AFdDe3A152E4E585560d797D16bddaFD13], 100000000000000 [1e14])
    #   #   # -- emit Transfer(from: Beekeeper: [0x7f826086F7225E2C30ae5325F7aa7f56605D2d39], to: overridereferral: [0xAc3d52AFdDe3A152E4E585560d797D16bddaFD13], amount: 100000000000000 [1e14])
    #   #   # -- [Return] true
    #   # -- [24782] 0xd6D83aF58a19Cd14eF3CF6fe848C9A4d21e5727c::transfer(0x0000000000000000000000000000000000080085, 9900000000000000 [9.9e15])
    #   #   # -- emit Transfer(from: Beekeeper: [0x7f826086F7225E2C30ae5325F7aa7f56605D2d39], to: 0x0000000000000000000000000000000000080085, amount: 9900000000000000 [9.9e15])
    #   #   # -- [Return] true
    #   # -- emit FeesDistributed(recipient: overridereferral: [0xAc3d52AFdDe3A152E4E585560d797D16bddaFD13], token: 0xd6D83aF58a19Cd14eF3CF6fe848C9A4d21e5727c, amount: 100000000000000 [1e14])
    #   # -- emit FeesDistributed(recipient: 0x0000000000000000000000000000000000080085, token: 0xd6D83aF58a19Cd14eF3CF6fe848C9A4d21e5727c, amount: 9900000000000000 [9.9e15])
    #   # -- [Stop] 
    # -- [558] 0xd6D83aF58a19Cd14eF3CF6fe848C9A4d21e5727c::balanceOf(overridereferral: [0xAc3d52AFdDe3A152E4E585560d797D16bddaFD13]) [staticcall]
    #   # -- [Return] 100000000000000 [1e14]
    # -- [0] console::log(100000000000000 [1e14]) [staticcall]
    #   # -- [Stop] 
    # -- [0] VM::assertTrue(false, "Fee distributed with standardfeeshare instead of the referrerFeeShare") [staticcall]
    #   # -- [Revert] Fee distributed with standardfeeshare instead of the referrerFeeShare
    # -- [Revert] Fee distributed with standardfeeshare instead of the referrerFeeShare
}
```
&nbsp;

&nbsp;


**Recommendation**

```Diff
function distributeFees(address _referrer, address _token, uint256 _amount) external payable {
        bool isBera = _token == address(0);
        if (!isBera && _token.code.length == 0) revert NoCodeForToken();
        // if not an authorized referrer, send everything to treasury
        if (!isReferrer[_referrer]) {
            isBera ? STL.safeTransferETH(treasury, _amount) : STL.safeTransfer(_token, treasury, _amount);
            emit FeesDistributed(treasury, _token, _amount);
            return;
        }
        // use the referrer fee override if it exists, otherwise use the original referrer
        address referrer = referrerOverrides[_referrer] != address(0) ? referrerOverrides[_referrer] : _referrer;
 -      uint256 referrerFeeShareInBps = referrerFeeShare(referrer);
 +      uint256 referrerFeeShareInBps = referrerFeeShare(_referrer);

        uint256 referrerFee = (_amount * referrerFeeShareInBps) / 10000;

        if (isBera) {
            STL.safeTransferETH(referrer, referrerFee);
            STL.safeTransferETH(treasury, _amount - referrerFee);
        } else {
            STL.safeTransfer(_token, referrer, referrerFee);
            STL.safeTransfer(_token, treasury, _amount - referrerFee);
        }

        emit FeesDistributed(referrer, _token, referrerFee);
        emit FeesDistributed(treasury, _token, _amount - referrerFee);
    }
```

