---
layout:             post
title:              "SEED Labs 2.0: Smart Contract Reentrancy Attack Lab Writeup"
category:           "Computing Systems"
tags:               ethereum blockchain
permalink:          /smart-contract-seedlab/
---

The DAO was a decentralized autonomous organization (DAO) that was launched in 2016 on the Ethereum blockchain. After raising $$\$150$$ million USD worth of ether (ETH) through a token sale, The DAO was hacked due to vulnerabilities in its code base. The Ethereum blockchain was eventually hard forked to restore the stolen funds, but not all parties agreed with this decision, which resulted in the network splitting into two distinct blockchains: Ethereum and Ethereum Classic.

Reentrancy played a major role in the attack. A reentrancy attack occurs when a malicious contract calls back into a vulnerable contract before the original function invocation is complete. The EVM doesn't permit concurrency, meaning two contracts involved in a message call cannot run simultaneously. An external call pauses the calling contract's execution and memory until the call returns, at which point execution proceeds normally. Consider a simple smart contract ("Victim") that allows anyone to deposit and withdraw ETH:

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

This contract exposes a `withdraw()` function to allow users to withdraw ETH previously deposited in the contract. When processing a withdrawal, the contract performs the following operations:
- Checks the user's ETH balance;
- Sends funds to the calling address;
- Resets their balance to $$0$$, preventing additional withdrawals from the user.

If `withdraw()` is called from an externally owned account (EOA), the function executes as expected. However, if `msg.sender` is a smart contract account that calls `withdraw()`, sending funds using `msg.sender.call.value()` will also trigger code stored at that address to run. Because the caller's balance isn't set to $$0$$ until the function execution completes, subsequent invocations will succeed and allow the caller to withdraw their balance multiple times. Reentrancy attacks are still a critical issue for smart contracts on Ethereum today.

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Environment Setup

![Choose one of the three providers to link a Web3.py instance with an Etheurem node.](/assets/images/web3py-and-nodes.png)

"Ethereum node" and "Ethereum client" are used interchangeably. In either case, it refers to the software that a participant in the Ethereum network runs. This software can read block data, receive updates when new blocks are added to the chain, broadcast new transactions, and more. Technically, the client is the software, the node is the computer running the software. Ethereum clients can be configured to be reachable by IPC (uses local filesystem: fastest and most secure), HTTP (supported by more nodes), or Websockets (works remotely, faster than HTTP), so Web3.py will need to mirror this configuration.