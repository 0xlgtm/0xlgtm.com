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

My main criticism of many of these articles is their preference for breadth over depth i.e. they do not adequately delve into the technical details behind the optimizations discussed. Therefore, this article will focus solely on a few of the most impactful gas saving strategies. A comprehensive breakdown of each technique will also be provided, allowing the reader to appreciate and understand how these tricks work.

We will begin by covering the foundational knowledge necessary to understand the concepts outlined later in this article. Once we've laid a solid groundwork, we'll delve into several optimizations, examining how these techniques work and some considerations to be mindful of when applying these techniques.



## Storage Packing

talk about sstore and sload, warm vs cold access, dirty vs clean writes.

https://github.com/wolflo/evm-opcodes/blob/main/gas.md


### Why it works

### Things to note


smart storage packing vs not so smart storage packing

# 



# Acknowledgements

https://snappify.com/view/f9a681c7-834c-467e-b34d-5ad443a893f2