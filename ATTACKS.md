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
| 2 | `gasleft()` probe asymmetry | No solo test; appears in v15 mega-hook (Test 16) | Layer A GAS (+35). Solo combinedRisk ≈ 85 — UNDER 100 threshold | **YES** — solo deployment bypasses Layer A. Day 10 Fix 1 (EVM_GLOBAL_BEFORESWAP_ONLY +50) closes this once landed |
| 3 | `block.coinbase == address(0)` | No solo test; in v15 | Layer A COINBASE (+35). Solo ≈ 85 — UNDER | **YES** — same fix path as #2 |
| 4 | Swap-count gating | Yes — Tests 2-8 (v10, v11, v12 variants) | Layer A SSTORE_BEFORESWAP_ONLY (+60), Layer C post-microsell re-probe | None |
| 5 | Long GRACE window | Yes — Test 17 (v16, GRACE=100) | Day 9 Fix A multi-horizon probe `[50, 300, 1500]`; Day 9 Fix C GRACE_PATTERN (+40); Day 9 Fix B sell-time guard | None for grace ≤ 1500 blocks |
| 6 | `tx.origin == address(0)` | No solo test; in v15 | Layer A ORIGIN (+30). Solo ≈ 80 — UNDER | **YES** — same fix path as #2 |
| 7 | Manual `activate()` flag flip | No solo test; in v15 | Day 9 Fix B sell-time guard (30% threshold) — catches **post-buy** | **PARTIAL** — bot still opens the position before the flag flips; loss limited to whatever the sell trap returns vs buy cost. Document, don't fix (would require a sell-time probe with mock-state). |
| 8 | Conditional revert at specific future block | No solo test; in v15 | delayedTaxProbe treats future-block revert as `detected: true, dropPct: 100` (current code line 253) — should already catch | Verify with v23 test |
| 9 | `basefee == 0` / `blockhash == 0` | No solo test; in v15 | Layer A BASEFEE (+30) + BLOCKHASH (+30). Solo combined ≈ 110-130 — softSkip | Verify with v17 test (border-case scoring) |
| 10 | Wallet-tracker (per-wallet counter) | No solo test; in v15 | Layer A SSTORE_BEFORESWAP_ONLY (+60) when hook only has beforeSwap (solo ≈ 120 — softSkip) | Verify with v18 test. Hook with afterInitialize + per-wallet counter would bypass SSTORE_BEFORESWAP_ONLY — that combo not yet thought through. |

---

## Section B — New vector from real-world incident

| # | Vector | Tested | Defense layers | Gap |
|---|---|---|---|---|
| 11 | Liquidity rug-pull (Clanker template, hookless V4, fee=0, tickSpacing=1) | Yes — Test 13/15 real Base mainnet incident (~0.012 ETH loss) | Day 8 Check 0.9 Clanker-pattern guard; Day 8 Layer F LP-drop monitor (30 s tick); Day 8 Layer E2 sell-side gas re-check | Layer F + Layer E2 not yet live-verified post-fix. Test 16 only verified Check 0.9 indirectly via v15 mega-hook |

---

## Section C — New vectors brainstormed but not in original 10

| # | Vector | Tested | Defense layers | Gap |
|---|---|---|---|---|
| 12 | Token-side time-bomb (no hook; ERC20 `transfer` reverts or escalates fee after stored expiry) | No | Day 10 Fix 2 (proposed) — extend `delayedTaxProbe` to hookless V4 pools so future-block Quoter call exposes token-side time-gating | **YES** until Fix 2 lands. After Fix 2, V3 hookless time-bombs still uncovered (no V3 future-block probe) |
| 13 | Token rebase post-buy (deployer's `transferFrom` dilutes bot's holdings after buy) | No | None — `analyzed_tokens` cache snapshot is static; no runtime check on totalSupply drift | **YES (full gap)**. Mitigation: manual blacklist of known rebase token contracts. Reliable detection is impractical (every rebase token has its own pattern) |
| 14 | Fee-on-transfer escalation by amount (low buy fee, exponential sell fee that scales with amountIn) | No | Layer E2 catches if the escalation hits the gas limit. Pure fee-only escalation (gas stays normal) bypasses | **PARTIAL** — Layer E2 misses pure-fee variants. Day 11 candidate: Quoter quote at micro-test scale vs at real-size scale; ratio drop > slippage threshold = trap |
| 15 | Permit2 trap (deployer signs `Permit2.permit` revoking bot's allowance mid-trade) | No | Day 10 Fix 3 (proposed) — re-fetch Permit2 allowance immediately before each real sell (max 60 s cache) | **YES** until Fix 3 lands |
| 16 | Liquidity-switching (deployer adds malicious 2nd pool with hook; bot routes sells through it) | No | `_sellV4` reconstructs PoolKey from `position.{poolId,hooks,tickSpacing,fee}` — pinned to the original pool. Cannot be switched. | None expected — verify in audit |
| 17 | `afterSwap` return-delta drain (hook returns BalanceDelta that steals bot's output) | No | `hookAnalyzer` flags `AFTER_SWAP_RETURNS_DELTA` permission bit as critical (`isCritical = true`, softSkip threshold drops to 80) | Probably covered; verify with a test hook |
| 18 | Sandwich / MEV (mempool front-run on bot's micro-test or real buy) | No | None — bot uses public RPC submission, no flashbots or private mempool | **YES (full gap)**. Day 12 candidate: per-network MEV-protected RPC option (mevblocker / merkle / Cow) |
| 19 | Token blacklist post-buy (`transfer` adds bot wallet to internal blacklist on first receive) | No | Micro-test buy succeeds; first real sell from THIS wallet would already be in blacklist → reverts. Layer E2 catches the gas-trap-style revert pattern but not all blacklist patterns (some just `revert()` cleanly which is a "definitive revert" code path → bot aborts cleanly but holds the bag) | **PARTIAL**. Day 11 candidate: dual-wallet micro-test (buy from wallet A, sell from wallet A in micro-test, real buy from wallet A but with new burner check). Operationally expensive. |
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

### Day 10 — Closing canonical solo gaps + token time-bomb + token blacklist post-buy

**Status: IMPLEMENTED 2026-05-31.** Offline regression on CleanFee /
LaunchBlock / RewardTracker / v16: no false positives (combined risks
60 / 85 / 70 / 125 respectively — only v16 grace solo trips softSkip).

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

| Test | Hook source | Purpose | Cost (sepolia ETH) |
|---|---|---|---|
| v17 | solo vector #9 (basefee + blockhash) | Verify Layer A scores 110+ → softSkip | ~0.04 |
| v18 | solo vector #10 (wallet-tracker) | Verify Layer A SSTORE_BEFORESWAP_ONLY softSkip | ~0.04 |
| v19 | solo vector #2 (gasleft) | Verify Day 10 Fix 1 EVM_GLOBAL_BEFORESWAP_ONLY catches | ~0.04 |
| v20 | solo vector #3 (coinbase) | Same | ~0.04 |
| v21 | solo vector #6 (tx.origin) | Same | ~0.04 |
| v22 | solo vector #7 (manual activate) | Confirm Day 9 Fix B sell-time guard catches at 30% threshold | ~0.04 |
| v23 | solo vector #8 (conditional revert) | Verify delayedTaxProbe handles future-revert correctly | ~0.04 |
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
