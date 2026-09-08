# RangeRent

**Makes a tick range something one provider holds, by renting it to them exclusively.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://range-rent.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/RangeRentHook.sol`](src/hooks/RangeRentHook.sol)
- **Licence:** Apache-2.0

## How it works

Concentrated liquidity is a commons. Anybody may add to any range at any time, and the moment a range becomes profitable everybody piles into it, which is how just-in-time liquidity works: watch for a large swap, add liquidity for one block at exactly the right ticks, take a share of the fee, and withdraw. The capital was never at risk and it collected as though it had been.

The providers who sat in that range through the quiet week are diluted by somebody who arrived for one transaction. Every defence so far has taxed the symptom: a fee that punishes short-lived positions, a lock-up, a tenure weight. They all penalise legitimate providers who happen to leave, and none of them stop the strategy, because the payoff scales with the swap and the penalty does not.

This removes the commons instead. A range can be leased, and while it is leased nobody but the lessee may add liquidity to it. The lease is bought for a period at a rent the market sets by competition, since anybody may take an unleased range and anybody may outbid an expiring one.

Just-in-time liquidity is not made expensive; it is made impossible, because the attacker cannot add at the ticks that matter. The rent goes to the pool, donated to whoever is providing when it settles. So a provider who wants to be alone in a range pays for the privilege, and the payment goes to the providers who are sharing the rest of the pool with them.

Unleased ranges stay a commons, which is the point: a pool wearing this hook is not closed, it is one where the ranges worth defending can be defended.

## Prior art

Just-in-time liquidity is well documented and the mitigations are all penalties: time-weighted fee shares, withdrawal delays, and this catalogue's own TenureWeightedFees. Auction-managed AMMs sell the right to set a pool's fee, and this catalogue's TickHarberger sells that right per range. Renting exclusive *provision* rights to a tick range, so that nobody else may add liquidity there at all, is the contribution here, and it is a different right from the one TickHarberger sells.

## Where it does not help

Exclusivity is enforced against the address that calls the pool, which for a router-carried position is the router rather than the person behind it, so a lessee has to provide through an address they control and a lease taken out on a shared router is shared with everybody using it. Beyond that, exclusivity is genuine and so is its cost: a leased range holds only one provider's capital, so a pool whose best ranges are all leased is thinner than one where anybody could join. Leases are fixed-term rather than continuously contestable, so an incumbent holds their range until it expires however valuable it becomes; the term is the pool's choice and a long one is a long monopoly. Rent reaches providers through `donate`, which credits whoever is in range at settlement rather than through the lease, and `settleRent` is callable by anyone precisely so that gap stays small. And a lessee who leaves their range empty has bought silence rather than liquidity, which is a legitimate thing to buy and worth knowing is possible.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    RangeRentHook.Config({
        rangeWidth: /* int24 */ 0,
        rentPerSecond: /* uint128 */ 0,
        minTerm: /* uint32 */ 0,
        maxTerm: /* uint32 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `rangeWidth` | `int24` |  |
| `rentPerSecond` | `uint128` |  |
| `minTerm` | `uint32` |  |
| `maxTerm` | `uint32` |  |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `CallbackNotPoolManager()` | Only the `PoolManager` may drive the unlock callback. |
| `InvalidRangeWidth()` | A range narrower than the tick spacing, or not a whole number of them, cannot be a range. |
| `InvalidTerms()` | A rent of zero would make exclusivity free, and terms must be a real interval. |
| `NothingToSettle()` | There is nothing to settle. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `PositionTooWide()` | A position spanning more ranges than this cannot be checked within a sensible gas budget. |
| `RangeLeased(address,uint64)` | Somebody else holds this range until `until`. |
| `RangeNotYours(int256,address)` | The position overlaps a range leased to somebody else. |
| `SafeCastOverflowedIntToUint(int256)` | An int value doesn't fit in a uint of `bits` size. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |
| `TermOutOfRange(uint32,uint32)` | The requested term is outside what this pool leases. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `beforeAddLiquidity`

Mask: `0x1800`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # RangeRent
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # liquidity, jit-defence, exclusivity, concentrated, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/range-rent
cd range-rent
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
