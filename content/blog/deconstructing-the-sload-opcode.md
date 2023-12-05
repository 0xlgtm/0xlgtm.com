+++
title = "Deconstructing the SLOAD opcode"
date = 2023-09-20
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
outdate_alert = true
outdate_alert_days = 120
display_tags = true
truncate_summary = true
+++

Gas auditors should possess a deep understanding of opcode pricing, particularly when it comes to opcodes with dynamic gas costs like SLOAD and SSTORE. Understanding how these opcodes work is pivotal for optimizing a contract's gas costs. In this article, we will explore the intricacies of the SLOAD opcode, shedding light on its operation and gas cost dynamics.

## What is SLOAD?

SLOAD is an Ethereum opcode that allows smart contracts to read data from the blockchain's storage. Given an index as an argument, SLOAD will access the specified storage slot and load the corresponding 256-bit value from storage onto the stack. The Berlin hardfork in 2021 introduced [EIP-2929](https://eips.ethereum.org/EIPS/eip-2929) which repriced SLOAD as follows:

- 2,100 gas for a cold access
- 100 gas for a warm access

A cold access occurs when a specific key at a particular address is accessed for the first time during the execution of a transaction. This `(address, key)` pair is then added to the `touched_storage_slot` list. Subsequent accesses to the same pair are categorized as warm accesses. Let's walk through a simple example to explain these concepts.

## Warm vs. Cold Access, Optimization Off

```solidity
// Solidity code with optimizer off
pragma solidity 0.8.19;

contract StorageExample {
    uint256 x = 0;
    uint256 y = 1;

    function coldAccess() public view returns (uint256 a) {
        a = x;
    }

    function coldAndWarmAccess() public view returns (uint256 a) {
        a = y;
        a = y;
    }
}
```

*Before we continue, take a look at the contract above and think which function is more costly of the two and why.*

With optimizations turned off, we can observe that the `coldAccess()` function costs 2,415 gas, while the `coldAndWarmAccess()` function costs 2,545 gas. This cost difference aligns with our expectations, as the `coldAndWarmAccess()` function accesses the `y` variable twice i.e. a cold access and a warm access. However, when the compiler's optimization is enabled, the situation becomes more complex.

## Warm vs. Cold Access, Optimization On

With optimizations enabled (10,000 runs), gas costs for `coldAccess()` and `coldAndWarmAccess()` are reduced to 2,247 and 2,280 respectively. Astute readers would have noticed that the difference in the gas costs is now 33 which begets the question — why is the difference less than the cost of a warm SLOAD operation when the `coldAndWarmAccess()` function is clearly making two SLOAD calls? Delving into the Remix debugger reveals two intriguing findings:

- The optimized `coldAndWarmAccess()` function only makes a single SLOAD call.
- Function selectors can impact gas costs.

## How Many SLOADs Are There?

Previously, we assumed that `coldAndWarmAccess()` makes two SLOAD calls because of the double access to `y`. However, this is true only when the optimizer is turned off. With optimization on, the compiler recognizes that `a = y;` is called twice but optimizes it to perform only one SLOAD operation since the value assigned to `a` after the first SLOAD remains unchanged. However, what if we were to access two different variables? Would the compiler still make one SLOAD call?

```solidity
// Solidity code with optimizer on, 10000 runs
pragma solidity 0.8.19;

contract StorageExample {
    uint256 x = 0;
    uint256 y = 1;
    uint256 z = 2;

    function coldAccess() public view returns (uint256 a) {
        a = x;
    }

    // Renamed from coldAndWarmAccess() to coldAccess2()
    function coldAccess2() public view returns (uint256 a) {
        a = y;
        a = z;
    }
}
```

In this modified contract, despite two assignments to `a`, one for `y` and another for `z`, the gas cost of executing `coldAccess2()` remains the same at 2,280. This behaviour suggests that the compiler ignores the first assignment statement, considering only the final assignment. But, this raises yet another question — if both functions make only a single SLOAD call, why is there a difference in gas costs? Let's return to the debugger to unravel this mystery.

## How Function Selectors Can Affect Gas Costs

Before we proceed further, we need to first understand function selectors. A function selector is the first eight characters of the hash of a function's signature. This signature comprises of a function's name and parameter's names and types.

For example, the signature of the ERC20 transfer function is `transfer(address sender, uint256 amount)`. Hashing this signature (without the parameter names) i.e. `keccak256Hash(transfer(address,uint256))` will yield a hexidecimal string of `a9059cbb2ab09eb219583f4a59a5d0623ade346d962bcd4e46b11da047c9049b`. The first 8 characters of this string is the function selector i.e. `a9059cbb`.

It is also important to note that hexadecimal characters can be converted to regular integers and vice versa. For instance, `0x1f` corresponds to the decimal number 31 and `a9059cbb` is equivalent to the integer value 2835717307. This allows any hexidecimal number to be "sorted" based on their integer values. Keep this at the back of your head while we go through the next part.

To answer the earlier question as to why two seemingly identical functions can have different gas costs, we will need to take a look at this sequence of characters `60003560e01c80637219c606146037578063e717341c14604d57`, which can be found in the contract's bytecode.

```markdown
| -------------- | ---------------------------------------------------------- |
| Sequence       | Explanation                                                |
| -------------- | ---------------------------------------------------------- |
| 600035         | Load the transaction's calldata                            |
| -------------- | ---------------------------------------------------------- |
| 60e01c         | Extract the transactions's function selector (fn sel)      |
| -------------- | ---------------------------------------------------------- |
| 80637219c60614 | Compare the current fn sel with the first fn sel 7219c606  |
| -------------- | ---------------------------------------------------------- |
| 603757         | If it matches, jump to instruction 37                      |
| -------------- | ---------------------------------------------------------- |
| 8063e717341c14 | Compare the current fn sel with the second fn sel e717341c |
| ---------------| ---------------------------------------------------------- |
| 604d57         | If it matches, jump to instruction 4d                      |
| -------------- | ---------------------------------------------------------- |

```

The summary table above describes an algorithm for matching function selectors. It will attempt to match the current transaction's function selector with one of the functions selectors in the contract. Once a match is found, it will jump to another part of the bytecode to continue its execution. 

The two function selectors correspond to the two functions in the contract. `7219c606` is the selector for the `coldAccess()` function while `e717341c` belongs to `coldAccess2()`. As one would imagine, this algorithm will grow in "size" based on the number of functions available in the contract. 

If we are trying to execute `coldAccess2()`, the algorithm will first compare its selector with `7219c606` and see if it is a match. Since it is not a match, it will do nothing and continue to execute the next comparison check. Since this is a match, it will jump to instruction 4d.

From this, we can deduce why the `coldAccess()` function costs less gas — it has fewer selectors to match against. Now you might be wondering why the algorithm is checking `7219c606` first? This is because function selectors are sorted in ascending order so the `7219c606` will always be higher up the sorting order. One can easily verify this by locating a function with a selector smaller than `7219c606`.

```solidity
// Solidity code with optimizer on, 10000 runs
pragma solidity 0.8.19;

contract StorageExample {
    uint256 x = 0;
    uint256 y = 1;
    uint256 z = 2;

    function coldAccess() public view returns (uint256 a) {
        a = x;
    }

    // Renamed from coldAccess2 to coldAccessTwo()
    function coldAccessTwo() public view returns (uint256 a) {
        a = y;
        a = z;
    }
}
```
The new `coldAccessTwo()` function has a function selector of `6d76c428` which is smaller than `7219c606` so as expected, it costs 2,247 gas while the original `coldAccess()` function costs 2,280 gas!

## How to Trigger Warm Access with Optimizations On
In an earlier example, the `coldAndWarmAccess()` function makes a single SLOAD call when optimizations are turned on but what if we wanted to intentionally trigger a warm access? We need to provide the compiler with a "reason" for the contract to execute the first assignment. For example:

```solidity
// Solidity code with optimizer on, 10000 runs
pragma solidity 0.8.19;

contract StorageExample {
    uint256 x = 0;
    uint256 y = 1;
    uint256 z = 2;

    function coldAccess() public view returns (uint256 a) {
        a = x;
    }

    function coldAndWarmAccess() public view returns (uint256 a) {
        a = y;
        a = a + y;
    }
}
```

With this new revised contract, the `coldAndWarmAccess()` function necessitates the execution of two SLOAD calls to calculate the final value assigned to `a`. Consequently, when this function is executed, it incurs a gas cost of 2363. The difference of 116 gas confirms that the updated `coldAndWarmAccess()` function executes two SLOAD calls even with optimizations enabled.

## Key Learnings

Throughout this article, we have uncovered a few findings that would not have been possible were it not for debugging tools. If you want to make a career out of doing gas optimizations, I highly recommend to familiarize oneself with such tools. Personally, I really like [Foundry's debugging interface and UX](https://book.getfoundry.sh/forge/debugger).

Furthermore, we've also highlighted some quirks of deploying a contract with optimizations turned on. When in doubt, the best strategy is to craft a simple proof of concept and leverage on debugging tools to scrutinize the underlying opcodes.

Perhaps the most important takeaway is the large disparity in gas costs between cold and warm SLOAD operations. This underscores the need for implementing optimization strategies such as variable packing to reduce on the number of cold SLOAD operations.

My exploration into the intricacies of SLOAD has proven to be a very fun and rewarding journey. I hope that you learnt something from this article. Follow me on [twitter](https://twitter.com/0xlgtm) if you want to see more gas optimization content like this.