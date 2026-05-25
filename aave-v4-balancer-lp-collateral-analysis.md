# Onboarding Balancer v3 LP Tokens (BPTs) as Collateral on Aave v4

### A deep analysis and a TEMP CHECK–ready proposal for a dedicated, collateral‑only Balancer Spoke

**Author:** Balancer (marcus@balancerlabs.dev)
**Date:** May 2026
**Status:** Draft for internal review → TEMP CHECK
**Target venue:** Aave v4, Ethereum mainnet

---

> **How to read this document.** Sections 1–7 are written to become a public Aave governance post (TEMP CHECK, then ARFC). Sections 8–10 and the appendices are **INTERNAL** Balancer strategy — they contain candid governance politics, a self‑critique log, and verification to‑dos that should be trimmed before anything is posted to `governance.aave.com`. Every load‑bearing number is tagged with a source and, where it could not be fetched live in this environment, a **[VERIFY]** flag.

---

## 0. Honest preface — why this is a hard sell in May 2026, and why it is still worth doing

Any serious reader on the Aave side will open with three objections, so we open with them too:

1. **Balancer was exploited for ~$128M on 3 November 2025.** The bug was a rounding/precision inconsistency in the **v2 Composable Stable Pool** invariant. **Balancer v3 pools were not affected.** ([Check Point](https://research.checkpoint.com/2025/how-an-attacker-drained-128m-from-balancer-through-rounding-error-exploitation/), [OpenZeppelin](https://www.openzeppelin.com/news/understanding-the-balancer-v2-exploit), [Certora](https://www.certora.com/blog/breaking-down-the-balancer-hack))
2. **Balancer Labs (the corporate entity) announced a wind‑down in March 2026**, BAL emissions ended and veBAL was dismantled; the protocol continues under a leaner DAO/"OpCo." ([CoinDesk, 2026‑03‑24](https://www.coindesk.com/tech/2026/03/24/balancer-labs-will-shut-down-as-corporate-entity-became-a-liability-after-usd110-million-exploit), [The Defiant](https://thedefiant.io/news/defi/balancer-proposes-labs-shutdown-bal-tokenomics-shift))
3. **Aave has tried LP‑token collateral before and walked away.** The Aave v2 "AMM Market" (2021) accepted Uniswap‑ and Balancer‑pool LP tokens, saw low usage, and was deprecated (LTs set to 0, assets frozen). ([Aave AMM Market](https://medium.com/aave/aave-amm-market-released-73ae76a7cbc0), [deprecation ARFC](https://governance.aave.com/t/arfc-deprecate-aave-v2-amm-market/12830))

This proposal does **not** ask Aave to re‑accept the thing that broke. It asks Aave to accept a **narrow, fairly‑priced, quarantined subset of Balancer v3 LP tokens** — specifically the correlated‑LST, stable/pegged, and **Aave‑wrapped "100% Boosted"** pools — inside a **dedicated v4 Spoke** whose blast radius is capped by the Liquidity Hub. The single strongest argument: **a Balancer v3 Boosted‑pool BPT is already, under the hood, a basket of Aave aTokens.** Accepting it as collateral is closer to re‑collateralizing Aave's own deposits than to taking on an exotic outside asset.

The honest bottom line, stated once, up front: **on today's post‑exploit TVL the dollars are small** (low‑single‑digit‑millions of incremental Aave revenue in the base case). The case is **strategic and forward‑leaning** — it is the right structure to have in place as Balancer v3 TVL recovers, it deepens the exact GHO/LST liquidity Aave already cares about, and it is a low‑footprint way for Aave v4 to demonstrate its headline feature (isolated, risk‑priced collateral) on a real, friendly counterparty.

---

## 1. Summary (TL;DR)

- **Ask:** Establish a dedicated **"Balancer Collateral Spoke"** on Aave v4 (Ethereum) that accepts a whitelist of **Balancer v3** BPTs **as collateral only** (the BPT itself is non‑borrowable; users borrow Hub assets / **GHO** against it).
- **Scope (v3 only):** (A) LST/LRT correlated pools, (B) stable/pegged pools, (C) **100% Boosted / Aave‑wrapped** pools. **Explicitly excludes all v2 pools** and all volatile/uncorrelated pools.
- **Risk containment:** dedicated Spoke with a **capped Liquidity‑Hub credit line**, conservative caps sized to live pool depth, a high **Collateral Risk score** (v4 Risk Premium) so borrowers pay for the risk rather than socializing it, isolation‑style configuration, and a **pause/freeze** integration.
- **Oracle (the crux):** **invariant‑based fair pricing** (never spot reserves) × **Chainlink / exchange‑rate** feeds, with **CAPO** growth bounds on rate providers, a **read‑only‑reentrancy guard** (revert if the Balancer Vault is unlocked/in‑context), and `getActualSupply()`.
- **Why Aave benefits:** organic **GHO** borrow demand; deeper, stickier LST/stable liquidity (collateral keeps earning swap fees); a flagship demonstration of v4 risk isolation; tighter Balancer↔Aave↔GHO alignment.
- **Why Balancer benefits:** a new utility sink for v3 BPTs (collateral that keeps LPing), sticky TVL, and a concrete reason to route v3 liquidity through Aave‑integrated pools.
- **Rollout:** staged — TEMP CHECK → ARFC with a risk provider → **pilot Spoke with deliberately tiny caps on the 2–3 safest pools** → scale caps with TVL recovery and observed performance.

---

## 2. Strategic rationale — why this pairing, and why now

### 2.1 Aave is already the home of correlated‑ETH leverage

Aave is *the* venue where users lever correlated‑ETH assets. As of mid‑May 2026:
- **weETH deposits ≈ $4.47B** on Aave Ethereum (2nd‑largest asset); weETH holders drive ~57.9% of all WETH borrowing (~$3.27B debt). ([ChainCatcher, Feb 2026](https://www.chaincatcher.com/en/article/2248017))
- **wstETH** carries a supply cap of **810,000 wstETH (≈ $3B+)**; Aave holds roughly two‑thirds of all wstETH used as DeFi collateral. ([Aave gov supply caps, 2026‑05‑18](https://governance.aave.com/t/risk-stewards-supply-cap-changes-on-aave-v3-2026-05-18/24939), [Lido×Aave](https://aave.com/blog/lido-aave-case-study))
- Combined, LSTs/LRTs back ~97.6% of WETH debt on Aave. **[VERIFY]**

A Balancer **wstETH/WETH** or **rETH/wstETH** v3 pool is the *same* underlying risk profile Aave already concentrates in at multi‑billion scale — **plus** the BPT earns swap fees. The marginal risk Aave takes on by accepting the correlated‑LST BPT (with a sound oracle) is small relative to what it already holds; the marginal yield to the user is real.

### 2.2 The Boosted‑pool synergy is the headline

Balancer **v3 "100% Boosted Pools"** route ~100% of idle pool liquidity into **Aave v3** as wrapped aTokens (e.g. `waUSDC`, `statATokens`), via Vault‑level **ERC‑4626 buffers**; LPs earn swap fees **on top of** Aave lending yield. ([Beets/Balancer "100% Boosted"](https://medium.com/balancer-protocol/100-boosted-pools-powered-by-aave-53fe5b920833), [The Block launch](https://www.theblock.co/post/330379/balancer-v3-launches-aave), [buffer docs](https://docs.balancer.fi/concepts/vault/buffer.html))

Implication: **the collateral Aave would accept for a Boosted‑pool BPT is, by construction, mostly Aave aTokens.** Aave is being asked to lend against a tokenized claim on its own deposits plus a thin swap‑fee wrapper. This is the lowest‑risk, highest‑synergy slice and should be the pilot's first asset. (It also introduces a **reflexivity** caveat — see §6.4.)

### 2.3 GHO demand

GHO is Aave v4's **native settlement asset** (~$584M supply, mid‑May 2026). ([DefiLlama/Aave](https://defillama.com/protocol/aave)) Borrowing against BPT collateral can be routed into GHO mint demand. A GHO/USDC **Boosted** pool already exists on Base ([Balancer Report](https://medium.com/balancer-protocol/the-balancer-report-e3937a2f735f)), which makes the loop self‑reinforcing: BPT collateral → GHO borrow → GHO liquidity that can itself sit in a Boosted pool. Across the opportunity scenarios (§5) this is **+$6M to +$70M** of incremental GHO demand (~1–12% of current supply).

### 2.4 Aave v4 was built for exactly this

Aave v4 went live on Ethereum mainnet **30 March 2026** with a **Hub‑and‑Spoke** design: a single immutable **Liquidity Hub** holds assets and enforces solvency; upgradeable **Spokes** are user‑facing modules with their own collateral rules, caps, oracle config, and a Hub‑granted **credit line**. ([Aave: Understanding v4 Architecture](https://aave.com/blog/understanding-aave-v4s-architecture), [v4 GitHub overview](https://github.com/aave/aave-v4/blob/main/docs/overview.md), [The Block, 2026‑03‑30](https://www.theblock.co/post/395617/aave-v4-launches-ethereum-mainnet)) Two features make a BPT listing far safer than it would have been on v3:
- **Risk isolation by Spoke.** A dedicated Spoke's exposure to the Hub is bounded by its credit line; a Balancer failure cannot drain the shared Hub or contaminate blue‑chip markets. ([Aave: v4 Risk Isolation](https://aave.com/blog/aave-v4-risk-isolation))
- **Risk Premiums.** Each asset carries a **Collateral Risk score (0–1000%)**; a borrower pays a premium on top of the base rate proportional to their collateral's risk — so BPT risk is **priced to the borrower, not socialized** across the market. ([Aave: v4 Risk Premiums](https://aave.com/blog/aave-v4-risk-premiums))

Precedent that this is how v4 is actually being used: a **TEMP CHECK to establish a dedicated "Babylon Spoke" for native BTC** ([gov thread 24911](https://governance.aave.com/t/temp-check-establish-babylon-spoke-and-onboard-babylon-native-btc/24911)) and live **Pendle PT** onboarding to v4 ([PT‑USDG ARFC](https://governance.aave.com/t/arfc-onboard-pt-usdg-24sep2026-to-aave-v4-on-ethereum/24942)). A Balancer collateral‑only Spoke is squarely within this pattern.

---

## 3. Proposal specification

### 3.1 Structure: a dedicated, collateral‑only Balancer Spoke

- A purpose‑built **Spoke** that lists a **whitelist of Balancer v3 BPTs as collateral only** (`borrowable = false` for every BPT reserve). Users supply a BPT and borrow **Hub assets and/or GHO** against it.
- The Spoke draws a **capped credit line** from the Liquidity Hub. This is the hard ceiling on systemic exposure: even a total loss inside the Spoke cannot exceed its credit line. Start the line **small** and raise it by governance as the book proves out.
- This is explicitly **"not the broad Spoke we have discussed"** — it is a constrained, single‑purpose collateral venue, not a general Balancer money market.

### 3.2 Eligible collateral (v3 only) — three tranches

| Tranche | Examples | Why eligible | Initial treatment |
|---|---|---|---|
| **C — Boosted / Aave‑wrapped** | Aave GHO/USDC, aUSDC/aUSDT boosted stable pools | BPT ≈ basket of Aave aTokens; lowest marginal risk | **Pilot first**, highest LTV within reason |
| **A — LST/LRT correlated** | wstETH/WETH, rETH/wstETH (established LSTs only) | Same risk profile as Aave's existing multi‑$B LST book; minimal IL | Pilot second; exchange‑rate oracle |
| **B — Stable / pegged** | Blue‑chip stable pools, GHO pairs | Low volatility, deep proportional exits | Pilot third; StableSurge‑hook pools preferred |

**Hard exclusions:** all **v2** pools (the exploited surface); algorithmic or newly‑launched LRTs without a mature redemption mechanism; any pool with a mutable/un‑renounced rate provider or admin key (see §6.7); volatile/uncorrelated 80/20‑style pools (out of scope by audience decision).

### 3.3 Oracle design — the make‑or‑break (specified, not hand‑waved)

A BPT is only as safe as its price feed. The proposed oracle for each listed BPT MUST:

1. **Price from the invariant, never from spot reserves.** Compute BPT value from the pool invariant and **independent external prices** (Chainlink market feeds and/or rate‑provider exchange rates), not from manipulable in‑pool balances. For stable pools, `BPT ≈ D / actualSupply` (D = invariant); for weighted pools, the geometric‑mean fair‑value formula. ([Balancer: valuing BPT](https://docs-v2.balancer.fi/concepts/advanced/valuing-bpt/valuing-bpt.html), [BPT oracle example](https://docs.balancer.fi/concepts/core-concepts/bpt-oracles/bpt-oracles-example.html))
2. **Use `getActualSupply()`**, never raw `totalSupply()`.
3. **Guard against read‑only reentrancy.** Revert if the Balancer Vault is mid‑operation: on v2, call `VaultReentrancyLib.ensureNotInVaultContext`; on **v3**, exploit the structural fix (transient accounting / Vault‑as‑ERC20 makes balances+supply atomic) **and** set an immutable flag that reverts all price/TVL reads while the Vault is `unlock()`‑ed (the Gyroscope pattern). ([Balancer reentrancy notice](https://forum.balancer.fi/t/reentrancy-vulnerability-scope-expanded/4345), [Gyro E‑CLP oracle](https://docs.balancer.fi/concepts/explore-available-balancer-pools/gyroscope-pool/gyro-eclp.html))
4. **Bound rate‑provider growth with CAPO.** Wrap each yield‑bearing constituent's rate in Aave's **Correlated‑Asset Price Oracle**, capping `maxYearlyRatioGrowthPercent` and enforcing freshness so an inflated/stale rate cannot over‑value the BPT. ([CAPO framework](https://governance.aave.com/t/chaos-labs-correlated-asset-price-oracle-framework/16605), [bgd‑labs/aave‑capo](https://github.com/bgd-labs/aave-capo)) **Calibrate freshness carefully** — a stale CAPO reference caused a **$27M** mis‑liquidation on Aave on **10 March 2026** (wstETH valued at 1.19 vs 1.23 ETH). ([CoinDesk](https://www.coindesk.com/business/2026/03/10/defi-lending-platform-aave-sees-a-rare-usd27-million-liquidations-after-a-price-glitch))
5. **Value collateral at a stressed (depeg) basket composition** for LT‑setting, not at par.

Reference implementations to lean on: **Gyroscope's `EclpLPOracle`** (explicitly built to make Balancer BPTs usable as lending collateral, and the origin of the unlocked‑Vault revert flag), Balancer v3's **Geomean Oracle** hook, and Chainlink's staked‑ETH feeds (wstETH/rETH/cbETH). ([Gyro E‑CLP](https://docs.balancer.fi/concepts/explore-available-balancer-pools/gyroscope-pool/gyro-eclp.html), [Geomean oracle write‑up](https://www.beirao.xyz/blog/ENG5-BalancerV3_TWAP_Oracle))

### 3.4 Initial risk parameters (illustrative — final values set by a risk provider at ARFC)

These are **placeholders to anchor discussion**, deliberately conservative. They are *not* a substitute for a LlamaRisk/Chaos‑style parameterization (§8).

| Parameter | Tranche C (Boosted) | Tranche A (LST) | Tranche B (Stable) |
|---|---|---|---|
| LTV | 70–80% | 70–80% | 70–80% |
| Liquidation Threshold | +5–8pp over LTV | +5–8pp | +5–8pp |
| Liquidation Bonus | sized to proportional‑exit slippage (≈3–7%) | ≈3–7% | ≈3–7% |
| Supply cap | small % of **live** pool TVL | small % of live pool TVL | small % of live pool TVL |
| Collateral Risk (v4 premium) | moderate | moderate–high | moderate |
| Borrowable | **No (collateral only)** | No | No |
| Mode | dedicated Spoke / isolation | dedicated Spoke / e‑mode‑style | dedicated Spoke |

### 3.5 Liquidation path

Liquidators seize the BPT and unwind via **`removeLiquidity` (PROPORTIONAL)** — burn BPT, receive all underlying pro‑rata, **zero price impact, no swap fee** — then sell the basket (or, for Boosted pools, **redeem the underlying Aave aTokens directly** — natural for Aave). Single‑asset exits are avoided in liquidation due to slippage. Supply caps must be sized so a full liquidation is a small fraction of pool TVL, and the liquidation bonus must cover realistic proportional‑exit + sell slippage. ([add/remove liquidity types](https://docs.balancer.fi/concepts/vault/add-remove-liquidity-types.html)) A working liquidator unwind path must be demonstrated on testnet before caps are raised.

### 3.6 Staged rollout

1. **TEMP CHECK** — sentiment, scope agreement.
2. **ARFC** — risk provider attaches parameters; oracle implementation audited.
3. **Pilot AIP** — launch the Spoke with **one** Boosted pool (Tranche C), tiny caps, low credit line.
4. **Observe** — liquidation drills, oracle behavior, utilization, for a defined period.
5. **Scale** — add Tranche A/B pools and raise caps via Risk‑Steward‑style adjustments, gated on Balancer v3 TVL recovery and clean operation.

---

## 4. How Fluid does it — the benchmark, and why our design differs

The user asked specifically to study **Fluid (Instadapp)** as the reference for LP‑as‑collateral, while noting our proposal is deliberately *not* the same.

**What Fluid does.** Fluid is a **vertically integrated** DEX + lending system on a shared Liquidity Layer. Its **"Smart Collateral"** lets a deposited two‑token position simultaneously (a) back a loan, (b) earn lending APR, and (c) act as AMM liquidity earning swap fees; **"Smart Debt"** lets the *borrowed* side also be productive liquidity. It uses a **tick/range batch‑liquidation** engine (Uniswap‑v3‑style ticks; partial liquidation of a whole at‑risk range in one tx; ~0.1% penalty, ~5% liquidated/event), which is what lets it run **~95% LTV** on correlated pairs. TVL ≈ **$1.6–1.9B** (Q2 2026). ([cyber.fund](https://cyber.fund/content/fluid-1), [Mirador on liquidations](https://www.mirador.finance/p/fluid-liquidation-mechanism-a-closer), [Messari](https://messari.io/report/understanding-fluid-a-comprehensive-overview)) **[VERIFY TVL]**

**What transfers to our proposal:**
- **Oracle discipline:** Fluid prices conservatively and asymmetrically (lowest‑recent price for borrow/withdraw; current/contract price for liquidation) with rate limits — a portable pattern for a BPT oracle.
- **Liquidation engineering:** batching/range liquidation of correlated positions reduces gas, MEV, and penalty — relevant input to how Aave sizes the BPT liquidation bonus.
- **The Resolv post‑mortem (Mar 2025):** Fluid took **~$21M bad debt** when a composite collateral (wstUSR) was bought cheap during a depeg and posted **at an inflated oracle price** to borrow against. The loss was an **oracle/collateral‑quality failure, not a liquidation‑engine failure** — exactly the risk a BPT oracle must defend against. ([Blockonomi](https://blockonomi.com/fluid-covers-21m-bad-debt-after-resolv-exploit-outlines-recovery-plan/))

**What does NOT transfer (and why the comparison has limits):**
- Fluid **owns its DEX**; its collateral *is* its own internal liquidity, so it controls minting, reserves, rebalancing, and the oracle center‑price. **Aave would accept an *external* Balancer LP token it does not control** — no shared liquidity layer, no internal rebalancing, no ability to keep the position "productive as Aave's own AMM."
- **Smart Debt** is irrelevant (we accept LP **only as collateral**).
- Fluid's "39x effective liquidity" capital‑efficiency is an artifact of integration and does **not** carry over.

**Net:** borrow Fluid's *oracle conservatism* and *liquidation batching*, internalize the *Resolv lesson*, and discard the *vertical‑integration / Smart‑Debt* narrative — it presumes ownership of the venue producing the LP, which Aave will not have.

---

## 5. Market sizing and the opportunity model

> **Data caveat:** in this environment, live data hosts (Balancer GraphQL API, DefiLlama, balancer.fi) were not reachable, so figures below come from cited reporting and must be **re‑pulled live before publishing** — especially Balancer 24h/30d volume and per‑pool TVL (from `api-v3.balancer.fi` `poolGetPools`). All sizing in §5.2–5.4 is **ESTIMATE** with stated assumptions.

### 5.1 Current scale (mid‑May 2026)

| Metric | Value | Source |
|---|---|---|
| Balancer total TVL | ~$117M (v2 ~$41.75M + v3 ~$75.43M); ~$157M incl. CoW AMM/others **[VERIFY]** | [DefiLlama](https://defillama.com/protocol/balancer) |
| Balancer pre‑exploit TVL | ~$800M (Oct/Nov 2025) | [CoinDesk](https://www.coindesk.com/web3/2025/11/27/balancer-dao-starts-discussing-usd8m-recovery-plan-after-usd110m-exploit-cut-tvl-by-two-thirds) |
| Balancer 2021 peak | ~$3.5B | [CoinDesk](https://www.coindesk.com/tech/2026/03/24/balancer-labs-will-shut-down-as-corporate-entity-became-a-liability-after-usd110-million-exploit) |
| v3 share of TVL | ~64% (and rising) | derived from DefiLlama splits |
| Aave total TVL | ~$14.49B | [DefiLlama](https://defillama.com/protocol/aave) |
| Aave supplied / borrowed | ~$14.86B / ~$11.12B (~74.8% util.) | [DefiLlama](https://defillama.com/protocol/aave) |
| GHO supply | ~$584M | [DefiLlama](https://defillama.com/protocol/aave) |
| Aave protocol revenue (annualized) | ~$89–100M | [DefiLlama](https://defillama.com/protocol/aave-v3) |

### 5.2 Addressable BPT collateral (the safe‑ish subset)

| Category | Est. share of Balancer TVL | On ~$117M today | On ~$300M recovery |
|---|---|---|---|
| Stable/pegged | 35–45% | $41–53M | $105–135M |
| LST/LRT correlated | 20–30% | $23–35M | $60–90M |
| Boosted/Aave‑wrapped | 15–25% | $18–29M | $45–75M |
| **Addressable total (~70–90%)** | | **~$82–105M** | **~$210–270M** |

### 5.3 Uptake benchmark

When a deep, trusted venue accepts an LP token, a meaningful share of that pool's supply migrates in. Fluid (whose entire model is LP‑as‑collateral) scaled to ~$2B of collateral; Aave already captures ~two‑thirds of DeFi wstETH. A defensible planning range is **15–30% uptake** of the addressable BPT base over 12–18 months. ([Messari/Fluid](https://messari.io/report/understanding-fluid-a-comprehensive-overview))

### 5.4 Model (transparent; `Collateral = X·Y`, `Borrow = X·Y·Z`, `Aave rev ≈ Borrow × ~1–1.5%`)

**Forward / recovery base (Y = $250M addressable):**

| Scenario | Uptake X | LTV Z | Collateral | Borrow unlocked | Aave annual revenue |
|---|---|---|---|---|---|
| Low | 10% | 60% | $25M | $15M | $0.15–0.23M |
| Base | 20% | 70% | $50M | $35M | $0.35–0.53M |
| High | 35% | 80% | $87.5M | $70M | $0.70–1.05M |

**Today's base (Y = $100M addressable):** Borrow unlocked ranges **$6M–$28M**, Aave revenue **$0.06–0.42M**.

**GHO demand:** if borrowing routes to GHO, **+$6M to +$70M** of GHO mint demand (~1–12% of current supply).

**Balancer‑side uplift:** collateralized BPT keeps earning swap fees → **~$50K–$400K/yr** incremental, sticky fee revenue (ESTIMATE), and — more importantly — it deepens the LST/GHO pools Balancer wants to grow.

**Honest read:** the immediate revenue is modest. The justification is **strategic** (right structure for the recovery), **risk‑adjusted** (same risk Aave already holds, but priced and isolated), and **synergistic** (GHO + Aave‑wrapped collateral). The upside scales ~linearly with Balancer v3 TVL recovery.

---

## 6. Risk assessment (candid)

Severity is rated per tranche: **A** = LST/LRT, **B** = stable, **C** = Boosted.

| # | Risk | A | B | C | Primary mitigation |
|---|---|:--:|:--:|:--:|---|
| 1 | Oracle manipulation / mispricing | High | High | Med | Invariant fair pricing + Chainlink; reentrancy guard; CAPO bounds |
| 2 | Impermanent loss / divergence | Low | Low | Low | LTV buffer ≫ baseline IL (<0.4% at 20% divergence) |
| 3 | Depeg / correlation break | High | High | Med | Conservative LT vs stressed basket; isolation; circuit‑breaker; caps |
| 4 | Smart‑contract / dependency / contagion | Med | High(v2)/Med(v3) | Med | **v3‑only**; capped Spoke credit line; pause integration |
| 5 | Liquidation / unwind under stress | High | Med | Med | Proportional `exitPool`; caps ≪ pool depth; tested liquidator path |
| 6 | Liquidity / exit | Med | Med | Med | Dynamic supply caps tied to live pool TVL |
| 7 | Governance / param / admin keys | Med | Med | Med‑High | Whitelist only immutable/renounced rate providers; monitor & freeze |

### 6.1 Oracle manipulation — *the* risk
Naive spot‑reserve pricing is flash‑loan‑manipulable; the **2023 Balancer read‑only reentrancy** enabled real losses downstream (**Sentiment ~$1M, Apr 2023; Sturdy ~$800K, Jun 2023**). The mitigation set in §3.3 (invariant fair pricing, reentrancy guard, CAPO, `getActualSupply`) is mandatory and well‑precedented. ([CertiK/Sturdy](https://www.certik.com/resources/blog/oracle-dependency-decrypting-the-sturdy-finance-attack))

### 6.2 Impermanent loss — low
For correlated/stable baskets, IL is negligible (≈0.03% at 5% divergence; ≈0.4% at 20%). A 5–8pp LTV→LT buffer absorbs steady‑state IL comfortably.

### 6.3 Depeg — the scary tail
A StableSwap‑style pool absorbs a depegging constituent disproportionately, so a BPT can fall **faster than a naive average** exactly when exit liquidity is worst. Precedents: **stETH −6–7% (Jun 2022)**, **USDC −12–13% (11 Mar 2023)**, **ezETH −79% in <1hr (24 Apr 2024)** → **$56–65M liquidations** across Gearbox/Morpho. ([DLNews ezETH](https://www.dlnews.com/articles/defi/renzos-ezeth-loses-ether-peg-drops-79-in-under-one-hour/)) Mitigation: exchange‑rate (redemption‑rate) oracles for LSTs (ignore transient market dislocations), LT set against a stressed basket, isolation, freeze capability, and **exclusion of immature/algorithmic LRTs**. Residual: an exchange‑rate oracle over‑values a *fundamentally* broken asset — bounded by caps, not eliminable.

### 6.4 Contagion & the Boosted‑pool reflexivity caveat
The **Nov 2025 $128M** exploit hit **v2 Composable Stable Pools only** — listing **v3** sidesteps that exact code path, the single most important scoping decision in this proposal. Remaining: Aave inherits Balancer **v3 Vault** smart‑contract risk (bounded by the Spoke credit line). **Reflexivity:** a Boosted‑pool BPT holds Aave aTokens; using it as Aave collateral creates a loop where Aave‑side stress (utilization spike, aToken liquidity crunch) feeds back into the very collateral meant to backstop it. Mitigation: avoid double‑counting circular aToken exposure in risk scoring, monitor buffer health, and cap Tranche C conservatively despite its low headline risk.

### 6.5 Liquidation under stress
**Cautionary tale: CRV‑on‑Aave (Nov 2022)** — an engineered position left Aave with **~$1.6M bad debt** and permanently hardened Aave toward illiquid/manipulable collateral. ([The Defiant](https://thedefiant.io/news/defi/crv-trade-aave-bad-debt)) For BPTs: proportional exit is clean in normal conditions but returns a worst‑composition basket if the pool is itself imbalanced/under attack. Size caps so a full liquidation is a small fraction of pool TVL.

### 6.6 Liquidity/exit
Large BPT collateral can't be unwound without moving the price of the collateral itself. Bind supply caps to a small % of **live** pool TVL, ratcheting down if TVL shrinks.

### 6.7 Governance/admin‑key
Balancer governance / pool admins can change swap fees, the amplification factor, **pause** a pool, or **upgrade a rate provider** (≈ oracle compromise). Whitelist only pools with immutable/renounced or time‑locked, audited control, and freeze the Aave reserve on any privileged change.

### 6.8 Residual risk (stated plainly)
Even a perfectly scoped listing inherits: (a) Balancer v3 Vault contract risk, (b) the fundamental‑depeg tail where an exchange‑rate oracle over‑values a broken asset, and (c) **Aave's own oracle‑config risk** (cf. the Mar‑2026 CAPO $27M event). None is fully eliminable — only **bounded** by v3‑only scope, the capped Spoke, conservative caps, and risk‑priced premiums.

---

## 7. Anticipated objections & responses (forum‑facing)

| Objection | Response |
|---|---|
| "Balancer just got hacked for $128M." | The exploit was **v2 Composable Stable Pools**; **v3 was untouched**. This proposal is **v3‑only** and excludes the exact code path that broke. |
| "Balancer Labs is winding down — who maintains the contracts?" | Address head‑on (§8.3): v3 contracts are immutable/audited; the DAO/OpCo and ecosystem continue maintenance; the Spoke's exposure is capped regardless. *We need a crisp, sourced maintenance answer before TEMP CHECK.* |
| "Aave already deprecated LP collateral (v2 AMM market)." | That was v1‑era shared‑pool risk with naive oracles and no isolation. v4's Spoke isolation + Risk Premiums + a modern fair‑value oracle are a different regime; and Boosted BPTs (≈ Aave aTokens) didn't exist then. |
| "LP oracles are dangerous." | Correct historically — which is why §3.3 mandates invariant fair pricing, reentrancy guards, CAPO, and the Gyroscope unlocked‑Vault pattern, with an audited implementation as a gating condition. |
| "Liquidations will fail in a crisis." | Proportional exits are clean; caps are sized to pool depth; a liquidator drill is a gating condition before caps rise. |
| "The numbers are too small to bother." | True today; the ask is a **low‑footprint pilot** that scales with recovery and demonstrates v4's flagship isolation feature on a friendly counterparty, while growing GHO. |

---

## 8. INTERNAL — governance reality check (trim before posting)

### 8.1 The political map has gotten worse for us
- **ACI / Marc Zeller** — historically the governance kingmaker **and Balancer‑friendly** (authored the original AMM market) — **announced an exit in March 2026.** Losing ACI removes our most natural sponsor. ([ACI is leaving Aave](https://governance.aave.com/t/aci-is-leaving-aave/24205))
- **Chaos Labs** (risk) and **BGD Labs** (core dev) **both exited (~April 2026).** ([Chaos Labs leaving](https://governance.aave.com/t/chaos-labs-is-leaving-aave/24386))
- **LlamaRisk** is now the primary risk provider and the gate we must convince. ([LlamaRisk continuity](https://governance.aave.com/t/llamarisk-ensuring-continuity-of-aaves-risk-management/24397))
- **Aave Labs / Stani Kulechov** consolidated control via the **"Aave Will Win"** vote (finalized ~Apr 2026). **Aave Labs' buy‑in is now effectively the gate.** ([CoinDesk](https://www.coindesk.com/tech/2026/04/13/aave-passes-landmark-vote-ending-months-long-fight-over-who-controls-protocol-revenue))

**Implication:** secure informal Aave Labs + LlamaRisk alignment *before* a public TEMP CHECK. Lead with GHO and the v4‑isolation showcase (things Aave Labs wants), not with "help Balancer."

### 8.2 Our best levers
- The **live v3↔Aave↔GHO Boosted‑pool partnership** is real, mutually beneficial, and survived the exploit unscathed — lead with it. ([Balancer v3 × Aave](https://www.theblock.co/post/330379/balancer-v3-launches-aave))
- **Marcus is already engaged** in the v4 launch roadmap thread as "Marcus‑Balancer." ([v4 roadmap thread](https://governance.aave.com/t/aave-v4-launch-roadmap/23134/6)) Use that relationship.
- A prior **Gauntlet "Market Risks of Listing LP Tokens as Collateral"** analysis exists — read it and pre‑empt it. ([gov thread 10573](https://governance.aave.com/t/gauntlet-analysis-market-risks-of-listing-lp-tokens-as-collateral/10573))

### 8.3 Open items to close before TEMP CHECK
1. **Balancer v3 maintenance/security answer** post‑Labs‑windown (who patches, who runs the bug bounty, audit cadence). This is the first question an Aave delegate asks.
2. **Live numbers** from `api-v3.balancer.fi` (per‑pool TVL, 24h/30d volume, fees, the exact list of live Boosted pools and their rate providers/admin keys).
3. **Audited reference oracle** (Gyroscope‑style) for the specific pilot pool.
4. **Testnet liquidation drill** evidence.
5. Confirm whether the 2022 "Staked aTokens" BPT‑collateral integration ever shipped (evidence suggests **no**) — frame this as first‑of‑kind on v4.

---

## 9. Recommended path

1. **Do not rush to a public TEMP CHECK.** First close the §8.3 open items and get informal LlamaRisk + Aave Labs read‑throughs.
2. **Pilot‑first framing:** propose a single Boosted (Tranche C) pool, tiny caps, capped credit line — a demonstrably low‑risk showcase of v4 isolation that grows GHO.
3. **Scale on evidence:** add Tranche A/B and raise caps only on clean operation + Balancer v3 TVL recovery.
4. **Sequence the message:** GHO + v4‑isolation showcase first; Balancer benefit second.

---

## 10. Appendices

### Appendix A — Key sources
*(Consolidated; full inline citations above.)*
- **Aave v4:** [architecture](https://aave.com/blog/understanding-aave-v4s-architecture), [GitHub overview](https://github.com/aave/aave-v4/blob/main/docs/overview.md), [risk premiums](https://aave.com/blog/aave-v4-risk-premiums), [risk isolation](https://aave.com/blog/aave-v4-risk-isolation), [launch (The Block)](https://www.theblock.co/post/395617/aave-v4-launches-ethereum-mainnet)
- **Spoke precedents:** [Babylon Spoke TEMP CHECK](https://governance.aave.com/t/temp-check-establish-babylon-spoke-and-onboard-babylon-native-btc/24911), [PT‑USDG v4 ARFC](https://governance.aave.com/t/arfc-onboard-pt-usdg-24sep2026-to-aave-v4-on-ethereum/24942)
- **Balancer v3 / Boosted / buffers:** [v3 overview](https://docs.balancer.fi/partner-onboarding/balancer-v3/v3-overview.html), [100% Boosted](https://medium.com/balancer-protocol/100-boosted-pools-powered-by-aave-53fe5b920833), [buffers](https://docs.balancer.fi/concepts/vault/buffer.html), [v3×Aave launch](https://www.theblock.co/post/330379/balancer-v3-launches-aave)
- **BPT pricing / oracles:** [valuing BPT](https://docs-v2.balancer.fi/concepts/advanced/valuing-bpt/valuing-bpt.html), [BPT oracle example](https://docs.balancer.fi/concepts/core-concepts/bpt-oracles/bpt-oracles-example.html), [Gyro E‑CLP](https://docs.balancer.fi/concepts/explore-available-balancer-pools/gyroscope-pool/gyro-eclp.html), [reentrancy notice](https://forum.balancer.fi/t/reentrancy-vulnerability-scope-expanded/4345), [CAPO](https://github.com/bgd-labs/aave-capo)
- **Incidents:** [Balancer v2 $128M (Check Point)](https://research.checkpoint.com/2025/how-an-attacker-drained-128m-from-balancer-through-rounding-error-exploitation/), [OpenZeppelin](https://www.openzeppelin.com/news/understanding-the-balancer-v2-exploit), [ezETH depeg](https://www.dlnews.com/articles/defi/renzos-ezeth-loses-ether-peg-drops-79-in-under-one-hour/), [CRV‑on‑Aave](https://thedefiant.io/news/defi/crv-trade-aave-bad-debt), [Aave CAPO $27M](https://www.coindesk.com/business/2026/03/10/defi-lending-platform-aave-sees-a-rare-usd27-million-liquidations-after-a-price-glitch), [Sturdy/CertiK](https://www.certik.com/resources/blog/oracle-dependency-decrypting-the-sturdy-finance-attack)
- **Fluid:** [cyber.fund](https://cyber.fund/content/fluid-1), [Mirador liquidations](https://www.mirador.finance/p/fluid-liquidation-mechanism-a-closer), [Messari](https://messari.io/report/understanding-fluid-a-comprehensive-overview), [Resolv bad debt](https://blockonomi.com/fluid-covers-21m-bad-debt-after-resolv-exploit-outlines-recovery-plan/)
- **Numbers:** [DefiLlama Balancer](https://defillama.com/protocol/balancer) / [v3](https://defillama.com/protocol/balancer-v3) / [Aave](https://defillama.com/protocol/aave); [exploit/windown (CoinDesk)](https://www.coindesk.com/tech/2026/03/24/balancer-labs-will-shut-down-as-corporate-entity-became-a-liability-after-usd110-million-exploit); [LST concentration](https://www.chaincatcher.com/en/article/2248017)
- **Governance shifts:** [ACI exit](https://governance.aave.com/t/aci-is-leaving-aave/24205), [Chaos Labs exit](https://governance.aave.com/t/chaos-labs-is-leaving-aave/24386), [Aave Will Win](https://www.coindesk.com/tech/2026/04/13/aave-passes-landmark-vote-ending-months-long-fight-over-who-controls-protocol-revenue)

### Appendix B — Verification to‑do (data not fetchable in this environment)
Live Balancer per‑pool TVL/volume/fees and Boosted‑pool list (`api-v3.balancer.fi`); current Balancer total TVL reconciliation ($117M v2+v3 vs ~$157M incl. CoW AMM); Aave live collateral list; whether "Staked aTokens" (2022) shipped; LlamaRisk/Chaos scoring rubrics from primary docs.

### Appendix C — Devil's‑advocate stress test
*(See §11 below — this log is updated across review cycles.)*

---

## 11. Devil's‑advocate stress‑test log

*Cycle 0 (initial draft) complete. Adversarial cycles to follow.*
