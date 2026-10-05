# HACKATHON — TOKEN2049 Origins

> **Working name: TBD** ❓ · Status: design draft, no code yet

**TL;DR**
- A TypeScript backend scores ~47k Hyperliquid traders and ~80 serious vaults on risk-adjusted metrics.
- A Chainlink CRE workflow runs every 10 min: Claude reviews the shortlist and curates a risk bucket, and CRE emits a DON-signed rebalance report, also logged on HyperEVM.
- An executor verifies the report and trades a 5 HYPE portfolio on mainnet.
- A dashboard shows the picks, the reasoning and live PnL.

## 1. Vision
Use AI + open onchain data to rank Hyperliquid perps traders and vaults by **risk-adjusted** returns, then run a live portfolio that copies the winners — with every pick explained on a public dashboard.

**Hackathon themes hit:** AI x Crypto · DeFi · Infrastructure (Chainlink CRE).

**Non-goals (for the hackathon):** custody of other users' funds, a token, mobile app, HFT-grade latency, guaranteed returns.

**Demo in 3 minutes:**
1. Leaderboard → ~14.5k "profitable" addresses → show how our risk-adjusted filters cut them to a short list (and why most fail).
2. Click a finalist → metrics + Claude's rationale and red flags.
3. Pick a risk bucket → show the live mainnet portfolio copying it, with the CRE rebalance log.
4. Live PnL vs. benchmark since the start of the hackathon.

## 2. How it works (4 steps)

| # | Step | Output |
|---|------|--------|
| 1 | **Collect** performance data for HL traders & vaults | Normalized trader/vault history in our DB |
| 2 | **Score** — algorithmic screen, then LLM review | Ranked shortlist with reasons + red flags |
| 3 | **Curate & execute** — AI builds risk buckets, we trade one live on mainnet | Live portfolio (small capital) |
| 4 | **Show** analysis + live performance | Dashboard for demo |

```
  BACKEND (TypeScript)                      CHAINLINK CRE (every 10 min)              BACKEND
 ┌────────────────────────────┐            ┌──────────────────────────────┐        ┌───────────────┐
 │ HL leaderboard / Info API  │            │ fetch shortlist (consensus)  │ signed │ Executor      │
 │ HyperEVM vaults, HyperTrkr │──► DB ──►  │ Claude review + curate       │ report │ verify report │──► Hyperliquid
 │   ▼                        │ shortlist  │ target weights + drift check │───────►│ risk limits   │    (mainnet)
 │ Ingest ──► Quant screen    │   API      │ report() ──► HyperEVM log    │        │ sign + send   │
 └────────────────────────────┘            └──────────────────────────────┘        └───────┬───────┘
                 │                                                                          │
                 └──────────────────────────► DB ◄── fills / PnL ◄──────────────────────────┘
                                               │
                                          Dashboard
```

## 3. Components

### 3.1 Data ingestion
- **Sources:**
  - **HL Info API (`POST /info`)**
    - `portfolio`: account value and PnL history per window, i.e. the equity curve we score on.
    - `userFillsByTime`, `clearinghouseState`, `userFunding`, `vaultDetails`.
    - Fills: only the **10k most recent** per address are reachable; older fills are in the requester-pays S3 bucket `hl-mainnet-node-data`.
  - **HyperEVM RPC.**
  - **hyperliquidvaults.com**: ranks *HyperCore* vaults only.
  - **[HyperTracker](https://hypertracker.io)**: paid API with a free tier of 100 tokens/day, then $179+/month. Backup/cross-check only.
- **Rate limits:** 1200 weight/min per IP, and most info calls cost 20. That is **~60 history pulls/min**, so a few hundred shortlisted addresses refresh in minutes.
- **Candidate universe:** leaderboard addresses + vaults.
  - **Leaderboard data is available.** The [HL leaderboard](https://app.hyperliquid.xyz/leaderboard) UI was failing to load, but its backing file `GET https://stats-data.hyperliquid.xyz/Mainnet/leaderboard` returns 200. Checked 2026-10-06:
    - **~40 MB JSON, 47,495 addresses**, taking 35 s to 2 min+ to download. The size likely explains the UI failure. It is a static file refreshed every few minutes.
    - Each row has `ethAddress`, `accountValue`, `displayName`, and `pnl` / `roi` / `vlm` for `day` / `week` / `month` / `allTime`.
    - About 14.5k addresses have ≥ $10k account value *and* positive month + all-time PnL. **Simple "is profitable" filters barely narrow the field; risk-adjusted scoring has to do the work.**
    - Plan: pull it at most every few hours, cache it, and only pull Info API history for the pre-filtered top few hundred.
  - ⚠️ The endpoint is **undocumented** and could change → keep cached snapshots; HyperTracker is the fallback.
  - **HyperCore vaults:** `GET https://stats-data.hyperliquid.xyz/Mainnet/vaults` (~14 MB, ~11 s; also undocumented) lists 9,476 vaults with APR, PnL, TVL, leader and `isClosed`.
    - Only **3,091 are open**, and just **234 / 82 / 26** have TVL ≥ $10k / $100k / $1M (checked 2026-10-06).
    - The serious vault universe is small enough to score in full every run.
    - The HL docs now call these vaults **legacy**: perps only, no spot/HIP-3, 10% leader profit share.
    - Depositor lockup is **1 day** (4 days for HLP).
  - **ERC-4626 vaults on HyperEVM** are what the HL team now recommends. Most of today's are **yield/lending** vaults (Felix on Morpho, Hyperbeat on Midas, Euler, HyperLend, …), not trader strategies.
    - They can be listed via the [DefiLlama yields API](https://yields.llama.fi/pools) (chain "Hyperliquid L1"), the [tradingstrategy.ai ERC-4626 list](https://web3-ethereum-defi.tradingstrategy.ai/tutorials/erc-4626-vault-list), or by scanning `Deposit` event logs.
    - DefiLlama shows **529 pools, 75 with ≥ $1M TVL** (2026-10-06).
      - Mostly lending: Morpho 20, HyperLend 7.
      - A handful are **strategy vaults** closer to "copy a trader": Liminal (basis), Harmonix, D2 Finance.
    - ❓ *Are ERC-4626 yield vaults in scope (e.g. as the "Conservative" bucket's yield leg), or do we only copy perps strategies?*
- Store raw + normalized snapshots so scoring is reproducible.

### 3.2 Scoring (algo first, AI second)
1. **Hard filters:** minimum history (e.g. ≥30 days), min trade count, min account value, max leverage, max drawdown cap.
2. **Baseline ranking:** the same style of metrics the HL leaderboard ranks by (account value, PnL, ROI, volume over 1D / 7D / 30D / all-time windows), so our rankings are familiar and easy to check against.
3. **Risk-adjusted metrics:** Sharpe, Sortino, Calmar, max drawdown, win rate, PnL consistency (not one lucky trade), exposure/leverage profile.
   - Compute these from `portfolio` **PnL history**, not raw account value, so deposits and withdrawals don't look like returns.
   - ⚠️ `portfolio` is coarse: about **~100 points per window**, checked 2026-10-06. That is roughly daily for `month` and **roughly weekly for `allTime`** (and sparser for `perpAllTime`).
     - 30-day daily Sharpe/Sortino is fine.
     - Longer horizons are weekly-resolution only, unless we rebuild the curve from `userFillsByTime` + `userFunding` (capped at the 10k most recent fills).
4. **LLM reviewer (Claude):** reads the stats and trade patterns of the **top ~20 finalists** (to fit CRE's 25 KB consensus limit). It flags red flags (martingale, wash-like behavior, single-asset concentration) and writes a plain-English rationale for each pick.

**Backtest caveat:** the leaderboard is only a *current* snapshot. Any address we backtest is by definition a survivor, so backtests will look too good. Mitigations:
- Rank on an earlier window and evaluate on a later one (e.g. rank on days −60…−30, evaluate on −30…0; weekly points from `allTime`).
- Start saving our own snapshots now, which gives an honest forward test by demo time.

❓ *Exact weights/thresholds — to be tuned on the split above.*

### 3.3 Portfolio curation (AI agent)
- Agent assembles **risk buckets** from the scored shortlist, e.g. **Conservative / Balanced / Aggressive**, each with target allocations.
- **MVP — static buckets:** agent publishes the buckets with rationale; the user picks one. We trade one bucket live.
- **Stretch — conversational onboarding:** agent asks the user a few questions (risk tolerance, horizon, drawdown comfort) and places them in a bucket.

### 3.4 Execution (Hyperliquid mainnet, small capital)
- Copy targets: **vaults (deposit) and/or individual traders (mirror positions, scaled)**. ❓ *Vaults, traders, or both?*
  - **Vaults:** small universe (~80 with ≥ $100k TVL), one deposit per target, but a 1-day lockup.
  - **Traders:** ~47k addresses and more novel, but scaled-down mirroring hits the $10 minimum.
- Use a Hyperliquid **API/agent wallet** to limit blast radius. It signs orders and other L1 actions, but withdrawals and transfers need the master wallet.
  - ❓ *Can the agent wallet sign `vaultTransfer` (vault deposits)? It is an L1 action, so probably yes, but this is undocumented → spike.*
- **Risk controls:** per-target allocation cap, max portfolio leverage, max slippage, global kill switch, executor dedupe by report ID.
- **Starting capital: 5 HYPE** (≈ $470 at the HYPE mid of ~$94 on 2026-10-06).
  - **Collateral:** standard perps margin is USDC. Portfolio margin, which would let HYPE back positions, needs > $10k account value, so it is out of reach. **Recommended:** sell HYPE → USDC on spot pair `@107` (HYPE/USDC) at start. ❓ *Confirm, or keep HYPE exposure on purpose?*
  - **Minimums:** the HL minimum order is **$10**, so ≈ $470 supports roughly **5–8 targets** with room to rebalance.
    - Mirroring a trader means scaling their positions down to our allocation, and legs under $10 get skipped. So mirror only each trader's **largest positions**.
    - Vault deposit minimum not yet verified ❓.
  - ❓ *Who funds the wallet?*

### 3.5 Orchestration (Chainlink CRE)
CRE runs the **10-minute decision loop**. Heavy data work happens off-CRE, because of CRE's quotas.

**Decision: CRE decides, the executor signs.** CRE produces a DON-signed rebalance report. A small executor service checks the signatures, applies risk limits, and signs/sends the Hyperliquid orders with the API wallet key.
- *Why not sign inside CRE?* A deployed workflow decrypts secrets into every node's memory. Single-send POSTs (`cacheSettings`) are only best effort, so duplicate orders are possible. Chainlink's own portfolio-rebalancing template uses the same decide/execute split.

**Loop (cron `0 */10 * * * *`):**
1. Fetch the compact top-N candidate file from our backend, using HTTP GET with identical-result consensus.
2. Call Claude with structured JSON output and field-level consensus. Free-text output would fail consensus.
3. Compute target weights and the drift vs. current positions. Trade only if drift is above a threshold.
4. `runtime.report()` → POST to the executor, **and** write the decision hash to a consumer contract on **HyperEVM**. HyperEVM is a supported CRE chain, and this gives an onchain, auditable decision log.

**CRE limits that shape the design** ([quotas](https://docs.chain.link/cre/service-quotas)):

| Limit | Value |
|---|---|
| Run time | 5 min |
| HTTP calls | 15 per run |
| Response size | **250 KB** |
| Data into consensus | 25 KB |
| Memory | 100 MB |

The full HL leaderboard is ~40 MB, so ingestion and scoring **must** run in our backend. CRE only consumes a pre-scored shortlist.

**Runtime notes:** workflows are TypeScript (`@chainlink/cre-sdk`) running on QuickJS/WASM, so there is no `node:crypto` (use Noble/viem). Develop with `cre workflow simulate`. Deploying to a live DON needs **Early Access approval** (`cre account access`).
- ❓ *Request Early Access now. If it doesn't arrive in time, demo the workflow in simulation mode with `--broadcast` and run the same loop live from the backend.*

### 3.6 Dashboard
- Leaderboard of scored traders/vaults with metrics + LLM rationale.
- Risk buckets and their composition.
- Live portfolio: positions, PnL vs. benchmark (e.g. hold BTC / HLP), rebalance log.
- ❓ *Frontend stack (Next.js suggested).*

### 3.7 Module contracts
Each module reads the previous module's output from the DB, so modules can be built and tested independently against fixture data.

| Module | Input | Output (stored) |
|--------|-------|-----------------|
| `ingest` | HL APIs, vault lists | `Snapshot`: address, kind (trader / HyperCore vault / ERC-4626 vault), account value, PnL/ROI/volume per window, equity curve, fills |
| `score` | Snapshots | `Candidate`: passed filters?, metrics, composite score |
| `review` | Top-N candidates | `Review`: keep/drop, red flags, rationale (Claude, structured JSON) |
| `curate` | Kept candidates | `Bucket`: risk level, target weights per address |
| `execute` | Selected bucket + current positions | `RebalanceLog`: intended vs. filled orders, fees, errors |
| `dashboard` | All of the above | — |

## 4. Stack
- **Language:** TypeScript everywhere.
- **Hyperliquid:** [`@nktkas/hyperliquid`](https://github.com/nktkas/hyperliquid). There is no official TS SDK; this one is listed in the HL docs and actively maintained. *(Inside CRE, use raw HTTP; the SDK is for the backend/executor.)*
- **Orchestration:** Chainlink Runtime Environment, `@chainlink/cre-sdk` (TS on QuickJS/WASM), `cre` CLI.
- **Onchain log:** small consumer contract on HyperEVM that receives CRE reports.
- **AI:** Claude API (Anthropic TS SDK) for the reviewer and the bucket curator.
- **Storage:** ❓ *Leaning **Supabase (hosted Postgres)**: the backend, executor and dashboard all need the same DB from different machines, and its realtime feed makes live PnL easy. SQLite only if everything runs on one box.*
- **Coinbase AgentKit:** ❓ *Open, leaning skip.* It has **no Hyperliquid action provider**; its TS providers as of 2026-10-06 include `vaultsfyi`, `morpho`, `defillama`, `across`, `erc20`, `x402`, …
  - It's only worth it if we include ERC-4626 vaults: `vaultsfyi` / `defillama` could feed vault discovery to the curator agent.
  - ❓ *Does vaults.fyi cover HyperEVM?*

## 5. Working rules
- Small, independently tested modules; integrate only tested pieces.
- Budget **~1h testing/bugfix per 2h of feature work**.
- Fallback for every live dependency (cached data for the demo if an API flakes).

## 6. Milestones
0. **First thing:**
   - request CRE Early Access (`cre account access`), since approval takes time
   - fund the wallet and swap HYPE → USDC
   - cache a leaderboard snapshot
1. **Spikes:**
   - `portfolio` history pull for 1 address and 1 vault
   - CRE cron + HTTP GET in `cre workflow simulate`
   - $10 HL order from the API wallet
   - `vaultTransfer` from the API wallet
   - CRE report → executor signature check
2. **MVP pipeline:** ingest → quant score → ranked list in DB.
3. **AI layer:** LLM reviewer + bucket curator.
4. **Live:** execution on mainnet with risk caps; dashboard shows live PnL.
5. **Stretch:** conversational bucket onboarding, position mirroring, onchain score attestation.

❓ *Map milestones to actual hackathon hours once the schedule is confirmed.*

## 7. Risks
- **Survivorship / overfitting:** past winners ≠ future winners → use out-of-sample backtest window.
- **Copy latency & slippage** when mirroring traders.
- **Key security** on mainnet → agent wallet, small capital, kill switch.
- **CRE Early Access may not arrive in time** → simulation mode for the demo; the same loop runs live from the backend.
- **CRE quotas** (250 KB responses, 15 HTTP calls, 25 KB consensus) → keep the shortlist CRE sees small (top ~20).
- **Duplicate/replayed orders** → the executor dedupes by report ID; HL also rejects reused nonces.
- **API rate limits** on HL Info API → fetch history only for the pre-filtered shortlist; cache aggressively.
- **Data source availability:** the leaderboard file is undocumented and slow (~40 MB) → cached snapshots + HyperTracker fallback.
- **Tiny capital:** ≈ $470 against the $10 minimum order limits us to ~5–8 targets, and fees are a large share of PnL → trade only past a drift threshold.
- **Legacy-vault dependence:** HyperCore vaults are marked legacy; the vaults endpoint is undocumented.

## 8. Decisions & open questions (summary)

**Decided**
- [x] Network: HL mainnet, starting capital 5 HYPE
- [x] Orchestration: CRE runs the 10-minute decision loop; ingestion + scoring run in the backend (CRE quotas)
- [x] Signing: CRE emits a signed report → executor verifies and signs HL orders; decision hash logged on HyperEVM
- [x] Leaderboard data: `stats-data.hyperliquid.xyz/Mainnet/leaderboard` works (~40 MB, ~47.5k addresses)
- [x] AI: quant screen first → Claude reviewer + bucket curator
- [x] Curation UX: static buckets for MVP, conversational onboarding as stretch
- [x] Sourcing: leaderboard + vaults, HL-leaderboard-style metrics, HyperTracker as fallback
- [x] HL SDK: `@nktkas/hyperliquid` (backend/executor); raw HTTP inside CRE

**Open**
- [ ] ❓ Project name
- [ ] ❓ CRE Early Access granted in time? (else simulation demo)
- [ ] ❓ HyperTracker API access/terms (fallback data; paid, free tier 100 tokens/day)
- [ ] ❓ ERC-4626 yield vaults in scope? (they list via DefiLlama; most are lending/yield, not trading)
- [ ] ❓ Can the API wallet sign `vaultTransfer`? Vault deposit minimum?
- [ ] ❓ Copy vaults, traders, or both
- [ ] ❓ Confirm HYPE → USDC swap at start (recommended); who funds the wallet
- [ ] ❓ Storage (leaning Supabase)
- [ ] ❓ AgentKit in or out (no HL support; only useful for ERC-4626 vault discovery; does vaults.fyi cover HyperEVM?)
- [ ] ❓ Frontend stack
- [ ] ❓ Scoring weights/thresholds
- [ ] ❓ Milestone → hackathon-hours mapping

## 9. References
- HL API: [info endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint) · [rate limits](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits) · [nonces & API wallets](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/nonces-and-api-wallets) · [vaults](https://hyperliquid.gitbook.io/hyperliquid-docs/hypercore/vaults) · [portfolio margin](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/portfolio-margin)
- CRE: [service quotas](https://docs.chain.link/cre/service-quotas) · [HTTP client (TS)](https://docs.chain.link/cre/reference/sdk/http-client-ts) · [non-determinism](https://docs.chain.link/cre/concepts/non-determinism-ts) · [supported networks](https://docs.chain.link/cre/supported-networks-ts) · [deploy access](https://docs.chain.link/cre/account/deploy-access) · [portfolio-rebalancing template](https://docs.chain.link/cre-templates/automated-portfolio-rebalancing)
- Data: [HL leaderboard](https://app.hyperliquid.xyz/leaderboard) · [hyperliquidvaults.com](https://hyperliquidvaults.com) · [HyperTracker](https://hypertracker.io) · [DefiLlama yields](https://yields.llama.fi/pools)
- SDKs: [`@nktkas/hyperliquid`](https://github.com/nktkas/hyperliquid) · [Coinbase AgentKit](https://github.com/coinbase/agentkit)
