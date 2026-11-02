# GEMBA Sniper Bot

Public documentation for **Gemba Sniper Bot** (https://gembabots.com), a hosted DeFi sniper bot for
Uniswap V3 and V4 on Ethereum, Base and Sepolia. Every newly detected pool goes through a multi-stage
honeypot pipeline (bytecode heuristics, V4 hook analysis, delayed-tax and gas-trap probes, transfer
simulation and a real $0.03 on-chain buy/sell micro-test) before a real buy is placed.

**The source code is private** (proprietary software of Gemba EOOD). This repository holds
documentation only — this overview and the red-team attack catalogue in [`ATTACKS.md`](ATTACKS.md).
It contains no code.

## Wallet custody — said plainly

The service is **custodial**. A sniper has to sign while its user is offline, so the bot wallet's
private key is held on the operator's server:

- The key is generated or imported in the user's browser and sealed there to a separate custody
  service. Only that service can open it; it keeps the key encrypted and signs on the user's behalf.
- It signs only the bot's own whitelisted operations — swaps whose output pays back to the same bot
  wallet, WETH wrap / unwrap, approvals to the routers the bot uses — within per-trade and daily
  limits, with a hash-chained audit trail, an emergency stop and key rotation.
- There is no key export, and transfers to other addresses are refused. Users keep their own copy of
  the key to move funds outside the bot.

Earlier versions of this README described a key wrapped with a value derived from the user's email.
That scheme was replaced on 2 November 2026 and is no longer used.

## Other facts

- **MEV:** on Ethereum mainnet trades are sent only through private relays (Flashbots Protect, then
  MEV Blocker); if none accepts a trade, it is not sent. Base has no public mempool, so no relay is
  used there.
- **Fees:** subscriptions are billed through GembaPay (currently in test mode). Sells through Uniswap
  V3 pools pay a 1 % on-chain platform fee.
- **Status (2 November 2026):** mainnet signing is switched off; only Sepolia is enabled.
- **No profit claims.** Trading newly launched tokens is high-risk; users can lose all funds the bot
  trades. See https://gembabots.com/compliance.

## License

Proprietary — see [`LICENSE`](LICENSE).
