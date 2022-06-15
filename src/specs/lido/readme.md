# Spec for Lido Bridge

## What is Lido?
[Lido](https://lido.fi/) is a project that build liquid staking derivatives. They allow users to deposit stakeable assets (Eth, Sol etc) and in return get a representation of the staked asset which will grow with staking rewards. We will only be working with `stEth` (staked ether), so we will be using that for explanations going on. 

`stEth` is a rebasing ERC20 token, and a users balance of it will grow/shrink based on accrued staking rewards or slashing events. After the Merge and the hardfork making it possible to withdraw staked ether from the beacon chain, `stEth` can be burned to redeem an equal amount of `eth`. Until then, `stEth` cannot be redeemed at lido directly, but can instead be traded at a secondary market, such as [curve.fi](https://curve.fi/steth).

## What is the flow of the bridge?
There are two flows of Lido Bridge, namely deposits and withdraws. 

![Lido flows](./LidoBridge.svg)


### Deposit 
If the bridge receives `eth` as the input token it will deposit it into the Lido contract along a referall address. In return, the bridge will receive `stEth` which it will wrap to `wstEth` before returnr the tuple `(wstEthAmount, 0, false)` and have the rollup pull the `wstEth` tokens.

**Edge cases**:
- There is a limit on the size of deposits, so a very large whale might not be able to enter
- The limit is subject to change, so it might influence smaller users over time
- If Lido don't provide sufficient `stEth` for the given `eth` deposit, the transaction will revert. 

### Withdrawal
If the bridge receives `wstEth` as the input token, it will unwrap them to `stEth` before going to curve, where it will swap it (NOTICE: it accepts any slippage). It will then transfer the eth received to the `ROLLUP_PROCESSOR` for the given `interactionNonce`. 

**Edge cases**
- If the balance of the bridge is less than the value returned by `exchange` on curve, the transaction will revet. E.g., if curve transfers fewer tokens than it tell us in the returnvalue. 

## Properties
- The bridge is synchronous, and will always return `isAsync = false`. 

- *Note*: Because `stEth` is rebasing, we wrap/unwrap it to `wstEth` (wrapped staked ether). This is to ensure that values are as expected when exiting from or transferring within the Rollup.

- The bridge itself is built to *not* hold significant tokens, but will leave some dust to save gas on future usage as the storage slot for the balances are updated instead of written from 0.

- The Bridge perform token pre-approvals in the constructor to allow the `ROLLUP_PROCESSOR`, `WRAPPED_STETH` and `CURVE_POOL` to pull tokens from it. This is to reduce gas-overhead when performing the actions. It is safe to do, as the bridge is not holding funds itself.

### Can tokens balances be impacted by external parties, if yes, how?
As we are using the wrapped variation of `stEth` it will not directly be impacted by rewards or slashing. However, the amount of `stEth` it can be unwrapped to might deviate from the expected if there has been a slashing event. The bridge itself need to handle it.

## What about withdrawing after the merge and hardfork?
When Lido can support withdraws directly, a new bridge can be made that performs this interaction. Because the bridge don't hold the tokens, the user is free to take his shielded L2 `wstEth` 
