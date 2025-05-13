<!--
---
title: Security Review Report
author: 4th05
date: January 31, 2025
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
    {\Huge D.A.A.O\par} 
    \vspace{0.5cm}
    {\Huge\itshape Security Review Report\par}
    \vfill
    {\Large \ 31 January 2025\par}
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
  - [\[M1\] All the amounts sent through the `receive()` after deadline are lost](#m1-all-the-amounts-sent-through-the-receive-after-deadline-are-lost)
  - [\[L1\] `Daoo::contribute` reverts when it could take `effectiveamount` instead](#l1-daoocontribute-reverts-when-it-could-take-effectiveamount-instead)
  - [\[L2\] Missing check in the `Daoo::extendFundraisingDeadline` would allow to brake a `require` constructor statement](#l2-missing-check-in-the-daooextendfundraisingdeadline-would-allow-to-brake-a-require-constructor-statement)
  - [\[L3\] Wrong require condition in `Daoo::refund`](#l3-wrong-require-condition-in-daoorefund)
  - [\[I1\] Could be useful to add a new `require` in `Dao::extendFundExpiry`](#i1-could-be-useful-to-add-a-new-require-in-daoextendfundexpiry)
  - [\[I2\] Remove the `secondToken` variable as not used](#i2-remove-the-secondtoken-variable-as-not-used)
  - [\[I3\] Missing check in the `Dao::addToWhitelist` may lead to misalignment with the contibution `tiers`](#i3-missing-check-in-the-daoaddtowhitelist-may-lead-to-misalignment-with-the-contibution-tiers)

# Protocol Summary

DAAO is a decentralized autonomous agentic organization protocol that enables automated fundraising and liquidity management through Uniswap V3. The protocol allows projects to raise funds through configurable fundraising rounds (both whitelisted and public), automatically creates and locks Uniswap V3 positions with the raised funds, and implements a sophisticated treasury management system with built-in tax mechanisms and fee collection from liquidity positions. The smart contract architecture ensures secure fund management while providing flexibility for DAO governance and automated market making.

# Disclaimer
A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

For private audits or security consulting, please reach out to me on Twitter [@0x4th05](https://x.com/0x4th05).

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
| LOC              |                   517                    |
| Language         |                 Solidity                 |
| Commit           | 85b759da2eda23a71032c3dac023a73e63afa167 |
| Previous audits  |               LightChaser                |


# Scope

- CLPoolRouter.sol
- Daao.sol
- DaaoToken.sol
- interface.sol

# Issues found

| Severity | Number of issues found |
| :------- | :--------------------: |
| High     |           0            |
| Medium   |           1            |
| Low      |           3            |
| Info     |           3            |

# Findings

## [M1] All the amounts sent through the `receive()` after deadline are lost

**Summary**
All the amounts sent through the `receive()` after deadline are lost.

**Finding Description**
All the contributions sent to the contract after the `fundraisingDeadline` do not contribute to `totalRaised` and are not even sent back to users. 

**Impact Explanation**
Loss of funds for the user

**Likelihood Explanation**
Medium considering that `receive()` has been designed as a method through which contribute to the fundraising for the user and that (because of its simplicity), user may want to use it also if they are close to the deadline without running risks to loose their funds.

**Recommendation**
```diff
-    receive() external payable {
+    receive() external payable nonReentrant {
        if (!goalReached && block.timestamp < fundraisingDeadline) {
            contribute();
        } 
+       else {
+        payable(msg.sender).transfer(msg.value);
+       }
    }
```


## [L1] `Daoo::contribute` reverts when it could take `effectiveamount` instead

**Summary**

`Daoo::contribute` reverts even if it could have taken an `effectiveamount` increasing the `totalRaised`.

**Finding Description**

Within `Daoo::contribute` there are `require` statements related to the total contribution amount of the `msg.sender`. 

```solidity
function contribute() public payable nonReentrant {
        require(!goalReached, "Goal already reached");
        require(block.timestamp < fundraisingDeadline, "Deadline hit");
        require(msg.value > 0, "Contribution must be greater than 0");

        // Must be whitelisted
        WhitelistTier userTier = userTiers[msg.sender];
        require(userTier != WhitelistTier.None, "Not whitelisted");
        // Contribution must boolow teir limit
        uint256 userLimit = tierLimits[userTier];
    @>    require(
            contributions[msg.sender] + msg.value <= userLimit,
            "Exceeding tier limit"
        );

        if (maxWhitelistAmount > 0) {
    @>    require(
                contributions[msg.sender] + msg.value <= maxWhitelistAmount,
                "Exceeding maxWhitelistAmount"
            );
        } else if (maxPublicContributionAmount > 0) {
    @>    require(
                  contributions[msg.sender] + msg.value <=
                    maxPublicContributionAmount,
                "Exceeding maxPublicContributionAmount"
            );
        }
```

All these `require` statements take into account the `msg.value` data of the transaction.
However after these statements it is given the possibility to the user to contribute with just a fraction of the `msg.value` increasing this way `totalRaised` by less then `msg.value` amount. 
It could be that all these `require` statements would have been satisfied with the `effectiveamount` instead of considering the whole `msg.value`.
Placeing all these `require` statements before knowing the final value of `effectiveamount` prevent the contract to get additional funds satisfying all the require.


```solidity 
        uint256 effectiveContribution = msg.value;
        if (totalRaised + msg.value > fundraisingGoal) {
            effectiveContribution = fundraisingGoal - totalRaised;
            payable(msg.sender).transfer(msg.value - effectiveContribution);
        }

        if (contributions[msg.sender] == 0) {
            contributors.push(msg.sender);
        }

        contributions[msg.sender] += effectiveContribution;
        totalRaised += effectiveContribution;

        if (totalRaised == fundraisingGoal) {
            goalReached = true;
        }

        emit Contribution(msg.sender, effectiveContribution);
    }
```

**Impact Explanation**

A transaction that could be finalized increasing the `totalRaised` will be reverted with a loss of funds for the contract that is equal to the `effectiveamount`.

**Likelihood Explanation**

Medium as users may do not know which are their current contribution limits, considering they could potentially be changed at any time during fundraising.

**Recommendation**

```diff
function contribute() public payable nonReentrant {
        require(!goalReached, "Goal already reached");
        require(block.timestamp < fundraisingDeadline, "Deadline hit");
        require(msg.value > 0, "Contribution must be greater than 0");

        // Must be whitelisted
        WhitelistTier userTier = userTiers[msg.sender];
        require(userTier != WhitelistTier.None, "Not whitelisted");
        // Contribution must boolow teir limit
        uint256 userLimit = tierLimits[userTier];
-        require(
-            contributions[msg.sender] + msg.value <= userLimit,
-            "Exceeding tier limit"
-        );

-        if (maxWhitelistAmount > 0) {
-        require(
-                contributions[msg.sender] + msg.value <= maxWhitelistAmount,
-                "Exceeding maxWhitelistAmount"
            );
-        } else if (maxPublicContributionAmount > 0) {
-        require(
-                  contributions[msg.sender] + msg.value <=
-                    maxPublicContributionAmount,
-                "Exceeding maxPublicContributionAmount"
-            );
-        }

        uint256 effectiveContribution = msg.value;
        if (totalRaised + msg.value > fundraisingGoal) {
            effectiveContribution = fundraisingGoal - totalRaised;
+        require(
+            contributions[msg.sender] + effectiveContribution <= userLimit,
+            "Exceeding tier limit"
+        );

+        if (maxWhitelistAmount > 0) {
+        require(
+                contributions[msg.sender] + effectiveContribution <= maxWhitelistAmount,
+                "Exceeding maxWhitelistAmount"
            );
+        } else if (maxPublicContributionAmount > 0) {
+        require(
+                  contributions[msg.sender] + effectiveContribution <=
+                    maxPublicContributionAmount,
+                "Exceeding maxPublicContributionAmount"
+            );
+        }
            payable(msg.sender).transfer(msg.value - effectiveContribution);
        }
+       else {
+        require(
+            contributions[msg.sender] + msg.value <= userLimit,
+            "Exceeding tier limit"
+        );

+        if (maxWhitelistAmount > 0) {
+        require(
+                contributions[msg.sender] + msg.value <= maxWhitelistAmount,
+                "Exceeding maxWhitelistAmount"
+            );
+        } else if (maxPublicContributionAmount > 0) {
+        require(
+                  contributions[msg.sender] + msg.value <=
+                    maxPublicContributionAmount,
+                "Exceeding maxPublicContributionAmount"
+            );
+        }
+       }

        if (contributions[msg.sender] == 0) {
            contributors.push(msg.sender);
        }

        contributions[msg.sender] += effectiveContribution;
        totalRaised += effectiveContribution;

        if (totalRaised == fundraisingGoal) {
            goalReached = true;
        }

        emit Contribution(msg.sender, effectiveContribution);
    }
```




## [L2] Missing check in the `Daoo::extendFundraisingDeadline` would allow to brake a `require` constructor statement

**Summary**

In the `Daoo::extendFundraisingDeadline` the `owner/protocolAdmin` can set a new deadline `>fundExpiry` which would brake the protocol core functionality.

```solidity
constructor(
        uint256 _fundraisingGoal,
        string memory _name,
        string memory _symbol,
        uint256 _fundraisingDeadline,
        uint256 _fundExpiry,
        address _daoManager,
        address _liquidityLockerFactory,
        uint256 _maxWhitelistAmount,
        address _protocolAdmin,
        uint256 _maxPublicContributionAmount
    ) Ownable(_daoManager) {
        require(
            _fundraisingGoal > 0,
            "Fundraising goal must be greater than 0"
        );
        require(
            _fundraisingDeadline > block.timestamp,
            "_fundraisingDeadline > block.timestamp"
        );
@>        require(
            _fundExpiry > fundraisingDeadline,
            "_fundExpiry > fundraisingDeadline"
        );
        name = _name;
        symbol = _symbol;
        fundraisingGoal = _fundraisingGoal;
        fundraisingDeadline = _fundraisingDeadline;
        fundExpiry = _fundExpiry;
        liquidityLockerFactory = ILockerFactory(_liquidityLockerFactory);
        maxWhitelistAmount = _maxWhitelistAmount;
        protocolAdmin = _protocolAdmin;
        maxPublicContributionAmount = _maxPublicContributionAmount;

        // Teir allocation
        tierLimits[WhitelistTier.Platinum] = PLATINUM_DEFAULT_LIMIT;
        tierLimits[WhitelistTier.Gold] = GOLD_DEFAULT_LIMIT;
        tierLimits[WhitelistTier.Silver] = SILVER_DEFAULT_LIMIT;
    }
```

**Finding Description**

In `Daoo::extendFundraisingDeadline` there is not `require` for `newFundraisingDeadline<fundExpiry`.

**Impact Explanation**

Would brake core functionality related to the deployment done in `Daoo::finalizeFundraising`.
```solidity
address lockerAddress = liquidityLockerFactory.deploy(
            address(POSITION_MANAGER),
            owner(),
@>          uint64(fundExpiry),
            tokenId,
            lpFeesCut,
            address(this)
        );
```

**Likelihood Explanation**

Depends on the time window between `fundExpiry` value and the `fundraisingDeadline`. However, `owner && protocolAdmin` roles are trusted.

**Recommendation**

```diff
function extendFundraisingDeadline(
        uint256 newFundraisingDeadline
    ) external {
        require(
            msg.sender == owner() || msg.sender == protocolAdmin,
            "Must be owner or protocolAdmin"
        );
        require(!goalReached, "Fundraising goal was reached");
        require(
            newFundraisingDeadline > fundraisingDeadline,
            "new fundraising deadline must be > old one"
        );
+       require(newFundraisingDeadline<fundExpiry "newFundraisingDeadline<fundExpiry");
        fundraisingDeadline = newFundraisingDeadline;
    }
```


## [L3] Wrong require condition in `Daoo::refund`

**Summary**

The `Daoo::refund` function does not allow refund when `block.timestamp == fundraisingDeadline`.

**Finding Description**

In the `Daoo::refund` the requirement based on the `block.timestamp` does not allow refunds when `block.timestamp == fundraisingDeadline`.

```solidity
function refund() external nonReentrant {
        require(!goalReached, "Fundraising goal was reached");
@>      require(
        block.timestamp > fundraisingDeadline,
            "Deadline not reached yet"
        );
        require(contributions[msg.sender] > 0, "No contributions to refund");

        uint256 contributedAmount = contributions[msg.sender];
        contributions[msg.sender] = 0;

        payable(msg.sender).transfer(contributedAmount);

        emit Refund(msg.sender, contributedAmount);
    }
```
However to be coherent with the `Daoo::contribute` and `Dao::receive` it should be `require(block.timestamp >= fundraisingDeadline, "Deadline not reached yet");`. 

```solidity
function contribute() public payable nonReentrant {
        require(!goalReached, "Goal already reached");
@>      require(block.timestamp < fundraisingDeadline, "Deadline hit");
        require(msg.value > 0, "Contribution must be greater than 0");
```

```solidity
receive() external payable {
@>      if (!goalReached && block.timestamp < fundraisingDeadline) {
            contribute();
        }
    }
```

**Impact Explanation**

When `block.timestamp==fundraisingDeadline` users cannot do anything neither contributing nor refunding their ETH.

**Likelihood Explanation**

The likelihood is very low as `block.timestamp==fundraisingDeadline` lasts 1 second.

**Recommendation**


```diff
function refund() external nonReentrant {
        require(!goalReached, "Fundraising goal was reached");
-        require(
-        block.timestamp > fundraisingDeadline,
-            "Deadline not reached yet"
-        )
+       require(
+       block.timestamp >= fundraisingDeadline,
+           "Deadline not reached yet"
+        );
        require(contributions[msg.sender] > 0, "No contributions to refund");

        uint256 contributedAmount = contributions[msg.sender];
        contributions[msg.sender] = 0;

        payable(msg.sender).transfer(contributedAmount);

        emit Refund(msg.sender, contributedAmount);
    }
```



## [I1] Could be useful to add a new `require` in `Dao::extendFundExpiry`

**Summary**

In case of `!fundraisingFinalized` the `Dao::extendFundExpiry` will revert 

**Finding Description**

If the `owner` calls the `Dao::extendFundExpiry` when `!fundraisingFinalized` 
the function will revert because `liquidityLocker=address(0)`. 

```diff
function extendFundExpiry(uint256 newFundExpiry) external onlyOwner {
        require(newFundExpiry > fundExpiry, "Must choose later fund expiry");    
        fundExpiry = newFundExpiry;
@>      ILocker(liquidityLocker).extendFundExpiry(newFundExpiry);
    }
```

Indeed the variable `liquidityLocker` is changed by its default value only if `Daoo::finalizeFundraising` is called. 

```solidity
function finalizeFundraising(int24 initialTick, int24 upperTick) external { 

     .
     .
     .

    emit TokenTransferredToLocker(tokenId, lockerAddress);

        // Initialize the locker
        ILocker(lockerAddress).initializer(tokenId);
        emit LockerInitialized(tokenId);

@>      liquidityLocker = lockerAddress;
        emit DebugLog("Finalize fundraising complete");
    }
```

**Recommendation**

```diff
function extendFundExpiry(uint256 newFundExpiry) external onlyOwner {
        require(newFundExpiry > fundExpiry, "Must choose later fund expiry");
+       require(fundraisingFinalized, "!fundraisingFinalized`);       
        fundExpiry = newFundExpiry;
        ILocker(liquidityLocker).extendFundExpiry(newFundExpiry);
    }
```


## [I2] Remove the `secondToken` variable as not used 

**Recommendation**

Remove the `secondToken` variable as it is not used at all but only set.

```solidity
    address public secondToken;
```
```solidity
function setSecondToken(address _daoToken) external onlyOwner {
        require(_daoToken != address(0), "Invalid second token address");
        require(secondToken == address(0), "DAO token already set");
        secondToken = _daoToken;
    }
```


## [I3] Missing check in the `Dao::addToWhitelist` may lead to misalignment with the contibution `tiers`

**Summary**

Calling `Dao::addToWhitelist` could create a misalignment between `userLimits` and `tierslimits` in terms of any contribution already done.

**Finding Description**

Using `Dao::addToWhitelist` the trusted role may overwrite a whitelisted user and lead to the inconsistency `contributions[msg.sender] >= userLimit`.

**Impact Explanation**

Inconsistency between `max contribution tiers` and all the contributions done by single `whitelisted` users

**Likelihood Explanation**

Medium 

**Recommendation**

```diff
for (uint256 i = 0; i < _addresses.length; i++) {
            require(_addresses[i] != address(0), "Invalid address");
            require(_tiers[i] != WhitelistTier.None, "Invalid tier");
+            uint256 userLimit = tierLimits[_tiers[i] ];            
+            require(contributions[msg.sender] <= userLimit, "Invalid tier");
            
            userTiers[_addresses[i]] = _tiers[i];
```
