# ABI, selectors and event topics

All addresses come from `config-api.md`. Use the chain of the product / package for every RPC call. `view` unless marked **write**. Verify the 4-byte selector of any calldata before signing.

## Shared ERC-20 / shares

```solidity
function symbol() view returns (string);            // 0x95d89b41
function decimals() view returns (uint8);           // 0x313ce567  — read before scaling any token (USDG = 6)
function totalSupply() view returns (uint256);      // 0x18160ddd
function balanceOf(address) view returns (uint256); // 0x70a08231
event Transfer(address indexed from, address indexed to, uint256 value);
// topic0 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
```

Liquid-share tokens (Coffer and AutoVault) are 18 decimals; `vault.totalSupply()` / `vault.balanceOf()` forward to them.

## Coffer

### CofferVault (`config.coffer.vaults[].vault`)

```solidity
function liquidShares() view returns (address);                        // 0x2eeb7f36
function strategyCount() view returns (uint256);                       // 0x22068b44
function strategies(uint256 i) view returns (address strat, uint16 targetWeightBps, bool retired); // 0xd574ea3d
function balance() view returns (uint256);                             // 0xb69ef8a8  vault value, WETH-notional
function depositETH() payable returns (uint256 shares);                // 0xf6326fb3  write
function withdraw(uint256 shares) returns (uint256 received);          // 0x2e1a7d4d  write
event Deposit(address indexed user, uint256 wethNotional, uint256 shares, uint256 acc);
// topic0 0x36af321ec8d3c75236829c5317affd40ddb308863a1236d2d277a4025cccee1e
event Withdraw(address indexed user, uint256 shares, uint256 outAmount, uint256 acc);
// topic0 0x02f25270a4d87bea75db541cdfe559334a275b4a233520ed6c0a2429667cca94
```

`depositETH`: wraps ETH, routes to under-weight legs, shares priced at `min(spot, TWAP-gated)`, credit capped at WETH deposited; reverts `TwapUnavailable` when a leg's TWAP gate is closed. No caller parameters.
`withdraw`: clamps `shares > balanceOf(user)` to the balance; `balanceOf(user) − shares < 1e10` raw → full exit; takes `withdrawalFeeBps` per leg first; per leg two swap hops (pair → quote, quote → WETH) each with an **in-kind fallback**; `outAmount` is the WETH delta only; one leg reverting reverts the whole call. No caller parameters.

### CofferStrategy (each `strategies(i).strat`; V3 and V4 legs share this surface)

```solidity
function vault() view returns (address);                 // 0xfbfa77cf  must equal the vault
function ASSET() view returns (address);                 // 0x4800d97f  current pair token
function quoteToken() view returns (address);            // 0x217a4b70  WETH or USDG
function poolFee() view returns (uint24);                // 0x089fe6aa
function mode() view returns (uint8);                    // 0x295a5212  0 ACTIVE, 1 IDLE
function poolValue() view returns (uint256);             // 0xb8615071
function UniswapFeesCollected() view returns (uint256);  // 0x2943bcec  cumulative, WETH-notional
function isAllowedToken(address) view returns (bool);    // 0xcbe230c3
function withdrawalFeeBps() view returns (uint256);      // 0x04336bb3  on the user's slice at exit
function protocolFeeBps() view returns (uint256);        // 0x35659fb8  on LP fees, not principal
function protocolFeeOn() view returns (bool);            // 0x8b4dc730
function swapSlippageBps() view returns (uint16);        // 0x2ae1c8ef  exits use min(×3, 1000)
function maxTwapDeviationBps() view returns (uint256);   // 0x51e80e86  exits use ×3
function twapSeconds() view returns (uint32);            // 0x4b85f96c
event AllowedTokenSet(address indexed token, bool allowed);
// topic0 0xa9f6e6144a80408b3df65a073af7832dc126e06e3ace724934f66bc1e1e6c711
```

## AutoVault

### AutoFactory (`config.autovault.packages[].factory`)

```solidity
function registry(address asset) view returns (...);     // 0x038defd7
function assetsLength() view returns (uint256);          // 0xf50477a2
function assets(uint256 i) view returns (address);       // 0xcf35bdd0
```

`registry` returns **5 words on V4** (`strategy, vault, liquidShares, shareStaking, active`) and **6 words on V3 / Sushi** (`strategy, vault, liquidShares, shareStaking, poolFee, active`). Decode by word count: words 0–2 are addresses, the **last** word is `active`; word 3 is `shareStaking` (zero → none); on 6-word returns word 4 is `poolFee`. `vault == 0` → not registered.

### AutoVault (`packages[].vault`)

```solidity
function asset() view returns (address);                               // 0x38d52e0f
function strategy() view returns (address);                            // 0xa8c62e76
function liquidShares() view returns (address);                        // 0x2eeb7f36
function shareStaking() view returns (address);                        // 0x9aee7d47
function balance() view returns (uint256);                             // 0xb69ef8a8  vault value, ETH-notional
function depositETH() payable returns (uint256 shares);                // 0xf6326fb3  write
function withdraw(uint256 shares, bool asAsset) returns (uint256 out); // 0x38d07436  write
event Deposit(address indexed user, uint256 wethNotional, uint256 shares, uint256 acc);
// topic0 0x36af321ec8d3c75236829c5317affd40ddb308863a1236d2d277a4025cccee1e
event Withdraw(address indexed user, uint256 shares, bool indexed asAsset, uint256 outAmount, uint256 acc);
// topic0 0x0017ee693025347648efd221cc811dabd86946b9d9f157510eb39a91e05389d5
```

`withdraw(shares, false)` pays WETH (Base / RhV3 / Sv3) or **native ETH** (RhV4, `withdrawNativeEth: true`); `withdraw(shares, true)` pays the vault's token. No min-out / deadline.

### AutoStrategy (`packages[].strategy`)

```solidity
function UniswapFeesCollected() view returns (uint256);  // 0x2943bcec
function poolValue() view returns (uint256);             // 0xb8615071
function poolFee() view returns (uint24);                // 0x089fe6aa  V3 / Sushi only
function rangeBelowTicks() view returns (uint256);       // 0xe9551fad
function rangeAboveTicks() view returns (uint256);       // 0xd3f6270e
function withdrawalFeeBps() view returns (uint256);      // 0x04336bb3  (if exposed by the package)
```

### ShareStaking (`packages[].shareStaking`, may be null)

```solidity
function balanceOf(address) view returns (uint256);          // 0x70a08231  staked liquid shares
function currentEpoch() view returns (uint256);              // 0x76671808
function epochFinalized(uint256) view returns (bool);        // 0xcafef3b5
function claimed(uint256 epoch, address) view returns (bool);// 0x120aa877
function claim(uint256 epoch) returns (uint256 assetOut);    // 0x379607f5  write — reward in the vault's token
event Claimed(address indexed user, uint256 indexed epoch, uint256 assetAmount);
// topic0 0x987d620f307ff6b94d58743cb7a7509f24071586a77759b77c2d4e29f75a2f9a
```

`stake` / `unstake` need an ERC-20 approval and are out of scope.

## UFloat

### UFloatStrategyFactoryV4 (`config.ufloat.factory`)

```solidity
enum StratMethod { ReBalanceOnly, OffensiveOnly, DefensiveOnly, OffensiveDefensive } // 0..3
function deployStrategy(uint8 stratMethod, address[] tokens) returns (address strategy, uint256 keeperId); // 0x39ecb2cd  write
function getStrategies(address owner) view returns (address[]);                                            // 0x61819d52
event StrategyDeployed(address indexed strategy, address indexed owner, address indexed firstAsset, uint256 keeperId, uint256 allowedTokenCount);
// topic0 0xba04dcce7dfa966e3c7160c26fd13e95a872ff0bacda15763dc4a2a3426414b9
```

### UFloatSwapRouter (`config.ufloat.swapRouter`, preflight only)

```solidity
function hasV4PoolConfig(address asset) view returns (bool); // 0xfdd931c9
```

### UFloatStrategy (clone)

```solidity
function owner() view returns (address);                     // 0x8da5cb5b
function ASSET() view returns (address);                     // 0x4800d97f
function mode() view returns (uint8);                        // 0x295a5212  0 NORMAL, 1 DEFENSIVE, 2 OFFENSIVE, 3 STABLE
function totalValueWeth() view returns (uint256);            // 0x0edaffb1
function poolValue() view returns (uint256);                 // 0xb8615071
function balanceOfIdle() view returns (uint256);             // 0x2e7f60ed
function balanceOfPool() view returns (uint256 assetAmt, uint256 wethAmt); // 0x11588086
function getPositionId() view returns (uint256);             // 0x492bc041
function isAllowedToken(address) view returns (bool);        // 0xcbe230c3
function allowedTokenCount() view returns (uint256);         // 0x7e14d485
function allowedTokens(uint256 i) view returns (address);    // 0x5e5f2e26
function stratMethod() view returns (uint8);                 // 0x2a1c5ac4
function targetAssetBps() view returns (uint256);            // 0xca316f83
function rangeBelowTicks() view returns (uint256);           // 0xe9551fad
function rangeAboveTicks() view returns (uint256);           // 0xd3f6270e
function stopLoss() view returns (uint256);                  // 0x527a1565
function feeReserveBps() view returns (uint256);             // 0xec66fee1
function reserveAddress() view returns (address);            // 0xf79ed94b
function tickSpacing() view returns (int24);                 // 0xd0c93a7c
function minFloorTickCount() view returns (uint256);         // 0xa7cd656f
function offensiveStaleDuration() view returns (uint256);    // 0x62cf5749
function offensiveAssetBps() view returns (uint256);         // 0xbbdfa347
function minRangeBelowTicks() view returns (uint256);        // 0x3732ff27

// owner writes
function depositETH() payable;                               // 0xf6326fb3
function withdrawWeth(uint256 wethAmount);                   // 0xdd0bea13  type(uint256).max = full exit
function addAllowedToken(address token);                     // 0x4178617f
function removeAllowedToken(address token);                  // 0x90469a9d
function changeAsset(address newAsset);                      // 0x10bbb707
function mintPosition(address token);                        // 0x36b0bb1c
function exitToStable();                                     // 0x91d3b523
function setMintParams(uint256 targetAssetBps, uint256 rangeBelowTicks, uint256 rangeAboveTicks, uint256 stopLoss, uint256 feeReserveBps, address reserveAddress); // 0xf77e9aca
function setOffensiveParams(uint256 minFloorTickCount, uint256 offensiveStaleDuration, uint256 offensiveAssetBps, uint256 minRangeBelowTicks); // 0x339dbe7d
function setStratMethod(uint8 method);                       // 0x86386893
```

## Preview recipes (read-only, `from` = user)

| Purpose | Call |
|---|---|
| Chain | `eth_chainId` → `0x2105` (8453) or `0x1237` (4663) |
| Code | `eth_getCode(addr)` non-empty for every contract touched |
| Deposit share estimate | `wei × liquidShares.totalSupply() / vault.balance()` (1:1 if supply 0) |
| Deposit simulation / gas | `eth_call` / `eth_estimateGas` `{ from, to: vault, data: depositETH(), value: wei }` |
| Withdraw estimate / gas | `eth_call` / `eth_estimateGas` `{ from, to: vault, data: withdraw(...) }` → returned amount (estimate) |
| UFloat deploy simulation | `eth_call { from, to: factory, data: deployStrategy(method, tokens) }` |

## Decoding receipts

Only a mined receipt with `status == 1` on the right chain is an outcome. Match logs by `address` (the contract), `topic0` (above) and the indexed user (`topic1`, left-padded). Non-indexed fields are 32-byte words in `data`, in declaration order. For every ERC-20 `Transfer` with `topic2 == user`, group by token address and scale by that token's live `decimals()`. Solidity return values are not in receipts; `eth_call` results are simulations.
