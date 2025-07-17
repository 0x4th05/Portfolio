<!--
---
title: Security Review Report
author: 4th05
date: May 26, 2025
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
    {\Huge Upside \par} 
    \vspace{0.5cm}
    {\Huge\itshape Security Review Report\par}
    \vfill
    {\Large \ 26 May 2025\par}
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
  - [\[L1\] Creation of `UpsideMetaCoin` vulnerable to reorg attacks](#l1-creation-of-upsidemetacoin-vulnerable-to-reorg-attacks)

&nbsp;





# Protocol Summary

Upside is a social prediction market (think Polymarket + reddit) where you win money by being early on viral tweets, Spotify songs, YouTube videos, etc.


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
| Contest platform |                Code4rena                 |
| LOC              |                   379                    |
| Language         |                 Solidity                 |
| Commit           | 9b733293823beebe0cc6813dcfb7bbbb2454d60a |
| Previous audits  |              Hans, 4naly3er              |



# Scope
- contracts/UpsideMetaCoin.sol
- contracts/UpsideProtocol.sol


# Issues found

| Severity | Number of issues found |
| :------- | :--------------------: |
| High     |           0            |
| Medium   |           0            |
| Low      |           1            |
| Info     |           0            |




# Findings

## [L1] Creation of `UpsideMetaCoin` vulnerable to reorg attacks


**Finding description and impact (Write a detailed description of the root cause and impact(s) of this finding)**

In the `UpsideProtocol` contract the creation of a `metacoin` contract through the `tokenize` function is vulnerable to reorg attacks because it is always done using a `CREATE` opcode. The address of the new contract is always computed using only the `address sender` and the `nonce`.

`metaCoinAddress = hash(sender, nonce)`

```solidity
    function tokenize(string calldata _url, address _tokenizeFeeAddress) external returns (address metaCoinAddress) {
        if (urlToMetaCoinMap[_url] != address(0)) {
            revert MetaCoinExists();
        }

        FeeInfo storage fee = feeInfo;

        // Handle tokenize fee
        // @dev We want to be able to accept many different ERC20 tokens as opposed to just one
        if (fee.tokenizeFeeEnabled) {
            if (tokenizeFeeMap[_tokenizeFeeAddress] == 0) {
                revert TokenizeFeeInvalid();
            }
            IERC20Metadata(_tokenizeFeeAddress).safeTransferFrom(
                msg.sender,
                fee.tokenizeFeeDestinationAddress,
                tokenizeFeeMap[_tokenizeFeeAddress]
            );
        }

       metaCoinAddress = address(
@>              new UpsideMetaCoin(
                META_COIN_DEFAULT_NAME,
                META_COIN_DEFAULT_SYMBOL,
                META_COIN_DEFAULT_TOTAL_SUPPLY,
                address(this)
            )
        );
        urlToMetaCoinMap[_url] = metaCoinAddress;

        metaCoinInfoMap[metaCoinAddress] = MetaCoinInfo(
            msg.sender,
            false,
            INITIAL_LIQUIDITY_RESERVES,
            META_COIN_DEFAULT_TOTAL_SUPPLY,
            block.timestamp
        );
        IUpsideStaking(stakingContractAddress).whitelistStakingToken(metaCoinAddress);

        emit UrlTokenized(metaCoinAddress, _url);
        return metaCoinAddress;
    }
```
This open the door to a reorg attack because neither the `url` in input nor the `user address` will be considered for the computation of a new `metaCoinAddress`.
This means that wether a reorg takes place after that a malicious user may create a contract at the same address, but linked with another `url` before the call of the swap function by the initial user thus making him to swap the funds for a wrong `metacoin`. 

**Proof of Concept**

```solidity
// SPDX-License-Identifier: BUSL-1.1
/*
Licensor:           Moai Labs LLC
Licensed Works:     This Contract
Change Date:        4 years after initial deployment of this contract.
Change License:     GNU General Public License v2.0 or later
*/

pragma solidity 0.8.24;

import "../contracts/UpsideMetaCoin.sol";
import {Test, console} from "../lib/forge-std/src/Test.sol";
import "../contracts/UpsideProtocol.sol";
import "../contracts/UpsideMetaCoin.sol";
import "../contracts/UpsideStaking.sol";
import "../contracts/UpsideStakingStub.sol";

contract TestUpsideProtocol is Test {

UpsideProtocol public upsideprotocol; 
UpsideStaking public upsideStaking;

address public owner;
address public Alice;
address tokenizeFeeAddress;

address public metacoin1;
address public metacoin2;

function setUp() public {
owner = makeAddr("owner");
Alice = makeAddr("Alice");
tokenizeFeeAddress = makeAddr("tokenizeFeeAddress");

upsideprotocol = new UpsideProtocol(owner);
upsideStaking = new UpsideStaking(address(upsideprotocol), Alice);

vm.startPrank(owner);
upsideprotocol.setStakingContractAddress(address(upsideStaking));
vm.stopPrank();
}

function test_reorgAttackMetacoin1() public {
vm.startPrank(owner);
metacoin1 = upsideprotocol.tokenize("Testmetacoin1", tokenizeFeeAddress);
vm.stopPrank();
}

function test_reorgAttackMetacoin2() public {
metacoin1 = 0x104fBc016F4bb334D775a19E8A6510109AC63E00;
vm.startPrank(owner);
metacoin2 = upsideprotocol.tokenize("Testmetacoin2", tokenizeFeeAddress);
vm.stopPrank();
assertEq(metacoin1, metacoin2, "The addresses are not the same");
}
}
```

To prove that in case of reorg the `metaCoinAddress` does not change using different `urls`, we firstly run a test function `test_reorgAttackMetacoin1` with an `url = "Testmetacoin1"`, then we use the output address `metacoin1` in a second function that uses the new `url = "Testmetacoin2"`. Comparing the 2 output we prove that the `metaCoinAddress` created is the same.


```solidity
[1538257] TestUpsideProtocol::test_reorgAttackMetacoin1()
    ├─ [0] VM::startPrank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [1503338] UpsideProtocol::tokenize("Testmetacoin1", tokenizeFeeAddress: [0x172536CdC4cBCEF1D189a90Dc4C8eE96CEc57B64])
    │   ├─ [1321984] → new UpsideMetaCoin@0x104fBc016F4bb334D775a19E8A6510109AC63E00
    │   │   ├─ emit OwnershipTransferred(previousOwner: 0x0000000000000000000000000000000000000000, newOwner: UpsideProtocol: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f])
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: UpsideProtocol: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], value: 1000000000000000000000000 [1e24])
    │   │   └─ ← [Return] 5790 bytes of code
    │   ├─ [25774] UpsideStaking::whitelistStakingToken(UpsideMetaCoin: [0x104fBc016F4bb334D775a19E8A6510109AC63E00])
    │   │   ├─ emit WhitelistSet(metaCoinAddress: UpsideMetaCoin: [0x104fBc016F4bb334D775a19E8A6510109AC63E00])
    │   │   └─ ← [Stop] 
    │   ├─ emit UrlTokenized(metaCoinAddress: UpsideMetaCoin: [0x104fBc016F4bb334D775a19E8A6510109AC63E00], url: "Testmetacoin1")
    │   └─ ← [Return] UpsideMetaCoin: [0x104fBc016F4bb334D775a19E8A6510109AC63E00]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.41ms (211.13µs CPU time)
```

```solidity
Ran 1 test for test/4th05.POC.t.sol:TestUpsideProtocol
[PASS] test_reorgAttackMetacoin2() (gas: 1561468)
Traces:
  [4498460] TestUpsideProtocol::setUp()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266]
    ├─ [0] VM::label(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266], "owner")
    │   └─ ← [Return] 
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] Alice: [0xBf0b5A4099F0bf6c8bC4252eBeC548Bae95602Ea]
    ├─ [0] VM::label(Alice: [0xBf0b5A4099F0bf6c8bC4252eBeC548Bae95602Ea], "Alice")
    │   └─ ← [Return] 
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] tokenizeFeeAddress: [0x172536CdC4cBCEF1D189a90Dc4C8eE96CEc57B64]
    ├─ [0] VM::label(tokenizeFeeAddress: [0x172536CdC4cBCEF1D189a90Dc4C8eE96CEc57B64], "tokenizeFeeAddress")
    │   └─ ← [Return] 
    ├─ [3415531] → new UpsideProtocol@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   ├─ emit OwnershipTransferred(previousOwner: 0x0000000000000000000000000000000000000000, newOwner: owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 16939 bytes of code
    ├─ [887862] → new UpsideStaking@0x2e234DAe75C793f67A35089C9d99245E1C58470b
    │   ├─ emit OwnershipTransferred(previousOwner: 0x0000000000000000000000000000000000000000, newOwner: UpsideProtocol: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f])
    │   ├─ emit ProtocolFeeRecipientSet(oldRecipient: 0x0000000000000000000000000000000000000000, newRecipient: Alice: [0xBf0b5A4099F0bf6c8bC4252eBeC548Bae95602Ea])
    │   └─ ← [Return] 4197 bytes of code
    ├─ [0] VM::startPrank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [23752] UpsideProtocol::setStakingContractAddress(UpsideStaking: [0x2e234DAe75C793f67A35089C9d99245E1C58470b])
    │   ├─ emit StakingContractAddressSet(stakingContractAddress: UpsideStaking: [0x2e234DAe75C793f67A35089C9d99245E1C58470b])
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 

  [1561468] TestUpsideProtocol::test_reorgAttackMetacoin2()
    ├─ [0] VM::startPrank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [1503338] UpsideProtocol::tokenize("Testmetacoin2", tokenizeFeeAddress: [0x172536CdC4cBCEF1D189a90Dc4C8eE96CEc57B64])
    │   ├─ [1321984] → new UpsideMetaCoin@0x104fBc016F4bb334D775a19E8A6510109AC63E00
    │   │   ├─ emit OwnershipTransferred(previousOwner: 0x0000000000000000000000000000000000000000, newOwner: UpsideProtocol: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f])
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: UpsideProtocol: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], value: 1000000000000000000000000 [1e24])
    │   │   └─ ← [Return] 5790 bytes of code
    │   ├─ [25774] UpsideStaking::whitelistStakingToken(UpsideMetaCoin: [0x104fBc016F4bb334D775a19E8A6510109AC63E00])
    │   │   ├─ emit WhitelistSet(metaCoinAddress: UpsideMetaCoin: [0x104fBc016F4bb334D775a19E8A6510109AC63E00])
    │   │   └─ ← [Stop] 
    │   ├─ emit UrlTokenized(metaCoinAddress: UpsideMetaCoin: [0x104fBc016F4bb334D775a19E8A6510109AC63E00], url: "Testmetacoin2")
    │   └─ ← [Return] UpsideMetaCoin: [0x104fBc016F4bb334D775a19E8A6510109AC63E00]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::assertEq(UpsideMetaCoin: [0x104fBc016F4bb334D775a19E8A6510109AC63E00], UpsideMetaCoin: [0x104fBc016F4bb334D775a19E8A6510109AC63E00], "The addresses are not the same") [staticcall]
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.95ms (316.34µs CPU time)

Ran 1 test suite in 103.47ms (1.95ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

**Recommended mitigation steps**

The creation of `metacoins` instances should be done employing a `CREATE2` opcode with a `salt` calculated based on `url` and `msg.sender`.


