---
date: '2025-05-02T18:21:19-07:00'
draft: false
title: 'Bug Disclosure: Reentrancy Lock Bypass'
math: false
---

## Summary

On April 22, [Cyfrin](https://www.cyfrin.io/) informed the Bunni team of a critical issue that allowed attackers to bypass the reentrancy lock in `BunniHub`. This issue enabled attackers to steal all assets in `BunniHub`.

The Bunni team responded by pausing the function that allowed the reentrancy lock to be bypassed, preventing any theft of assets.

## Issue 1: Malicious Rebalance

The culprit is the `BunniHub::unlockForRebalance()` function. Specifically, `BunniHub` has these two functions that allowed the hook of a pool to access the reentrancy lock of `BunniHub`:

```solidity
/// @inheritdoc IBunniHub
function lockForRebalance(PoolKey calldata key) external notPaused(6) {
    if (address(_getBunniTokenOfPool(key.toId())) == address(0)) revert BunniHub__BunniTokenNotInitialized(); // `key` must correspond to a valid Bunni pool
    if (msg.sender != address(key.hooks)) revert BunniHub__Unauthorized(); // msg.sender must by the hook of the pool
    _nonReentrantBefore(); // locks BunniHub's reentrancy lock
}

/// @inheritdoc IBunniHub
function unlockForRebalance(PoolKey calldata key) external notPaused(7) {
    if (address(_getBunniTokenOfPool(key.toId())) == address(0)) revert BunniHub__BunniTokenNotInitialized(); // `key` must correspond to a valid Bunni pool
    if (msg.sender != address(key.hooks)) revert BunniHub__Unauthorized(); // msg.sender must by the hook of the pool
    _nonReentrantAfter(); // unlocks BunniHub's reentrancy lock (!!!)
}
```

The two functions were introduced as a fix to issue C-03 in a [previous audit](https://github.com/pashov/audits/blob/master/team/pdf/Bunni-security-review-August.pdf) by Pashov Audit Group. Issue C-03 was a possible reentrancy attack that allowed a rebalancer to reenter `BunniHub` during the rebalance order’s execution when `BunniHub` is at an inconsistent state. This led to possible theft of funds. In the recommended fix, `lockForRebalance()` and `unlockForRebalance()` were added to `BunniHub` so that `BunniHook` could lock `BunniHub` before a rebalance order is executed and unlock it after the order is executed.

This fix was flawed. `BunniHub` does not maintain a whitelist of hooks, a new pool can be initialized with any hook contract that satisfies some basic interface requirements. This meant that if an attacker deployed a Bunni pool with a malicious hook contract, the malicious hook can call `BunniHub::unlockForRebalance()` at any time and **disable the reentrancy lock entirely**. `BunniHub` only has a single global reentrancy lock instead of implementing a per-pool lock, so given that an attacker can trivially disable the lock `BunniHub`'s reentrancy lock is effectively useless.

The first possible attack Cyfrin disclosed to us was near identical to issue C-03 in the Pashov audit. The malicious rebalancer could simply use a malicious hook to call `BunniHub::unlockForRebalance()` to disable the reentrancy lock and execute the same attack as in C-03.

Bunni does maintain a whitelist of rebalancers, but it also allows the am-AMM manager of a pool to fill rebalance orders. Anyone could become the manager of a pool as long as am-AMM is enabled for the pool, so this attack could be executed by anyone.

## Our First Response

After receiving the bug disclosure, the Bunni team verified that the issue could be reproduced locally and then looked for the best path forward.

Since the attack required a malicious rebalancer, we decided to swap out the existing `BunniZone` contract. `BunniZone` implements the access control for who can rebalance orders:

```solidity
/// @inheritdoc IZone
/// @dev Only allows whitelisted fulfillers and am-AMM manager of the pool to fulfill orders.
function validate(IFloodPlain.Order calldata order, address fulfiller) external view returns (bool) {
    // extract PoolKey from order's preHooks
    IBunniHook.RebalanceOrderHookArgs memory hookArgs =
        abi.decode(order.preHooks[0].data[4:], (IBunniHook.RebalanceOrderHookArgs));
    PoolKey memory key = hookArgs.key;
    PoolId id = key.toId();

    // query the hook for the am-AMM manager
    IAmAmm amAmm = IAmAmm(address(key.hooks));
    IAmAmm.Bid memory topBid = amAmm.getTopBid(id);

    // allow fulfiller if they are whitelisted or if they are the am-AMM manager
    return isWhitelisted[fulfiller] || topBid.manager == fulfiller;
}
```

Basically, only whitelisted addresses and the am-AMM manager of a pool can fill the pool’s rebalance orders. We replaced `BunniZone` with one that only allowed whitelisted addresses to rebalance:

```solidity
/// @inheritdoc IZone
/// @dev Only allows whitelisted fulfillers.
function validate(IFloodPlain.Order calldata, /* order */ address fulfiller) external view returns (bool) {
    return isWhitelisted[fulfiller];
}
```

Thus the attack is prevented. We thought this fix was the least disruptive to Bunni’s users, since all features including rebalancing will still be available as usual.

At this point we considered disclosing the issue to the Bunni community, but we decided to be cautious. If the reentrancy lock of `BunniHub` is now useless, surely there are other reentrancy attacks that are now also possible right?

We asked Cyfrin about this possibility. On April 25, Cyfrin gave us the PoC for a different attack.

## Issue 2: Malicious Vault + Hook

The second attack required the attacker deploy a Bunni pool with a malicious hook as well as a malicious ERC-4626 vault. The malicious hook calls `BunniHub::hookHandleSwap()`, which is a function that allows hooks to access the assets in a pool. Normally the amount of assets a hook can access is limited to the assets deposited into the pool that uses the hook, so that even if the hook was malicious it couldn’t steal assets in pools using honest hooks. However with no reentrancy guard, it is possible for an attacker to reenter the function at an inconsistent state, bypassing the pool asset accounting and allowing the malicious hook to access more assets than usual.

The attack is a bit complicated to explain, so here’s the explanation Cyfrin sent us:

> 1. We have a malicious pool configured with one malicious vault (can use a worthless corresponding token) and one legitimate vault used by other pools.
>
> 2. The target ratios of the malicious pool are such that token0 will always be withdrawn from the target vault (100% raw balance) and token1 will always be deposited to the malicious vault (0% raw balance).
>
> 3. Our malicious hook calls hookHandleSwap() with an output amount that requires tokens to be withdrawn from the target vault reserve.
>
> 4. We re-enter over the cached state in _updateRawBalanceIfNeeded() which calls _updateVaultReserveViaClaimTokens() again to push raw balance to the malicious vault.
>
> 5. The malicious hook is called by the malicious vault during deposit to invoke unlockForRebalance(). The BunniHub is now unlocked so it then repeats the process, calling hookHandleSwap() with 1 wei of input token (to trigger the deposit from which to re-enter) and an excessive output amount for x iterations until the reserve is drained.
>
> 6. Outcome: the raw balance of legitimate pools are drained by a single pool configured with a malicious hook and vault.

Essentially the attack is a classic reentrancy attack: the pool’s internal balance is updated after transferring tokens out and other external calls, so a reentrancy attack could repeatedly withdraw tokens without depleting the internal balance.

This attack made it possible to drain all of `BunniHub`'s assets and steal all of Bunni’s TVL.

## Our Second Response

We understood the urgency of resolving this issue, so we immediately got to work.

 `BunniHub`  allows the contract owner (i.e. the Bunni team) pause functions individually. We recognized that `BunniHub::unlockForRebalance()` was the crux of the issue, so we planned to pause this function. The effect on users would be that rebalancing is disabled, but all other features (depositing, withdrawing, swapping) would still be available.

After confirming with Cyfrin that this was a valid solution, we executed it on all networks. It was done ~15 mins after we saw the PoC disclosure from Cyfrin. All funds were now safe.

## Conclusion

Security is a top priority at Bunni. The Bunni contracts were audited by Pashov Audit Group and Trail of Bits, and they’re currently being audited by Cyfrin as part of the [Uniswap Foundation Security Fund](https://uniswapfoundation.mirror.xyz/jEBcJKZf8OEOiLB-svh9MNoopqOSUh0Nf3M4qAABUk8). Once the audit is finished, we will deploy the fixed contracts which will have rebalancing enabled again.