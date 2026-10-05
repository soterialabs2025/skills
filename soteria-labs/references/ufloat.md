# UFloat — owner-managed Uniswap v4 strategies on Base (`8453`)

Read `SKILL.md` Safety first. Infra: `config.ufloat` (`factory`, `swapRouter`, `keeper`, `weth`, `defaultDeployTokens`). Calls: `abi.md`.

**Model.** A UFloat strategy is a clone the user **owns**. No vault, no shares, no other depositors. ETH in, WETH out, only by the owner. A keeper rebalances; the owner sets the token allowlist, rotates the active token, and tunes mint parameters.

Per-user strategy addresses come from `factory.getStrategies(user)` or a pasted `0x`. Never invent one. Page link: `site + ufloat.hrefStrategy` with `{strategy}` replaced; directory `site + ufloat.hrefDirectory`.

## Core rules

1. **Deploy** uses the factory from the config. **Manage** needs a strategy address (deployed now, from `getStrategies(user)`, or pasted).
2. Owner-only writes require `strategy.owner() == user`; if not, read only and say who owns it.
3. **Deposit = ETH only** (`depositETH()` with `value`). **Withdraw = WETH only** (`withdrawWeth(wethAmount)`), paid to the owner as WETH, never native ETH, never the token.
4. Token arguments are **ERC-20 addresses on Base**, not names; each must pass `swapRouter.hasV4PoolConfig(token)` or the call reverts.
5. Deploy installs on-chain **default** mint params; tune afterwards with `setMintParams` / `setOffensiveParams`.

## Deploy

"deploy a ufloat strategy" / "create a ufloat"

1. Ask which token addresses (not WETH). Order matters: **`tokens[0]` is the initial ASSET**. Offer `ufloat.defaultDeployTokens` as candidates (still preflighted). Confirm the list back.
2. `stratMethod = 0` (`ReBalanceOnly`) unless the user picks `1 OffensiveOnly`, `2 DefensiveOnly`, `3 OffensiveDefensive`.
3. Preflight: chain `8453`, code on `factory` and `swapRouter`, `hasV4PoolConfig(t) == true` for every `t`. Any `false` → stop; that token is not configured on the UFloat router.
4. Preview: factory, method, ordered tokens, gas; simulate `deployStrategy(stratMethod, tokens)` from the user. Authorize.
5. Send `factory.deployStrategy(uint8 stratMethod, address[] tokens)`; caller becomes owner.
6. Receipt: `StrategyDeployed(strategy, owner, firstAsset, keeperId, allowedTokenCount)` on the factory (topic1 strategy, topic2 owner, topic3 firstAsset). Report **strategy address**, initial **ASSET**, allowlist, `keeperId`, page link. Then offer: deposit ETH, tune params, manage tokens.

## Reads (all on `strategy`)

| User asks | Call |
|---|---|
| value / NAV / what is it worth | `totalValueWeth()` (WETH-notional, 18 dec) |
| what is in the pool | `balanceOfPool()` → `(assetAmt, wethAmt)` |
| idle | `balanceOfIdle()` |
| current ASSET | `ASSET()` (label via `symbol()`) |
| allowed tokens | `allowedTokenCount()` + `allowedTokens(i)` |
| mode | `mode()` → `0 NORMAL`, `1 DEFENSIVE`, `2 OFFENSIVE`, `3 STABLE` |
| owner | `owner()` |
| position id | `getPositionId()` |
| my strategies | `factory.getStrategies(user)` |
| params | `stratMethod()`, `targetAssetBps()`, `rangeBelowTicks()`, `rangeAboveTicks()`, `stopLoss()`, `feeReserveBps()`, `reserveAddress()`, `tickSpacing()`, `minFloorTickCount()`, `offensiveStaleDuration()`, `offensiveAssetBps()`, `minRangeBelowTicks()` |

## Deposit

"Deposit 0.1 ETH into strategy 0x…" → owner check → wei → preview (amount, gas, simulation) → authorize → `strategy.depositETH()` with `value = wei` → receipt `status 1` → report hash, then re-read `totalValueWeth()`. No other deposit path.

## Withdraw

`withdrawWeth(wethAmount)` to the owner, in WETH.

| User asks | `wethAmount` |
|---|---|
| X WETH / X ETH-notional | `X × 1e18` |
| Y % | `totalValueWeth() × Y / 100`, `0 < Y ≤ 100` |
| all / exit fully | `2^256 − 1` |

Owner check; `nav = totalValueWeth()`; `nav == 0` → do not send. Cap computed amounts at `nav` (full exit uses max). Preview amount, gas, simulation; authorize; send; report WETH received from the receipt's WETH `Transfer` to the owner (raw + human). Tell the user it is **WETH**, not ETH.

## Asset management (owner writes)

| User asks | Call |
|---|---|
| allow token | `addAllowedToken(token)` (preflight `hasV4PoolConfig`) |
| remove token | `removeAllowedToken(token)` |
| change asset / rotate to | `changeAsset(token)` (must be allowlisted) |
| mint / remint position | `mintPosition(token)` (same ASSET remints; another allowlisted token rotates; reverts if NAV ≤ stopLoss) |
| exit to stable / flatten | `exitToStable()` |

Before a rotation, show `ASSET()`, allowlist, `mode()`, `totalValueWeth()`. Each write: owner check, preview, simulate, authorize, send, receipt.

## Mint params (owner writes)

```
setMintParams(targetAssetBps, rangeBelowTicks, rangeAboveTicks, stopLoss, feeReserveBps, reserveAddress)
setOffensiveParams(minFloorTickCount, offensiveStaleDuration, offensiveAssetBps, minRangeBelowTicks)
setStratMethod(method)   // 0 ReBalanceOnly, 1 OffensiveOnly, 2 DefensiveOnly, 3 OffensiveDefensive
```

Rules the contract enforces (reject before sending): `targetAssetBps` and `offensiveAssetBps` `< 10_000` (`10_000 = 100%`); range ticks non-zero, `< 10_000`, and a multiple of `tickSpacing()` (read it; default `200`); `feeReserveBps ≤ 9000`; `reserveAddress != 0`. If the user says "update" without every value, read the current params first and fill the rest from chain, then show the full set for authorization.

## Response rules

After deploy: strategy address, initial ASSET, link. On manage writes: strategy + current ASSET. If a deploy / add / rotate reverts on router config, say the token is not configured on the UFloat router — do not try another address.

## Do not call from this skill

Keeper functions (`keeperCheck`, `harvestBoolean`, `setWatched`), `sellAsset`, router / factory owner setters, any ERC-20 `approve` (no flow here needs one).
