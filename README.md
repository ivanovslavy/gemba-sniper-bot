# GEMBA Sniper Bot

Professional DeFi sniper bot offered as a SaaS product. Each subscriber gets their own bot instance running on a dedicated wallet, controlled through a React dashboard, with a layered honeypot defense pipeline that aborts trades before real ETH is committed wherever possible.

The product runs on Ethereum mainnet, Base mainnet, and Sepolia (testnet). It targets Uniswap V3 and Uniswap V4 pools (the V4 path includes hook-honeypot defenses that the V3 path does not need).

---

## What the bot does

1. **Detects new liquidity** by subscribing to `PoolCreated` (V3 factory) and `Initialize` (V4 PoolManager) events over WebSockets, plus a mempool listener for pending `addLiquidity`/`createPool` transactions.
2. **Filters the opportunity** through a multi-stage detection pipeline (described below). Anything that looks like a honeypot, rug-pull template, or thin-liquidity trap is rejected before the buy fires.
3. **Probes the pool with a $0.03 round-trip** (micro test buy + sell on the real chain) so any bypass that survived static analysis still has to survive a real on-chain trade.
4. **Executes a real buy** via Uniswap Universal Router (V4) or SwapRouter02 / FeeRouter (V3), sized by the user's `max_buy_eth` (or `stablecoin_buy_amount` for stable-paired pools).
5. **Manages the open position** with real-time price tracking (Swap-event subscription), take-profit, stop-loss, timeout, and partial-sell exits.
6. **Recovers stuck positions** with a separate sweeper that runs every 30 minutes (`scripts/auto_unstuck.js` via systemd timer), with a configurable age gate and write-off detection for drained pools and gas-trap tokens.
7. **Reports everything** through Telegram notifications and a React dashboard with live activity feed, analytics, and per-network status.

---

## Architecture

Two long-running Node processes plus a separate dashboard process.

```
                          gemba-sniper-engine.service
                         (Node ESM, no HTTP listener)
                                    |
              +---------------------+---------------------+
              |                                           |
   poolDetector  mempoolMonitor               positionManager  walletManager
   (WS subscribe)  (pending tx)              (price + exits)   (decrypt PK)
              |                                           |
              v                                           v
        sniper._onOpportunity --------> executor.buy / sell --------> chain
        (detection pipeline)            (V3 SwapRouter / V4 UR)

                                                          ^
                                                          |
                          gemba-sniper-api.service        |
                        (Node ESM, Express on :3010)      |
                                    |                     |
        Dashboard <----JWT/HTTPS----+--- /bot, /settings, /trades, ...
        (Vite preview on :3012)               + Apache reverse proxy

                                            PostgreSQL on :5432
                                  (shared by API + Engine + auto_unstuck)
```

The engine and the API never call each other directly. Their contract is the `bot_instances` table: the API writes `is_running = true`, the engine polls that table every 10 s, picks up the row, instantiates a Sniper, and writes `last_heartbeat` every 60 s. Stopping is symmetric: API flips `is_running = false`, engine sees it on the next poll, tears the Sniper down.

A third process — `gemba-sniper-dashboard.service` — runs the Vite preview build of the React dashboard on port 3012. It is served behind Apache as `sniper.gembabots.com` and talks to the API at `api-sniper.gembabots.com`.

### Per-network bot instance

The bot is keyed by `(user_id, network)`. A user can have one instance on Ethereum, one on Base, one on Sepolia in parallel — each consumes its own slot in `bot_instances`, its own WebSocket subscriptions, and its own position book. The Sniper class is instantiated once per row.

### Sibling repositories

| Repository | Purpose |
|---|---|
| `~/projects/gemba-sniper-bot/` | This repo — Node backend (engine + API + DB + infrastructure) |
| `~/projects/gemba-sniper-dashboard/` | React + Vite dashboard, served at `sniper.gembabots.com` |
| `~/projects/test-hook-sepolia/` | Foundry project with controlled honeypot hooks used to red-team the defenses |
| `contracts/` (inside this repo) | Hardhat workspace for the `FeeRouter` contract that wraps SwapRouter02 |

---

## Defense pipeline

Every pool detection runs through the same ordered set of checks in `src/bot/sniper.js` (`_onOpportunity`). Each check is wired to a Telegram pipeline-report and a dashboard event so the operator sees exactly which layer rejected (or admitted) a token.

The checks are numbered with decimals so retro-fitted layers can slot in between existing checks without renumbering. Defense **layers** (A, B, C, D, E, F) are named per the RED_TEAM playbook and may span multiple checks.

```
Check 0.9  Clanker-pattern guard          (Base only; hooks=0x0, fee=0, tickSpacing=1)
Check 1    Fee tier cap                   (user max_fee_percent)
Check 2    V4 dynamic-fee re-check        (PoolManager.getSlot0 lpFee)
Check 3    Liquidity USD floor            (LiquidityChecker)
Check 3.4  Hook blacklist precheck         [Layer A entry]   (db.getHookBlacklist*)
Check 3.5  V4 hook capability + bytecode  [Layer A scanner]  (HookAnalyzer + hookBytecodeScanner)
Check 3.6  Delayed-tax probe              [FIX 34]           (DelayedTaxProbe.probe — multi-horizon, Day 9)
Check 3.7  Caller-spoof probe             [Layer B]          (WalletSimProbe via eth_simulateV1)
Check 4    Bytecode honeypot heuristics                      (ContractAnalyzer)
Check 5    Honeypot detector (eth_call sim)                  (HoneypotDetector — cache-first)
Check 6    Transfer simulator                                (TransferSimulator — eth_call + stateOverride)
Check 7    Micro-test buy + sell                             (executor.microTestBuy — $0.03 round-trip)
Check 7.5  Post-microsell re-probe         [Layer C]         (DelayedTaxProbe.reprobeAfterMicroSell)
Check 7.6  Gas-trap probe                  [Layer E]         (GasTrapProbe — eth_estimateGas with future block)
Check 8    Real-size vs micro-test pro-rata                  (sniper._guardRealBuyVsMicro)

   -> REAL BUY fires only if every gate above passes
   -> Layer D phantom-TP guard at sell time                  (executor._sellV4)
   -> Layer E2 sell-side gas re-check                        (executor._sellV4, Day 8)
   -> Layer F LP-drop monitor (every 30 s on open positions) (positionManager.checkAllPositions, Day 8)
```

### What each layer catches

**Check 0.9 — Clanker-pattern guard (Day 8).** Skips V4 pools matching `hooks=0x0 AND fee=0 AND tickSpacing=1` on networks listed in `config.bot.clankerGuardNetworks` (default: `['base']`). Configured launchpad template that consistently rug-pulls 2-3 minutes after launch. Zero RPC cost on detection. Trade-off: a small number of legitimate Clanker tokens get rejected.

**Check 3 — Liquidity floor.** Reads PoolManager state for V4 or pool reserves for V3; rejects pools below the user's `min_liquidity_usd` (default 2000 USD; setting to 0 disables the gate).

**Check 3.4 — Hook blacklist precheck.** Looks up the hook address AND its bytecode hash in `hook_blacklist`. A hit aborts the pipeline immediately — no probes, no micro-test, no gas spent. Catches re-deploys of known traps even when the attacker uses a fresh CREATE2 salt to mint a new hook address (because the bytecode hash is stable).

**Check 3.5 — Layer A bytecode scanner (Day 2 + Day 5 + Day 9).** `src/detection/hookBytecodeScanner.js` walks the hook runtime bytecode, skipping PUSH data bytes, and scores against the patterns below. The score is added to a base "flag risk" (derived from which permission bits the hook has) and an "unknown deployment" penalty. Soft-skip if combined risk crosses thresholds: 80 for hooks with critical permissions (ReturnDelta) or 100 otherwise.

  | Pattern | Score | Catches |
  |---|---|---|
  | `KNOWN_ADDRESS_EMBEDDED` (PUSH20 of Quoter/UR) | +70 | v9 canonical Quoter-bypass |
  | `SSTORE_PRESENT` | +10 | noise floor for state-mutating hooks |
  | `SSTORE_BEFORESWAP_ONLY` | +60 | v10 swap-count trap (state-mutating callback only) |
  | `ORIGIN` | +30 | `tx.origin` deployer whitelist |
  | `COINBASE` | +35 | `block.coinbase` probe asymmetry |
  | `GAS` (gasleft) | +35 | gasleft() probe asymmetry |
  | `BASEFEE` | +30 | basefee probe asymmetry |
  | `BLOCKHASH` | +30 | blockhash probe asymmetry |
  | `GASLIMIT` | +30 | gaslimit probe asymmetry |
  | `NUMBER` (Day 9) | +15 | `block.number` time-gating signal |
  | `TIMESTAMP` (Day 9) | +15 | `block.timestamp` time-gating signal |
  | `GRACE_PATTERN` (Day 9) | +40 | SLOAD followed by NUMBER/TIMESTAMP within 16 ops — v7/v16 launchBlock + grace fingerprint |

  Detection writes the hook into `hook_blacklist` with reason `bytecode_heuristic` so subsequent encounters short-circuit at Check 3.4.

**Check 3.6 — Delayed-tax probe (FIX 34, multi-horizon since Day 9).** Calls V4 Quoter twice for each swap direction:

  - `latest` block — establishes baseline `currentOut`
  - Each future horizon in `config.networks[net].delayTaxProbeHorizons` (sepolia/ethereum: `[50, 300, 1500]`; base: `[300, 1800, 9000]`) — uses `eth_call` with `BlockOverrides` `{number, time}` to simulate the same swap at that future block

  Returns the worst (highest) drop across all horizons. A drop above 5% on either swap direction marks the hook as a delayed-tax honeypot. Pre-Day-9 the probe used a single horizon (50 blocks on sepolia) and could be bypassed by any grace longer than that. The multi-horizon variant defeats any grace shorter than the largest horizon (~5 hours of wall clock on every chain).

**Check 3.7 — Layer B caller-spoof (Day 4).** Compares two Quoter quotes: one from the canonical Quoter address (Check 3.6 baseline), one from a synthetic Quoter that the bot constructs via `eth_simulateV1` with `stateOverrides[SPOOFED_ADDR].code = getCode(realQuoter)`. For clean hooks the quotes match. For caller-discriminating hooks (canonical v9 OR obfuscated v13 where the Quoter address sits in a storage slot instead of as PUSH20) the spoofed quote drops by ~99% and the trap is detected. Silently skipped if the RPC does not support `eth_simulateV1`.

**Check 4 — ContractAnalyzer.** Bytecode-level heuristics on the **token** contract (not the hook): proxy patterns, hidden external calls, dangerous admin selectors (`setTax`, `pauseTrading`, etc.). Score-based; user threshold default 40. Hard-scam toggles (`hard_scam_on_proxy`, `hard_scam_on_hidden`, `hard_scam_on_dangerous_admin`) escalate any single hit to an immediate SCAM regardless of score.

**Check 5 — HoneypotDetector.** Cache-first: looks up `analyzed_tokens` for prior verdicts before running fresh `eth_call` simulations of buy + sell. Combines several signals into a score; verdict ranges are SAFE (≤20), SUSPICIOUS (between SAFE and the honeypot threshold), HONEYPOT (≥ user `honeypot_score_threshold`, default 40).

**Check 6 — TransferSimulator.** `eth_call` with `stateOverride` to grant the bot wallet a synthetic token balance, then simulates `transfer` and `approve`. Detects buy/sell tax asymmetry, blocked sells, and amount-manipulation patterns. Toggleable via `transfer_sim_enabled`.

**Check 7 — Micro-test buy + sell.** A real on-chain round-trip with `microTestAmountEth` (default 0.00001 ETH ≈ $0.03) or `microTestAmountStable` ($0.01) for stable-paired pools. Fails if either leg reverts. If the round-trip loss exceeds `micro_test_max_loss_percent` (default 5%), the token is marked as a hook honeypot and the real buy is aborted. Retry policy: up to 3 attempts when the failure looks transient (pool not yet seeded, hook anti-snipe gate active). Toggleable via `micro_test_enabled`.

  Failed sells leave the dust in `orphan_balances` so the user can recover it manually from the Wallet tab.

**Check 7.5 — Layer C post-microsell re-probe.** After the micro-test sell confirms, `DelayedTaxProbe.reprobeAfterMicroSell` quotes the SELL direction again. The micro-sell mutated on-chain storage; if a v10-style swap-count trap was watching, it is now armed. A drop > 5% versus the original Check 3.6 baseline aborts the real buy and blacklists the hook with reason `post_microsell_trap_armed`.

**Check 7.6 — Layer E gas-trap probe (Day 7).** `eth_estimateGas` of the planned real-size sell with `stateOverride` to grant the bot the projected token balance, plus a second estimate with `BlockOverrides` set 300 blocks into the future (catches time-gated traps that arm post-launch). Threshold 5,000,000 gas. Skipped if the RPC does not support `stateOverride` on `eth_estimateGas`.

**Check 8 — Real-size vs micro-test guard.** Compares the Quoter's real-size quote to the pro-rata extrapolation from the micro-test rate. If the real-size buy would receive < 70% of the pro-rata amount (configurable via `defaultMaxBuyVsMicroSlippagePercent` = 30), the pool is thin-liquidity-trapped and the buy aborts.

**Layer D phantom-TP guard at sell time (executor._sellV4).** Extended in Day 9 (Fix B) to cover all non-forced sell reasons:

  - `TAKE_PROFIT` / `PARTIAL_TP`: strict — Quoter quote ≤ pro-rata breakeven → hold position + blacklist hook with reason `phantom_tp_at_sell`
  - `TIMEOUT` / `STOP_LOSS`: wide — quote < 30% of breakeven → hold + blacklist (the user's normal SL exit accepts up to 10–20% loss, so 30% sits well below normal acceptable loss)
  - `MANUAL` / `AUTO_UNSTUCK` / `LP_DROP`: skip (user explicitly wants a forced exit, even at a loss)

**Layer E2 sell-side gas re-check (Day 8, executor._sellV4).** Before submitting a real sell transaction, the executor's existing `provider.estimateGas` is interpreted. If it returns > 5,000,000 gas or throws with a gas-trap error pattern, the executor writes the token to `analyzed_tokens` as HONEYPOT (reason `gas_trap_at_sell`, layer E2), sends a Telegram alert, and throws an error with the prefix `GAS_TRAP_AT_SELL`. `scripts/auto_unstuck.js` recognizes that marker and writes off the position on the first attempt instead of looping through 3 retries.

**Layer F LP-drop monitor (Day 8, positionManager.checkAllPositions).** Every 30 s the position manager calls `PoolManager.getLiquidity(poolId)` for each open V4 position and compares to the buy-time baseline stored in `trades.initial_liquidity`. If current liquidity drops below `(initial × lpDropThresholdPercent / 100)` (default 30%), the manager flags `_lpDropFired = true` (one-shot), fires an emergency sell with reason `LP_DROP`, and sends a Telegram alert via `notifier.sendLpDropAlert`. Catches deployer LP rug-pulls within the 30 s tick window. Skipped for V3 positions and for any row without a baseline.

### Write-on-detect

Every defense layer writes its catch into `hook_blacklist` (with a reason enum value) so the next encounter — whether by the same user or another, whether by a different token with the same hook address or a re-deployed hook with the same bytecode — is short-circuited at Check 3.4. The pipeline therefore compounds: a defense that catches a class of trap once makes every future instance free to detect.

| Layer / check | hook_blacklist reason | Side effect |
|---|---|---|
| Check 3.5 softSkip | `bytecode_heuristic` | + analyzed_tokens HONEYPOT |
| Check 3.6 detected | `delayed_tax_detected` | + analyzed_tokens HONEYPOT |
| Check 3.7 detected | `caller_discrimination` | + analyzed_tokens HONEYPOT |
| Check 7.5 trap armed | `post_microsell_trap_armed` | + analyzed_tokens HONEYPOT |
| Check 7.6 gas trap | (none — no hook) | analyzed_tokens HONEYPOT only |
| Layer D phantom-TP | `phantom_tp_at_sell` | + analyzed_tokens HONEYPOT (first sighting only) |
| Layer E2 sell-side gas | (none — no hook) | analyzed_tokens HONEYPOT only |

### Two paths around the pipeline

- **Trusted tokens** (`trusted_tokens` table, dashboard tab "Trusted Tokens"): per-user whitelist. `check_mode = 'skip_checks'` bypasses the entire pipeline; `'run_checks'` runs the pipeline with relaxed thresholds.
- **Watch-only mode** (`user_settings.watch_only = true`): bot analyzes every detection, sends a `WatchAlert` Telegram event, but never fires a real buy. Useful for measurement before committing to autonomous trading.

---

## Trade execution

### Buy path

**V3.** `executor.buy(opportunity, settings, quoteToken)` resolves the quote token (default WETH, accepts USDC/USDT/EURC/DAI for stable pairs), pre-flight checks the wallet balance, approves the SwapRouter (or FeeRouter) for the exact `amountIn`, fetches a Quoter quote, applies `amountOutMinimum = quote × (1 - slippage%)`, estimates gas with a 50% buffer (250k floor), submits `exactInputSingle`, waits 2 confirmations, parses the `Transfer` log to confirm tokens received. Sepolia uses `amountOutMinimum = 0` since testnet pools are too thin to enforce slippage meaningfully.

**V4.** `_buyV4` resolves which currency the pool considers `currency0` vs `currency1` by hashing the PoolKey against the on-chain `poolId`; if WETH and native-ETH both look plausible it tries both. Builds a Universal Router command sequence:

- ETH pair with WETH-side pool: `WRAP_ETH(UR address) + V4_SWAP`
- Native-ETH pool (currency = `0x0`): `V4_SWAP` only (msg.value funds the SETTLE)
- Stable pair: `V4_SWAP` only (Permit2-approved stable supplies the input)

V4_SWAP actions: `SWAP_EXACT_IN_SINGLE + SETTLE/SETTLE_ALL + TAKE_ALL`. `amountOutMin` comes from `opportunity._realQuoteAmount` (the sniper's Check-8 quote × slippage). Definitive reverts (`PoolNotInitialized`, `HookCallFailed`, `PriceLimitAlreadyExceeded`, `CurrencyNotSettled`) abort the buy immediately; opaque reverts fall back to a 500k gas floor.

**Permit2 chain (FIX 19).** Both V4 buy and sell use a cached allowance check that skips re-approval when the on-chain values are already at max. First trade per token pays the two approvals (token → Permit2, Permit2 → UR); subsequent trades go straight to the swap until the 30-day Permit2 grace lapses.

**Stable-paired buys.** `sniper.ensureQuoteTokenBalance` pre-funds the bot wallet with the required stablecoin via an `exactOutputSingle` ETH → stable swap on the canonical 0.05% tier pool, then refunds leftover ETH via `refundETH`. Slippage applied to the input side.

### Sell path

**V3.** Probes fee tiers `[500, 3000, 10000]` plus `opportunity.fee` and picks the one with the highest quoted output. If the network has a FeeRouter deployment and the call is not a micro test, the sell routes through FeeRouter (which deducts a 1% platform fee inside the same transaction); otherwise direct SwapRouter02. Approval is per-sell, then `exactInputSingle`, then gas + 50% buffer, then submit. After a WETH-side sell, an `auto-unwrap` step calls `WETH.withdraw` for any balance over `10^-5` ETH so the wallet holds native ETH.

**V4 (`_sellV4`).** Reconstructs the PoolKey, fetches a Quoter quote, applies the **trap detector** (Layer D + Day 9 Fix B — see above) which may abort the sell and hold the position. If the quote passes, applies `amountOutMin = quote × (1 - slippage%)`, then `provider.estimateGas` which doubles as **Layer E2** (Day 8 sell-side gas re-check) — a return value above 5,000,000 or a gas-trap error pattern aborts with `SELL_GAS_TRAP_MARKER`. Builds the same SWAP_EXACT_IN_SINGLE + SETTLE_ALL + TAKE_ALL action bundle and submits via UR.

### Micro-test buy + sell

`executor.microTestBuy` spends `microTestAmountEth` (default 0.00001 ETH) or `microTestAmountStable` ($0.01) on a real on-chain buy, waits 3 blocks, then attempts up to 2 sells with 3-block gaps. If both sells revert, the leftover token balance is logged to `orphan_balances` for manual recovery from the Wallet tab. `WrappedError` (V4 hook revert blob) is decoded to expose which callback rejected, so the SKIPPED reason carries the hook address and callback name. Failure modes are surfaced as flags: `isHoneypot`, `isBuyFail`, `isPoolNotInit`, `isPoolEmpty`, `isHookReject`, `hook`.

### Exit reasons

| Reason | Trigger | Threshold |
|---|---|---|
| `TAKE_PROFIT` | priceChangePercent ≥ `take_profit_percent` | user setting (default 50%) |
| `PARTIAL_TP` | TP hit and `sell_all_at_once = false` and not yet partially sold | sells `partial_sell_percent` (default 50%) |
| `STOP_LOSS` | priceChangePercent ≤ `-stop_loss_percent` | user setting (default 20%) |
| `TIMEOUT` | `timeoutAt` reached and abs price change < 2% in the window | `position_timeout_minutes` (default 5; resets on >2% moves) |
| `LP_DROP` | Layer F: live `getLiquidity(poolId)` < `initial × lpDropThresholdPercent` | default 30% |
| `MANUAL` | user-initiated sell | — |
| `AUTO_UNSTUCK` | sweeper script forcing an exit | runs every 30 min via systemd timer |

### Wallet key handling

Trading keys are stored in `user_wallets` as AES-256-GCM ciphertext. The wrapping key is derived in the browser via `PBKDF2(emailLowercased, "gemba-sniper", 100k iterations, 32 bytes)` and is never sent to the server in plaintext. The Executor decrypts at instantiation:

1. Derive wrapping key from `user.email.toLowerCase()`
2. Split the stored hex into ciphertext (everything but the last 16 bytes) and auth tag (last 16 bytes)
3. `createDecipheriv('aes-256-gcm', key, iv) + setAuthTag + update + final`
4. Plaintext goes to `new ethers.Wallet(...)`; the local string drops out of scope

FIX 16 (2026-05-30) serializes all `wallet.sendTransaction` calls through a promise chain so concurrent pipelines do not collide on the same nonce.

---

## Dashboard

`gemba-sniper-dashboard` is a React + Vite SPA served at `sniper.gembabots.com`. The visual language uses CSS-variable design tokens that match the `gembatools.io` product family (no Tailwind). Login is email-only with a 6-digit OTP delivered over SMTP and protected by Cloudflare Turnstile.

The dashboard has a flat horizontal navbar with the following tabs (in order):

| Tab | What it shows | API endpoints |
|---|---|---|
| Overview | Three BotCards (Sepolia / Ethereum / Base). Each shows running status, last heartbeat, open positions, last 10 trades, Start/Stop buttons. | `/bot/status`, `/bot/positions`, `/bot/start`, `/bot/stop`, `/trades?limit=10`, `/settings` |
| Live (Activity) | Real-time event feed: pipeline reports, contract analysis, honeypot analysis, transfer sim, micro-test results, buy/sell/position events. Filters: network and event type. | `/activity/recent`, `/activity/stream` (SSE) |
| Analytics | Per-network PnL chart (Recharts), summary cards (total PnL, ROI%, win rate), paginated trade list (20/page). Date range selectable; default 7 days. | `/trades/analytics?network=&from=&to=` |
| Analyzer | One-off token analysis form. Returns verdict (SAFE / SUSPICIOUS / HONEYPOT) with full audit trail. Lists the 50 most recently analyzed tokens. | `/token/analyze`, `/token/analyzed`, `/trusted` |
| Watchlist | Per-user watched tokens. When `watch_only` is on the bot only alerts on these; when off the bot also buys (still subject to the full pipeline). | `/watchlist`, `/watchlist` (POST), `/watchlist/:addr?network=` (DELETE) |
| Settings | Trading-side configuration (see Settings tab fields below) and a button to send a Telegram test message. | `/settings`, `/notifications/test` |
| Risk | Risk-side configuration: score thresholds, hard-scam toggles, micro-test overrides, "Reset to defaults" preview/confirm. | `/settings`, `/settings/reset-risk` |
| Wallet | Setup (generate a new wallet OR import a private key, encrypted client-side before sending) and view (address, balance, network selector). Lists orphan balances with a "Resolve" action. | `/wallet`, `/wallet/encrypt`, `/wallet/orphans`, `/wallet/orphans/:id/resolve` |
| Unstuck | Auto-unstuck job audit trail and configuration: master toggle, age-gate mode (Aggressive 15 min / Smart timeout+10 / Conservative 60 min), Run Now button. | `/unstuck/settings`, `/unstuck/attempts?limit=100`, `/unstuck/run-now` |
| Threats | Operator-facing view of `hook_blacklist` and HONEYPOT-verdict `analyzed_tokens`. Network filter, refresh button, reason-to-layer mapping shows which defense layer fired. Read-only — manual un-blacklisting is done via `psql`. | `/threats?network=&limit=` |
| Trusted Tokens | Per-user whitelist. Each entry has a `check_mode` (`skip_checks` bypasses everything, `run_checks` runs the pipeline with relaxed thresholds). | `/trusted`, `/trusted` (POST), `/trusted/:addr?network=` (DELETE) |
| Plans | Subscription plan grid (Trial, Daily, Weekly, Monthly, Annual). Per-network status cards. Free-trial activation (one-shot, only when no other active subscription). Checkout flow with 5 s payment polling and 10-min timeout. | `/subscriptions/plans`, `/subscriptions/status`, `/subscriptions/checkout`, `/subscriptions/activate-free-trial` |

### Settings tab — user-configurable fields

| Field | Type | Bounds | Default | Purpose |
|---|---|---|---|---|
| Max Buy ETH | number | 0.001–10 | 0.1 | ETH spend per buy for WETH-paired pools |
| Stablecoin Buy Amount | number | $5–$100,000 | $200 | USD spend for stable-paired pools |
| Take Profit % | number | 10–1000 | 50 | Exit at gain ≥ this % |
| Stop Loss % | number | 5–90 | 20 | Exit at loss ≤ -this % |
| Max Open Positions | int | 1–10 | 3 | Simultaneous slot cap |
| Position Timeout (min) | int | 1–60 | 5 | TIMEOUT exit when price has not moved >2% |
| Min Liquidity (USD) | number | 0–1,000,000 | 2000 | Pools below this are rejected; 0 disables the gate |
| Slippage % | number | 1–49 | 5 | Swap slippage tolerance |
| Gas Multiplier | number | (no DB clamp) | 1.5 | Multiplier on Quoter gas estimate |
| Partial Sell % | number | 1–99 | 50 | Portion to sell at PARTIAL_TP (hidden when sell-all-at-once is on) |
| Active Network | enum | ethereum, base, sepolia | ethereum | Currently selected network |
| Sell All At Once | bool | — | true | All-or-nothing TP vs partial |
| Auto-approve Tokens | bool | — | false | Skip per-token approval dialogs |
| Watch-Only Mode | bool | — | false | Alerts only, never buy |
| Telegram Chat ID | text | — | (server default) | Per-user chat for alerts |

### Risk Settings tab — risk thresholds and toggles

| Field | Type | Bounds | Default | Purpose |
|---|---|---|---|---|
| Max Pool Fee % | float | 0.1–5.0 | 1.0 | Pools above this fail Check 1 |
| Min Liquidity (USD) | number | 100–1,000,000 | 2000 | Same as Settings tab, kept for risk-side adjustment |
| Contract Score Threshold | int | 10–100 | 40 | Check 4 / ContractAnalyzer cutoff |
| Honeypot Score Threshold | int | 25–100 | 40 | Check 5 / HoneypotDetector cutoff for HONEYPOT verdict |
| Micro Test Max Loss % | number | 1–95 | 5 | Check 7 round-trip loss cutoff |
| Reject Proxy Contracts | bool | — | on | Hard-scam toggle for EIP-1167 / EIP-1967 patterns |
| Reject Hidden Contracts | bool | — | on | Hard-scam toggle for tokens that reference hidden deployed contracts |
| Reject Dangerous Admin | bool | — | on | Hard-scam toggle for `setTax` / `pauseTrading` / etc. selectors |
| Transfer Sim Enabled | bool | — | on | Run Check 6 |
| Micro Test Enabled | bool | — | on | Run Check 7 (the $0.03 round-trip) |
| Micro Test Amount ETH | number | 0.000001–0.01 | (config default) | Override Check 7 spend amount |
| Micro Test Amount Stable | number | 0.001–5 | (config default) | Override Check 7 stable spend |

The "Reset to defaults" button shows a modal preview before clearing every risk-side field to its config default.

---

## REST API

All routes are mounted under `/` on the API process (port 3010 internally, reverse-proxied as `api-sniper.gembabots.com`). Auth is JWT bearer in the `Authorization` header unless noted. Global middleware: CORS allowlist (`sniper.gembabots.com`, `localhost:3012`, `localhost:5173`), 500 req / 15 min per IP rate limit, 256 KB JSON body cap.

### Public routes (no auth)

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Returns `{ ok: true }` |
| POST | `/auth/request-otp` | Body `{ email, turnstileToken }`. Creates user if new, generates 6-digit OTP, sends email. Rate-limited 3/hour/email |
| POST | `/auth/verify-otp` | Body `{ email, code }`. Validates OTP, issues a 7-day JWT. Auto-activates a 24h free trial on first verification |
| GET | `/subscriptions/plans` | Returns the public pricing list |
| POST | `/subscriptions/webhook` | GembaPay HMAC-signed webhook (header `x-gembapay-signature`). Activates the subscription on `payment.completed` |

### Authenticated routes

| Method | Path | Body / params | What it does |
|---|---|---|---|
| GET | `/bot/status` | — | One row per network: `{ isRunning, lastHeartbeat, startedAt }` |
| POST | `/bot/start` | `{ network? }` | Registers the bot instance; gates on subscription + wallet (ADMIN_EMAIL bypasses) |
| POST | `/bot/stop` | `{ network \| 'all' }` | Marks the bot instance as stopped |
| GET | `/bot/positions` | — | Open positions for the user across all networks |
| GET | `/settings` | — | User settings (snake_case keys) |
| PUT | `/settings` | partial settings (camelCase) | Merges with COALESCE-per-field. Tri-state semantics for nullable fields (micro-test amount overrides) |
| POST | `/settings/reset-risk` | — | Clears risk-side fields to config defaults |
| GET | `/wallet` | — | `{ walletAddress }` — encrypted key is never sent |
| POST | `/wallet/encrypt` | `{ encryptedPrivateKey, iv, walletAddress }` | Stores the client-encrypted key blob |
| GET | `/wallet/orphans` | `?includeResolved` | Lists stuck-dust tokens from failed micro-test sells |
| POST | `/wallet/orphans/:id/resolve` | `{ resolveTx? }` | Marks an orphan as resolved after the user manually sells |
| GET | `/trades` | `?limit` (default 50, max 500) | The user's last N trades |
| GET | `/trades/analytics` | `?from&to&network&limit` | Aggregated PnL: total, win rate, by-network breakdown, best/worst trade |
| GET | `/trusted` | — | The user's whitelist |
| POST | `/trusted` | `{ tokenAddress, network, label?, checkMode? }` | Adds a whitelist entry |
| DELETE | `/trusted/:addr` | `?network=` | Removes a whitelist entry |
| GET | `/watchlist` | `?network` | The user's watch list |
| POST | `/watchlist` | `{ tokenAddress, network, label? }` | Adds a watch entry |
| DELETE | `/watchlist/:addr` | `?network=` | Removes a watch entry |
| GET | `/token/analyzed` | — | Global cache of the 50 most-recent analyses |
| POST | `/token/analyze` | `{ tokenAddress, network, poolAddress? }` | Runs ContractAnalyzer + HoneypotDetector; caches the result |
| GET | `/threats` | `?network&limit` | `hook_blacklist` rows + HONEYPOT-verdict `analyzed_tokens` |
| GET | `/unstuck/settings` | — | `{ enabled, mode }` |
| POST | `/unstuck/settings` | `{ enabled?, mode? }` | Saves auto-unstuck toggle and age-gate mode |
| GET | `/unstuck/attempts` | `?limit` (default 100, max 500) | Audit trail of sweeper attempts |
| POST | `/unstuck/run-now` | — | Spawns the auto-unstuck script (detached); requires `enabled = true` |
| GET | `/notifications/test` | — | Sends a Telegram test message |
| GET | `/activity/recent` | `?limit&network&since` | Recent in-memory pipeline events |
| GET | `/activity/stream` | `?token=` | Server-Sent Events live stream |
| POST | `/activity/internal` | event payload + `x-internal-secret` | Engine pushes events to the live feed (internal use only) |
| GET | `/subscriptions/status` | — | Per-network subscription state |
| POST | `/subscriptions/checkout` | `{ plan, network }` | Creates a pending subscription + GembaPay payment request |
| POST | `/subscriptions/activate-free-trial` | — | One-shot trial activation |
| POST | `/subscriptions/activate-manual` | `{ targetEmail, plan, network }` | Admin-only (gated on ADMIN_EMAIL) — comp a subscription |

---

## Database schema

PostgreSQL 12+. Schema in `src/db/schema.sql`; migrations in `src/db/migrations/00X_*.sql` (currently 1, 3-22 — migration `002` was renumbered).

### Tables

| Table | PK | Purpose |
|---|---|---|
| `users` | `id` (uuid) | Email-OTP accounts |
| `user_settings` | `user_id` (uuid FK) | Per-user trading + risk configuration |
| `user_wallets` | `user_id` (uuid FK) | AES-encrypted trading key + on-chain address |
| `bot_instances` | `(user_id, network)` | Engine-API handshake; `is_running` + `last_heartbeat` |
| `trades` | `id` (uuid) | Every buy / sell / partial / timeout. UNIQUE `(tx_hash, action)` |
| `analyzed_tokens` | `(token_address, network)` | Permanent contract analysis cache; `analyzed_count` ticks on every hit |
| `hook_blacklist` | `(hook_address, network)` | V4 hooks confirmed as honeypots. `bytecode_hash` index catches re-deploys |
| `watched_tokens` | `id` + UNIQUE `(user_id, token_address, network)` | Per-user watch list |
| `trusted_tokens` | `id` + UNIQUE `(user_id, token_address, network)` | Per-user whitelist; `check_mode` ∈ `{skip_checks, run_checks}` |
| `subscriptions` | `id` + UNIQUE `(gembapay_order_id)` | Per-user, per-network plan with `expires_at`. Six reminder-sent flags for the cron job |
| `orphan_balances` | `id` + UNIQUE `(user_id, network, token_address, buy_tx_hash)` | Stuck tokens after failed micro-test sells |
| `unstuck_attempts` | `id` | Auto-unstuck sweeper audit trail |

### Trades — V4 metadata (migration 016) and Layer F baseline (migration 022)

`trades` rows for V4 buys also persist `version`, `pool_id`, `hooks`, `tick_spacing`, `fee`, and (since Day 8) `initial_liquidity`. These let `positionManager.loadPositionsFromDB` reconstruct the position across an engine restart with enough information to re-establish the Swap-event subscription and route a sell back through `_sellV4`. Pre-migration rows have nulls; the position still resurrects but uses V3-shaped fallbacks.

### Hook blacklist — write reasons

CHECK constraint on `reason`:

```
post_microsell_trap_armed | caller_discrimination | bytecode_heuristic |
delayed_tax_detected | phantom_tp_at_sell | manual
```

### Trades — exit reasons

CHECK constraint on `action`:

```
buy | sell | partial_sell | timeout_sell
```

The reason for the sell (TAKE_PROFIT, STOP_LOSS, LP_DROP, etc.) is logged via the pipeline-report stream and Telegram, not persisted on the row itself.

### Subscriptions — plans

CHECK constraint on `plan`:

```
daily | weekly | monthly | annual | trial
```

---

## Telegram notifications

`src/telegram/notifier.js` exposes one method per event class. All messages go through a 1-message-per-second queue, so a burst of detections never trips Telegram's rate limit.

| Method | Fires on |
|---|---|
| `sendOpportunity` | New pool detection (used by debug paths) |
| `sendContractAnalysis` | Check 4 result |
| `sendHoneypotAnalysis` | Check 5 result |
| `sendTransferSimulation` | Check 6 result |
| `sendMicroTestResult` | Check 7 success or failure |
| `sendAnalysisResult` | TokenAnalyzer dashboard request |
| `sendTradeExecuted` | Real buy confirmed |
| `sendTradeClosed` | Position closed (any exit reason); includes PnL |
| `sendBotStatus` | Engine start / stop |
| `sendWatchAlert` | Watched-token detection while `watch_only = true` |
| `sendMicroTestFail` | Legacy thin wrapper for the failure path |
| `sendTest` | Notifications-test endpoint |
| `sendPipelineReport` | One bundled message per opportunity with all checks + final verdict |
| `sendScamAlert` | HONEYPOT verdict (eager — fires before pipeline report) |
| `sendHookHoneypotAlert` | Any Layer A/B/C/D/E2 detection; includes layer, reason, addresses, explorer links |
| `sendLpDropAlert` | Layer F LP-drop emergency sell |
| `sendError` | Background error |

Per-user Telegram chat IDs are supported via `user_settings.telegram_chat_id`; falling back to the global `TELEGRAM_CHAT_ID` env var.

---

## Configuration

### Environment variables (`.env`)

All read at process start; the API and engine each read the same file.

**Network RPCs / WebSockets.** Comma-separated lists; the first entry is preferred, the `ethers.FallbackProvider` rotates on RPC failure, and monitors rotate the WS URL on each reconnect. Singular variants are back-compat fallbacks.

```
ETH_RPC_URLS=, ETH_WS_URLS=
BASE_RPC_URLS=, BASE_WS_URLS=
SEPOLIA_RPC_URLS=, SEPOLIA_WS_URLS=
ETH_RPC_URL=, ETH_WS_URL=  (back-compat)
BASE_RPC_URL=, BASE_WS_URL=
SEPOLIA_RPC_URL=, SEPOLIA_WS_URL=
```

**Sepolia stable tokens** (used by `test/simulate.js`): `SEPOLIA_USDC`, `SEPOLIA_USDT`, `SEPOLIA_EURC`.

**Hardhat forks.** Per-network persistent fork URLs used by some test scripts. Default ports match the systemd units: `HARDHAT_BASE_URL=http://localhost:8546`, `HARDHAT_ETH_URL=http://localhost:8547`, `HARDHAT_SEPOLIA_URL=http://localhost:8545`. Override for off-box testing.

**Telegram.** `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` (default for users without their own chat ID).

**Database.** `DATABASE_URL`.

**Auth.** `JWT_SECRET`, `ENCRYPTION_KEY`.

**SMTP** (for OTP login). `SMTP_HOST` (default `mail.gembamail.com`), `SMTP_PORT` (587), `SMTP_SECURE=false`, `SMTP_USER`, `SMTP_PASS`, `FROM_EMAIL`, `FROM_NAME`.

**Cloudflare Turnstile.** `TURNSTILE_SECRET_KEY` — leave blank in dev to bypass the bot check on `/auth/nonce`.

**Bot defaults.** `DEFAULT_SLIPPAGE=5`, `DEFAULT_GAS_MULTIPLIER=1.5`, `DEFAULT_MAX_BUY_ETH=0.1`, `DEFAULT_STOP_LOSS_PERCENT=20`, `DEFAULT_TAKE_PROFIT_PERCENT=50`, `MICRO_TEST_MAX_LOSS_PERCENT=5`, `MAX_BUY_VS_MICRO_SLIPPAGE_PERCENT=30`.

**Service ports.** `SNIPER_API_PORT=3010`, `SNIPER_BOT_PORT=3011` (reserved, not bound), `SNIPER_DASHBOARD_PORT=3012`. Port range `3020–3022` is reserved for a future arbitrage bot.

**FeeRouter deploy.** `FEE_WALLET_ADDRESS`, `DEPLOY_PRIVATE_KEY`, `ETHERSCAN_API_KEY`, `BASESCAN_API_KEY`. Live addresses are hard-coded in `config.js`; the env vars override for staging.

**Subscriptions / GembaPay.** `GEMBAPAY_API_URL`, `GEMBAPAY_TEST_API_KEY`, `GEMBAPAY_LIVE_API_KEY` (LIVE preempts TEST), `GEMBAPAY_WEBHOOK_SECRET`, `APP_BASE_URL`, `APP_API_URL`, `ADMIN_EMAIL` (bypasses subscription gate and is the only address allowed to call `/subscriptions/activate-manual`).

### config.js bot section

`src/utils/config.js` `config.bot` exposes the operational defaults and the API-side clamp bounds. Notable groups:

- Trading defaults: `defaultSlippagePercent`, `defaultGasMultiplier`, `defaultMaxBuyEth`, `defaultStopLossPercent`, `defaultTakeProfitPercent`, `defaultSellAllAtOnce`, `defaultPartialSellPercent`
- Stable defaults: `defaultStablecoinBuyAmount = 200`, bounds `[5, 100_000]`
- Hard bounds: `minBuyEth = 0.001`, `maxBuyEth = 10`, `takeProfitMin/Max`, `stopLossMin/Max`, `slippageMin/Max`
- Position management: `defaultMaxOpenPositions = 3`, `defaultPositionTimeoutMinutes = 5`
- Micro test: `microTestEnabled = true`, `microTestAmountEth = '0.00001'`, `microTestAmountStable = 0.01`, override bounds
- Risk thresholds: `defaultMaxFeePercent = 1.0`, `defaultContractScoreThreshold = 40`, `defaultHoneypotScoreThreshold = 40`, hard-scam toggles, transfer-sim toggle
- Day 8 / Day 9 defenses: `clankerGuardEnabled = true`, `clankerGuardNetworks = ['base']`, `lpDropMonitorEnabled = true`, `lpDropThresholdPercent = 30`, per-network `delayTaxProbeHorizons`

### config.networks

Per-network blocks describing the chain (chainId, RPC list, WS list, factory and PoolManager addresses, Quoter v2 and v4, Universal Router, WETH, FeeRouter, supported quote tokens, block time, delay-tax probe horizons). Ethereum, Base, and Sepolia are configured today.

---

## Deployment

### Three systemd services on the host

| Unit | Process | Listens on |
|---|---|---|
| `gemba-sniper-api.service` | `node src/api/server.js` | `127.0.0.1:3010` |
| `gemba-sniper-engine.service` | `node src/index.js` | — (WS only, DB poll) |
| `gemba-sniper-dashboard.service` | `node node_modules/.bin/vite preview --port 3012 --host 0.0.0.0` (working dir `gemba-sniper-dashboard/`) | `0.0.0.0:3012` |

All three are `Restart=always` with `RestartSec=10`. The engine and the API each start their own DB pool; both read `EnvironmentFile=/home/slavy/projects/gemba-sniper-bot/.env`. The dashboard reads `VITE_*` env vars at build time, so a config change requires `npm run build && systemctl restart`.

### Two timer-driven jobs

| Unit | What it does | Schedule |
|---|---|---|
| `gemba-sniper-unstuck.timer` | Triggers `gemba-sniper-unstuck.service` which runs `scripts/auto_unstuck.js` | every 30 min |
| `gemba-sniper-unwrap-weth.timer` | Triggers `auto_unwrap_weth.js` which converts dust WETH on bot wallets back to native ETH | (see unit) |

### Two Hardhat fork services

`gemba-hardhat-base.service` and `gemba-hardhat-eth.service` run persistent forks on ports 8546 and 8547 respectively, used by some validation scripts and the `test/simulate.js` smoke test. Sepolia does not ship a dedicated unit — spin up `npx hardhat node --fork $SEPOLIA_RPC_URLS` manually when needed.

### Apache reverse proxy

Two vhosts behind Cloudflare with an Origin CA certificate at `/etc/ssl/gembabots/{cert,key}.pem` (valid through 2041-05-24):

| Hostname | Origin path | Notes |
|---|---|---|
| `api-sniper.gembabots.com` | `http://localhost:3010` | REST API; `X-Frame-Options: DENY` |
| `sniper.gembabots.com` | `http://localhost:3012` (+ `/ws` proxied) | Dashboard; `X-Frame-Options: SAMEORIGIN` |

Both vhosts redirect `:80` → `:443` permanently.

### Daily operations

The full operations cookbook is in [`OPERATIONS.md`](OPERATIONS.md). Quick reference:

```bash
# Three services at a glance
systemctl is-active gemba-sniper-api gemba-sniper-engine gemba-sniper-dashboard
sudo journalctl -u gemba-sniper-api -u gemba-sniper-engine -f

# Health
curl -s https://api-sniper.gembabots.com/health

# DB peek
sudo -u postgres psql -d gemba_bot -c "SELECT user_id, network, is_running, last_heartbeat FROM bot_instances;"
sudo -u postgres psql -d gemba_bot -c "SELECT id, action, network, amount_eth, status, created_at FROM trades ORDER BY created_at DESC LIMIT 20;"

# Dashboard rebuild + restart
cd /home/slavy/projects/gemba-sniper-dashboard && \
  npm run build && \
  sudo systemctl restart gemba-sniper-dashboard
```

---

## Red-team testing

`~/projects/test-hook-sepolia/` contains a Foundry workspace with controlled honeypot hook variants (`DynamicFeeHook.sol.v7.back` through `v16.back`), legitimate hook patterns (`CleanFeeHook.sol`, `LaunchBlockHook.sol`, `RewardTrackerHook.sol`), and the deploy / liquidity / pump / withdraw scripts used to deploy them to Sepolia. The full attack catalog and per-test results live in [`RED_TEAM.md`](RED_TEAM.md).

The original RED_TEAM playbook lists ten canonical attack vectors against the delayed-tax probe and micro-test. As of 2026-05-31 every one of them is covered, either directly (vectors 1, 4 — actively red-team-tested live) or aggregated (vectors 2, 3, 5, 6, 7, 8, 9, 10 — covered by the v15 mega-hook smoke test plus v16 solo grace test).

Three defense additions came out of red-teaming and now ship in the pipeline:

- **Day 8** — Clanker-pattern guard (Check 0.9), sell-side gas re-check (Layer E2), LP-drop monitor (Layer F). Closed the Test 13 / Test 15 incident on Base mainnet where Clanker-template rug-pulls drained ~0.012 ETH.
- **Day 9** — Multi-horizon delayed-tax probe (Fix A), Layer A grace-pattern fingerprint (Fix C), sell-time trap detector extended beyond TAKE_PROFIT (Fix B). Closed the Test 17 (v16) gap where a 100-block grace window survived the single-50-block probe horizon.

---

## File map

### Backend (this repo)

```
src/index.js                         # Engine entrypoint (gemba-sniper-engine.service)
src/api/server.js                    # API entrypoint (gemba-sniper-api.service)
src/api/routes/                      # auth, bot, settings, wallet, trades, trusted,
                                     # token, watchlist, activity, threats, unstuck,
                                     # subscriptions, notifications
src/api/middleware/auth.js           # JWT bearer middleware

src/bot/sniper.js                    # Detection pipeline orchestration
src/bot/executor.js                  # V3 + V4 buy / sell paths, micro-test, phantom-TP guard
src/bot/positionManager.js           # TP/SL/TIMEOUT/LP_DROP, real-time price tracking
src/bot/walletManager.js             # AES-GCM decrypt
src/bot/pipelineReport.js            # Per-opportunity report bundle

src/monitor/poolDetector.js          # V3 PoolCreated + V4 Initialize subscriber
src/monitor/mempoolMonitor.js        # Pending tx filter

src/detection/hookAnalyzer.js        # V4 hook capability + soft-skip orchestration
src/detection/hookBytecodeScanner.js # Layer A opcode walk
src/detection/walletSimProbe.js      # Layer B caller-spoof
src/detection/delayedTaxProbe.js     # Check 3.6 multi-horizon + Layer C re-probe
src/detection/gasTrapProbe.js        # Layer E (Day 7) + shared GAS_TRAP_THRESHOLD
src/detection/liquidityChecker.js    # Check 3 USD-floor (+ getPoolLiquidityUsd)
src/detection/contractAnalyzer.js    # Check 4 token bytecode heuristics
src/detection/honeypotDetector.js    # Check 5 eth_call buy+sell sim
src/detection/transferSimulator.js   # Check 6 transfer/approve simulation

src/db/schema.sql                    # Canonical schema
src/db/migrations/00*.sql            # Ordered migrations
src/db/queries.js                    # All DB helpers

src/telegram/notifier.js             # 1 msg/s queued Telegram client

src/utils/config.js                  # Networks, bot defaults, env parsing
src/utils/providers.js               # FallbackProvider builder
src/utils/logger.js                  # Winston
src/utils/mailer.js                  # SMTP for OTP
src/utils/turnstile.js               # Cloudflare verification
src/utils/priceCache.js              # ETH/USD via Chainlink
src/utils/activityPublisher.js       # SSE broadcast
src/utils/subscriptionReminder.js    # Hourly expiry-reminder cron

scripts/auto_unstuck.js              # Sweeper (gemba-sniper-unstuck.timer)
scripts/auto_unwrap_weth.js          # WETH dust unwrap
scripts/sell_*.js                    # Ad-hoc recovery scripts
contracts/                           # Hardhat workspace for FeeRouter

infrastructure/systemd/              # All systemd units
infrastructure/apache/               # vhost configs
infrastructure/scripts/setup.sh      # Install units + vhosts
infrastructure/scripts/install-cert.sh
infrastructure/scripts/deploy.sh
```

### Frontend (sibling repo `gemba-sniper-dashboard/`)

```
src/main.jsx + src/App.jsx           # Entry + token-driven router
src/pages/Login.jsx                  # OTP login (Turnstile)
src/pages/Dashboard.jsx              # Tab shell
src/pages/tabs/Overview.jsx          # Per-network BotCards
src/pages/tabs/Activity.jsx          # Live event stream
src/pages/tabs/Analytics.jsx         # PnL chart + trade list
src/pages/tabs/TokenAnalyzer.jsx     # Manual analysis tool
src/pages/tabs/Watchlist.jsx
src/pages/tabs/Settings.jsx
src/pages/tabs/RiskSettings.jsx
src/pages/tabs/Wallet.jsx            # Setup / view / orphans
src/pages/tabs/StuckPositions.jsx    # Auto-unstuck panel
src/pages/tabs/Threats.jsx
src/pages/tabs/TrustedTokens.jsx
src/pages/tabs/Subscription.jsx
src/components/DarkSelect.jsx        # Cross-browser dropdown replacement
src/api/client.js                    # apiFetch + JWT helpers
src/utils/walletCrypto.js            # AES-256-GCM PBKDF2 (browser-side)
src/index.css                        # Design tokens + component CSS
```

---

## License

Proprietary. Internal use only.
