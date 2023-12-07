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

My main criticism of these articles lies in their inclination towards breadth rather than depth. Specifically, they fall short in adequately explaining the technical details behind the optimizations discussed. Consequently, readers often resort to memorization rather than acquiring the ability to discern potential optimization opportunities.

Furthermore, some of these techniques may be compiler-version-dependent and typically yield minimal gas savings. Therefore, it is logical to prioritize the most impactful optimizations and gain a thorough understanding of how and why they work.

We will begin by covering the foundational knowledge necessary to grasp the techniques outlined later in this article. These techniques share a common underlying concept, making a solid groundwork crucial for a better understanding and appreciation of these strategies. Additionally, we will briefly touch upon some considerations to keep in mind when applying these optimizations.

# Prerequisite Knowledge

{% tip(header="Tip") %}
If you understand the SSTORE and SLOAD opcodes and their dynamic pricing algorithm, you can skip ahead to the next section.
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

To the untrained eye, this paragraph might appear nonsensical. However, it contains the code for the aforementioned contract. Each pair of hexadecimal characters constitutes one byte, and each byte corresponds to an operation code, or opcode for short. For example, `60` stands for the `PUSH1` opcode and `80` is the input for `PUSH1`.

Opcodes are the basic instructions executed by the Ethereum Virtual Machine (EVM) and each opcode has a gas cost associated with it. In this article, we will focus on strategies that minimizes the use of two of the most expensive opcodes, namely `SSTORE` and `SLOAD`.

## SLOAD 101

Given the index of a specific slot in a contract's storage, the `SLOAD` opcode is used to retrieve the 256-bit word located at that index. For example, the ERC20 `balanceOf()` function executes one `SLOAD` operation to retrieve a user's balance. Unlike other opcodes with a fixed gas price, the `SLOAD` opcode has a dynamic pricing as follows:

- 2,100 gas for a cold access
- 100 gas for a warm access

Notably, a cold `SLOAD` is 20 times more costly than a warm `SLOAD`. To leverage on this difference, we must first understand the distinction between a cold and a warm access.

### Cold vs. Warm Access

Before a transaction execution begins, an empty set called the `accessed_storage_keys` is initialized. Whenever a storage slot of a contract is accessed, the `(address, storage_key)` pair is first check against the `accessed_storage_keys` set. If it is not already present in the set, it is classified as a cold access. Conversely, if it is already part of the set, it is categorized as a warm access.

Let's demonstrate this with [two simple contract](https://github.com/0xlgtm/gas-optimization-deep-dive-source-code/blob/main/src/ColdVsWarm.sol).

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

Both contracts are very similar except the `getX()` function in the `ColdAndWarmAccess` contract makes two `SLOAD` calls to storage slot 0. Therefore, the gas costs between the two `getX()` functions should differ by at least 100 gas.

{% tip(header="Tip") %}
The difference in gas costs cannot be exactly 100 because there may be some miscellaneous operations required to rearrange the stack. This is dependent on the compiler version and whether or not optimizations are turned on.
{% end %}

Generating the gas report for the two `getX()` function reveals a gas cost of 2246 and 2353 respectively. As expected, the `ColdAndWarmAccess:getX()` function costs 107 gas more. Now that you have a good understanding of how the `SLOAD` opcode works, we can now proceed to untangle the intricacies of the most expensive and complex opcode, the `SSTORE` opcode.



## Storage Packing

talk about sstore and sload, warm vs cold access, dirty vs clean writes.

https://github.com/wolflo/evm-opcodes/blob/main/gas.md


### Why it works

### Things to note


smart storage packing vs not so smart storage packing

# 



# Acknowledgements

https://snappify.com/view/f9a681c7-834c-467e-b34d-5ad443a893f2