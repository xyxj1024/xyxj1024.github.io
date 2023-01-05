---
layout:             post
title:              "SEED Labs 2.0: Smart Contract Reentrancy Attack Lab Writeup"
category:           "Computing Systems"
tags:               ethereum blockchain
permalink:          /smart-contract-seedlab/
---

The DAO was a decentralized autonomous organization (DAO) that was launched in 2016 on the Ethereum blockchain. After raising $$\$150$$ million USD worth of ether (ETH) through a token sale, The DAO was hacked due to vulnerabilities in its code base. The Ethereum blockchain was eventually hard forked to restore the stolen funds, but not all parties agreed with this decision, which resulted in the network splitting into two distinct blockchains: Ethereum and Ethereum Classic.

Reentrancy played a major role in the attack. A reentrancy attack occurs when a malicious contract calls back into a vulnerable contract before the original function invocation is complete. The EVM doesn't permit concurrency, meaning two contracts involved in a message call cannot run simultaneously. An external call pauses the calling contract's execution and memory until the call returns, at which point execution proceeds normally. Consider a simple smart contract ("Victim") that allows anyone to deposit and withdraw ether:

```solidity
contract Victim {
    mapping (address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 amount = balances[msg.sender];
        (bool success, ) = msg.sender.call.value(amount)("");
        require(success);
        balances[msg.sender] = 0;
    }
}
```

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}
