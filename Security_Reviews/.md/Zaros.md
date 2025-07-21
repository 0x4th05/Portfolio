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
  - [\[M1\] `GlobalConfigurationBranch::updatePerpMarketConfiguration` missing update of the `priceFeedHeartbeatSeconds` value](#m1-globalconfigurationbranchupdateperpmarketconfiguration-missing-update-of-the-pricefeedheartbeatseconds-value)

&nbsp;

&nbsp;


# Protocol Summary

Zaros is a Perpetuals DEX powered by Boosted (Re)Staking Vaults. It seeks to maximize LPs yield generation, while offering a top-notch trading experience on Arbitrum (and Monad in the future).

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
| Contest platform |                Codehawks                 |
| LOC              |                   2878                   |
| Language         |                 Solidity                 |
| Commit           | d687fe96bb7ace8652778797052a38763fbcbb1b |
| Previous audits  |               Cyfrin audit               |



# Scope

 - src/utils/Math.sol
 - src/utils/Constants.sol
 - src/utils/Errors.sol
 - src/tree-proxy/leaves/RootUpgrade.sol
 - src/tree-proxy/leaves/Branch.sol
 - src/tree-proxy/leaves/LookupTable.sol
 - src/tree-proxy/RootProxy.sol
 - src/tree-proxy/branches/UpgradeBranch.sol
 - src/tree-proxy/branches/LookupBranch.sol
 - src/external/chainlink/ChainlinkUtil.sol
 - src/external/chainlink/keepers/BaseKeeper.sol
 - src/external/chainlink/interfaces/ILogAutomation.sol
 - src/external/chainlink/interfaces/IStreamsLookupCompatible.sol
 - src/external/chainlink/interfaces/IAutomationCompatible.sol
 - src/external/chainlink/keepers/market-order/MarketOrderKeeper.sol
 - src/account-nft/AccountNFT.sol
 - src/external/chainlink/interfaces/IFeeManager.sol
 - src/external/chainlink/interfaces/IOffchainAggregator.sol
 - src/external/chainlink/interfaces/IVerifierProxy.sol
 - src/external/chainlink/interfaces/IAggregatorV3.sol
 - src/external/chainlink/keepers/liquidation/LiquidationKeeper.sol
 - src/usd/USDToken.sol
 - src/perpetuals/PerpsEngine.sol
 - src/account-nft/interfaces/IAccountNFT.sol
 - src/perpetuals/branches/TradingAccountBranch.sol
 - src/perpetuals/branches/LiquidationBranch.sol
 - src/perpetuals/branches/OrderBranch.sol
 - src/perpetuals/branches/PerpMarketBranch.sol
 - src/perpetuals/leaves/OrderFees.sol
 - src/perpetuals/leaves/CustomReferralConfiguration.sol
 - src/perpetuals/branches/SettlementBranch.sol
 - src/perpetuals/branches/GlobalConfigurationBranch.sol
 - src/perpetuals/leaves/MarginCollateralConfiguration.sol
 - src/perpetuals/leaves/GlobalConfiguration.sol
 - src/perpetuals/leaves/PerpMarket.sol
 - src/perpetuals/leaves/Position.sol
 - src/perpetuals/leaves/SettlementConfiguration.sol
 - src/perpetuals/leaves/MarketConfiguration.sol
 - src/perpetuals/leaves/TradingAccount.sol
 - src/perpetuals/leaves/Referral.sol
 - src/perpetuals/leaves/MarketOrder.sol
 - src/perpetuals/leaves/FeeRecipients.sol
 - src/perpetuals/leaves/OffchainOrder.sol


# Issues found

| Severity | Number of issues found |
| :------- | :--------------------: |
| High     |           0            |
| Medium   |           1            |
| Low      |           0            |
| Info     |           0            |




# Findings

## [M1] `GlobalConfigurationBranch::updatePerpMarketConfiguration` missing update of the `priceFeedHeartbeatSeconds` value 


**Summary**

In the `GlobalConfigurationBranch.sol`, calling the `updatePerpMarketConfiguration` function, the `priceFeedHeartbeatSeconds` set in the `MarketConfiguration.sol` cannot be updated as all the other parameters.

&nbsp;

**Vulnerability Details**

In the `GlobalConfigurationBranch.sol::updatePerpMarketConfiguration` the `priceFeedHeartbeatSeconds` input value does not get updated because it is not assigned to the right `uint 32 priceFeedHeartbeatSeconds` variable in the `MarketConfiguration.sol::update`.

That input it is not assigned to any variable at all.

```Solidity
    function updatePerpMarketConfiguration(
        uint128 marketId,
        UpdatePerpMarketConfigurationParams calldata params
    )
        external
        onlyOwner
        onlyWhenPerpMarketIsInitialized(marketId)
    {
        PerpMarket.Data storage perpMarket = PerpMarket.load(marketId);
        MarketConfiguration.Data storage perpMarketConfiguration = perpMarket.configuration;

        if (abi.encodePacked(params.name).length == 0) {
            revert Errors.ZeroInput("name");
        }
        if (abi.encodePacked(params.symbol).length == 0) {
            revert Errors.ZeroInput("symbol");
        }
        if (params.priceAdapter == address(0)) {
            revert Errors.ZeroInput("priceAdapter");
        }
        if (params.maintenanceMarginRateX18 == 0) {
            revert Errors.ZeroInput("maintenanceMarginRateX18");
        }
        if (params.maxOpenInterest == 0) {
            revert Errors.ZeroInput("maxOpenInterest");
        }
        if (params.maxSkew == 0) {
            revert Errors.ZeroInput("maxSkew");
        }
        if (params.initialMarginRateX18 == 0) {
            revert Errors.ZeroInput("initialMarginRateX18");
        }
        if (params.initialMarginRateX18 <= params.maintenanceMarginRateX18) {
            revert Errors.InitialMarginRateLessOrEqualThanMaintenanceMarginRate();
        }
        if (params.skewScale == 0) {
            revert Errors.ZeroInput("skewScale");
        }
        if (params.minTradeSizeX18 == 0) {
            revert Errors.ZeroInput("minTradeSizeX18");
        }
        if (params.maxFundingVelocity == 0) {
            revert Errors.ZeroInput("maxFundingVelocity");
        }
  @>    if (params.priceFeedHeartbeatSeconds == 0) {
            revert Errors.ZeroInput("priceFeedHeartbeatSeconds");
        }

  @>      perpMarketConfiguration.update(
            MarketConfiguration.Data({
                name: params.name,
                symbol: params.symbol,
                priceAdapter: params.priceAdapter,
                initialMarginRateX18: params.initialMarginRateX18,
                maintenanceMarginRateX18: params.maintenanceMarginRateX18,
                maxOpenInterest: params.maxOpenInterest,
                maxSkew: params.maxSkew,
                maxFundingVelocity: params.maxFundingVelocity,
                minTradeSizeX18: params.minTradeSizeX18,
                skewScale: params.skewScale,
                orderFees: params.orderFees,
  @>            priceFeedHeartbeatSeconds: params.priceFeedHeartbeatSeconds
            })
        );

        emit LogUpdatePerpMarketConfiguration(msg.sender, marketId);
    }

```

&nbsp;

**Impact**

The owner is not able to update the `priceFeedHeartbeatSeconds` parameter in case of high market volatility or lower available heartbeat of all the `oracles`, accepting this way outdated prices for the market

&nbsp;

**Tools Used**

Manual review

&nbsp;

**Recommendations**

Add a  line of code in the `MarketConfiguration.sol::update` to update the `priceFeedHeartbeatSeconds`

```diff
function update(Data storage self, Data memory params) internal {

self.name = params.name;

self.symbol = params.symbol;

self.priceAdapter = params.priceAdapter;

self.initialMarginRateX18 = params.initialMarginRateX18;

self.maintenanceMarginRateX18 = params.maintenanceMarginRateX18;

self.maxOpenInterest = params.maxOpenInterest;

self.maxSkew = params.maxSkew;

self.maxFundingVelocity = params.maxFundingVelocity;

self.minTradeSizeX18 = params.minTradeSizeX18;

self.skewScale = params.skewScale;

self.orderFees = params.orderFees;

+ self.priceFeedHeartbeatSeconds = params.priceFeedHeartbeatSeconds

}

```
