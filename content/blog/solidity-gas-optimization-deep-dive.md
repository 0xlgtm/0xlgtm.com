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

My main criticism of many of these articles is their preference for breadth over depth i.e. they do not adequately delve into the technical details behind the optimizations discussed. As such, my article will focus only on a few but highly impactful gas saving strategies. A comprehensive breakdown of each technique will also be provided, allowing the reader to appreciate and understand how these tricks work.

We will begin by covering the foundational knowledge necessary to understand the concepts outlined later in this article. Once we've laid a solid groundwork, we will delve into several optimizations, examining how these techniques work and some considerations to be mindful of when applying these techniques.

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

Compiling the Storage contract will result in the following [runtime](https://ethereum.stackexchange.com/questions/32234/difference-between-bytecode-and-runtime-bytecode) bytecode:

`6080604052348015600e575f80fd5b50600436106030575f3560e01c80632e64cec11460345780636057361d146048575b5f80fd5b5f5460405190815260200160405180910390f35b605760533660046059565b5f55565b005b5f602082840312156068575f80fd5b503591905056fea264697066735822122078bcdcb3640a135d107fd506fad0323eea506128357e97975d901550f030118e64736f6c63430008140033`

To the untrained eye, this paragraph might appear nonsensical. However, it contains the code for the aforementioned contract. Each pair of hexadecimal characters constitutes one byte, and each byte corresponds to an operation code, or opcode for short.

Opcodes are the basic instructions executed by the Ethereum Virtual Machine (EVM) and each opcode has a gas cost associated with it. In this article, we will focus on the strategies that minimizes the use of the most expensive opcodes, `SSTORE` and `SLOAD`.


## Storage Packing

talk about sstore and sload, warm vs cold access, dirty vs clean writes.

https://github.com/wolflo/evm-opcodes/blob/main/gas.md


### Why it works

### Things to note


smart storage packing vs not so smart storage packing

# 



# Acknowledgements

https://snappify.com/view/f9a681c7-834c-467e-b34d-5ad443a893f2