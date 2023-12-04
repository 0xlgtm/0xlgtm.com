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

Before we begin, let's talk about the elephant in the room — why write another gas optimization article when there are already [so](https://www.rareskills.io/post/gas-optimization) [many](https://www.alchemy.com/overviews/solidity-gas-optimization) [resources](https://coinsbench.com/comprehensive-guide-tips-and-tricks-for-gas-optimization-in-solidity-5380db734404) [out](https://betterprogramming.pub/solidity-gas-optimizations-and-tricks-2bcee0f9f1f2) [there](https://www.cyfrin.io/blog/solidity-gas-optimization-tips)?

The biggest gripe that I have with most of these articles is that they do not go sufficiently in-depth to explain the technicalities behind the various optimizations. Moreover, some of the optimizations covered are also version specific and [may no longer be applicable](https://twitter.com/solidity_lang/status/1717213166210588706). Instead, this article will only focus on the most impactful and version agnostic optimizations. A thorough breakdown of each technique will also be included so that the reader can understand how and why these tricks work.

Prerequisite knowledge about the EVM and opcodes are required so if you need a refresher on this topic, I can highly recommend Nox's [EVM deep dive series](https://noxx.substack.com/p/evm-deep-dives-the-path-to-shadowy).



# Storage Packing

## Storage Packing

talk about sstore and sload, warm vs cold access, dirty vs clean writes.

https://github.com/wolflo/evm-opcodes/blob/main/gas.md


### Why it works

### Things to note


smart storage packing vs not so smart storage packing

# 



# Acknowledgements

https://snappify.com/view/f9a681c7-834c-467e-b34d-5ad443a893f2