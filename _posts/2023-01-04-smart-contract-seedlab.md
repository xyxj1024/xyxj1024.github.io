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

If `withdraw()` is called from an externally owned account (EOA), the function executes as expected. However, if `msg.sender` is a smart contract account that calls `withdraw()`, sending funds using `msg.sender.call.value()` will also trigger code stored at that address to run. Because the caller's balance isn't set to $$0$$ until the function execution completes, subsequent invocations will succeed and allow the caller to withdraw their balance multiple times with a fallback function[^1]:

```solidity
function () payable {
    if (Victim.balance > 1 ether) {
        Victim.withdraw(1 ether);
    }
}
```

Reentrancy attacks are still a critical issue for smart contracts on Ethereum today.

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Environment Setup

### Ethereum Nodes and Accounts

![web3-providers](/assets/images/web3py-and-nodes.png)
<p style="color:gray; font-size:80%;">
图片来源：ethereum.org
</p>

"Ethereum node" and "Ethereum client" are sometimes used interchangeably. In either case, it refers to the software that a participant in the Ethereum network runs. This software can read block data, receive updates when new blocks are added to the chain, broadcast new transactions, and more. Technically, the client is the software, the node is the computer running the software. Ethereum clients can be configured to be reachable by IPC (uses local filesystem: fastest and most secure), HTTP (supported by more nodes), or Websockets (works remotely, faster than HTTP), so Web3.py will need to mirror this configuration.

The SEED Internet Emulator enables the HTTP server on four Ethereum nodes that can only be accessed using a port on the SEED VM:

```text
localhost:8545 --> <IP 1>:8545 : Attacker (eth0)
localhost:8546 --> <IP 2>:8545 : Victim   (eth1)
localhost:8547 --> <IP 3>:8545 : User 1   (eth2)
localhost:8548 --> <IP 4>:8545 : User 2   (eth3)
```

Geth (go-ethereum) is an Ethereum *execution client* meaning it handles transactions, deployment and execution of smart contracts and contains the EVM. Running Geth alongside a consensus client turns a computer into an Ethereum node. By default, Geth accepts connections from the local loopback interface (`127.0.0.1`) using the port `8545`. Geth supports **Clique**, a Proof-of-Authority (PoA) consensus mechanism. The following code snippet shows how to connect to the Victim node:

```python
port = 8546 # the Victim node
web3 = SEEDWeb3.connect_to_geth_poa('http://127.0.0.1:{}'.format(port))
```

The `connect_to_geth_poa()` method in the second line is a wrapper function included in `SEEDWeb3.py`:

```python
# Connect to a geth node
def connect_to_geth_poa(url):
    web3 = Web3(Web3.HTTPProvider(url))
    if not web3.isConnected():
        sys.exit("Connection failed!") 
    web3.middleware_onion.inject(geth_poa_middleware, layer=0)
    return web3
```

where

```python
# /web3/types.py
Middleware = Callable[[Callable[[RPCEndpoint, Any], RPCResponse], "Web3"], Any]
MiddlewareOnion = NamedElementOnion[str, Middleware]

# /web3/datastructures.py
class NamedElementOnion(Mapping[TKey, TValue]):
    """
    Add layers to an onion-shaped structure. Optionally, inject to a specific layer.
    This structure is iterable, where the outermost layer is first, and innermost
    is last.
    """
    
    ...

    def inject(
        self, element: TValue, name: Optional[TKey] = None, layer: Optional[int] = None
    ) -> None:
        """
        Inject a named element to an arbitrary layer in the onion.
        The current implementation only supports insertion at the innermost layer,
        or at the outermost layer. Note that inserting to the outermost is equivalent
        to calling :meth:`add` .
        """
        if not is_integer(layer):
            raise TypeError("The layer for insertion must be an int.")
        elif layer != 0 and layer != len(self._queue):
            raise NotImplementedError(
                f"You can only insert to the beginning or end of a {type(self)}, "
                f"currently. You tried to insert to {layer}, but only 0 and "
                f"{len(self._queue)} are permitted. "
            )

        self.add(element, name=name)

        if layer == 0:
            if name is None:
                name = cast(TKey, element)
            self._queue.move_to_end(name, last=False)
        elif layer == len(self._queue):
            return
        else:
            raise AssertionError(
                "Impossible to reach: earlier validation raises an error"
            )

    ...
```

and

```python
# web3/middleware/geth_poa.py
geth_poa_middleware = construct_formatting_middleware(
    result_formatters={
        RPCEndpoint("eth_getBlockByHash"): geth_poa_cleanup,
        RPCEndpoint("eth_getBlockByNumber"): geth_poa_cleanup,
    },
)
```

All the accounts in the emulator are encrypted, and the password is `admin`. After connecting to an Ethereum node, we can access all its accounts via the `web3.eth.accounts[]` array:

```python
sender_account = web3.eth.account[1] # use the account indexed 1
web3.geth.personal.unlockAccount(sender_account, "admin")
```

and get the balance of an account:

```python
web3.eth.get_balance(Web3.toChecksumAddress(address))
```

## Task 1: Getting Familiar with the Victim Smart Contract

## Task 2: The Attacking Contract

## Task 3: Launching the Reentrancy Attack

## Task 4: Countermeasures

## Notes

[^1]: Andreas M. Antonopoulos and Gavin Wood, "Mastering Ethereum," 2018, [https://github.com/ethereumbook/ethereumbook](https://github.com/ethereumbook/ethereumbook). See also [Smart Contract Security](https://ethereum.org/en/developers/docs/smart-contracts/security/) from Ethereum official documents.