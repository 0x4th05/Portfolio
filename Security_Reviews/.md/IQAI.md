<!--
---
title: Security Review Report
author: 4th05
date: February 7, 2025
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
    {\Huge IQAI\par} 
    \vspace{0.5cm}
    {\Huge\itshape Security Review Report\par}
    \vfill
    {\Large \ 7 February 2025\par}
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
  - [\[I1\] Useless transaction consuming gas fee](#i1-useless-transaction-consuming-gas-fee)

&nbsp;

&nbsp;

&nbsp;

&nbsp;



# Protocol Summary

IQ AI built the Agent Tokenization Platform (ATP) to empower developers to launch tokenized AI agents that can own and manage digital and physical assets—from cryptocurrencies and DeFi strategies to robotics and IoT automation.


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
| LOC              |                   778                    |
| Language         |                 Solidity                 |
| Commit           | b16b866d4c8d3e4a69b37a02c4e396d4b294537e |
| Previous audits  |                 4naly3er                 |


# Scope

 - /src/AIToken.sol
 - /src/Agent.sol
 - /src/AgentFactory.sol
 - /src/AgentRouter.sol
 - /src/BootstrapPool.sol
 - /src/LiquidityManager.sol
 - /src/TokenGovernor.sol



# Issues found

| Severity | Number of issues found |
| :------- | :--------------------: |
| High     |           0            |
| Medium   |           0            |
| Low      |           0            |
| Info     |           1            |


# Findings

## [I1] Useless transaction consuming gas fee

**Relevant GitHub Links**

https://github.com/code-423n4/2025-01-iq-ai/blob/main/src/AgentFactory.sol#L116

**Finding description**

```solidity
@>    if (mintToDAOAmount > 0) token.safeTransfer(address(this), mintToDAOAmount);
        if (mintToAgentAmount > 0) token.safeTransfer(address(agent), mintToAgentAmount);
        token.safeTransfer(address(manager), initialLiquidity);
        manager.initializeBootstrapPool();
```
The transfer of the `mintToDAOAmount` is from the `factoryAgent` to the `factoryAgent` itself so it could be removed to use this way less gas in the `createAgent` function.