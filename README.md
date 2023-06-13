# Uniswap V3: Accounting of Positions and Fees

## Table of Contents
1. [Introduction](#introduction)
2. [Accounting of Positions](#accounting-of-positions)
3. [Fee Mechanism](#fee-mechanism)
4. [NFT Mechanism for Representing Positions](#nft-mechanism-for-representing-positions)
5. [Conclusion](#conclusion)
6. [References](#references)

## Introduction
- Brief overview of Uniswap V3, with a mention of v3-core and v3-periphery repositories.
- Statement of the document's focus on v3-core.

Uniswap V3 is a prominent decentralized exchange (DEX) on the Ethereum blockchain, known for its innovative features and mechanisms that set it apart from its predecessors and competitors. This document aims to delve into the intricacies of Uniswap V3, particularly focusing on how it handles the accounting of positions and fees.

Th focus is blah blah. Liquidity Provider (LP), Pool, token0 and token1. 

### V2 vs V3
The key novelty of Uniswap V3 is the introduction of *concentrated liquidity* in V3. In Uniswap V2, liquidity is distributed across an infinite range of prices, which can lead to capital inefficiency (as some of the provided liquidity may never be used). Uniswap V3 addresses this issue by allowing liquidity providers to specify a price range for their liquidity, effectively concentrating their capital where it's most likely to be needed.

This is achieved through a mechanism called *ticks*. The entire price range is divided into discrete 'ticks', and liquidity providers can choose the upper and lower ticks between which their liquidity will be active. This means that their liquidity is only used when the market price of the pair is within their chosen range, making their capital more efficient.

Another significant difference is that Uniswap V3 uses a square root price function, which simplifies the calculation of prices and liquidity amounts and reduces the potential for rounding errors. This also allows for more precise and flexible liquidity provision.

These improvements in Uniswap V3 make it more capital efficient and flexible than V2, and they form the basis for the mechanisms of accounting, positions, and fees that we will discuss in this document. For a more detailed explanation of these concepts, you can refer to [the Uniswap V3 Development Book](#references).


### Code repos
The Uniswap V3 protocol is primarily composed of two repositories: `v3-core` and `v3-periphery`. The `v3-core` repository contains the core contracts and logic for the protocol, including the contracts for pools, positions, and fees. On the other hand, the `v3-periphery` repository includes additional contracts that interact with the core contracts, providing extra functionality such as multicall, swapping, and liquidity management.

In this document, we will primarily focus on the `v3-core` repository, as it forms the backbone of the Uniswap V3 protocol and is directly responsible for the mechanisms of accounting, positions, and fees. However, it's important to note that the `v3-periphery` repository also plays a significant role in the overall functionality of the protocol.

In the following sections, we will explore how Uniswap V3 keeps track of positions, how it calculates and distributes fees, and how it uses Non-Fungible Tokens (NFTs) to represent liquidity positions. Each section will provide a detailed explanation of the mechanisms involved and will include links to the relevant code in the Uniswap V3 repository for further exploration.


## Pools and Positions
- Explanation of how Uniswap V3 keeps track of positions.
- Description of the Position struct and its fields.

In Uniswap V3, every Pool is an instance of [`contract UniswapV3Pool`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L30). We will discuss some of its variables that are relevant to our report. Let's first start with the *immutables*:

```solidity
/// @notice The first of the two tokens of the pool, sorted by address
address public immutable override token0;
/// @notice The second of the two tokens of the pool, sorted by address
address public immutable override token1;
/// @notice The pool's fee in hundredths of a bip, i.e. 1e-6
uint24 public immutable override fee;
```
[[link]](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L44:)






```solidity
struct Info {
  // the amount of liquidity owned by this position
  uint128 liquidity;
  // fee growth per unit of liquidity as of the last update to liquidity or fees owed
  uint256 feeGrowthInside0LastX128;
  uint256 feeGrowthInside1LastX128;
  // the fees owed to the position owner in token0/token1
  uint128 tokensOwed0;
  uint128 tokensOwed1;
}
```
https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Position.sol#L13


- Explanation of how positions are represented by unique token IDs.
- Link to the relevant code in the Uniswap V3 repository.

## Fee Mechanism
- Explanation of how fees are calculated and distributed in Uniswap V3.
- Description of the fee growth mechanism.
- Explanation of how fees are tracked and accumulated.
- Link to the relevant code in the Uniswap V3 repository.

## NFT Mechanism for Representing Positions
- Explanation of how Uniswap V3 uses NFTs to represent liquidity positions.
- Description of the minting process and the role of the NonfungiblePositionManager contract.
- Link to the relevant code in the Uniswap V3 repository.

## Conclusion
- Summary of the key points discussed in the document.
- Reflection on the implications of Uniswap V3's mechanisms for accounting, positions, and fees.

## References
Links to the Uniswap V3 code repos and any other resources used to produce this document:
- [Uniswap V3 core contracts](https://github.com/Uniswap/v3-core)
- [Uniswap V3 periphery contracts](https://github.com/Uniswap/v3-periphery)
- [The Uniswap V3 book](https://uniswapv3book.com/)
- [Uniswap V3 whitepaper](https://uniswap.org/whitepaper.pdf)
