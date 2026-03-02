<!--
---
title: Security Review Report
author: 4th05
date: January 23, 2025
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
    {\Huge Plaza Finance\par} 
    \vspace{0.5cm}
    {\Huge\itshape Security Review Report\par}
    \vfill
    {\Large \ 23 January 2025\par}
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
  - [\[L1\] Wrong decrease of the `freePsmAssets` in the `sell` function](#l1-wrong-decrease-of-the-freepsmassets-in-the-sell-function)
  - [\[L2\] The requirement in the constructor regarding `halfLife` bypassed.](#l2-the-requirement-in-the-constructor-regarding-halflife-bypassed)


&nbsp;

&nbsp;

# Protocol Summary
Monolith is a stablecoin-as-a-service platform being launched by Inverse Finance, enabling permissionless creation of immutable, over-collateralized stablecoins using any collateral with an on-chain price feed. Monolith's design features interest-bearing vaults for stablecoin holders, autonomous interest rate controllers, and deployer fee access, all optimized for security, scalability, and cross-chain expansion.


# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended. 

&nbsp;

&nbsp;


# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

&nbsp;

&nbsp;

# Overview 
|                  |                                          |
| ---------------- | :--------------------------------------: |
| Contest platform |                 Sherlock                 |
| LOC              |                   1091                   |
| Language         |                 Solidity                 |
| Commit           | 227bff2f0b5942f1a56806e9ed3475caa627743b |
| Previous audits  |                  yaudit                  |

&nbsp;

&nbsp;

# Scope

src
- Coin.sol
- Factory.sol
- InterestModel.sol
- Lender.sol
- Lens.sol
- Vault.sol

&nbsp;

&nbsp;

# Issues found

| Severity | Number of issues found |
| :------- | :--------------------: |
| High     |           0            |
| Medium   |           0            |
| Low      |           2            |
| Info     |           0            |

&nbsp;



# Findings

## [L1] Wrong decrease of the `freePsmAssets` in the `sell` function


**Summary**

In the `sell` function the `freePsmAssets` is wrongly updated. This has an impact on profits that are overestimated when calcuated through the function `accruePsmProfit`.

**Relevant GitHub Links**

https://github.com/sherlock-audit/2025-12-monolith-stablecoin-factory/blob/main/Monolith/src/Lender.sol#L510

**Root Cause**

In the `sell` function of the Monolith contract, the `freePsmAssets` variable is decreased by the total amounts of the assets sold, without considering the actual amount that was transferred to the vault.  
This `freePsmAssets`, as done in the `buy` function should be decreased by the `actualAssetOutToSeller` value and not the `assetOut` amount.

```solidity
function sell(uint coinIn, uint minAssetOut) external returns (uint assetOut) {
        require(psmAsset != ERC20(address(0)), "PSM asset was not set");
        accrueInterest();
        assetOut = getSellAmountOut(coinIn);
        require(assetOut >= minAssetOut, "insufficient amount out");
        // get and burn coins from caller
        coin.transferFrom(msg.sender, address(this), coinIn);
        coin.burn(coinIn);
        // give assets to caller
        if(psmVault != ERC4626(address(0))) {
            // Convert to shares out without taking into account fees, which will be paid by the seller by receiving less than assetOut
            uint256 sharesOut = psmVault.convertToShares(assetOut);
            uint256 actualAssetOutToSeller = psmVault.redeem(sharesOut, msg.sender, address(this));
@>          freePsmAssets -= assetOut;
            require(actualAssetOutToSeller >= minAssetOut, "redeem failed");
        } else {
            freePsmAssets -= assetOut;
            psmAsset.safeTransfer(msg.sender, assetOut);
        }
        emit Sold(msg.sender, coinIn, assetOut);
    }
```
```solidity 
   function buy(uint assetIn, uint minCoinOut) external beforeDeadline returns (uint coinOut) {
        require(psmAsset != ERC20(address(0)), "PSM asset was not set");
        accrueInterest();
        uint coinFee;
        (coinOut, coinFee) = getBuyAmountOut(assetIn);
        require(coinOut >= minCoinOut, "insufficient amount out");

        if(coinFee > 0) accruedLocalReserves += uint120(coinFee);

        // get assets from caller
        psmAsset.safeTransferFrom(msg.sender, address(this), assetIn);
        if(psmVault != ERC4626(address(0))) {
            require(psmVault.totalSupply() > minTotalSupply, "PSM vault total supply below minimum");
            uint256 shares = psmVault.deposit(assetIn, address(this));
            require(shares > 0, "PSM deposit failed");
@>          freePsmAssets += psmVault.previewRedeem(shares);
        } else {
            freePsmAssets += assetIn;
        }
        // give coins to caller
        coin.mint(msg.sender, coinOut);
        emit Bought(msg.sender, assetIn, coinOut);
    }
```

**Attack Path**

- A user call the sell function to get back some assets 
- `freePsmAssets` is wrongly updated
- `accruePsmProfit` whenever called overestimate profits by the amount of `assetOut- actualAssetOutToSeller` because of using a wrong `freePsmAssets`

**Impact**

This wrong update of the `freePsmAssets` variable impacts the `accruePsmProfit` increasing profits through the `accruedLocalReserves` value because it decreases the `freePsmAssets` more than what it should.

```solidity
function accruePsmProfit() internal {
        if(address(psmVault) != address(0)){
            uint assets = psmVault.previewRedeem(psmVault.balanceOf(address(this)));
            if(assets <= freePsmAssets) return; // avoids underflow in case of loss
@>          uint profit = assets - freePsmAssets;
@>          accruedLocalReserves += uint120(normalizePsmAssets(profit));
            freePsmAssets = assets;
        } else if(psmAsset != ERC20(address(0))) {
            // we do this in case the underlying asset may be a rebasing token that accrues profit
            uint bal = psmAsset.balanceOf(address(this));
            if(bal <= freePsmAssets) return; // avoids underflow in case of loss
            uint profit = bal - freePsmAssets;
            accruedLocalReserves += uint120(normalizePsmAssets(profit));
            freePsmAssets = bal;
        }
    }
```       
Therefore, the `accruePsmProfit` function does not provide the real profits because of this issue.

&nbsp;

**Mitigation**

```diff
function sell(uint coinIn, uint minAssetOut) external returns (uint assetOut) {
        require(psmAsset != ERC20(address(0)), "PSM asset was not set");
        accrueInterest();
        assetOut = getSellAmountOut(coinIn);
        require(assetOut >= minAssetOut, "insufficient amount out");
        // get and burn coins from caller
        coin.transferFrom(msg.sender, address(this), coinIn);
        coin.burn(coinIn);
        // give assets to caller
        if(psmVault != ERC4626(address(0))) {
            // Convert to shares out without taking into account fees, which will be paid by the seller by receiving less than assetOut
            uint256 sharesOut = psmVault.convertToShares(assetOut);
            uint256 actualAssetOutToSeller = psmVault.redeem(sharesOut, msg.sender, address(this));
-          freePsmAssets -= assetOut;
+          freePsmAssets -= assetOactualAssetOutToSeller;
            require(actualAssetOutToSeller >= minAssetOut, "redeem failed");
        } else {
            freePsmAssets -= assetOut;
            psmAsset.safeTransfer(msg.sender, assetOut);
        }
        emit Sold(msg.sender, coinIn, assetOut);
    }
```

## [L2] The requirement in the constructor regarding `halfLife` bypassed.


**Summary**

The requirement written in the constructor, can be bypassed using the `setHalfLife`. 


**Relevant GitHub Links**

https://github.com/sherlock-audit/2025-12-monolith-stablecoin-factory-0x4th05/blob/main/Monolith/src/Lender.sol#L917

https://github.com/sherlock-audit/2025-12-monolith-stablecoin-factory-0x4th05/blob/main/Monolith/src/Lender.sol#L127

**Root Cause**

In the constructor it is written that `halfLife >= 24 hours && params.halfLife <= 30 days`. 
However, the `Operator` and the `manager` can bypass it calling before the deadline, the function `setHalfLife` having a different constraint.

```solidity
constructor(LenderParams memory params) {
        require(params.collateralFactor <= 8500, "Invalid collateral factor");
        require(params.timeUntilImmutability < 1460 days, "Max immutability deadline is in 4 years");
@>      require(params.halfLife >= 24 hours && params.halfLife <= 30 days, "Invalid half life");
           .
           .
           .
}
```
```solidity
function setHalfLife(uint64 halfLife) external onlyOperatorOrManager beforeDeadline {
        accrueInterest();
@>      require(halfLife >= 12 hours && halfLife <= 30 days, "Invalid half life");
        expRate = uint64(uint(wadLn(2*1e18)) / halfLife);
        emit HalfLifeUpdated(halfLife);
    }
```


**Attack Path**

- The contract is deployed with `halfLife = 25 hours`
- Operator or manager call `setHalfLife` immediately after the deployment setting it to a value `>12 hours && <24 hours` bypassing therefore, the check of the constructor.

**Impact**

The bypass of the check written in the constructor regarding the `halfLife` variable has an impact on all the other functions that use the value of this variable in their execution

&nbsp;

**Mitigation**

```diff
function setHalfLife(uint64 halfLife) external onlyOperatorOrManager beforeDeadline {
        accrueInterest();
-      require(halfLife >= 12 hours && halfLife <= 30 days, "Invalid half life");
+      require(halfLife >= 24 hours && halfLife <= 30 days, "Invalid half life");
        expRate = uint64(uint(wadLn(2*1e18)) / halfLife);
        emit HalfLifeUpdated(halfLife);
    }
```