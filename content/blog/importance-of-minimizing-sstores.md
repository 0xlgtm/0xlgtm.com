+++
title = "A case study on gas optimization design ft. SSTORE"
date = 2023-10-02
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

Modern decentralised applications are becoming progressively complex and gas costs are on the rise. It is essential to consider gas optimization as an integral part of a system's design from the very beginning. In this article, we will explore the significance of the SSTORE opcode and examine an approach for reducing the frequency of SSTORE operations. An example contract will be used to demonstrate this strategy and its potential gas savings. To wrap up, we will highlight some considerations / tradeoffs to bear in mind when employing gas optimization.

## What is the SSTORE opcode

SSTORE is the opcode used to modify a blockchain state. Whenever a function wants to update the contract's storage, an SSTORE opcode is executed. For instance, the ERC20 `transfer()` function executes two SSTORE operations. The first reduces the sender's balance and the second is to increase the recipient's balance. In the case of the `transferFrom()` function, it requires three SSTOREs: two for modifying balances and one for updating allowances. ERC20 tokens are considered one of the most primitive building blocks yet it already uses up to three SSTOREs. It stands to reason that more sophisticated dapps, such as decentralized exchanges or lending platforms, would inherently require a greater number of SSTORE operations.

## Why minimize SSTOREs

There are a multitude of strategies that can be employed to reduce a contract's gas costs, but one of the most straightforward and effective methods is to minimize the frequency of SSTORE calls. Why the emphasis on SSTORE? It's due to the fact that SSTORE represents the most expensive opcode to execute. Changing the state from zero to a non-zero value carries a significant cost of 20k gas, while transitioning from a non-zero state to another non-zero state incurs a cost (albeit lower but still expensive) of 5k gas. In contrast, basic mathematical opcodes like add and sub come at a much lower cost of 3 gas each. As pareto's principle puts it, 80% of your gas savings comes from 20% of the optimization. Therefore, it is prudent to try reducing the number of SSTORE operations before considering other gas optimization strategies.

## Gas optimization strategy

One approach we can adopt to decrease the frequency of SSTORE operations is by employing a technique known as bit packing. The core idea behind bit packing is to combine multiple smaller bits into a single storage slot. For example, we can compress eight uint32 values into a single `uint256` value. Writing this value to the blockchain would only require a single SSTORE operation instead of eight! Before delving into the intricacies of bit packing, it's essential to establish a foundational understanding of what a "bit" represents.

## Bits and Bitwise Operations

In the digital space, all data is depicted using a combination of 1s and 0s. For example, the number 14 is represented as `00001110`. Each individual value within this binary representation is referred to as a "bit" and eight bits is collectively known as a "byte". For the purpose of this article, you do not need to understand how to a number's binary representation is derived. As long as you understand that every number has a corresponding binary representation, you are ready for the next section.

```md
|--------|-----|----|----|----|---|---|---|---|
| Number | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |
|--------|-----|----|----|----|---|---|---|---|
| 14     | 0   | 0  | 0  | 0  | 1 | 1 | 1 | 0 |
|--------|-----|----|----|----|---|---|---|---|
| 28     | 0   | 0  | 0  | 1  | 1 | 1 | 0 | 0 |
|--------|-----|----|----|----|---|---|---|---|

```

Notice that the binary representation of 28 is essentially the same as that of 14, except it's shifted one bit to the left. Shifting a value to the left by one bit is equivalent to multiplying it by 2. The left shift operation is an example of a bitwise operation, as it manipulates data at the bit level. Bit packing relies on three key bitwise operations: left shift (`<<`), right shift (`>>`) and OR (`|`).

We've already seen left shift in action earlier, but more generally, the left shift operation relocates the bits of a number to the left by a specified number of positions. Similarly, the right shift operation moves the bits of a number to the right by a specified number of positions. For example, if you shift 28 to the right by two bits, you'll obtain the binary representation of `00000111`.

The bitwise OR operator compares each bit of the first operand to the corresponding bit of the second operand. If either of the bits is 1, the resulting bit is set to 1; otherwise, it's set to 0. The bitwise OR operator is used to "combine" two binary representations. For example, performing a bitwise OR operation between 28 and 14 (`00011100 | 00001110`) yields `00011110`.

Now that we have a clear understanding of what a bit is and the essential bitwise operations involved in bit packing, let's put it all together. Storage slots in Ethereum are 32 bytes in size, resulting in 253 unused bits when we storing the number 28. Leveraging on left or right shifts allows us to rearrange the bit positions to create space for combining with other numbers using the bitwise OR operation. To demonstrate this gas optimization technique, we will first look at an unoptimized contract and its gas costs, before examining an optimized version of it that employs bit packing.

### Unoptimized Shop Contract

We will create an unoptimized shop contract. In this shop, we are able to list items for sale. Each item is identified by a unique `uint256` number and as the owner of the shop, you can update the price and quantity of items to be sold. The unoptimized shop contract might look something like this:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.19 <0.9.0;

contract Shop {
    error NotOwner();

    address public owner;

    mapping(uint256 => uint256) public priceOf;
    mapping(uint256 => uint256) public quantityOf;

    constructor(uint256 id, uint256 price, uint256 quantity) {
        owner = msg.sender;
    }

    function addPriceAndQuantityNoPacking(uint256 id, uint256 price, uint256 quantity) public {
        if (msg.sender != owner) revert NotOwner();
        priceOf[id] = price;
        quantityOf[id] = quantity;
    }

    function getPriceAndQuantityNoPacking(uint256 id) public view returns (uint256 price, uint256 quantity) {
        price = priceOf[id];
        quantity = quantityOf[id];
    }
    // ---snip---
}
```

The `addPriceAndQuantityNoPacking()` function can be called to update the price and quantity of the specified item id. This function will execute two SSTORE operations to update the `priceOf` and `quantityOf` mappings. At this point, we can ignore the auxiliary `getPriceAndQuantityNoPacking()` function, as it will be utilized later for a gas comparison test.

Optimizing this contract involves reassessing the use of `uint256` for price and quantity. The maximum value of a `uint256` value is ~10^77 so even with a payment token having 18 decimals, the maximum possible payment amount remains irrationally large at ~10^59. One potential strategy is to consolidate both price and quantity within a single `uint256`.

### Optimized Shop Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.19 <0.9.0;

contract PackedShop {
    error NotOwner();

    address public owner;

    mapping(uint256 => uint256) public priceAndQuantityOf;

    constructor(uint256 id, uint256 price, uint256 quantity) {
        owner = msg.sender;
    }

    function addPriceAndQuantityWithPacking(uint256 id, uint128 price, uint128 quantity) public {
        if (msg.sender != owner) revert NotOwner();
        // First 128 bits contains the price
        // Last 128 bits contain the quantity
        priceAndQuantityOf[id] = uint256(price) << 128 | quantity;
    }

    function getPriceAndQuantityWithPacking(uint256 id) public view returns (uint128 price, uint128 quantity) {
        uint256 priceAndQuantity = priceAndQuantityOf[id];
        price = uint128(priceAndQuantity >> 128);
        quantity = uint128(priceAndQuantity);
    }
    // ---snip---
}

```

As you can see from the `PackedShop` contract above, we are accepting two `uint128` variables for price and quantity. `uint128` uses half the number of bits of a `uint256` variable so we can fit two `uint128` into a single `uint256` slot. Here's a visual illustration of this.

```md
|----------|-------|-----------------------|----------------|
|          | Value | Binary representation | Notes          |
|----------|-------|-----------------------|----------------|
| Price    | 7     | 0x000...000111        | 125 leading 0s |
|----------|-------|-----------------------|----------------|
| Quantity | 5     | 0x000...000101        | 125 leading 0s |
|----------|-------|-----------------------|----------------|

```

Let's assume that we want to set a price of 7 and a quantity of 5. In binary, price and quantity are represented as 0x000...000111 and 0x000...000101 respectively. Both binary representations have 125 leading 0s. Next, we need to figure out how to combine these two numbers together.


```md
|-----------------------------------|-----------------------|-------------------------------------------|
| Action                            | Binary Representation | Notes                                     |
|-----------------------------------|-----------------------|-------------------------------------------|
| uint256(price)                    | 0x000...0000111       | 253 leading 0s                            |
|-----------------------------------|-----------------------|-------------------------------------------|
| uint256(price) << 128             | 0x000...0011100...000 | 125 leading 0s, 111, 128 trailing 0s      |
|-----------------------------------|-----------------------|-------------------------------------------|
| uint256(price) << 128 | quantity  | 0x000...0011100...101 | 125 leading 0s, 111, 125 trailing 0s, 101 |
|-----------------------------------|-----------------------|-------------------------------------------|
```

The first step is to convert price (of type `uint128`) to type `uint256`. This means that instead of 125 leading 0s, it will now have 253 leading 0s. Next, we will shift left by 128 bits so that there is 128 0s trailing 111. The trailing 0s are then replaced with the binary representation of quantity through the use of the bitwise OR operation.

The combined binary representation is therefore `0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000011100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000101`. Since we've combined both values into a single `uint256` value, we can now make a single storage call to store both values instead of two.

### Performance Comparison

The following gas report generated by foundry with `forge test --match-contract PackedShopTest --gas-report`.

```md
| src/PackedShop.sol:PackedShop contract |                 |       |        |       |         |
|----------------------------------------|-----------------|-------|--------|-------|---------|
| Deployment Cost                        | Deployment Size |       |        |       |         |
| 276713                                 | 1223            |       |        |       |         |
| Function Name                          | min             | avg   | median | max   | # calls |
| addPriceAndQuantityNoPacking           | 46802           | 46802 | 46802  | 46802 | 1       |
| addPriceAndQuantityWithPacking         | 24776           | 24776 | 24776  | 24776 | 1       |
| getPriceAndQuantityNoPacking           | 4701            | 4701  | 4701   | 4701  | 1       |
| getPriceAndQuantityWithPacking         | 2476            | 2476  | 2476   | 2476  | 1       |
```

As expected, adding the price and quantiy via the `addPriceAndQuantityWithPacking()` function costs at least 20k gas less than the `addPriceAndQuantityNoPacking()` function because it only calls SSTORE once instead of twice. Retrieving the price and quantity from the packed value is also cheaper because, like when storing the packed value, the `getPriceAndQuantityWithPacking()` function only makes a single SLOAD call instead of two. 

## Conclusion

Significant gas savings can be achieved by reducing the frequency of SSTORE calls, and there are additional unexplored strategies, such as caching values and consolidating related variables to minimize SSTORE/SLOAD operations. Nevertheless, regardless of the chosen strategy, gas optimization inevitably involves trade-offs.

One of the biggest trade-offs arises from the use of inline assembly for optimization. While assembly can lead to further code optimization, such as eliminating over/underflow checks, it often results in code that is less readable and maintainable. This is particularly important for projects who are worked on by a team as other team members may not necessarily be as proficient in assembly. Contrary to common belief, assembly is not necessarily always required for optimization. Substantial gas savings can be achieved simply by restructuring contracts to minimize unnecessary calls.

Another crucial consideration when implementing complex optimizations is the potential exposure to security risks. For example, when a storage variable combines multiple values, there will most likely be multiple functions to update this variable. Extreme caution is essential to prevent inadvertent or deliberate modifications to the other values who are sharing the same storage slot during updates.

Lastly, gas optimizations can potentially make an upgraded contract less gas efficient. For instance, if the first version of a contract combines two values into a single storage slot, and the second version deprecates one of these values, a simple SLOAD operation is now unintentionally more expensive (due to it using the same bit retrieval process) unless a complete storage migration is performed.