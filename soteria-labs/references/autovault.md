# AutoVault — one vault per token (Base `8453` · Robinhood `4663`)

Read `SKILL.md` Safety first. Packages: `config.autovault.packages` (see `config-api.md`). Calls: `abi.md`.

**Model.** Each AutoVault package manages one token against WETH on one DEX: `Bv3` / `Bv4` (Base, Uniswap V3 / V4), `RhV3` / `RhV4` (Robinhood, Uniswap V3 / V4), `Sv3` (Robinhood, SushiSwap V3). A package = `asset`, `vault`, `strategy`, `liquidShares`, optional `shareStaking`. **Names (`surplus`, `bnkr`, `cashcat`, `nvda` …) are the token's ERC-20 symbol**, lower-cased, exactly as the config returns them; they select chain and package by themselves — never ask the user for chain or version.

Only **active** packages are in the config. A name that is not there is not a live vault (it may have been retired when the factory rotated). Say so and offer the list; do not scan factories or reuse an older address.

## Resolve

`config-api.md` §Resolving step 3. If the same name exists on two packages, show both (`pkg`, chain, vault) and ask. Always echo **name · pkg · chain · vault · page link** (`site + href`; add `?tab=claim` for claim UX).

## Reads

| User asks | Call | Notes |
|---|---|---|
| vault value / total value of surplus | `vault.balance()` | ETH-notional, 18 dec |
| fees earned | `strategy.UniswapFeesCollected()` | cumulative lifetime |
| total liquid shares | `liquidShares.totalSupply()` | not `vault.totalSupply()` |
| my liquid shares | `liquidShares.balanceOf(user)` | |
| my staked shares | `shareStaking.balanceOf(user)` | only if `shareStaking != null` |
| current band / range | `strategy.rangeBelowTicks()`, `rangeAboveTicks()` | optional, when asked |
| APR | `vault-snapshots?vault&chain={snapshotsChain}` → 7-day fee APR | `config-api.md`; else fees + value + link |
| what have I earned | `user-vault-position?chain={snapshotsChain}` | `503` → say history unavailable |
| what vaults exist | list `packages` grouped by chain: name, pkg, link | |

## Deposit

"Deposit 0.01 ETH into surplus"

1. Resolve the package; `active` is guaranteed by the config but re-check `factory.registry(asset)` in preflight.
2. Amount must be ETH (convert and show if the user gave USD). Preflight: chain `p.chainId`, code checks, AutoVault wiring checks.
3. **Preview**: ETH to send (wei + human), gas, estimated shares `wei × liquidShares.totalSupply() / vault.balance()` (estimate; TWAP-gated pricing can mint fewer), fees (none on deposit; withdrawal fee and protocol fee are strategy settings — read `withdrawalFeeBps()` live if the strategy exposes it, otherwise say fees are shown on the vault page), simulation must succeed. No min-shares / deadline argument exists.
4. Authorize → send `vault.depositETH()` with `value = wei`.
5. Receipt: `Deposit(user, wethNotional, shares, acc)` on the vault → shares minted, WETH credited, hash, link. Status 0 → reverted. Missing log → unverified.

## Claim / withdraw

Liquid shares are the claim ticket. `vault.withdraw(shares, asAsset)` **burns** them and pays:

| Package | `asAsset = false` | `asAsset = true` |
|---|---|---|
| Bv3 / Bv4 | WETH (ERC-20) | the vault's token |
| RhV3 / Sv3 | WETH (aeWETH ERC-20) | the vault's token |
| RhV4 (`withdrawNativeEth: true`) | **native ETH** | the vault's token |

Default `asAsset = false`. "as token", "as ASSET", or naming the vault's token → `true`.

Sizing: `x` shares → `x × 1e18` if spoken in whole shares; `x%` → `balanceOf(user) × x / 100`, `0 < x ≤ 100`; "all" → `balanceOf(user)`. Cap at balance; `0` → do not send. Disclose the cap before authorization.

**Preview**: shares to burn, out token (per table), estimated out via `eth_call vault.withdraw(shares, asAsset)` from the user (estimate), gas, simulation must succeed. No min-out / deadline argument exists — stop if the user requires one.

**Send** `vault.withdraw(shares, asAsset)`, `value = 0`.

**Receipt**: `Withdraw(user, shares, asAsset, outAmount, acc)` on the vault (topic1 user, topic2 asAsset). Report **shares burned**, **token received** (WETH / ETH / token name + address), **amount** (`outAmount`, raw + human by live `decimals()`), hash, name, pkg, chain, vault, link. If the log is missing, report the user's WETH / ETH / token balance delta across the tx as received and say the event was not found. Never only a hash.

## ShareStaking (only when the user asks about staking / epoch rewards)

`shareStaking` (from the package) holds staked liquid shares and pays epoch rewards in the **vault's token**. This skill may read `balanceOf(user)`, `currentEpoch()`, `epochFinalized(epoch)`, `claimed(epoch, user)` and send `claim(epoch)` for a finalized epoch `< currentEpoch`. Receipt: `Claimed(user, epoch, assetAmount)`. Staking / unstaking needs an ERC-20 approval and is **not** done here — send the user to the page link. Do not confuse epoch `claim` with vault withdraw; it burns no shares.

## Do not call from this skill

Factory `deployVaultPackage` / setters, keepers (`performUpkeep`, `performHarvest`), strategy owner functions (`setBandParams`, rotation), `stake` / `unstake`, any ERC-20 `approve` / permit. Non-ETH deposit paths are out of scope.
