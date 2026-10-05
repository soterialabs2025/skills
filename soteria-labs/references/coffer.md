# Coffer — multi-strategy ETH vaults on Robinhood (`4663`)

Read `SKILL.md` Safety first. Addresses: `config.coffer` (see `config-api.md`). Calls: `abi.md`.

**Model.** Several Coffer vaults, each with its own share token and 2–3 strategy legs (a meme/volatile leg quoted in WETH, stock legs quoted in USDG, a USDe leg). A share is a claim on the **whole vault**, not one leg. There is **no factory**; the live list is `config.coffer.vaults`.

## Which vault?

Every Coffer request needs one vault. Resolve per `config-api.md` §Resolving. If the user says only "Coffer" and the action is a **read**, answer for all vaults in a short list. If it is a **write**, list `id · label · page link` and ask which one — never pick for them.

Legs: `n = vault.strategyCount()`; for `i < n`, `(strat, targetWeightBps, retired) = vault.strategies(i)`; then on `strat`: `ASSET()`, `quoteToken()`, `poolFee()`, `mode()`. Label `ASSET` from `config.coffer.vaults[].sleeves[].label` when the address matches, else `symbol()` read live. Show quote as **WETH** (`chains.4663.weth`) or **USDG** (`chains.4663.usdg`). Rotator legs change `ASSET()` over time — the sleeve `label` in the config is the deploy pair, the live pair is `ASSET()`.

- `retired == true` → no new deposits, **still counts in vault value and still pays out on withdraw**. Never omit it.
- `mode() == 1` (IDLE) → position exited, keeper skips it; may still hold pair token, so read balances rather than assume.

## Reads

| User asks | Call | Reply as |
|---|---|---|
| vault value / total value | `vault.balance()` (18 dec) | **vault value** in WETH |
| total liquid shares | `liquidShares.totalSupply()` | total shares |
| my shares | `liquidShares.balanceOf(user)` | your shares |
| fees earned | Σ `strat.UniswapFeesCollected()` over all legs | fees earned (WETH) |
| what's in it / pairs / stock pairs | legs loop above | pair + quote, weight, retired/IDLE |
| rotating tokens | config sleeve labels for that leg + `strat.isAllowedToken(addr)` for a given token | rotating tokens; current pair is `ASSET()` |
| APR / performance | `vault-snapshots?chain=coffer` → 7-day fee APR (`config-api.md`) | APR or "not enough history" |
| what have I earned | `user-vault-position?chain=coffer` | fees accrued / lifetime |

Fee and execution parameters are **live reads on every leg, every time** (`withdrawalFeeBps`, `protocolFeeBps`, `protocolFeeOn`, `swapSlippageBps`, `maxTwapDeviationBps`, `twapSeconds`). Do not quote remembered values.

## Deposit ETH

"Deposit 0.05 ETH into Coffer vault 8"

1. Resolve the vault. Amount must be ETH; a USD figure needs a conversion the user sees and authorizes.
2. Preflight: chain `4663`, code checks, Coffer wiring checks (`config-api.md`).
3. **Preview** (reads only):
   - ETH to send (wei + human), gas from `eth_estimateGas` of the exact call.
   - Estimated shares `wei × liquidShares.totalSupply() / vault.balance()` (1:1 if supply is 0). Label it an **estimate**: the vault prices at `min(spot, TWAP-gated)` value and caps credit to the WETH deposited.
   - Fees: **no deposit fee**. Withdrawal fee (`withdrawalFeeBps`, live per leg) applies on exit; protocol fee (`protocolFeeBps`) is taken from LP fees, not principal.
   - Limits: `depositETH()` has **no min-shares / deadline** argument. If the user needs one, stop.
   - Simulation: `eth_call` with the same `from`/`value` must succeed; `TwapUnavailable` or any revert stops the flow.
4. Authorize the exact amount and terms.
5. Send `vault.depositETH()` with `value = wei`, `to` = the vault.
6. Receipt (chain `4663`): `status 0` → reverted, nothing deposited. `status 1` → decode `Deposit(user, wethNotional, shares, acc)` where `address == vault` and topic1 == user; report **shares minted** and **WETH credited** from the event, tx hash, vault, page link. No matching log → outcome unverified, do not guess. No receipt → `pending`.

The vault routes WETH across under-weight legs; the user does not pick a leg.

## Withdraw

`vault.withdraw(shares)` **burns** shares and pays the pro-rata slice of **every** leg (retired included). It aims to pay WETH but the payout can be **mixed**.

**1. Size the shares.** `x` shares → `x × 1e18` if spoken in whole shares; `x%` → `balanceOf(user) × x / 100` with `0 < x ≤ 100`; "all" → `balanceOf(user)`. If `shares == 0` do not send. If `shares > balanceOf(user)` or `balanceOf(user) − shares < 1e10` raw, the vault performs a **full exit** — show the clamped figure and say so before authorization.

**2. Preflight** as for deposit.

**3. Preview** (reads on every leg):
- **Withdrawal fee** `withdrawalFeeBps()` per leg, taken from the user's slice before conversion.
- **Exit floor**: each leg sells pair token → quote → WETH behind a widened TWAP floor (`maxTwapDeviationBps × 3`, haircut `min(swapSlippageBps × 3, 1000)` bps). This is the contract's own per-swap protection, **not** a user slippage limit or loss cap.
- **What can be paid**: any refused hop pays that leg **in kind**, shares still burn. Possible: WETH only; WETH + USDG (6 dec); WETH + pair tokens; or zero WETH and only in-kind tokens.
- **Estimated WETH**: `eth_call vault.withdraw(shares)` from the user → `received` (18 dec); an estimate that excludes in-kind legs.
- **Gas**: `eth_estimateGas`. **Limits**: no min-out, no deadline. If the user requires WETH-only, a minimum, a slippage limit, or a deadline — **stop**.

**4. Authorize** the exact (clamped) share count, fee, and the mixed-payout possibility.

**5. Send** `vault.withdraw(shares)`, `value = 0`.

**6. Receipt.** `status 0` → reverted, no shares burned. `status 1` → from the receipt only:
1. **Shares burned** — `shares` in `Withdraw(user, shares, outAmount, acc)` (`address == vault`, topic1 == user); may differ from the request if clamped.
2. **WETH received** — `outAmount` from the same log; report even when `0`.
3. **Other tokens** — every ERC-20 `Transfer` log with `to == user` and token ≠ WETH: address, label (config sleeve / `symbol()`), raw, and amount scaled by **live `decimals()`** (USDG = 6).
4. Tx hash, vault, page link.

Never say "paid in WETH" when the receipt shows other tokens. Never report only a hash.

## Do not call from this skill

`CofferKeeper.performUpkeep / performHarvest`, `changeAsset`, `exitToQuote`, `setBandParams`, `setTargetWeight`, `setRetired`, `addStrategy`, any ERC-20 `approve` / permit (neither user flow needs one). Coffer has no ShareStaking and no `asAsset` toggle.
