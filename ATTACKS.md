# Attack Coverage Matrix

Living document. Every attack vector the bot might face, current test
status, current defense status, and known gaps. Appended over time, never
truncated.

Three orthogonal questions per vector:

1. **Tested?** — was a controlled hook / token deployed to a real chain
   and run against a live bot instance? (Tests in `RED_TEAM.md`.)
2. **Defense implemented?** — does at least one layer in the current
   pipeline have a code path that could reject it?
3. **Gap?** — is there a credible scenario where the defense slips and the
   bot loses funds?

A "yes" to (2) without (1) means the defense exists on paper but has not
been validated end-to-end. A "yes" to (3) means we have known unfinished
work.

---

## Section A — Canonical 10 vectors (`RED_TEAM.md` original table)

| # | Vector | Tested | Defense layers | Gap |
|---|---|---|---|---|
| 1 | Quoter sender bypass | Yes — Test 1 (v9 canonical), Test 10 (v13/v14 obfuscated) | Layer A KNOWN_ADDRESS_EMBEDDED (+70), Layer B walletSimProbe (eth_simulateV1 caller spoof) | None |
| 2 | `gasleft()` probe asymmetry | **Yes — Test 20 (v19, 2026-05-31)** | Layer A GAS (+35) + Day 10 Fix 1 EVM_GLOBAL_BEFORESWAP_ONLY (+50) → combinedRisk **135** → softSkip ✅ verified live | Closed by Day 10 Fix 1 |
| 3 | `block.coinbase == address(0)` | **Yes — Test 21 (v20, 2026-05-31)** | Layer A COINBASE (+35) + Day 10 Fix 1 → combinedRisk **135** → softSkip ✅ verified live | Closed by Day 10 Fix 1 |
| 4 | Swap-count gating | Yes — Tests 2-8 (v10, v11, v12 variants) | Layer A SSTORE_BEFORESWAP_ONLY (+60), Layer C post-microsell re-probe | None |
| 5 | Long GRACE window | Yes — Test 17 (v16, GRACE=100) | Day 9 Fix A multi-horizon probe `[50, 300, 1500]`; Day 9 Fix C GRACE_PATTERN (+40); Day 9 Fix B sell-time guard | None for grace ≤ 1500 blocks |
| 6 | `tx.origin == address(0)` | **Yes — Test 22 (v21, 2026-05-31)** | Layer A ORIGIN (+30) + Day 10 Fix 1 → combinedRisk **130** → softSkip ✅ verified live | Closed by Day 10 Fix 1 |
| 7 | Manual `activate()` flag flip | **Yes — Test 23 (v22, 2026-05-31)** | Day 9 Fix B sell-time guard intended; in practice Layer A's SSTORE_BEFORESWAP_ONLY caught it at combinedRisk **120** because v22's deployer-flag SSTORE trips the heuristic. Fix B sell-time path remains code-reviewed-correct but not exercised on this vector. | Closed by Layer A coincidence; Fix B remains backstop for hypothetical proxy-pattern variants |
| 8 | Conditional revert at specific future block | **Yes — Test 24 (v23, 2026-05-31)** | Layer A GRACE_PATTERN (Day 9 Fix C) caught it at combinedRisk **115** because v23's SLOAD+NUMBER compare matches the launchBlock+grace fingerprint. delayedTaxProbe future-revert handling (code line 253) remains correct backstop. | Closed by Layer A coincidence |
| 9 | `basefee == 0` / `blockhash == 0` | **Yes — Test 18 (v17, 2026-05-31)** | Layer A BASEFEE + BLOCKHASH + NUMBER + Day 10 Fix 1 → combinedRisk **175** → softSkip ✅ verified live | None |
| 10 | Wallet-tracker (per-wallet counter) | **Yes — Test 19 (v18, 2026-05-31)** | Layer A SSTORE_PRESENT + SSTORE_BEFORESWAP_ONLY + ORIGIN + Day 10 Fix 1 → combinedRisk **200** → softSkip ✅ verified live | None |

---

## Section B — New vector from real-world incident

| # | Vector | Tested | Defense layers | Gap |
|---|---|---|---|---|
| 11 | Liquidity rug-pull (Clanker template, hookless V4, fee=0, tickSpacing=1) | Yes — Test 13/15 real Base mainnet incident (~0.012 ETH loss) | Day 8 Check 0.9 Clanker-pattern guard; Day 8 Layer F LP-drop monitor (30 s tick); Day 8 Layer E2 sell-side gas re-check | Layer F + Layer E2 not yet live-verified post-fix. Test 16 only verified Check 0.9 indirectly via v15 mega-hook |

---

## Section C — New vectors brainstormed but not in original 10

| # | Vector | Tested | Defense layers | Gap |
|---|---|---|---|---|
| 12 | Token-side time-bomb (no hook; ERC20 `transfer` reverts or escalates fee after stored expiry) | **Yes — Test 25 (v24, 2026-05-31)** | Day 10 Fix 2 (probe hookless pools) insufficient — V4 Quoter doesn't call `transfer` in simulation. **Day 10 Fix 4 added**: gasTrapProbe treats future-block estimateGas clean reverts as time-bomb. `provider.estimateGas` of a real swap DOES call transfer. Verified live. | Closed by Day 10 Fix 4 (V4 only; V3 still uncovered) |
| 13 | Token rebase post-buy (deployer's `transferFrom` dilutes bot's holdings after buy) | No | None — `analyzed_tokens` cache snapshot is static; no runtime check on totalSupply drift | **YES (full gap)**. Mitigation: manual blacklist of known rebase token contracts. Reliable detection is impractical (every rebase token has its own pattern) |
| 14 | Fee-on-transfer escalation by amount (low buy fee, exponential sell fee that scales with amountIn) | Partial — Test 27 (v26) test contract had a design flaw (PoolManager-exempt path) | **Day 11 Fix 7** — post-buy receive verification: after the real buy confirms, compare `trade.amountTokens` to pro-rata extrapolation from the micro-test (`micro.tokensReceived × realEth / microEth`). If actual < `100 - max_buy_vs_micro_slippage_percent`% (default 70%) → persist HONEYPOT (`post_buy_receive_short`), Telegram alert, set `opportunity._honeypotForceExit`. After `positionManager.addPosition` returns, immediately call `_sellFull(reason='AUTO_UNSTUCK')` so the trap surfaces in dashboard + Telegram instead of waiting for TIMEOUT. AUTO_UNSTUCK reason bypasses Layer D's phantom-TP gate. | Closed by Day 11 Fix 7 |
| 15 | Permit2 trap (deployer signs `Permit2.permit` revoking bot's allowance mid-trade) | No | Day 10 Fix 3 (proposed) — re-fetch Permit2 allowance immediately before each real sell (max 60 s cache) | **YES** until Fix 3 lands |
| 16 | Liquidity-switching (deployer adds malicious 2nd pool with hook; bot routes sells through it) | No | `_sellV4` reconstructs PoolKey from `position.{poolId,hooks,tickSpacing,fee}` — pinned to the original pool. Cannot be switched. | None expected — verify in audit |
| 17 | `afterSwap` return-delta drain (hook returns BalanceDelta that steals bot's output) | **Yes — Test 28 (v27, 2026-05-31)** | Layer A `isCritical: true` at combinedRisk 110 → softSkip ✅ verified live. Address-bit signature alone sufficient. | Closed |
| 18 | Sandwich / MEV (mempool front-run on bot's micro-test or real buy) | Operator-driven (set `*_MEV_RPC_URLS` env) | **Day 12 Fix 8** — per-network `mevProtectedRpcUrls` config. `getMevProvider(network)` returns a dedicated `JsonRpcProvider` bound to the configured MEV RPC (e.g. `https://rpc.mevblocker.io`, Cow, Flashbots Protect, Merkle). Executor's `wallet.sendTransaction` patch (the existing nonce-lock wrapper) now ALSO routes through MEV: `populateTransaction` + `signTransaction` locally, `broadcastTransaction` via MEV provider. Public read path (FallbackProvider) unchanged. On MEV broadcast failure (RPC down, rejected) falls back to public submission — better to land in public mempool than to lose the trade. Empty config = previous behaviour (public submission). | Closed (operator opt-in) |
| 19 | Token blacklist post-buy (`transfer` adds bot wallet to internal blacklist on first receive) | **Yes — Test 26 (v25, 2026-05-31) + re-test (Day 11)** | Day 10 Fix 5 (rev 2) verified live. ethers v6 `tx.wait()` THROWS `CALL_EXCEPTION` on status=0 (not returns) — catch wraps `wait()`, classifies `err.receipt.status === 0` as definitive revert, persists HONEYPOT (`source: sell_tx_revert_at_execution`), throws `SELL_DEFINITIVE_REVERT`. Day 11 also added matching handler in `positionManager._sellFull` catch — marker recognition writes off the open position immediately so the 30 s tick doesn't keep retrying. | Closed |
| 20 | Reentrancy via `beforeSwap` callback | No | V4 PoolManager has unlock/lock pattern; nested swaps inside beforeSwap revert | Untested but architecturally blocked |
| 21 | Pool-state oracle manipulation (hook reads sqrtPriceX96 to compute fee, attacker flash-loans to skew before bot's swap) | No | None | **YES (full gap)**. Rare in practice for sniper-target pools (too low liquidity to be worth a flash loan). Document. |
| 22 | Multi-hook composition (hook A calls hook B that the scanner did not visit) | No | hookAnalyzer fetches getCode of `opportunity.hooks` only; doesn't trace external calls | **YES (deep gap)**. Day 13 candidate: add an `extcodesize`/CALL static-scan to hookBytecodeScanner — flag any hook that performs external CALLs |
| 23 | Hook upgrade (proxy hook whose implementation changes mid-position) | No | hook addresses in V4 are bound to permission bits in the address itself, so a proxy that changes implementation cannot change permissions — but CAN change behavior of beforeSwap/afterSwap | **YES (latent)**. Document; combined-risk should already be high because proxies show DELEGATECALL pattern; verify |

---

## Section D — Pipeline coverage summary

What every layer / check protects against today (as of 2026-05-31, post Day 9):

| Layer / check | Vectors covered |
|---|---|
| Check 0.9 Clanker guard (Day 8) | #11 |
| Check 1 Fee tier | (UI safety, not a vector) |
| Check 2 V4 dynamic-fee | (UI safety) |
| Check 3 Liquidity floor | (low-LP traps, partial #11) |
| Check 3.4 Hook blacklist | All vectors after first sighting (compounding defense) |
| Check 3.5 Layer A bytecode scanner | #1 (canonical), #4, #9, #10 (likely); #2, #3, #6 only when aggregated (caught in v15 Test 16) |
| Check 3.6 delayedTaxProbe multi-horizon (Day 9 Fix A) | #5 (any GRACE ≤ 1500), #8 (future-block revert); #12 partial after Day 10 Fix 2 |
| Check 3.7 Layer B walletSimProbe | #1 (obfuscated v13/v14 storage-slot variant); does NOT catch caller-blind traps |
| Check 4 ContractAnalyzer (token bytecode) | proxy / hidden contract / dangerous admin selectors |
| Check 5 HoneypotDetector | buy/sell tax asymmetry, blocked sells, basic patterns |
| Check 6 TransferSimulator | stateOverride-based transfer / approve revert detection |
| Check 7 Micro-test buy + sell | catches anything visible in a $0.03 round-trip; misses scale-dependent traps |
| Check 7.5 Layer C post-microsell re-probe | #4 (after micro-sell arms storage trap) |
| Check 7.6 Layer E gas-trap probe (Day 7) | bytecode gas traps; partial #11; partial #14 |
| Check 8 Real-size guard | thin-liquidity traps |
| Layer D phantom-TP (Day 9 Fix B extended) | #7 post-buy, any sell-time trap that quotes ≤ 30% of breakeven |
| Layer E2 sell-side gas re-check (Day 8) | bytecode gas traps that arm post-buy; partial #19 |
| Layer F LP-drop monitor (Day 8) | #11 (rug-pulls detected within 30 s) |

---

## Section E — Implementation roadmap

### Day 11+12 — Post-buy receive verification + Layer F StateView fix + MEV-protected RPC

**Status: IMPLEMENTED 2026-05-31.**

- **Fix 6** (sniper.js + positionManager.js + config.js): Layer F's `getLiquidity(bytes32)` was being called on the V4 PoolManager, which does NOT expose that function externally — every call reverted with `require(false)` and `initial_liquidity` was always NULL. Switched both call sites to V4 StateView (`uniswapV4StateView` per-network in config; ethereum `0x7ffe…`, base `0xa3c0…`, sepolia `0xE1Dd…`). Verified live: initial probe captured `5e20` for a 0.5-ETH-funded pool, LP-pull triggered `LP_DROP` emergency sell within 26 s.
- **Fix 7** (sniper.js): post-buy receive verification. After real buy confirms, compares `trade.amountTokens` to micro-test pro-rata. Below `100 - max_buy_vs_micro_slippage_percent`% → HONEYPOT + immediate `_sellFull(AUTO_UNSTUCK)`. Closes vector #14 (fee-on-transfer escalation that Quoter's quote-time math misses).
- **Fix 8** (providers.js + config.js + .env.example + executor.js): MEV-protected RPC for tx submission. Per-network `mevProtectedRpcUrls`; `getMevProvider()` builds a dedicated JsonRpcProvider; executor's nonce-lock `sendTransaction` patch detects MEV provider presence and routes through `populateTransaction` + `signTransaction` + `broadcastTransaction`. Falls back to public submission on MEV broadcast error.
- **Fix 5 v2** (executor.js): ethers v6 `tx.wait()` THROWS `CALL_EXCEPTION` on status=0 rather than returning the receipt. Day 10 Fix 5 v1's `if (receipt.status !== 1)` check never executed. Wrapped `wait()` in try/catch; catches `err.code === 'CALL_EXCEPTION'` AND `err.receipt?.status === 0` → persist HONEYPOT + throw `SELL_DEFINITIVE_REVERT_MARKER`. Also added matching handler in `positionManager._sellFull` catch: marker recognition writes off the position immediately so the 30 s tick doesn't keep retrying. Verified live with v25 re-test.

### Day 10 — Closing canonical solo gaps + token time-bomb + token blacklist post-buy

**Status: IMPLEMENTED 2026-05-31** (5 fixes total: Fix 1 + Fix 2 + Fix 3 initially planned; Fix 4 + Fix 5 added during STAGE 5 testing). Offline regression on CleanFee / LaunchBlock / RewardTracker / v16: no false positives (combined risks 60 / 85 / 70 / 125 respectively — only v16 grace solo trips softSkip).

Two additional fixes shipped during STAGE 5 (Tests 25-28) red-team
testing in response to gaps surfaced by v24 + v25:

- **Fix 4** — `gasTrapProbe.js`: future-block clean-revert = time-bomb detection. Closes vector #12 (V4 path) where v24 exposed that Day 10 Fix 2 alone was insufficient because Quoter doesn't trigger `transfer`.
- **Fix 5** — `executor.js _sellV4`: post-submit `receipt.status=0` triggers HONEYPOT persist + `SELL_DEFINITIVE_REVERT_MARKER` throw. Closes vector #19 tail case where v25 exposed Day 10 Fix 3's estimateGas-only gating missed actual tx-time reverts.

Plus a third change required by Fix 2: **sniper.js gating fix** removed
the hookless V4 short-circuit at the `!hasHooks` branch so hookless
pools actually reach the delayedTaxProbe call.

| Fix | Location | What it does | Closes vectors |
|---|---|---|---|
| **Fix 1** | `hookBytecodeScanner.js` | New pattern `EVM_GLOBAL_BEFORESWAP_ONLY` (+50) — fires when any single EVM-global opcode (GAS/COINBASE/ORIGIN/BASEFEE/BLOCKHASH/GASLIMIT) is present AND the hook permissions are ONLY beforeSwap. Combined with the global's own +30/35 → ~130 solo combined risk → softSkip | #2, #3, #6, partial #9/#10 |
| **Fix 2** | `delayedTaxProbe.js` | Remove the `if no hooks skip` early return. Run the multi-horizon probe for hookless V4 pools too. Token-side time-bombs that revert `transfer` post-launch show up as a future-block Quoter revert → `detected: true` | #12 (V4 path only) |
| **Fix 3** | `executor.js _sellV4` (definitive-revert branch) | When `provider.estimateGas` returns a definitive revert at sell time (token blacklisted the bot wallet, clean revert in `transfer`), persist HONEYPOT verdict to `analyzed_tokens` + Telegram alert + throw with `SELL_DEFINITIVE_REVERT_MARKER` so auto_unstuck writes off on first attempt instead of looping. Same shape as Day 8 Layer E2 but for clean reverts (not gas-trap reverts) | #19 token blacklist post-buy |

> **Re-classified during Day 10 implementation: vector #15 (Permit2 trap) is
> not a real attack.** `_ensurePermit2Approval` reads on-chain allowance
> fresh on every call (no in-memory cache despite the function name's
> "cache" reference — it caches the *on-chain approval*, not a JS object).
> Permit2 allowance can only be reduced by the owner (bot wallet) via
> `Permit2.approve` or `Permit2.lockdown`, or by signing a permit message
> — none of which an attacker can do on someone else's wallet. The
> previously planned Fix 3 (Permit2 freshness gate) is therefore not
> needed. The slot is repurposed for the token-blacklist defense above.

### Day 11+ — Larger or out-of-scope items

| Fix | Closes | Cost |
|---|---|---|
| Quoter dual-scale check (micro vs real-size ratio) | #14 fee escalation | small — extends Check 8 |
| MEV-protected RPC option | #18 sandwich / MEV | medium — per-network config + provider |
| `extcodesize`/CALL static scan in hookBytecodeScanner | #22 multi-hook composition | medium — bytecode walker extension |
| V3 hookless future-block probe | #12 V3 path | medium — V3 quoter has no BlockOverrides support on all RPCs |
| Rebase token blacklist (manual curated list) | #13 | small — config-driven |
| Pool-state oracle manipulation defense | #21 | hard — generally requires private-mempool execution |

### Permanent "won't fix" entries

| Vector | Why no defense |
|---|---|
| #20 Reentrancy via beforeSwap | V4 PoolManager unlock/lock pattern blocks it architecturally |
| #21 Flash-loan price manipulation | Economically infeasible for sniper-target liquidity sizes; better solved by waiting for deeper pools |

---

## Section F — Test plan (after defenses land)

| Test | Hook source | Purpose | Status | Cost (sepolia ETH) |
|---|---|---|---|---|
| v17 (Test 18) | solo vector #9 (basefee + blockhash) | Verify Layer A scores 110+ → softSkip | **DONE — combinedRisk 175 ✅** | ~0.04 |
| v18 (Test 19) | solo vector #10 (wallet-tracker) | Verify Layer A SSTORE_BEFORESWAP_ONLY softSkip | **DONE — combinedRisk 200 ✅** | ~0.04 |
| v19 (Test 20) | solo vector #2 (gasleft) | Verify Day 10 Fix 1 EVM_GLOBAL_BEFORESWAP_ONLY catches | **DONE — combinedRisk 135 ✅ smoking gun** | ~0.04 |
| v20 (Test 21) | solo vector #3 (coinbase) | Same | **DONE — combinedRisk 135 ✅** | ~0.04 |
| v21 (Test 22) | solo vector #6 (tx.origin) | Same | **DONE — combinedRisk 130 ✅** | ~0.04 |
| v22 (Test 23) | solo vector #7 (manual activate) | Confirm Day 9 Fix B sell-time guard catches at 30% threshold | **DONE — combinedRisk 120 via SSTORE_BEFORESWAP_ONLY coincidence ✅** | ~0.04 |
| v23 (Test 24) | solo vector #8 (conditional revert) | Verify delayedTaxProbe handles future-revert correctly | **DONE — combinedRisk 115 via GRACE_PATTERN coincidence ✅** | ~0.04 |
| v24 | token-side time-bomb (no hook) | Verify Day 10 Fix 2 catches | ~0.04 |
| v25 | Permit2 revocation race | Verify Day 10 Fix 3 catches | ~0.05 (needs deployer permit signing flow) |
| v26 | fee-on-transfer escalation | Document partial coverage; design Day 11 fix from result | ~0.04 |
| v27 | `afterSwap` return-delta drain | Verify hookAnalyzer critical-flag catches | ~0.04 |
| Live verification | Check 0.9 on real Clanker on Base | Confirm rejection in production traffic | $0 (passive) |
| Live verification | Layer F controlled LP-pull | Trigger 70%+ LP removal on bot-held position; expect emergency sell | ~0.05 (deploy + LP + remove) |
| Live verification | Layer E2 re-test on gas-trap | Re-deploy time-bomb-style token; expect single-attempt writeoff | ~0.05 |

Estimated total: ~0.5 sepolia ETH + ~10-15 hours wall-clock.

---

## Section G — Historical decisions

| Day | Decision | Reason |
|---|---|---|
| Day 1 | Layer A blacklist precheck + Layer C re-probe + Layer D phantom-TP | Closing v10 swap-count trap surfaced in Test 2 |
| Day 2 | Layer A bytecode opcode scanner | Catches v9/v10 fingerprints before any probe RPCs spend |
| Day 3 | Write-on-detect from every defense layer to `hook_blacklist` | Compounding defense — every catch makes future re-deploys free |
| Day 4 | Layer B `eth_simulateV1` caller-spoof | Closes obfuscated v9 (storage-slot Quoter) |
| Day 5 | Layer A score re-calibration (SSTORE_PRESENT 40→10, SSTORE_BEFORESWAP_ONLY 30→60) | RewardTrackerHook false positive at 100 threshold |
| Day 6 | Dashboard Threats tab | Operator-facing observability |
| Day 7 | Layer E gas-trap probe with future-block | Test 13 base mainnet gas-trap incident |
| Day 8 | Check 0.9 Clanker guard + Layer E2 sell-side + Layer F LP-drop | Test 15 root-cause revision: rug-pulls, not gas-traps |
| Day 9 | Multi-horizon probe (Fix A) + GRACE_PATTERN (Fix C) + sell-time guard extended (Fix B) | Test 17 confirmed solo grace bypass |
| Day 10 | (in progress) EVM_GLOBAL_BEFORESWAP_ONLY + hookless probe + Permit2 freshness | This document; closes #2/#3/#6/#12/#15 |
