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

Before we begin, the code examples have been uploaded to the [deep dives repo](https://github.com/0xlgtm/gas-optimization-deep-dive-source-code/). Feel free to clone it if you wish to follow along with the code examples. We will be using Solidity compiler version 0.8.22 with optimizations enabled for 10,000 runs.

# Prerequisite Knowledge

## Understanding Opcodes

{% tip(header="Tip") %}
If you understand the SSTORE and SLOAD opcodes and their dynamic pricing algorithm, you can skip to the [optimization](#optimizations) section.
{% end %}

Assume that you want to deploy the following contract:

```solidity
// https://github.com/0xlgtm/gas-optimization-deep-dive-source-code/blob/main/src/Storage.sol
pragma solidity 0.8.22;

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

Compiling the `Storage` contract, with the command `forge inspect Storage deployedBytecode`, result in the following [runtime](https://ethereum.stackexchange.com/questions/32234/difference-between-bytecode-and-runtime-bytecode) bytecode:

`6080604052348015600f57600080fd5b506004361060325760003560e01c80632e64cec11460375780636057361d14604c575b600080fd5b60005460405190815260200160405180910390f35b605c6057366004605e565b600055565b005b600060208284031215606f57600080fd5b503591905056fea2646970667358221220f830518cc265c932d00bc09e305ea281ef5d24fe16cb6b04364d484451e3582164736f6c63430008160033`

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

Let's demonstrate a cold and warm access with some code examples.

```solidity
// https://github.com/0xlgtm/gas-optimization-deep-dive-source-code/blob/main/src/ColdVsWarm.sol
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
The difference cannot be exactly 100 gas as additional operations are required e.g. moving things around the stack.
{% end %}

We can generate the gas report using forge tests i.e. `forge test --match-contract ColdVsWarmTest --gas-report`. Executing this command reveals a gas cost of 2,246 and 2,353 respectively. As expected, the `getX()` function of the `ColdAndWarmAccess` contract costs 107 gas more. Now that you understand how the `SLOAD` opcode works, we can proceed to untangle the most expensive and complex opcode, the `SSTORE` opcode.

## Deciphering SSTORE

`SSTORE` is the opcode used to modify a blockchain state. Whenever a function wants to update the contract's storage, an `SSTORE` opcode is executed. For example, the `transfer()` function in ERC20 executes two `SSTORE` operations to update the balances of both the sender and the recipient. The gas cost of `SSTORE` can be derived from this [algorithm](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#a7-sstore) but for brevity, the key points are:

1) A storage update from zero to a non-zero value costs 22,100 gas*.
2) A storage update from a non-zero value to the same value costs 2,200 gas*.
3) A storage update from a non-zero value to a different non-zero value costs 5,000 gas*.
4) All subsequent writes to a slot which has been modified previously in the same transaction costs 100 gas.

<sub>* These are only applicable to the first write to a specific storage slot.</sub>

Based on the summary above, it is clear that the initial `SSTORE` operation to a specific slot, also known as a clean write, is prohibitively expensive. For example, setting the values of two different slots from zero to non-zero costs 44,200 gas.

Let's walk through some code examples to illustrate the key points described above.

```solidity
// https://github.com/0xlgtm/gas-optimization-deep-dive-source-code/blob/main/src/SstoreCost.sol
pragma solidity 0.8.22;

contract ZeroToNonZero {
    uint256 public x;

    // Costs 20k + 2.1k gas
    // 2.1k is for the cold access
    function setX() public {
        x = 1;
    }
}

contract NonZeroToSameNonZero {
    uint256 public x = 1;

    // Costs 100 + 2.1k gas
    // 2.1k is for the cold access
    function setX() public {
        x = 1;
    }
}

contract NonZeroToDiffNonZero {
    uint256 public x = 1;

    // Costs 2.9k + 2.1k gas
    // 2.1k is for the cold access
    function setX() public {
        x = 2;
    }
}

contract MultipleSstores {
    uint256 public x = 1;

    // Since the compiler's optimization is turned on,
    // it knows to skip the first 2 assignments.
    // If you want to see the unoptimized cost
    // i.e. 2.9k + 2.1k + 100 + 100
    // turn off optimizations and run the command
    // `forge debug src/SstoreCost.sol --tc "MultipleSstores" --sig "setX()"`
    function setX() public {
        x = 4;
        x = 3;
        x = 2;
    }
}
```

The code snippet above contains four contracts, each corresponding to one of the four summarized points. Each contract contains a slightly different `setX()` function, which is responsible for updating the storage variable `x`.

Similarly, we can generate a gas report for this test using `forge test --match-contract SstoreCostTest --gas-report`. Executing this command reveals a gas cost of 22,238, 2,338, 5,138, and 5,138 respectively.

As expected, setting `x` from its default value of zero to a non-zero value the most expensive. Conversely, it is the cheapest when you try to update `x` to the same value. This is considered a no-op so you only need to pay for (cold) accessing the storage slot.

Astute readers may notice that the cost for the `setX()` function of the last two scenarios is identical.This is because, with optimizations enabled, the compiler is smart enough to recognize that earlier assignments in the `MultipleSstores` contract can be disregarded. If compiler optimization is disabled in the foundry.toml file, you will able to see the additional gas costs arising from the multiple assignments.

As demonstrated in the examples above, the cost of executing the `SSTORE` opcode is exceedingly high, surpassing the costs of nearly every other opcode. This observation highlights a potential optimization opportunity: how can we minimize the use of `SSTORE`?

## Storage Packing

talk about sstore and sload, warm vs cold access, dirty vs clean writes.

https://github.com/wolflo/evm-opcodes/blob/main/gas.md


### Why it works

### Things to note


smart storage packing vs not so smart storage packing

# 



# Acknowledgements

https://snappify.com/view/f9a681c7-834c-467e-b34d-5ad443a893f2