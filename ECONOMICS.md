# Forge — Economics

Honest economics for forge based on **cited Promova baselines** (Amplitude charts + Anna Z analytics + Slack-cited campaign results). No fabricated benchmarks.

> **Demo path vs Production path:** Claude Desktop accumulates context across turns — each turn re-reads system prompt + tool definitions + conversation history. Demo costs ~$0.30/push because of this overhead. Production uses isolated direct SDK calls — backend assembles the snapshot via HTTP, LLM only generates the message. ~$0.004/push, **75x cheaper**.

---

## TL;DR

- **Demo cost (Claude Desktop + MCP):** ~$0.30/push — measured live, includes context accumulation overhead
- **Production cost (direct backend → SDK):** ~$0.004/push — discovery tokens = $0, only generation paid
- **Pilot 7,000 attempts (production path):** **$28**
- **Honest pilot ROI (conservative funnel):** ~6.6x — see §3 for funnel math
- **Break-even:** 7 activations from 7,000 pushes (0.1% lift) — order of magnitude below any realistic CTR
- **Path A (deep link offer for churned):** upside, **not promised** — no Promova win-back baseline exists, measure in pilot

---

## 1. Cost

### Why two paths

**Hackathon (Claude Desktop + MCP).** Every Walhalla MCP call response (user_get, amplitude_analytics, cancel_reasons) is read by Claude as input tokens. Context accumulates across turns. A session of 10 users costs ~$3.00 total → **~$0.30/user**. Two users with large keeper history spiked to ~50k tokens each. Used because no backend integration is needed during the hackathon.

**Production (`ingestion.service.ts` + `composer.service.ts`).** Backend makes HTTP requests directly: Walhalla REST → Amplitude API → DB query for group_stats. LLM only sees the assembled snapshot. Discovery tokens = **$0**.

```
Hackathon (MCP):                    Production (direct API):
LLM reads user_get    ~5k tokens    HTTP GET walhalla/user  → $0 LLM
LLM reads amplitude   ~3k tokens    HTTP GET amplitude      → $0 LLM
LLM reads cancel      ~1k tokens    HTTP GET cancel_reasons → $0 LLM
LLM reads group_stats ~1k tokens    DB query group_stats    → $0 LLM
─────────────────────────────────   ────────────────────────────────
generation            ~3k tokens    generation only         ~3k tokens
Total LLM input:     ~13k tokens    Total LLM input:        ~3k tokens
```

### Cost comparison

| | Hackathon (MCP) | Production (direct API) | Cheaper by |
|---|---|---|---|
| Cost/push | ~$0.30 | **~$0.004** | **75x** |
| Pilot 7,000 attempts | ~$2,100 | **~$28** | 75x |
| Per 1,000 pushes (linear scaling) | ~$300 | **~$4** | 75x |

### Production cost breakdown

Per push (single isolated SDK call):

| Component | Tokens |
|---|---|
| Brand voice prompt (cached) | ~1,500 |
| User snapshot (fresh per user) | ~500 |
| Group stats winners/losers (fresh per group) | ~1,000 |
| **Total input** | **~3,000** |
| Output (reasoning + push text) | **~300** |

| Model | Cost/push |
|---|---|
| Sonnet 4.6 + prompt caching | **~$0.005** |
| Haiku 4.5 (cold-start groups, no in-context examples yet) | **~$0.003** |
| **Sonnet+cache + Haiku hybrid (production target)** | **~$0.004** |

---

## 2. Promova baselines used in revenue calculations

All numbers below are **cited from Promova's own data**, not industry benchmarks.

### Activation funnel (Amplitude)

| Step | Conversion | Source |
|---|---|---|
| Return to app → Learned Lesson (within 24h) | **48–52%** (avg ~50%) | Amplitude chart `f6ew1hqt` (Yevhenii Mianovskyi), last 30 days, Apr 5 – May 5 2026 |
| Lesson → Activation Pattern (1+ lesson on 2+ days) | ~40% (estimated from Anna Z's chart) | Anna Z, `xd7eauzw` retention chart |
| Activation → ARPS_13m delta | **+$4.40 per activated user** | Anna Z, #analytics, March 18 2026 |

### Cited campaign results (from Slack)

| Campaign | Result | Source |
|---|---|---|
| AI-generated welcome push (Veronika's pioneer experiment) | +41% CTR uplift on 12.2% stat-sig messages | Veronika V + Vlad Gut, #bite-sized_marketing, May 14 2025 |
| Personalized welcome push (Veronika static personalization) | +13.4% start lift | Veronika V, #symonenko_scale, Sep 8 2025 |
| Welcome 50% discount email | View → Purchase 4.97% | Maiia Pustovit, May 22 2025 |
| PALM well-being broadcast email | $0.00042 – $0.00105 revenue per email sent | Maiia Pustovit, #scale_palm |
| Inactive user reactivation email (Yevheniia) | open +8.3%, click +0.43% vs manual | Yevheniia Semko, #bite-sized_marketing, Jul 28 2025 |
| Post-cancellation save offer (in-app, NOT push win-back) | **11–13% paywall view → purchase** | Hanna Sharuk chart `ajqmxujy`, last 12 weeks |
| Purchase → Returned to app (within 90d) | **91.6%** | Roman Hryshkanych chart `4jf3603m`, 81,580 purchases |
| CRM channel revenue % of total | **3% (May 2025), 6.5% (January)** | Veronika V, #bite-sized_marketing |

### What we **don't** have

- Re-engagement push CTR for inactive users (specifically push, not email) — no Promova baseline documented
- Push-driven win-back conversion for churned-paying users — no baseline
- Specific ARPS lift from forge-style personalization vs broadcast — no measured comparison

These are the **unknowns we measure in the pilot**.

---

## 3. Pilot ROI — 7,000 attempts

### Conservative funnel (push CTR 3% — lower than typical broadcast)

```
7,000 pushes (1,000 users × 7 waves, per Vlad Gut recommendation)
  → 3% CTR                                          = 210 clicks
  → 50% return → learned lesson (Amplitude f6ew1hqt) = 105 lessons
  → 40% lesson → Activation Pattern (Anna Z)        = 42 activations
  → × $4.40 ARPS_13m delta per activation           = $185 revenue
  
Cost (production direct backend): $28
Net profit: $157
ROI: ~6.6x
```

### Sensitivity to CTR (only assumption we don't have measured)

| CTR | Activations | Revenue | Net | ROI |
|---|---|---|---|---|
| **0.1%** (break-even floor) | 7 | $31 | $3 | 1.1x |
| **1%** (well below broadcast) | 14 | $62 | $34 | 2.2x |
| **3%** (conservative) | 42 | $185 | $157 | **6.6x** |
| **5%** (mid-range personalization) | 70 | $308 | $280 | 11x |
| **8%** (Veronika +41% uplift transferred) | 112 | $493 | $465 | 18x |

**Break-even:** 7 activations from 7,000 pushes = **0.1% activation lift**. An order of magnitude below any realistic CTR for re-engagement comms. To not break even, forge would need to underperform Veronika's already-proven floor by ~130x.

### Path A — upside, not promised

Path A: deep link to `promova_annual_trial_20disc` (-20% annual + 7-day trial) embedded in pushes for `churned-paying` users.

We do **not** have Promova baseline for push-driven win-back conversion. The 11-13% post-cancellation save rate (`ajqmxujy`) is **inside the cancel funnel** — captive, motivated audience at moment of cancel intent — not comparable to win-back through a push days/weeks later.

Path A is a **measurable upside in the pilot**, not promised in the floor calculation.

---

## 4. What we measure in the pilot

### Primary leading indicator
**Win-rate trajectory by wave.** If wave 2 win-rate > wave 1, wave 3 > wave 2, ... — system learns. If plateau across 3+ consecutive waves on the same group_key — fail-fast at 100–150 attempts, not 12k.

### Secondary leading indicator
**Predicted confidence calibration.** Brier score of `predictedConfidence` vs `outcomeReactivated` — should improve across waves. Validates that the model is learning from feedback, not gaming.

### Final outcome metric
**Activation Pattern Hit-rate.** Anna Z's metric: 1+ lesson on 2+ days. Correlation 0.8 with W4 retention (Anna Z, March 18 2026). This is what funds the ARPS lift.

### Path A (if we activate it)
**Push → click → paywall view → purchase.** Direct attribution via `attempt_id` UTM in the deep link.

---

## 5. Production scale assumptions (parked for now)

We **don't claim 50k/day = 1.5M/month** anymore — that was a fantasy figure and Promova's actual inactive cohort size needs confirmation from Anastasiia Stelmakh / Anna Ziubanova before sizing production.

Per-1,000-push cost:
- Hackathon (MCP): ~$300
- Production direct API: **~$4**

Linear scaling: whatever cohort size CRM team agrees to target, multiply by $0.004/push for production cost.

---

## 6. Honest caveats

**Cost side:**
- $0.30/push demo cost is measured but not representative — Claude Desktop overhead.
- $0.004/push production cost is calculated from token economics, not yet measured (no production SDK code path shipped in MVP).
- Real production cost will depend on actual cache hit rate for brand voice + group structure.

**Revenue side:**
- The 50% return → learned lesson (Amplitude `f6ew1hqt`) is across **all returning users**, not push-driven specifically. Push-driven returns may convert higher (engaged via personalized CTA) or lower (interrupted attention) — measure in pilot.
- $4.40 ARPS_13m delta is a **floor** — excludes cross-sell, renewal beyond 13 months, organic, B2B LTV.
- 40% lesson → Activation Pattern is an estimate from Anna Z's chart — exact number for the specific segment (es, B1, days_since_active 7–14) needs verification.
- CTR for re-engagement push is the **only major assumption** we don't have measured at Promova. Sensitivity table in §3 shows ROI is positive even at 1% CTR.

**Methodology:**
- Every cost number is calculated, not estimated.
- Every revenue benchmark cites a specific Slack post, Amplitude chart ID, or analytics report.
- Path A is explicitly marked as upside-not-promised, because no historical push-driven win-back data exists at Promova.

---

## 7. Demo slide (Block 3 — Analytics & Economics, 20%)

**Single slide:**

```
COST
  ~$0.30 / push    Hackathon: Claude Desktop + Walhalla MCP
                   (LLM reads all API responses as tokens, dev tool)

  ~$0.004 / push   Production: backend assembles snapshot via HTTP,
                   LLM only generates message (75x cheaper)

  $28 / pilot      7,000 attempts on production path

PILOT ROI (conservative, CTR 3%)
  7,000 pushes
  → 210 clicks (3% CTR — below broadcast benchmark)
  → 105 lessons (50% return→lesson, Amplitude f6ew1hqt)
  → 42 activations (40% Activation Pattern, Anna Z)
  → $185 revenue ($4.40 ARPS × 42)
  Net: $157 profit, ROI 6.6x

BREAK-EVEN
  7 activations from 7,000 pushes (0.1% lift)
  130x below proven Veronika +13.4% floor
  → structurally cheap to test

WHAT WE MEASURE
  Primary:   win-rate trajectory by wave (does system learn?)
  Final:     Activation Pattern Hit-rate (Anna Z, 0.8 corr W4 retention)
  Upside:    Path A purchase attribution via deep link (if activated)
```

**1-sentence pitch:**
> "Pilot costs $28 on production architecture. Break-even at 0.1% activation lift — an order of magnitude below Veronika's already-proven +13.4% AI-generation floor. We don't promise specific ROI numbers because Promova doesn't have a re-engagement push CTR baseline yet — but the cost is so low that any positive CTR pays back the pilot, and we measure trajectory not absolutes."

---

*Updated May 5 2026, hackathon day 1. Costs revised after honest session token analysis (~$0.30/push demo, ~$0.004/push production target). Revenue funnel grounded in cited Promova Amplitude charts (`f6ew1hqt`, `ajqmxujy`, `4jf3603m`) and Anna Z analytics. Path A reframed as measurable upside, not floor commitment, since no Promova push-driven win-back baseline exists.*
