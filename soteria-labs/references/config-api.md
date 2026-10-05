# Config API — the only address source

Host: `https://www.soterialabs.io`. All routes are `GET`, JSON, CORS `*`, no auth.

```
/api/skill-config                              all products
/api/skill-config?product=coffer               one product (or a comma list)
/api/vault-snapshots?vault={vault}&chain={base|robinhood|coffer}&first=200
/api/user-vault-position?vault={vault}&user={user}&chain={base|robinhood|coffer}
```

Fetch `skill-config` at the start of a conversation and again immediately before any write (it is cached ~60 s server-side). Addresses are EIP-55 checksummed. Compare addresses case-insensitively.

## Shape (`schemaVersion` 1)

```jsonc
{
  "schemaVersion": 1,
  "generatedAt": "ISO-8601",
  "site": "https://www.soterialabs.io",
  "chains": {
    "8453": { "name": "Base",      "weth": "0x4200…0006", "explorer": "https://basescan.org" },
    "4663": { "name": "Robinhood", "weth": "0x0Bd7D308…", "usdg": "0x5fc5360D…", "explorer": "https://robinhoodchain.blockscout.com" }
  },
  "autovault": {
    "factories": { "Bv3": { "chainId": 8453, "address": "0x…" }, "Bv4": {…}, "RhV3": {…}, "RhV4": {…}, "Sv3": {…} },
    "packages": [ {
      "name": "surplus",            // lower-case ERC-20 symbol — what the user types
      "symbol": "Surplus",
      "pkg": "Bv4",                 // Bv3 | Bv4 | RhV3 | RhV4 | Sv3
      "chainId": 8453,
      "dex": "uniswap-v4",          // uniswap-v3 | uniswap-v4 | sushiswap-v3
      "factory": "0x…", "asset": "0x…", "vault": "0x…", "strategy": "0x…",
      "liquidShares": "0x…", "shareStaking": "0x…" | null,
      "poolFee": 10000 | null,      // V3 / Sushi only
      "active": true,               // only active packages are listed
      "withdrawNativeEth": false,   // true on RhV4: withdraw(shares,false) pays native ETH
      "href": "/dapp/auto-vaults-base/0x…",
      "snapshotsChain": "base"      // value for /api/vault-snapshots?chain=
    } ]
  },
  "coffer": {
    "chainId": 4663, "keeper": "0x…",
    "vaults": [ {
      "id": "vault3", "label": "COFFER · MU · AMZN · MEME",
      "vault": "0x…", "liquidShares": "0x…", "keeper": "0x…",
      "href": "/dapp/coffer/0x…", "snapshotsChain": "coffer",
      "sleeves": [ { "id": "mu", "label": "MU", "kind": "v3" | "v4", "strategy": "0x…", "asset": "0x…", "quote": "0x…", "targetWeightBps": 3334 } ]
    } ]
  },
  "ufloat": {
    "chainId": 8453, "factory": "0x…", "swapRouter": "0x…", "keeper": "0x…", "weth": "0x4200…0006",
    "defaultDeployTokens": ["0x…"],
    "hrefStrategy": "/dapp/u-strategy/{strategy}", "hrefDirectory": "/dapp/u-strat-dir"
  },
  "api": { "skillConfig": "…", "vaultSnapshots": "…", "userVaultPosition": "…" },
  "errors": { "autovault": "…" }   // present only when a product failed to build; treat it as unavailable
}
```

Page link = `site + href` (replace `{strategy}` for UFloat). Append `?tab=claim` for AutoVault claim UX.

## Resolving what the user said

1. Normalize: lower-case, trim, strip a trailing `vault`, `pool`, `token`, `strategy`.
2. **Coffer**: match `coffer.vaults[].id` (`vault3`, `vault 8`, `8`), a word in `label` (`nasdaq`, `multi-sector`, `sgov`), a sleeve `label` (`qqq`, `gld`), or the vault address. Several matches or none → list `id · label · vault` and ask.
3. **AutoVault**: match `autovault.packages[].name` or `symbol` (case-insensitive) or `asset` / `vault` address. One name can exist on two packages (e.g. a token on both Bv3 and Bv4) → list both with `pkg` and chain and ask. No match → say it is not a live package; offer the list. Do **not** scan factories by hand — a token on a retired factory is intentionally absent.
4. **UFloat**: a strategy address the user pasted, or `factory.getStrategies(user)` for the connected wallet. Never invent one.
5. A raw `0x` address with no product → try Coffer vault, then AutoVault vault/asset, then UFloat strategy (`owner()` + `totalValueWeth()` succeed) and confirm with the user.

## Wiring checks (run before any write; cheap, all `eth_call`)

Coffer vault `V` from config entry `c`:
- `V.liquidShares() == c.liquidShares`; `V.strategyCount() > 0`; for every `i`, `V.strategies(i).strat.vault() == V`.
- Chain `4663`; `eth_getCode` non-empty for `V`, `c.liquidShares`, WETH.

AutoVault package `p`:
- `factory(p.factory).registry(p.asset)` → `vault == p.vault`, `strategy == p.strategy`, `liquidShares == p.liquidShares`, `active == true` (decode by word count, see `abi.md`).
- `vault.liquidShares() == p.liquidShares`, `vault.strategy() == p.strategy`, `vault.asset() == p.asset`.
- Chain `p.chainId`; code non-empty for vault, strategy, liquidShares, (shareStaking if used).

UFloat strategy `S` (user-pasted or from `getStrategies`):
- Code non-empty for `S`, `ufloat.factory`, `ufloat.swapRouter`; `S.owner() == user` for owner writes.
- For deploy: `ufloat.swapRouter.hasV4PoolConfig(token) == true` for every token.

Any mismatch → stop; tell the user the live wiring does not match the published config and nothing was sent.

## When the config is down

Say live listing is unavailable. Accept only a user-pasted vault / strategy address, run the wiring checks above with the values **read from chain** (`vault.liquidShares()`, `vault.strategy()`, `strategy.vault()`), and proceed only if they are self-consistent. Never fall back to a remembered address table.

## Snapshots → APR

`GET /api/vault-snapshots?vault={vault}&chain={snapshotsChain}&first=200[&since={unix}]` → `{ vault, chain, snapshots: [{ valueWeth, fees, ts }] }` — `valueWeth` and `fees` (cumulative) are ETH decimal strings, `ts` unix seconds. Sort by `ts`, take the trailing ~7 days, sum `Δfees / valueWeth` per interval, annualize by `31,536,000 / elapsedSeconds`, report one decimal. Fewer than 2 points → no APR; report fees + vault value + page link instead.

`GET /api/user-vault-position?vault&user&chain` → `{ position: { shares, feesAccruedWeth, lifetimeFeesWeth, vault: { totalShares }, flows: [{ kind: DEPOSIT|WITHDRAW, shares, timestamp, … }] } }` (wei strings). Use it for "what have I earned"; `503` means the index is not configured for that chain — fall back to `liquidShares.balanceOf(user)` and say earnings history is unavailable.
