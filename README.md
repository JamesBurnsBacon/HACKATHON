# HACKATHON — TOKEN2049 Origins

> **Working name: TBD** ❓ · Status: design draft, no code yet

## 1. Vision
Use AI + open onchain data to rank Hyperliquid perps traders and vaults by **risk-adjusted** returns, then run a live portfolio that copies the winners — with every pick explained on a public dashboard.

**Hackathon themes hit:** AI x Crypto · DeFi · Infrastructure (Chainlink CRE).

## 2. How it works (4 steps)

| # | Step | Output |
|---|------|--------|
| 1 | **Collect** performance data for HL traders & vaults | Normalized trader/vault history in our DB |
| 2 | **Score** — algorithmic screen, then LLM review | Ranked shortlist with reasons + red flags |
| 3 | **Curate & execute** — AI builds risk buckets, we trade one live on mainnet | Live portfolio (small capital) |
| 4 | **Show** analysis + live performance | Dashboard for demo |

```
      ┌──────────── Chainlink CRE workflow (cron) ────────────┐
      │                                                       │
 HL Info API ──► Ingest ──► Quant screen ──► LLM reviewer ──► Bucket curator ──► Executor ──► Hyperliquid
 HL RPC / HyperEVM   │           │                │                 │               │        (mainnet)
 hyperliquidvaults ──┘           └────────────────┴─────► DB ◄──────┴───────────────┘
                                                          │
                                                     Dashboard (frontend)
```

## 3. Components

### 3.1 Data ingestion
- **Sources:** Hyperliquid Info API (account state, fills, funding, PnL history, vault details), Hyperliquid RPC/HyperEVM, hyperliquidvaults.com, [HyperTracker](https://app.coinmarketman.com/hypertracker) (backup/cross-check).
- **Candidate universe:** leaderboard addresses + vaults.
  - The [HL leaderboard](https://app.hyperliquid.xyz/leaderboard) UI wasn't loading on our last check. ❓ *Is the underlying data endpoint still up? If not, fall back to HyperTracker. Does HyperTracker have an API we can use, and what are its terms?*
  - **Vaults:** HyperCore (legacy) vaults are only a subset. **ERC-4626 vaults on HyperEVM** are the approach the HL team recommends, and they may show up as ordinary accounts. ❓ *How do we detect and label ERC-4626 vault addresses (factory events, hyperliquidvaults.com, manual list)?*
- Store raw + normalized snapshots so scoring is reproducible.

### 3.2 Scoring (algo first, AI second)
1. **Hard filters:** minimum history (e.g. ≥30 days), min trade count, min account value, max leverage, max drawdown cap.
2. **Baseline ranking:** the same style of metrics the HL leaderboard ranks by (account value, PnL, ROI, volume over 1D / 7D / 30D / all-time windows), so our rankings are familiar and easy to check against.
3. **Risk-adjusted metrics:** Sharpe, Sortino, Calmar, max drawdown, win rate, PnL consistency (not one lucky trade), exposure/leverage profile.
4. **LLM reviewer (Claude):** reads finalists' stats + trade patterns, flags red flags (martingale, wash-like behavior, single-asset concentration), writes a plain-English rationale for each pick.

❓ *Exact weights/thresholds — to be tuned by backtest.*

### 3.3 Portfolio curation (AI agent)
- Agent assembles **risk buckets** from the scored shortlist, e.g. **Conservative / Balanced / Aggressive**, each with target allocations.
- **MVP — static buckets:** agent publishes the buckets with rationale; the user picks one. We trade one bucket live.
- **Stretch — conversational onboarding:** agent asks the user a few questions (risk tolerance, horizon, drawdown comfort) and places them in a bucket.

### 3.4 Execution (Hyperliquid mainnet, small capital)
- Copy targets: **vaults (deposit) and/or individual traders (mirror positions, scaled)**. ❓ *Vaults, traders, or both? Vaults are simpler to execute; mirroring is more novel.*
- Use a Hyperliquid **API/agent wallet** (can trade, cannot withdraw) to limit blast radius.
- **Risk controls:** per-target allocation cap, max portfolio leverage, max slippage, global kill switch.
- **Starting capital: 5 HYPE.**
  - ❓ *HL perps margin is in USDC. Do we swap HYPE → USDC on HL spot, or keep HYPE exposure (e.g. HYPE-collateral markets / vaults)?*
  - ❓ *With 5 HYPE, how many targets can we copy before order-size and vault-deposit minimums bind?*
  - ❓ *Who funds the wallet?*

### 3.5 Orchestration (Chainlink CRE)
- CRE workflow on a **cron trigger**: fetch → score → curate → rebalance decision, with results recorded for the dashboard.
- ❓ *Can the CRE workflow sign Hyperliquid orders directly (secrets in CRE), or does CRE emit a verified rebalance decision that a small executor service signs?* — **spike this first, it shapes the architecture.**
- **Rebalance every 10 minutes** (~144 runs/day). Only trade when drift exceeds a threshold, to keep fees and API load down.

### 3.6 Dashboard
- Leaderboard of scored traders/vaults with metrics + LLM rationale.
- Risk buckets and their composition.
- Live portfolio: positions, PnL vs. benchmark (e.g. hold BTC / HLP), rebalance log.
- ❓ *Frontend stack — Chris's call (Next.js suggested).*

## 4. Stack
- **Language:** TypeScript everywhere.
- **Hyperliquid:** TS SDK ❓ *(which one — official API via raw HTTP vs a community SDK)*.
- **Orchestration:** Chainlink Runtime Environment (TS workflows).
- **AI:** Claude API (Anthropic TS SDK) for the reviewer and the bucket curator.
- **Storage:** ❓ *(SQLite/Postgres/Supabase — whatever is fastest)*.
- **Coinbase AgentKit:** ❓ *Open. Candidate uses: the curator agent's tool-calling layer, or funding/bridging USDC into HL. Skip if it doesn't earn its place.*

## 5. Team & ownership

| Who | Focus |
|-----|-------|
| James | Architecture, HL execution, CRE integration |
| Bradley | Data processing, backend APIs, scoring logic + tests |
| Masa | Backend ❓ *(ingestion? LLM agent? — to confirm)* |
| Chris | Frontend / dashboard |

James is away **midday on the 6th** — CRE/execution handoff notes needed before then.

## 6. Working rules
- Small, independently tested modules; integrate only tested pieces.
- Budget **~1h testing/bugfix per 2h of feature work**.
- Fallback for every live dependency (cached data for the demo if an API flakes).

## 7. Milestones
1. **Spikes:** HL data pull for 1 address/vault · CRE hello-world cron · HL order from agent wallet.
2. **MVP pipeline:** ingest → quant score → ranked list in DB.
3. **AI layer:** LLM reviewer + bucket curator.
4. **Live:** execution on mainnet with risk caps; dashboard shows live PnL.
5. **Stretch:** conversational bucket onboarding, position mirroring, onchain score attestation.

❓ *Map milestones to actual hackathon hours once the schedule is confirmed.*

## 8. Risks
- **Survivorship / overfitting:** past winners ≠ future winners → use out-of-sample backtest window.
- **Copy latency & slippage** when mirroring traders.
- **Key security** on mainnet → agent wallet, small capital, kill switch.
- **CRE ↔ Hyperliquid signing** may not be straightforward (see 3.5).
- **API rate limits** on HL Info API, especially with 10-minute runs.
- **Data source availability:** the HL leaderboard may be down → HyperTracker fallback + cached snapshots.
- **Tiny capital:** 5 HYPE may run into minimum sizes and makes fees a large share of PnL.

## 9. Open questions (summary)

**Decided**
- [x] Network: HL mainnet, starting capital 5 HYPE
- [x] Orchestration: CRE as core orchestrator, 10-minute rebalance
- [x] AI: quant screen first → Claude reviewer + bucket curator
- [x] Curation UX: static buckets for MVP, conversational onboarding as stretch
- [x] Sourcing: leaderboard + vaults, HL-leaderboard-style metrics, HyperTracker as fallback

**Open**
- [ ] ❓ Project name
- [ ] ❓ Is the HL leaderboard data endpoint up? HyperTracker API access/terms
- [ ] ❓ How to detect ERC-4626 vaults on HyperEVM
- [ ] ❓ Copy vaults, traders, or both
- [ ] ❓ HYPE → USDC swap vs. HYPE exposure; minimum sizes with 5 HYPE; who funds the wallet
- [ ] ❓ CRE signs orders vs. CRE → executor service
- [ ] ❓ HL SDK choice, storage
- [ ] ❓ AgentKit in or out
- [ ] ❓ Masa's ownership area; frontend stack
- [ ] ❓ Scoring weights/thresholds
- [ ] ❓ Milestone → hackathon-hours mapping
