# Uniswap V3: Accounting of Positions and Fees

## Table of Contents
1. [Introduction](#introduction)
2. [Pools and Positions](#pools-and-positions)
3. [Fee Mechanism](#fee-mechanism)
4. [Additional Notes](#additional-notes)
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
In Uniswap V3, every Pool is an instance of [`contract UniswapV3Pool`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L30). A pool contains many variables that define it or its state. Here, we will discuss some of those that are of relevance to this report. To begin with, let's look at some variables from the set of *immutables*:

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
In addition, there is the set of variables that describes the current *state* of the Pool (those are desribed in [`IUniswapV3PoolState`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/interfaces/pool/IUniswapV3PoolState.sol#LL7C11-L7C31)). Here, we will only discuss some of those, which are relevant to the purpose of this document. Firstly, we have `uint128 public override liquidity` which captures the liquidity available to the Pool in the current range. Moreover, each pool maintains (in storage) a mapping `mapping(bytes32 => Position.Info) public override positions` of the open positions of the participating LPs. Essentially, a position represents  the owner's liquidity between the lower and upper tick boundary. The mapping maps from the [`keccak(owner, lowerTick, upperTick)`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Position.sol#L36) to the `Position.Info` struct. The latter looks like this:
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

The amount of liquidity in position can be affected in two ways, by depositing more liquidity with [`mint()`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L457) or by withdrawing liquidity with [`burn()`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L517). Both of these methods will call [`_modifyPosition`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L306). The latter will first call `_updatePosition()`, which will update the liquidity of the position itself (more details on this in the next section where we talk about fees), and then also update `liquidity` [here](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#LL361C17-L361C27).
 

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
```solidity
/// @notice The fee growth as a Q128.128 fees of token0 collected per unit of liquidity for the entire life of the pool
uint256 public override feeGrowthGlobal0X128;
```
[[link to code]](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/interfaces/pool/IUniswapV3PoolState.sol#L36) This is represented as Q128.128 (fixed point number 128.128) and corresponds to fees of token0 per unit of liquidity. For a swap, the global fee tracker is updated here:
```solidity
// update global fee tracker
if (state.liquidity > 0)
  state.feeGrowthGlobalX128 += FullMath.mulDiv(step.feeAmount, FixedPoint128.Q128, state.liquidity);
```
[[link to code]](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L689) The `state` variable is a struct of type [`SwapState`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L561) which keeps information on the state of each swap. Note that a swap could also [affect](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L755) the `liquidity` variable (as it might move an amount of liquidity in/out of the current range). Finally, the relevant `feeGrowthGlobal` variable (and [protocol fees](#protocol-fees) - to be mentioned in the next section) is [re-assigned](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L759) from the info in the state variable. 

Let's now discuss how the fee growth of a certain position is calculated based on the global fee growth. The fees accrued by each Position are not calculated after every swap (as it would be very gas-intensive to do so for every Position). Instead, they are calculated every time an LP interacts with their position through `mint()`, `burn()`. As we discussed in the previous section, both of those methods call `_updatePosition()` down the stack:
```solidity
/// @dev Gets and updates a position with the given liquidity delta
/// @param owner the owner of the position
/// @param tickLower the lower tick of the position's tick range
/// @param tickUpper the upper tick of the position's tick range
/// @param tick the current tick, passed to avoid sloads
function _updatePosition(
  address owner,
  int24 tickLower,
  int24 tickUpper,
  int128 liquidityDelta,
  int24 tick
) private returns (Position.Info storage position) {
```
[[link to code]](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L379) This method will perform some tick operations to update ticks that have been crossed and eventually will calculate the fee growth *inside* the Position, with variables `feeGrowthInside0X128` and `feeGrowthInside1X128` by using the [`getFeeGrowthInside()`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Tick.sol#L60) method from the Tick library. Finally, the `update()` method of the Position library will be called:


## Additional Notes

In this section, we will briefly mention some additional details that did not make it in the main body of the document.

## Math Libraries

Uniswap V3 has a few libraries used for calculations. Those can be found in [`contracts/libraries`](https://github.com/Uniswap/v3-core/tree/main/contracts/libraries) folder. For example, `LiquidityMath` is used to calculate changes in liquidity and `TickMath` is used to calculate the (sqrt) price from ticks and vice versa. Many other math libraries can be found in the linked folder. 

### Protocol Fees
Not as important but write something about it? 

### NFT Mechanism for Representing Positions
- Explanation of how Uniswap V3 uses NFTs to represent liquidity positions.
- Description of the minting process and the role of the NonfungiblePositionManager contract.
- Link to the relevant code in the Uniswap V3 repository.


### Uniswap V4 

Uniswap V4 was announced while I was writing this document :smile_cat: <br>
https://blog.uniswap.org/uniswap-v4

## References
Links to the Uniswap V3 code repos and any other resources used to produce this document:
- [Uniswap V3 core contracts](https://github.com/Uniswap/v3-core)
- [Uniswap V3 periphery contracts](https://github.com/Uniswap/v3-periphery)
- [The Uniswap V3 book](https://uniswapv3book.com/)
- [Uniswap V3 whitepaper](https://uniswap.org/whitepaper.pdf)
