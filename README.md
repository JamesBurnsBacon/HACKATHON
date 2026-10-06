# PerpParrot 🦜

> Copy the best Hyperliquid perps traders and vaults, picked by quant screens and an AI agent, orchestrated by Chainlink CRE.
>
> **TOKEN2049 Origins Hackathon** · Tracks: **Chainlink (CRE)** · **AI x Crypto** · Status: design doc, no code yet

**TL;DR**
- Score Hyperliquid addresses (traders, HyperCore vaults, ERC-4626 vaults) on risk-adjusted performance.
- An LLM agent picks a set of **5–25 source wallets**. Two models are compared in the backtest, and the winner runs live.
- Every 10 minutes, Chainlink CRE verifies their positions and emits a signed rebalance report.
- An executor holds the **weighted, netted** copy of those positions in our own Hyperliquid account (5 HYPE, about $470).
- **The backtest is the proof:** in-sample selection vs. out-of-sample results over 2 weeks, 1 month, 6 weeks and 3 months, compared with holding BTC. The ~5 h live run shows the machinery working.

---

# Part 1: Design

## 1. Scope

**Build window:** 36 h, from **Tue 6 Oct 12:00 SGT** to **Thu 8 Oct 00:00 SGT**. Team of 3–4. The final demo is a **recorded video**.

**Live window:** go live at **~Wed 19:00 SGT**, so ~5 h and ~30 mirror runs before the deadline.

**In scope**
- Score and select source wallets.
- Backtest the selection.
- Live mirror of one bucket on mainnet, orchestrated by CRE.
- Dashboard.

**Non-goals:**
- managing other people's money
- depositing into vaults (Conservative's lending legs are paper-only during the hackathon)
- a token
- mobile
- HFT-grade latency
- guaranteed returns

**Video storyline (~3 min):**
1. **Funnel:** ~14.5k "profitable" leaderboard addresses → our filters → the source set. Show why most fail.
2. **Backtest:** OOS equity curves for 2 weeks, 1 month, 6 weeks and 3 months vs. holding BTC. *This is the value claim.*
3. **Finalist drill-down:** metrics, the agent's rationale, red flags.
4. **Live:** the CRE run log, HyperEVM decision hashes, executor fills, and per-source PnL attribution over the ~5 h live window.

## 2. Terminology

| Term | Meaning |
|---|---|
| **Source wallet** | An address we copy: a trader, a HyperCore vault, or the HyperCore account behind an ERC-4626 vault. We copy **5–25** of them. |
| **Slice** | Our scaled copy of one source's position in one asset. Kept in a **virtual ledger** per (source, asset). |
| **Position** | Our actual **net** per-asset holding on Hyperliquid, which is the sum of all slices for that asset. **Positions are the bottleneck**: ~5–10 net positions above the $10 minimum with ≈ $470. |
| **Bucket** | A risk tier: a weighted source set plus a leverage policy. |

## 3. How it works

```
 BACKEND (Railway)                    CRE: review (hourly)            CRE: mirror (every 10 min)         EXECUTOR (serverless)
┌──────────────────────────┐ shortlist ┌───────────────────────┐ notes ┌─────────────────────────────┐ signed ┌──────────────────────┐
│ leaderboard + vault list │──────────►│ LLM agent (2 models)  │──────►│ fetch positions snapshot (1)│ report │ verify DON signature │
│ ingest → label → score   │           │ monitor sources, flag │       │ spot-check ~10 sources (HL) │───────►│ dedupe report ID     │──► Hyperliquid
│ backtest · ledger        │ positions │ risks, write rationale│       │ slices → net → diff vs. ours│        │ sign w/ API wallet   │    (our account)
│ positions snapshot API   │──────────►└───────────────────────┘       │ report() → executor+HyperEVM│        └──────────┬───────────┘
└────────────┬─────────────┘                                           └─────────────────────────────┘                   │
             └───────────────────────────────► Supabase ◄──────────── fills / PnL / ledger ◄─────────────────────────────┘
                                                  │
                                        Dashboard (Next.js on Vercel)
```

## 4. Components

### 4.1 Ingest (backend)
- **Universe:**
  - the leaderboard file (~47.5k addresses)
  - the HyperCore vault list (~3.1k open)
  - keep only those with ≥ **$10k** account value / TVL
- **Label each address's kind:**
  - HyperCore vault if it's in the vault list.
  - **ERC-4626 vault** if `eth_getCode` on HyperEVM returns code *and* `asset()` / `totalAssets()` succeed. This runs on the shortlist only.
  - Trader otherwise.
- **History:** `portfolio` PnL history for the pre-filtered top few hundred. `clearinghouseState` for current positions.
- Cache the leaderboard every few hours and save every snapshot.

### 4.2 Score (backend)
1. **Hard filters:**
   - ≥ $10k account value / TVL
   - **≥ 30 days active**
   - **≥ 10 trades**
   - not closed
2. **Metrics:** computed from PnL history, not account value, so deposits and withdrawals don't count as returns.
   - Sharpe, Sortino, Calmar
   - max drawdown
   - PnL consistency
   - realized volatility
   - average leverage
3. **Composite score = average percentile rank** across metrics (Sortino, Calmar, consistency, −max drawdown, …). This is robust to outliers and needs no scale tuning. **Top ~25 → finalists** for the agent (§4.6).

### 4.3 Buckets (risk tiers)
| Bucket | Source universe | Exposure | Hackathon mode |
|---|---|---|---|
| **Aggressive** | Traders + vaults (HyperCore and ERC-4626) | Mirrored exactly, limited only by the 95% margin rule (§4.8) | **Live** |
| **Balanced** | **Identical source set to Aggressive** | Aggressive × a leverage multiplier `m < 1`, **tuned from the backtest** to hit a target vol/drawdown | **Backtest + paper-tracked** |
| **Conservative** | **A separate, lower-risk universe: vaults + lending only**, no individual traders | We **copy the sources' lending/yield positions** with the same ledger model, alongside their perp positions | **Backtest + paper-tracked** |

- The agent picks and weights sources within each universe and writes the rationale.
- ❓ *How do we read sources' lending positions? Current state via the HyperCore borrow/lend user-state Info call and EVM lending-vault balances; history for the backtest is unclear.*

### 4.4 Copy model: virtual ledger
- **Slice** for source `i`, asset `c`: `slice_i,c = wᵢ × (nᵢ,c / Eᵢ) × E_ours`.
  - `nᵢ,c` is the source's signed notional in asset `c`.
  - `Eᵢ` is the source's **current** equity, and `E_ours` is ours. Scaling by current equity means we copy the source's exposure ratio, so its deposits and withdrawals don't distort our size.
- **Flat is not a signal:** each run, weights are **renormalized over sources that currently hold positions**: `wᵢ' = wᵢ / Σ_active wⱼ`. One source going flat doesn't shrink our total exposure.
- **Position** in asset `c` = `Σᵢ slice_i,c`, netted only at order time.
- **Reconciliation:** the ledger is the *target* and the account is the *truth*. Each run diffs the targets against the real account, so partial fills, skipped legs and partial liquidations self-correct with no separate repair logic. Per-source PnL attributes realized fills pro-rata.
- **Trade a leg only if** the gap between the target and our position is **≥ $10 and ≥ 10%** of the target.
- **Which assets get mirrored:**
  - validator perps, plus **HIP-3 perps with USDC collateral**
  - **every asset must have ≥ $20M open interest**. The line is set so it just includes the Microsoft HIP-3 market.
  - Anything else (spot, HIP-3 with other collateral, thin markets) is skipped and shows up as tracking error.
  - **Isolated-only markets (`onlyIsolated`: `noCross` / `strictIsolated`) are excluded**, so we keep one cross margin pool. As of 2026-10-06 that leaves **59 assets**: 38 core perps + 21 HIP-3.
  - The OI floor is checked **at go-live (then daily)**. An intraday dip below $20M is ignored until the next check.
- The ledger is stored in Supabase. It enables per-source PnL attribution and clean removals (§4.5).

### 4.5 Source-set changes
**Hackathon: the set is fully frozen at go-live (~Wed 19:00 SGT).** No reselection and no ejections. The hourly agent run monitors and comments only.

**Production design** (reselection daily or weekly). Removals come in two distinct types:

| Type | Cause | Handling |
|---|---|---|
| **A: edged out** | A better performer takes the spot | Recompute targets with the new mix. Legs left **misaligned with the new allocation become reduce-only**: we follow only the old source's reductions and closes, so it stays the reference and nothing is orphaned. After a time limit, **DCA out in 2–3 trades over a few hours**. |
| **B: performed badly** | The source's **own account drawdown ≥ N%** from peak, measured on PnL history (not deposits/withdrawals). **N depends on the tier** (e.g. 15% Conservative / 25% Balanced / 40% Aggressive) | **Immediate exit** of its slices on the next run, netted against the other slices. |

❓ *What are the per-tier values of N, the type-A time limit, and the DCA spacing?*

### 4.6 AI layer: LLM agent inside the CRE `review` workflow (hourly)
This is a dedicated workstream, integrated into the CRE flow.
- **Role (before go-live):** the agent **picks 5–25 sources from the ~25 algo finalists** and assigns weights, with a rationale and red flags (martingale, wash-like behavior, concentration, near-liquidation). The algo sets the minimum bar; the agent adds judgment.
- **The agent also judges:**
  - **Diversification:** near-duplicate sources, using the return-correlation matrix plus vault ↔ leader address links.
  - **Time in market:** *flat is not a signal*, so avoid sources that are often flat.
- **After go-live:** monitoring and commentary only. The set stays frozen.
- **Inputs for each finalist:**
  - the metrics table and kind
  - an equity-curve summary (~30 daily points)
  - current positions (asset, size, leverage, distance to liquidation)
  - recent trade patterns (frequency, holding time, averaging down)
  - time in market
  - the finalists' correlation matrix and vault ↔ leader links
  - **Budget:** ≈ 25 × ≤ 4 KB, which keeps the prompt under CRE's **120 KB request limit**.
- **Models:** **two models are compared in the backtest; at least one is from OpenAI.** ❓ *Which two (e.g. Claude Sonnet 5.5 vs. an OpenAI model)?* **The backtest winner drives the live set.**
  - Structured JSON output.
  - API keys are CRE secrets, which is acceptable since every node can read them.
- **Consensus:** **Default: every DON node calls the model**, with **per-field consensus** (❓ still open). That's truly decentralized AI, at N× the API calls per run. The alternative is a single call via `cacheSettings`.*
- **Output format:** ❓ *Open. Options:*
  - *a discrete weight grid (e.g. 0–3 units), which works well with identical-result consensus*
  - *continuous weights with median consensus*
  - *a ranking plus a fixed weight formula*
- **Logging:** every run's full prompt and output are stored in Supabase with their hashes, for reproducibility and so the dashboard can show exactly what the model saw and said.
- **Proving value:** the backtest compares **algo-only top-N vs. algo + model A vs. algo + model B** (§4.9).
- **Live shadow:** after go-live, the losing model's picks are tracked **on paper** (no orders), so the video can show both side by side.
- **AI workstream's first deliverable:** a **CRE `review` workflow spike** that proves an LLM call plus consensus works in `cre workflow simulate`.
  - Then: I/O schemas in `packages/shared`, a prompt + offline eval, and the point-in-time backtest harness.

### 4.7 Mirror (CRE workflow, every 10 min, cron `0 */10 * * * *`)
1. Fetch the backend's **positions snapshot** for all sources (1 call; up to 25 sources). The backend **refreshes it just before each run**: a cron at `:x9`, one minute ahead of each 10-minute mark.
2. **Spot-check:** re-read `clearinghouseState` directly from Hyperliquid for ~10 random sources plus our own account. If they disagree beyond a tolerance, reject the run. ❓ *Tolerance, since positions can move between reads.*
3. Slices → net positions → diff against our account, applying the drift rule from §4.4.
4. `report()` → POST to the executor, plus the report hash to a HyperEVM consumer contract.

**Freeze commitment:** at go-live, the `review` workflow writes a hash of the **frozen source set + weights** to HyperEVM *before* the first trade. That proves the picks were made before the live results.

That totals **≤ 13 HTTP calls**, under CRE's limit of 15.

### 4.8 Execute (serverless executor)
- Verify the DON signature and dedupe by report ID.
- **Sanity bounds:** reject a report if its source set differs from the frozen set, or if total notional exceeds a bound (e.g. 50× equity).
- Send **IOC limit orders priced at mark ± a slippage cap**. Any unfilled remainder is retried on the next run.
- **Keys:** a fresh EOA. A human holds the master key; the executor holds only an **HL API wallet** key (trade, no withdraw).
- **Leverage management:**
  - **Cross margin for all assets.**
  - Each asset is set to **its max leverage** (`updateLeverage`; e.g. many HIP-3 markets default to 10x, BTC up to 40x). This locks the least margin per leg.
  - The sources' **exposure ratios are mirrored exactly**, unless margin binds.
  - **Feasibility check each run:** required initial margin `Σ |N_c| / maxLev_c` must stay **≤ 95% of our equity**. If it doesn't, **scale all targets down pro-rata** by `k = 0.95·E_ours / Σ |N_c| / maxLev_c`, which keeps the mix and under-copies everything.
- **Kill switch:** manual only, with two buttons that any team member can press (Supabase auth):
  - **Pause:** stop trading and keep positions.
  - **Flatten:** close everything.
- **Alerts:** a Telegram bot reports rejected runs, failed orders and big drawdowns, with a link to the dashboard.
- **Capital:** 5 HYPE, currently **on HyperEVM**.
  - Keep **~0.1 HYPE on HyperEVM** for the consumer contract's gas.
  - Move ~4.9 HYPE to HyperCore by sending it to the HYPE system address `0x2222…2222` (verify in a spike).
  - **Swap to USDC on HyperCore spot** (`@107`) to margin perps. Portfolio margin, which would let HYPE back positions, needs a $10k balance, and we have just under $500.
- **After submission:** keep running through judging, with live PnL on the public dashboard. Shut down after results.

### 4.9 Backtest (backend): the main value claim
- **Return-based:**
  - Portfolio return ≈ `Σ wᵢ · rᵢ` of the sources' PnL-history returns.
  - Subtract a fee and slippage haircut **based on modeled turnover**: estimate each source's turnover from volume / equity, then charge the HL taker fee plus a few bps of slippage per unit.
  - This follows from §4.4: equity-ratio scaling makes our return a weighted sum of source returns.
- **In-sample** is all of an address's history **before** the cut. **Out-of-sample** windows are the **last 2 weeks, 1 month, 6 weeks, and 3 months**.
- Report each window against holding BTC.
- **Algo-only vs. model A vs. model B:** at each OOS cutoff, each model sees **only data from before the cut** and picks from the algo finalists. Compare OOS returns; the winner selects the live set.
  - **One cutoff per window:** 4 cutoffs × 2 models, plus algo-only.
  - **No peeking:** prompts are **anonymized** (no addresses, display names, vault names or calendar dates) and **truncated** at the cut, so the models can't recall famous wallets or market events.
  - ⚠️ Point-in-time *positions and trade patterns* can only be rebuilt from fills (≤ 10k most recent). The backtest-time agent may therefore get only metrics plus the truncated equity curve. Disclose this.
- **Data resolution:** OOS windows within the last 30 days have ≈ daily points (`month` window). The 6-week and 3-month windows only have ≈ weekly points (`allTime`).
- **Caveat:** the leaderboard is a *current* snapshot, so every address is a survivor and the backtest flatters us. State this in the video.
- *Stretch:* position replay from fills (≤ 10k most recent per address), simulating the 10-minute loop, the $10 minimum and netting.

### 4.10 Paper books (backend only, not via CRE)
- **Books:**
  - **Balanced** (Aggressive × `m`)
  - **Conservative** (vaults + lending)
  - the **losing model's** Aggressive picks (shadow)
- **Sizes:** each book is tracked at **two starting equities**: **~$470**, the same as live including the $10-minimum effect, and **$10k**, which shows the strategy without minimum-order distortion. The live Aggressive book also gets a $10k paper twin.
- **Fills:** each 10-minute run uses the same positions snapshot. Paper fills happen at **HL mark price, plus the taker fee, plus the backtest's slippage bps**, with the $10 minimum applied.

### 4.11 Dashboard (Next.js + Tailwind on Vercel, Supabase realtime): **public, read-only**
The kill switch and admin actions sit behind auth.
- the funnel
- backtest charts for each OOS window vs. BTC
- finalist drill-down with the agent's rationale
- buckets
- live net positions vs. targets
- **per-source PnL from the ledger**
- the CRE run log with HyperEVM hashes

### 4.12 Module contracts
| Module | Input | Output (Supabase / API) |
|---|---|---|
| `ingest` | leaderboard, vault list, HL Info API, HyperEVM RPC | `snapshots` (kind, equity, PnL history, positions) |
| `score` | `snapshots` | `candidates` (filters, metrics, score, tier) |
| `backtest` | `snapshots`, `candidates` | `backtests` (per OOS window: curve, stats, vs. BTC) |
| `review` (CRE) | candidates | `reviews`, `buckets` |
| `positions` | source set | positions snapshot API |
| `mirror` (CRE) | positions snapshot + spot-checks + `ledger` | signed report + HyperEVM hash |
| `execute` | report | `orders`, `fills`, updated `ledger` |
| `paper` | positions snapshot, buckets, mark prices | `paper_books` (Balanced, Conservative, shadow; $470 and $10k) |
| `dashboard` | all of the above | — |

## 5. Stack
| Layer | Choice |
|---|---|
| Language | TypeScript everywhere |
| Hyperliquid | [`@nktkas/hyperliquid`](https://github.com/nktkas/hyperliquid) in the backend and executor; raw HTTP inside CRE |
| Orchestration | Chainlink CRE, `@chainlink/cre-sdk` + `cre` CLI, with DON access from the sponsor on-site |
| Onchain log | CRE consumer contract on HyperEVM |
| Prices | **Hyperliquid's own oracle/mark prices** via the Info API (`metaAndAssetCtxs` → `oraclePx` / `markPx`). BTC benchmark history comes from `candleSnapshot`. We don't use Chainlink Data Feeds. |
| AI | Two LLMs (at least one OpenAI; ❓ which), structured JSON. The backtest winner runs live. |
| Storage | Supabase (Postgres + realtime) |
| Hosting | Backend on Railway (the leaderboard download is too slow for serverless); executor + dashboard on Vercel |
| Frontend | Next.js + Tailwind |
| Optional infra | **NOWNodes** (`hype.nownodes.io`): HyperEVM RPC + an Info API copy that includes `clearinghouseState`. Use it if it helps with rate limits. **Not a target track.** |
| Not used | Coinbase AgentKit |
| Repo | **pnpm monorepo**: `packages/{backend, cre-workflows, executor, dashboard, contracts, shared}`. `shared` holds the types and JSON schemas, which keeps the AI and backend in sync. No license yet. |
| Testing | Unit tests on fixtures, then **tiny mainnet runs** ($10–20) before the freeze. No testnet. |
| Secrets | Railway/Vercel env vars, plus CRE secrets (Vault DON) for LLM keys. `.env.example` in the repo, never real values. |

## 6. Timeline (all times SGT)
Budget ~1 h of testing per 2 h of feature work. Integrate only tested modules. **Video ≤ 3 min.**

| When | Milestone |
|---|---|
| **Before Tue 12:00** | **Handoff notes** (one member is away midday Tue): repo scaffold, Supabase schema, env vars, task split |
| Tue 12–16 | DON access from the sponsor · fresh wallet + API wallet · move HYPE from HyperEVM to HyperCore (keep 0.1) and swap to USDC · spikes: `portfolio` pull, $10 IOC order, CRE cron + HTTP in simulation |
| Tue 16–24 | `ingest` + `score` · kind labeling · start saving snapshots |
| Wed 00–08 | **`backtest`** (return-based, 4 OOS windows, algo-only vs. model A vs. model B) · `review` workflow (agent in CRE) |
| Wed 08–15 | `positions` API · `mirror` workflow · executor (signature check, dedupe, ledger) · end-to-end dry run with tiny size |
| Wed 15–19 | Dashboard: funnel, backtest charts, ledger PnL, CRE log · HyperEVM consumer contract · **freeze the source set** |
| **Wed 19:00** | **Go live** on mainnet. **Gate:** the backtest winner is chosen and the set + weights hash is committed on HyperEVM; a $10–20 end-to-end mainnet run has passed (CRE report → executor → fill → ledger → dashboard) |
| Wed 19–Thu 00 | Monitor, record the video. **Before submitting:** make the README judge-facing (results, demo link, how to run) and move all design content to `docs/DESIGN.md` |

## 7. Risks
- **Backtest survivorship** → stated openly; our own snapshots start a forward record.
- **Short live window (~5 h)** → the value claim rests on the backtest; live only proves the mechanism.
- **No leverage cap and no automatic halt (Aggressive)** → liquidation is possible. *Mitigated* only by the 95% initial-margin rule and pro-rata scaling. *Accepted:* small capital, Pause/Flatten buttons, Telegram alerts, monitored throughout.
- **$10 minimum vs. ≈ $470** → only ~5–10 net positions are possible; small legs get skipped. Show tracking error.
- **Copy lag (≤ 10 min) and slippage** → IOC orders, slippage limit, drift rule, $20M OI floor.
- **No testnet** → fixtures plus $10–20 mainnet runs before the freeze.
- **Snapshot vs. spot-check mismatch** from positions moving between reads → tolerance and timestamps.
- **Undocumented data endpoints** → cached snapshots; HyperTracker/NOWNodes as fallbacks.
- **Duplicate orders** → report-ID dedupe plus HL nonces.

## 8. Decisions & open questions

**Decided**
- [x] Name **PerpParrot** · tracks: Chainlink + AI x Crypto · recorded video · go live ~Wed 19:00 SGT
- [x] Copy model: mirror positions of **5–25 source wallets** in our own account. Per-(source, asset) virtual ledger, scaled by current equity, netted at order time. Positions (~5–10) are the bottleneck.
- [x] Drift rule: trade a leg only if the gap is ≥ $10 and ≥ 10%
- [x] Buckets tiered by vol + leverage. Balanced = the Aggressive set at lower leverage. **Aggressive is live**, leverage mirrored exactly, manual kill switch.
- [x] Source set **fully frozen** during the hackathon. Production: type A (edged out) → reduce-only, then a DCA exit; type B (own drawdown ≥ N%) → immediate exit.
- [x] Backtest: return-based; IS = all history before the cut; OOS = 2 wk / 1 mo / 6 wk / 3 mo vs. BTC. Position replay is a stretch goal.
- [x] CRE: hourly `review` + 10-minute `mirror`. Mirror uses the backend snapshot plus a direct spot-check of ~10 sources. CRE decides, the executor signs; hash on HyperEVM.
- [x] Capital: 5 HYPE → USDC on HyperCore spot (portfolio margin needs $10k)
- [x] Prices: Hyperliquid oracle/mark prices via the Info API, not Chainlink oracles
- [x] AI: the agent picks 5–25 sources from ~25 algo finalists. It sees metrics, equity curve, positions, trade patterns, time in market and correlations, and judges diversification and flatness. Two models (≥ 1 OpenAI) are compared against algo-only in the backtest; the winner runs live.
- [x] Backtest haircut: modeled from turnover. Type-B threshold is tier-dependent.
- [x] Markets: validator perps + USDC-collateral HIP-3, every asset ≥ $20M OI. Orders: IOC limit with a slippage cap.
- [x] Snapshot refreshed just before each CRE run · keep 0.1 HYPE on HyperEVM for gas
- [x] Score = average percentile rank; filters ≥ $10k, ≥ 30 days active, ≥ 10 trades
- [x] Weights renormalized over active (non-flat) sources · ledger = target, account = truth
- [x] Executor sanity bounds (frozen set, notional cap) · Pause + Flatten buttons for any team member · Telegram alerts
- [x] Buckets: Aggressive (live) and Balanced share one source set; Balanced = Aggressive × `m`. Conservative uses a separate vaults + lending universe and copies the sources' lending positions. Balanced and Conservative are backtested + paper-tracked.
- [x] AI workstream starts with a CRE `review` workflow spike (LLM call + consensus in simulation)
- [x] Go-live gate: backtest winner chosen + set frozen and committed; tiny mainnet run passed · secrets in platform env vars + CRE secrets · before submission, the README becomes judge-facing and design moves to `docs/DESIGN.md`
- [x] Paper books (Balanced, Conservative, shadow model) run in the backend only, at ~$470 and $10k, with fills at mark + fees + slippage
- [x] Backtest: one cutoff per OOS window, anonymized + truncated prompts · the losing model is shadow-tracked on paper after go-live
- [x] Leverage: cross margin, each asset at max leverage, exposure mirrored exactly; initial margin ≤ 95% of equity, else pro-rata scale-down
- [x] Isolated-only HIP-3 markets excluded (59 eligible assets on 2026-10-06) · OI floor checked at go-live, then daily · source set + weights hash committed on HyperEVM at freeze · full LLM prompts/outputs logged with hashes
- [x] pnpm monorepo, no license yet · fixtures + tiny mainnet testing · public read-only dashboard · video ≤ 3 min · keep running through judging
- [x] Stack: TS, `@nktkas/hyperliquid`, two LLMs, Supabase, Next.js + Tailwind, Railway + Vercel. NOWNodes optional; no AgentKit.

**Open**
- [ ] ❓ Composite score weights
- [ ] ❓ Balanced leverage multiplier `m` (tune from backtest)
- [ ] ❓ How to read sources' lending positions (now and historically) for Conservative
- [ ] ❓ Per-tier type-B threshold N; type-A time limit and DCA spacing (production design)
- [ ] ❓ Spot-check tolerance between the snapshot and direct reads (tune in the dry run)
- [ ] ❓ Which two models (≥ 1 OpenAI)
- [ ] ❓ LLM in CRE: **default is every node + per-field consensus**; still open vs. a single cached call
- [ ] ❓ Agent output format: weight grid, continuous weights, or ranking

---


# Part 2: Research notes (checked 2026-10-06)

### Hyperliquid data
- **Leaderboard:** `GET https://stats-data.hyperliquid.xyz/Mainnet/leaderboard`. Undocumented static file, refreshed every few minutes.
  - **~40 MB, 47,495 rows.** Downloads take 35 s to 2 min+, which is probably why the [UI](https://app.hyperliquid.xyz/leaderboard) fails to load.
  - Each row has `ethAddress`, `accountValue`, `displayName`, and `pnl` / `roi` / `vlm` for `day` / `week` / `month` / `allTime`.
  - 20.9k rows have ≥ $10k account value. 14.5k have ≥ $10k *and* positive month + all-time PnL.
- **Vault list:** `GET https://stats-data.hyperliquid.xyz/Mainnet/vaults` (~14 MB, ~11 s, undocumented).
  - 9,476 vaults, 3,091 open. **234 / 82 / 26** open vaults have TVL ≥ $10k / $100k / $1M.
  - The HL docs call HyperCore vaults **legacy** (perps only, no spot/HIP-3, 10% leader profit share, 1-day depositor lockup).
- **Info API** (`POST https://api.hyperliquid.xyz/info`):
  - `portfolio`: account value + PnL history for `day` / `week` / `month` / `allTime` (+ `perp*` versions).
    - Only **~100 points per window**: ≈ daily for `month`, ≈ weekly for `allTime`.
  - `clearinghouseState` gives positions. `userFillsByTime` gives fills; only the 10k most recent are reachable, and older fills are in the requester-pays S3 bucket `hl-mainnet-node-data`. Also `userFunding` and `vaultDetails`.
- **Rate limits:** 1200 weight/min per IP. Most info calls cost 20; `clearinghouseState` / `allMids` cost 2.
- **HyperTracker** (CoinMarketMan): paid API with a free tier of 100 tokens/day, then $179–$1,999/month.

### Hyperliquid execution
- **Markets ≥ $20M OI on 2026-10-06:** 71 total.
  - 38 core perps, all of which allow cross margin.
  - 33 HIP-3: 32 on the `xyz` dex + `io:ANTH`, all with USDC collateral (`collateralToken` 0).
  - **12 HIP-3 markets are isolated-only:** CBRS, MSTR, HOOD, SOXL, CXMT, JPY, SMSN, NBIS, ORCL, ZHIPU, EUR (`noCross`); `io:ANTH` (`strictIsolated`).
  - HIP-3 max leverage ranges 6–50x (10x is common). MSFT sits just above the floor at $20.8M.
  - Source: `perpDexs` + `metaAndAssetCtxs` with a `dex` parameter.
- The minimum order is **$10**.
- API (agent) wallets sign orders and other L1 actions. Withdrawals and transfers need the master wallet.
- **Collateral:** standard perps margin is USDC.
  - HIP-3 markets can use other quote assets.
  - Portfolio margin (HYPE at LTV 0.65) needs > $10k account value or > $5M volume.
  - HYPE/USDC spot is `@107` (order asset id 10107).
- There is no official TS SDK. `@nktkas/hyperliquid` is the best-maintained community SDK and is listed in the HL docs.

### ERC-4626 vaults on HyperEVM
- This is the approach the HL team recommends now. Most of these vaults are lending/yield vaults (Felix/Morpho, Hyperbeat/Midas, Euler, HyperLend). A few are strategy vaults (Liminal basis, Harmonix, D2 Finance).
- DefiLlama yields API (chain "Hyperliquid L1"): 529 pools, 75 with ≥ $1M TVL.
- When a vault trades on HyperCore via CoreWriter, its HyperCore account shares the contract's address, so on the leaderboard it **looks like a normal address**. Hence the `eth_getCode` check.

### Chainlink CRE
- **SDK:** TypeScript SDK `@chainlink/cre-sdk` (v1.x), Go also supported. Not formally GA yet. Workflows run on QuickJS/WASM, so there is no `node:crypto` (use Noble/viem).
- **Cron:** 5- or 6-field expressions, minimum interval 30 s.
- **Quotas per run:**

  | Limit | Value |
  |---|---|
  | Run time | 5 min |
  | Memory | 100 MB |
  | HTTP calls | 15 |
  | Response size | 250 KB |
  | Request size | 120 KB |
  | Data into consensus | 25 KB |
  | Secret fetches | 5 |

- **HTTP:** every node sends every request by default. `cacheSettings` makes a POST single-send, but only best effort.
  - GET results are aggregated with identical, median, or per-field consensus.
- **LLM output** has to be structured JSON, with consensus run per field.
- **Secrets** are decrypted into every node's memory. That's why signing happens in the executor, not in CRE. Chainlink's portfolio-rebalancing template uses the same split.
- **Writes:** ~24 EVM chains, **including HyperEVM** (TS SDK v1.4.0+). Reports go through a forwarder to a consumer contract, max 50 KB.

### NOWNodes (optional infra)
- `hype.nownodes.io` has two parts:
  - **HyperEVM JSON-RPC:** `eth_getCode`, `eth_call`, `eth_getLogs`, `eth_sendRawTransaction`, …
  - **A copy of HL's Info API:** `clearinghouseState`, spot state, vault summaries, user vault equities, `webData2`, …
- Missing from the Info API copy: `portfolio`, fill history, `userFunding`. No `/exchange`, so orders go to HL directly.
- Paid plans advertise unlimited requests per second. It needs an API key. [Docs](https://docs.nownodes.io/hype)

### Coinbase AgentKit
- No Hyperliquid action provider. Its TS providers include `vaultsfyi`, `morpho`, `defillama`, `across`, `erc20`, `x402`. Not used.

### References
- HL: [info endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint) · [rate limits](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits) · [nonces & API wallets](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/nonces-and-api-wallets) · [vaults](https://hyperliquid.gitbook.io/hyperliquid-docs/hypercore/vaults) · [portfolio margin](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/portfolio-margin)
- CRE: [service quotas](https://docs.chain.link/cre/service-quotas) · [HTTP client (TS)](https://docs.chain.link/cre/reference/sdk/http-client-ts) · [non-determinism](https://docs.chain.link/cre/concepts/non-determinism-ts) · [supported networks](https://docs.chain.link/cre/supported-networks-ts) · [deploy access](https://docs.chain.link/cre/account/deploy-access) · [portfolio-rebalancing template](https://docs.chain.link/cre-templates/automated-portfolio-rebalancing)
- Data: [HL leaderboard](https://app.hyperliquid.xyz/leaderboard) · [hyperliquidvaults.com](https://hyperliquidvaults.com) · [HyperTracker](https://hypertracker.io) · [DefiLlama yields](https://yields.llama.fi/pools) · [tradingstrategy.ai ERC-4626 list](https://web3-ethereum-defi.tradingstrategy.ai/tutorials/erc-4626-vault-list)
- SDKs: [`@nktkas/hyperliquid`](https://github.com/nktkas/hyperliquid) · [Coinbase AgentKit](https://github.com/coinbase/agentkit)
