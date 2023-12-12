+++
title = "Solidity Gas Optimization Deep Dive"
date = 2023-12-04
draft = true

[taxonomies]
categories = ["solidity"]
tags = ["gas optimization"]

[extra]
lang = "en"
toc = true
comment = true
copy = false
math = false
mermaid = false
outdate_alert = false
outdate_alert_days = 120
display_tags = true
truncate_summary = true
+++

Before we begin, let's talk about the elephant in the room — why write another gas optimization article when there are already [so](https://www.rareskills.io/post/gas-optimization) [many](https://www.alchemy.com/overviews/solidity-gas-optimization) [resources](https://coinsbench.com/comprehensive-guide-tips-and-tricks-for-gas-optimization-in-solidity-5380db734404) [out](https://betterprogramming.pub/solidity-gas-optimizations-and-tricks-2bcee0f9f1f2) [there](https://0xmacro.com/blog/solidity-gas-optimizations-cheat-sheet/)?

My main criticism of these articles lies in their inclination towards breadth rather than depth. Specifically, they fall short in adequately explaining the technical details behind the optimizations discussed. Consequently, readers often resort to memorization and pattern matching instead of learning how to discern potential optimization opportunities. In this article, we aim to address this gap by thoroughly exploring the most impactful optimizations, unraveling their intricacies to understand how and why they work.

We will begin by covering the foundational knowledge necessary to grasp the techniques outlined later in this article. These techniques share a common underlying concept, making a solid groundwork crucial for a better understanding and appreciation of these strategies. Additionally, we will briefly touch upon some considerations to keep in mind when applying these optimizations.

# Prerequisite Knowledge

## Understanding Opcodes

{% tip(header="Tip") %}
If you understand the SSTORE and SLOAD opcodes and their dynamic pricing algorithm, you can skip to the [optimization](#optimizations) section.
{% end %}

Assume that you want to deploy the following contract:

```solidity
// SPDX-License-Identifier: GPL-3.0
// Source: https://remix.ethereum.org/
pragma solidity >=0.8.2 <0.9.0;

contract Storage {

    uint256 number;

    function store(uint256 num) public {
        number = num;
    }

    function retrieve() public view returns (uint256){
        return number;
    }
}
```

Compiling the `Storage` contract will result in the following [runtime](https://ethereum.stackexchange.com/questions/32234/difference-between-bytecode-and-runtime-bytecode) bytecode:

`6080604052348015600e575f80fd5b50600436106030575f3560e01c80632e64cec11460345780636057361d146048575b5f80fd5b5f5460405190815260200160405180910390f35b605760533660046059565b5f55565b005b5f602082840312156068575f80fd5b503591905056fea264697066735822122078bcdcb3640a135d107fd506fad0323eea506128357e97975d901550f030118e64736f6c63430008140033`

To the untrained eye, this paragraph might appear nonsensical. However, it contains the code for the `Storage` contract. Each pair of hexadecimal characters constitutes one byte, and each byte corresponds to an operation code, or opcode for short. For example, `60` stands for the `PUSH1` opcode and `80` is the input for `PUSH1`.

Opcodes are the basic instructions executed by the Ethereum Virtual Machine (EVM) and each opcode has a gas cost associated with it. In this article, we will focus on strategies that minimizes the use of two of the most expensive opcodes, namely `SSTORE` and `SLOAD`.

## Deconstructing SLOAD

### What is the SLOAD opcode?

Given the index to some position in a contract's storage, the `SLOAD` opcode is used to retrieve the 256-bit (32 bytes) word located at that slot. For example, the ERC20 `balanceOf()` function executes one `SLOAD` operation to retrieve a user's balance. Unlike other opcodes with a fixed gas price, the `SLOAD` opcode has a (simple) dynamic pricing model as follows:

- 2,100 gas for a cold access
- 100 gas for a warm access

Notably, a cold `SLOAD` is 20 times more costly than a warm `SLOAD`. To take advantage of this, we must first understand the distinction between a cold and a warm access.

### Cold Access vs. Warm Access

Before a transaction execution begins, an empty set called the `accessed_storage_keys` is initialized. Whenever a storage slot of a contract is accessed, the `(address, storage_key)` pair is first check against the `accessed_storage_keys` set. If it is present in the set, it is classified as a warm access. Conversely, if it is not present, it is categorized as a cold access.

Let's demonstrate these two types of access with [some examples](https://github.com/0xlgtm/gas-optimization-deep-dive-source-code/blob/main/src/ColdVsWarm.sol).

```solidity
// Compiler version 0.8.22, optimizer on with 10000 runs
// SPDX-License-Identifier: MIT
pragma solidity 0.8.22;

contract ColdAccess {
    uint256 x = 1;

    function getX() public view returns (uint256 a) {
        a = x;
    }
}

contract ColdAndWarmAccess {
    uint256 x = 1;

    function getX() public view returns (uint256 a) {
        a = x;
        a = a + x;
    }
}
```

The code snippet above contains two contracts, namely `ColdAccess` and `ColdAndWarmAccess`. The main difference lies in the `getX()` function of the `ColdAndWarmAccess` contract, which executes two `SLOAD` calls to storage slot 0. The first `SLOAD` is considered a cold access and thus costs 2,100 gas. The second `SLOAD` is a warm access and costs 100 gas therefore, the difference in gas costs between the two `getX()` functions should be at least 100 gas.

{% tip(header="Tip") %}
The difference cannot be exactly 100 gas as additional operations are required e.g. stack management.
{% end %}

We can generate the gas report using forge tests i.e. `forge test --match-contract ColdVsWarmTest --gas-report`. Executing this command reveals a gas cost of 2,246 and 2,353 respectively. As expected, the `getX()` function of the `ColdAndWarmAccess` contract costs 107 gas more. Now that you understand how the `SLOAD` opcode works, we can proceed to untangle the most expensive and complex opcode, the `SSTORE` opcode.

## SSTORE 101

`SSTORE` is the opcode used to modify a blockchain state. Whenever a function wants to update the contract's storage, an `SSTORE` opcode is executed. For example, the `transfer()` function in ERC20 executes two `SSTORE` operations to update the balances of both the sender and the recipient. Determining the gas cost of `SSTORE` involves a complex [algorithm](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#a7-sstore) so instead, here's a summary of the key points:

1) An `SSTORE` operation is always preceeded by an `SLOAD` because you need to access a slot when you write to it. This implies that we must add the gas costs from the `SLOAD` operation on top of the cost of `SSTORE`.
2) A storage update from zero to non-zero costs 20k gas.
3) A storage update from non-zero to a different non-zero costs 2.9k gas.
4) A storage update from non-zero to the same non-zero costs 100 gas.
5) If the slot has been modified previously in the same transaction i.e. a dirty write, all subsequent `SSTORE` to the same slot will only cost 100 gas.
6) A storage update from non-zero to zero will also refund the user with some gas*.

<sub>*The gas refund amount depends on a few factors, which we will address in a separate article.</sub>

From the summary above, it is evident that the very first update to storage, also known as a clean write, is prohibitively expensive. The cost for executing `SSTORE` far surpasses the costs associated with almost every other opcode. This insight suggests a potential optimization: how can we minimize the use of `SSTORE`s?

Let's walk through another contract to illustrate the gas costs described above.



## Storage Packing

talk about sstore and sload, warm vs cold access, dirty vs clean writes.

https://github.com/wolflo/evm-opcodes/blob/main/gas.md


### Why it works

### Things to note


smart storage packing vs not so smart storage packing

# 



# Acknowledgements

https://snappify.com/view/f9a681c7-834c-467e-b34d-5ad443a893f2