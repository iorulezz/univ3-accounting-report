# Uniswap V3: Accounting of Positions and Fees

## Table of Contents
1. [Introduction](#introduction)
2. [Accounting of Positions](#accounting-of-positions)
3. [Fee Mechanism](#fee-mechanism)
4. [NFT Mechanism for Representing Positions](#nft-mechanism-for-representing-positions)
5. [References](#references)

## Introduction
- Brief overview of Uniswap V3, with a mention of v3-core and v3-periphery repositories.
- Statement of the document's focus on v3-core.

Uniswap V3 is a prominent decentralized exchange (DEX) on the Ethereum blockchain, known for its innovative features and mechanisms that set it apart from its predecessors and competitors. This document aims to delve into the intricacies of Uniswap V3, particularly focusing on how it handles the accounting of positions and fees.

Write how we ignore price mechanics and link the book for details. 

Th focus is blah blah. Liquidity Provider (LP), Pool, token0 and token1. 

### V2 vs V3
The key novelty of Uniswap V3 is the introduction of *concentrated liquidity* in V3. In Uniswap V2, liquidity is distributed across an infinite range of prices, which can lead to capital inefficiency (as some of the provided liquidity may never be used). Uniswap V3 addresses this issue by allowing liquidity providers to specify a price range for their liquidity, effectively concentrating their capital where it's most likely to be needed.

This is achieved through a mechanism called *ticks*. The entire price range is divided into discrete 'ticks', and liquidity providers can choose the upper and lower ticks between which their liquidity will be active. This means that their liquidity is only used when the market price of the pair is within their chosen range, making their capital more efficient.

Another significant difference is that Uniswap V3 uses a square root price function, which simplifies the calculation of prices and liquidity amounts and reduces the potential for rounding errors. This also allows for more precise and flexible liquidity provision.

These improvements in Uniswap V3 make it more capital efficient and flexible than V2, and they form the basis for the mechanisms of accounting, positions, and fees that we will discuss in this document. For a more detailed explanation of these concepts, you can refer to [the Uniswap V3 Development Book](#references).


### Code repos
The Uniswap V3 protocol is primarily composed of two repositories: `v3-core` and `v3-periphery`. The `v3-core` repository contains the core contracts and logic for the protocol, including the contracts for pools, positions, and fees. On the other hand, the `v3-periphery` repository includes additional contracts that interact with the core contracts, providing extra functionality such as multicall, swapping, and liquidity management.

In this document, we will primarily focus on the `v3-core` repository, as it forms the backbone of the Uniswap V3 protocol and is directly responsible for the mechanisms of accounting, positions, and fees. However, it's important to note that the `v3-periphery` repository also plays a significant role in the overall functionality of the protocol.

- Should I write about the Callbacks? mint swap and flash all have callbacks. Ergo can only be called by another contract. Put this in the core vs periphery paragraph?


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
[[link to code]](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L44:)
<br><br>
In addition, each pool maintains (in storage) a mapping `mapping(bytes32 => Position.Info) public override positions` of the open positions of the participating LPs. Essentially, a position represents  the owner's liquidity between the lower and upper tick boundary. The mapping maps from the [`keccak(owner, lowerTick, upperTick)`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Position.sol#L36) to the info of the `Position`. The latter looks like this:
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
[[link to code]](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Position.sol#L13) Therefore, the position is defined by the owner and the lower and upper ticks, and it stores information on the amount of liquidity, the fee growth and the tokens owned (the latter parts to be discussed in the section). 
<br><br><br>

- write how liquidity is affected by mint() and burn()
 

## Fee Mechanism
- Explanation of how fees are calculated and distributed in Uniswap V3.
- Description of the fee growth mechanism.
- Explanation of how fees are tracked and accumulated.
- Link to the relevant code in the Uniswap V3 repository.

The fees are collected from users of the protocol who perform swaps (the concept of a flash swap is also defined and generates fees but let's deem it out of scope for this document). The method for swapping tokens looks like:
```solidity
/// @notice Swap token0 for token1, or token1 for token0
function swap(
  address recipient,
  bool zeroForOne,
  int256 amountSpecified,
  ...
) external returns (int256 amount0, int256 amount1);
```
[[link to code]](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L457) Note that we are ignoring price mechanics etc. Each swap charges the fee on the input token, i.e. the Pool keeps some of the input token as fee. This way the fees accumulate and will eventually be distributed to the LPs based on the amount of liquidity they are holding. The variables `feeGrowthGlobal0X128` and `feeGrowthGlobal1X128` keep account of the fee accumulation for this Pool. For example, `feeGrowthGlobal0X128` is defined as:
```
/// @notice The fee growth as a Q128.128 fees of token0 collected per unit of liquidity for the entire life of the pool
uint256 public override feeGrowthGlobal0X128;
```
[[link to code]](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/interfaces/pool/IUniswapV3PoolState.sol#L36) This is represented as Q128.128 (fixed point number 128.128) and corresponds to fees of token0 per unit of liquidity.


### Protocol Fees
Not as important but write something about it? 

## NFT Mechanism for Representing Positions
- Explanation of how Uniswap V3 uses NFTs to represent liquidity positions.
- Description of the minting process and the role of the NonfungiblePositionManager contract.
- Link to the relevant code in the Uniswap V3 repository.

## Notes
Uniswap V4 was announced while I was writing this document :smile_cat: <br>
https://blog.uniswap.org/uniswap-v4

## References
Links to the Uniswap V3 code repos and any other resources used to produce this document:
- [Uniswap V3 core contracts](https://github.com/Uniswap/v3-core)
- [Uniswap V3 periphery contracts](https://github.com/Uniswap/v3-periphery)
- [The Uniswap V3 book](https://uniswapv3book.com/)
- [Uniswap V3 whitepaper](https://uniswap.org/whitepaper.pdf)
