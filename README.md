# ORRERY

### **O**mni-Rate **R**isk **R**econciliation for **E**vidence, **R**ebalancing & **Y**ield

> **One Greek source. Four clock domains. Nine orders of magnitude.**
> The advisor, the rebalancer, the hedger and the margin engine read the same register.

---

## 0 · The Antikythera Problem

In 1901, sponge divers working a wreck off the Greek island of Antikythera brought up a corroded lump of bronze. It sat in a museum for fifty years before anyone understood it. When they finally did, the object turned out to contain at least thirty interlocking gears cut by hand, driving pointers that tracked the sun, the moon, the lunar phase, the Metonic cycle and the timing of eclipses — all from one input crank.

The remarkable thing is not that it computed. It is *how*.

The moon pointer and the Metonic pointer move at wildly different rates — one completes a cycle in under a month, the other takes nineteen years. They could have been built as two separate instruments. They were not. They share a gear train, which means they can never disagree. Turn the crank, and every pointer on the face is, by construction, consistent with every other pointer. There is no reconciliation step, because there is nothing to reconcile.

Now consider a modern wealth platform.

The advisor's quarterly report says a client's equity exposure is one number. The rebalancing optimiser, running its own analytics, says something slightly different. The hedging desk's risk system says a third thing. The margin engine — which is the only one of the four whose answer is legally binding, because it determines a cash call — says a fourth. Four systems, four models, four release cycles, four vendors. And a permanent, expensive, human function whose entire job is to explain the differences.

That function exists because somebody built four instruments instead of one gear train.

**ORRERY is the gear train.** A single hardware Greek source feeds four consumers running nine orders of magnitude apart in time, and the consumers never compute their own sensitivities. What they compute is *policy*. Arithmetic disagreement is designed out of the system rather than reconciled after the fact.

---

## 1 · The Domain Map

```
                    ┌──────────────────────────────────────────┐
                    │   D0 · GREEK SOURCE   (HELICON)          │
                    │   400 MHz · 1 nine-Greek vector/cycle    │
                    │   372.5 ns deterministic · zero jitter   │
                    │   PDE residual attested per result       │
                    └───┬──────┬──────────┬──────────┬─────────┘
      constraints ↓     │      │          │          │     ↑ measurements
                        │      │          │          │
        ┌───────────────▼─┐ ┌──▼────────┐ ┌▼─────────┐ ┌▼──────────────┐
        │ D1 HEDGING      │ │ D2 COLLAT │ │ D3 REBAL │ │ D4 ADVISORY   │
        │ µs – ms         │ │ s – min   │ │ hrs–days │ │ days–quarters │
        │ option select   │ │ margin    │ │ optimise │ │ risk framing  │
        │ $200M mandate   │ │ SIMM/HVaR │ │ $300M    │ │ $500M         │
        └─────────────────┘ └───────────┘ └──────────┘ └───────────────┘
              10⁻⁶ s            10⁰ s        10⁴ s          10⁶ s
```

Nine orders of magnitude from the fastest consumer to the slowest, and eleven from the source. The system's entire design problem is stated in that one fact: **how do you keep four processes that tick at radically different rates from ever holding inconsistent beliefs about the same portfolio?**

That is not a finance question. It is the **clock-domain crossing** problem from digital design, and it has known answers.

| Domain | Period | Consumer | What it decides | Greek dependency |
|:--|:--|:--|:--|:--|
| **D0** | 2.5 ns | HELICON | nothing — it measures | emits Δ Γ ν Θ ρ Vanna Volga Charm + residual |
| **D1** | µs–ms | Hedging | *which contract* | Γ$, ν$, Θ$ for instrument selection |
| **D2** | s–min | Collateral | *how much cash* | Δ, ν, curvature — the margin inputs |
| **D3** | hrs–days | Rebalancing | *which basket* | Δ$, Γ$ constraints on the optimiser |
| **D4** | days–qtrs | Advisory | *what to hold, and why* | Greek-space suitability box |

---

## 2 · Law 0 — One Greek Source

**No consumer computes its own sensitivities. Ever.**

This is the load-bearing rule, and it is worth being concrete about what it buys, because the benefit is usually described in vague terms ("a single source of truth") when it is in fact arithmetic.

Four systems pricing the same at-the-money-ish equity option, each with its own library, compiler flags, day-count convention and dividend assumption, will produce four deltas:

```
  advisory report        Δ = 0.6368
  rebalance optimiser    Δ = 0.6371
  hedge engine           Δ = 0.6368
  margin engine          Δ = 0.6355
  ────────────────────────────────────
  spread                     0.0016
```

Sixteen basis points of delta. On a single $50M position that is **$80,000 of phantom exposure** — exposure that one system believes exists and another does not. Across 2,500 positions averaging $400,000 notional, that is **$1.6M of daily break** that somebody has to look at, categorise, and sign off before the books close.

None of it is a modelling error. All four are "right." They are right about slightly different things, and the difference has no economic content whatsoever.

Under Law 0 that number is identically zero, because there is one register and four readers. What remains are the differences that *do* have content — a constraint the advisor set that the rebalancer honoured and the hedger could not, a policy conflict rather than a numerical one. Those are worth a human's attention. The other $1.6M was never worth anyone's.

### 2.1 What D0 Emits

Every cycle, for every instrument in scope, the source emits a fixed-shape record:

```
GREEK_VECTOR {
  instrument_id       u64
  as_of_seq           u64        gray-coded, monotone       (§4)
  price               Q24.40
  delta               Q24.40
  gamma               Q24.40
  vega                Q24.40
  theta               Q24.40
  rho                 Q24.40
  vanna               Q24.40
  volga               Q24.40
  charm               Q24.40
  sigma_implied       Q24.40     ← the inverse map (§8.2)
  pde_residual        Q24.40     ← conservation-law attestation
  window_flag         u1         ← convergence-domain guard
  model_id            u8         ← lognormal | bachelier | displaced
}
```

Two fields deserve emphasis because they are what make the record *trustworthy* rather than merely *fast*:

- **`pde_residual`** — the Black–Scholes PDE identity `Θ + (r−q)SΔ + ½σ²S²Γ − rC`, computed beside the output stage at no marginal latency. A correct vector has a residual at the noise floor (one double ULP, ~2.2e−16, on the reference case). A vector produced by a stuck ROM bit, a dropped iteration, or an input pushed outside the convergence window does not. Every consumer downstream can therefore **check the goods it was handed**, rather than trusting the supplier.
- **`model_id`** — because a hyperbolic logarithm has no value at a negative underlying, and that is not hypothetical. On 20 April 2020 the May WTI contract settled at **−$37.63** after an intraday low of **−$40.32**; CME Clearing published an advisory titled **"Switch to Bachelier Options Pricing Model"** on 21 April, effective for the margin cycle ending 22 April, and negative strikes down to −50 traded on the June future within days. ICE did the same. Any system that hard-codes log-space has embedded limited liability as a routing assumption, and routing assumptions do not update on a clearing advisory.

---

## 3 · Law I — Constraints Down, Measurements Up

**Information flows downward as bounds and upward as observations. Never sideways, and never downward as instructions.**

```
   D4 advisory   ──── sets ────►  Greek-space box: (Δ$, Γ$, ν$, Θ$/day) bounds
                 ◄─── reports ──  realised position, attribution, breaches

   D3 rebalance  ──── sets ────►  tradable universe, tax-lot state, basket policy
                 ◄─── reports ──  achieved weights, in-kind vs cash split

   D2 collateral ──── sets ────►  margin envelope, concentration limits
                 ◄─── reports ──  IM/VM consumed, headroom

   D1 hedging    ──── acts ─────► orders, within all inherited bounds
                 ◄─── reports ──  fills, residual exposure
```

The rule that a slow tier may only ever emit a **constraint** is what makes the language model safe to deploy at D4. A model that emits a constraint cannot emit an order. It can narrow the feasible set; it cannot pick a point in it. The picking happens at D1, inside an optimiser, inside a margin envelope, against a Greek vector with an attested residual.

The corollary is the useful part: **the advisory tier cannot recommend something the hedging tier cannot hold**, because both resolve against the same box. The oldest failure in wealth management — the advisor promises an outcome that operations cannot implement — becomes a type error rather than a discovery made in week three.

---

## 4 · Law II — Clock-Domain Crossing

**A slow reader must never sample a fast writer mid-update.**

In silicon this is metastability: a flip-flop clocked while its input is transitioning settles to an indeterminate value for an unbounded time. The standard fix is not to make the signal faster. It is to make the *crossing* disciplined — synchroniser chains, gray-coded pointers so only one bit changes per increment, and two-phase handshakes for multi-word payloads.

The wealth-platform version of metastability is a nightly batch that reads a position file while an intraday process is halfway through writing it, and produces a report that is internally impossible: a portfolio holding a hedge for an exposure it does not have. Everybody in the industry has seen it. Almost nobody names it correctly.

ORRERY's crossings are explicit:

| Crossing | Mechanism | Guarantee |
|:--|:--|:--|
| **D0 → D1** | lock-free ring, single producer, gray-coded write pointer | reader sees a complete vector or the previous one; never a torn one |
| **D1 → D2** | two-phase handshake on `as_of_seq` | margin is computed against a named, immutable snapshot |
| **D2 → D3** | snapshot isolation, copy-on-write position record | the optimiser runs against a frozen book |
| **D3 → D4** | append-only event log, no in-place update | every figure the advisor sees has a sequence number |

The `as_of_seq` field is gray-coded for exactly the reason a FIFO pointer is: a monotone counter crossing a domain boundary can be sampled during a multi-bit carry and read as a value that never existed. One bit changing per increment makes the worst case "off by one," not "arbitrary."

**Design law:** every number that reaches a client — in a report, a chat response, a statement, a margin call — carries the `as_of_seq` it was derived from. Two numbers with different sequence numbers are not comparable, and the system says so rather than silently averaging them.

---

## 5 · D4 — The Advisory Tier · $500M

### 5.1 The Constraint Nobody Budgets For

A generative advisory assistant looks like a language problem. It is not. It is an **evidence** problem wearing a language costume.

The binding constraint is stated most cleanly by what happened on 18 March 2024, when the SEC settled its first two AI-related actions against registered advisers. Delphia paid $225,000; Global Predictions paid $175,000; $400,000 total. The charges were under Advisers Act §206(2) and §206(4), the Marketing Rule (206(4)-1) and the Compliance Rule (206(4)-7). The finding in one case was that a firm could not **produce documents to substantiate** the claims it had made. In the Chair's framing, **"AI washing hurts investors."**

Read that as a systems requirement rather than a headline. The failure was not that a model said something false. It was that when asked to show the basis for a statement, the firm had nothing to show. **Substantiation is the deliverable.** The language is packaging.

### 5.2 The Two-Channel Architecture

ORRERY's advisory tier separates the **numeric channel** from the **linguistic channel** completely. The model writes prose and structure. It does not write numbers.

```
                        ┌─────────────────────────────┐
   client question ────►│  intent + slot extraction   │
                        └──────────────┬──────────────┘
                                       │
                    ┌──────────────────┴──────────────────┐
                    │                                     │
        ┌───────────▼──────────────┐        ┌─────────────▼─────────────┐
        │  NUMERIC CHANNEL         │        │  LINGUISTIC CHANNEL       │
        │  ─────────────────       │        │  ──────────────────       │
        │  · position record (SPR) │        │  · model generates prose  │
        │  · D0 Greek vectors      │        │  · emits {slot} tokens    │
        │  · tax-lot ledger        │        │  · NEVER emits a numeral  │
        │  · margin envelope (D2)  │        │  · grammar-constrained    │
        │  every value carries     │        │    decoding forbids       │
        │  record_id + as_of_seq   │        │    free digits            │
        └───────────┬──────────────┘        └─────────────┬─────────────┘
                    └──────────────┬──────────────────────┘
                                   │
                        ┌──────────▼───────────┐
                        │  BINDING + MONITOR   │
                        │  every {slot} must   │
                        │  resolve to a        │
                        │  (record_id, seq)    │
                        │  UNBOUND ⇒ BLOCK     │
                        └──────────┬───────────┘
                                   │
                        response + substantiation manifest
```

**The invariant: no unbound numeral.** A response containing a figure that does not resolve to a record identifier and a sequence number is not returned. It is blocked, logged, and the interaction falls back to a template.

This is the same idea as HELICON's PDE residual monitor, transposed one domain up. In both cases a correctness property is checked *beside* the output path rather than in front of it, costs almost nothing, and converts a probabilistic component into an auditable one. In silicon the invariant is a conservation law. Here it is provenance.

What the manifest looks like:

```
RESPONSE  "Your equity sleeve currently carries about {A} of exposure to a
           one-percent move, and the collar you asked about would reduce that
           to roughly {B} while costing about {C} per day in time decay."

MANIFEST
  {A}  $286,135      delta_dollar_1pct   rec:PF-8871  seq:0x4C1A93   D0@14:22:07.113
  {B}  $ 41,880      delta_dollar_1pct   rec:SIM-2210 seq:0x4C1A93   D1 candidate #7
  {C}  $ 36,477      theta_dollar_day    rec:SIM-2210 seq:0x4C1A93   D1 candidate #7
  MODEL lognormal    PDE residual 3.1e-16    window_flag OK
  BOUNDS respected: IPS box PF-8871-IPS-v4, margin envelope D2 seq:0x4C1A90
```

Every figure is reproducible from a named record at a named sequence. The books-and-records obligation under Rule 204-2 stops being a retrofitted logging project and becomes a byproduct of how the response was constructed.

### 5.3 Suitability, Expressed in Greek Space

Here is the piece that connects D4 mechanically to D1 rather than rhetorically.

Risk tolerance is conventionally captured by questionnaire, mapped to a label ("moderate growth"), mapped to a model portfolio, mapped to weights. Four translations, each lossy, none of which the hedging desk can evaluate.

ORRERY expresses the investment policy statement directly as a **box in dollar-Greek coordinates**:

```
IPS BOX  PF-8871-IPS-v4
  Δ$   (per 1% underlying move)     ∈  [  +1.2M ,  +2.4M ]
  Γ$   (per 1% move, second order)  ∈  [   -180k,   +400k ]
  ν$   (per 1 vol point)            ∈  [   -140k,   +140k ]
  Θ$   (per day)                    ∈  [    -55k,    +25k ]
  ρ$   (per 1 bp)                   ∈  [    -90k,    +90k ]
  concentration: no single name > 8% of Δ$
```

Six inequalities. Every one of them is checkable by the same arithmetic the hedging engine and the margin engine already run, against the same vectors, at the same sequence number.

The consequences:

- **A recommendation is feasible or it is not**, and the answer is arithmetic rather than judgement. No round trip to a human to discover the constraint.
- **The advisor and the trader speak one language.** "Moderate growth" is not a quantity. `Δ$ ∈ [1.2M, 2.4M]` is.
- **Drift is a breach with a timestamp**, not a quarterly observation. When the realised Δ$ leaves the box, the event is emitted upward with a sequence number, and D1 already knows the notional required to return it.
- **The Θ$ bound is the one clients have never had.** A yield-enhancement overlay is usually sold on premium collected. Its cost is a daily decay figure the client has never seen quantified against a bound they agreed to.

### 5.4 Mandate Arithmetic

```
  $500,000,000 advisory mandate

  avg household     $250,000  →  2,000 households
  avg household     $500,000  →  1,000 households
  avg household   $1,000,000  →    500 households
  avg household   $2,500,000  →    200 households

  fee   25 bps → $1,250,000/yr        fee   75 bps → $3,750,000/yr
  fee   50 bps → $2,500,000/yr        fee  100 bps → $5,000,000/yr

  2,000 households ×  12 interactions/yr =  24,000/yr =    95/business day
  2,000 households ×  26 interactions/yr =  52,000/yr =   206/business day
  2,000 households ×  52 interactions/yr = 104,000/yr =   413/business day
  2,000 households × 250 interactions/yr = 500,000/yr = 1,984/business day
```

The nightly Greek load for the advisory tier — 2,000 households × 45 positions × 5,000 stress scenarios — is **450 million evaluations**. Note that number. It comes back in §9.

---

## 6 · D3 — The Rebalancing Tier · $300M

### 6.1 The Ground Has Moved

Active ETFs are no longer a structure argument. As of mid-2026 they hold roughly **13% of the ~$16.1 trillion in US-listed ETF assets** while capturing **more than a third of all flows** — a figure that has climbed from 26% in 2024. In 2025 they took in approximately **$450–460 billion**, about a third of the $1.46 trillion industry total, and active funds have accounted for **more than 80% of new ETF launches**. Active ETF asset growth has compounded at over **59% annually across three years**, nearly double the industry rate.

Three regulatory events made that possible, and a fourth just changed the shape of the field:

| Date | Event | Consequence for D3 |
|:--|:--|:--|
| **25 Sept 2019** (eff. 23 Dec 2019) | Rule 6c-11 adopted | **Custom baskets** for all transparent active ETFs, subject to written policies; daily portfolio transparency; six-year records |
| ongoing | IRC §852(b)(6) | In-kind redemption recognises **no gain** at the fund |
| **17 Mar 2026** | SEC Exchange Act §36 relief supplementing 1940 Act share-class relief | Broker-dealer and market-structure relief permitting an ETF share class alongside non-exchange-traded classes without triggering Rules 10b-10, 14e-5 or §11(d)(1) |
| **27 Jul 2026** | Division of Investment Management no-action letter to ICI | Creation baskets accepted during a **passive concentration exceedance**, explicitly extended to actively managed ETFs |

That last one matters more than its length suggests. Active ETFs exercise portfolio discretion, which is precisely what complicates any argument that accepting a basket was non-volitional. Getting it in writing removes a constraint that had been quietly shaping basket policy.

### 6.2 The Objective Function Is After-Tax, and It Is Lot-Level

The rebalancing literature optimises weights. The client experiences after-tax return. Those are different objective functions, and the gap is measurable.

The most careful estimate in the literature evaluated CRSP top-500 data from **1926 to 2018** at long-term and short-term capital gains rates of 15% and 35%, and found a before-cost tax alpha of **1.08% per year**. Constrained by the wash-sale rule it fell to **0.82%**. Transaction costs subtract roughly **13 bps**, landing near **0.95%**.

```
  On a $300,000,000 sleeve:

    unconstrained          1.08%  →  $3,240,000 / yr
    wash-sale constrained  0.82%  →  $2,460,000 / yr
    net of transaction cost 0.95% →  $2,850,000 / yr

  Compare the management fee on the same assets:

    15 bps → $  450,000      35 bps → $1,050,000
    25 bps → $  750,000      50 bps → $1,500,000
```

**The tax channel is worth roughly two to six times the management fee.** Any optimiser whose objective is tracking error against a benchmark is optimising the smaller number.

ORRERY's D3 objective:

```
maximise   E[after-tax return]
         = E[pre-tax return]
         − Σ_lots  realised_gain(lot) · rate(holding_period(lot))
         + Σ_lots  harvested_loss(lot) · rate(offset_available)
         − transaction_cost
         − market_impact

subject to
   Δ$, Γ$, ν$, Θ$  ∈  inherited IPS box              (from D4)
   IM consumed     ≤  margin envelope                 (from D2)
   wash-sale window: no substantially-identical repurchase within ±30 days
   basket policy:   custom-basket parameters per 6c-11 written procedures
   concentration:   disclosed policy, with exceedance handling
```

The in-kind channel is a **free variable, not a post-process**. Every redemption basket the optimiser can construct is an opportunity to deliver out the lowest-basis lots without recognising gain. The decision of *which lots go out the door in kind* belongs inside the optimisation, not in a settlement script that runs afterward.

### 6.3 Why a Rebalancer Needs Greeks

A weight-based optimiser is correct only when exposure is linear in weight. The moment an active ETF holds a covered-call overlay, a convertible, a buffered structure, a preferred with a call feature, or any option position at all, weights stop describing exposure.

Concretely: two portfolios with **identical weights** in the same underlying can carry Δ$ that differ by a factor of two and Γ$ that differ in *sign*. A weight-space optimiser cannot see this. A Greek-constrained one cannot miss it.

So D3 inherits the IPS box from D4 in the same coordinates, and its feasible set is defined in Greek space rather than weight space. The vectors come from D0 — the same vectors D1 hedges against and D2 margins against.

### 6.4 Creation Unit Economics

```
  $300,000,000 fund

  NAV $25 → 12,000,000 shares    CU 25,000 sh = $  625,000    480 CUs outstanding
  NAV $30 → 10,000,000 shares    CU 25,000 sh = $  750,000    400 CUs outstanding
  NAV $50 →  6,000,000 shares    CU 50,000 sh = $2,500,000    120 CUs outstanding
```

At $300M the fund is small relative to a ~$2.1 trillion active-ETF category. **Capacity is not the constraint; granularity is.** With 400 creation units outstanding, a single CU is 25 bps of the fund. The in-kind channel's resolution — how finely the optimiser can steer lots out the door — is quantised by creation-unit size, and that quantisation is the real limit on tax-lot surgery at this scale. It is a design parameter, chosen at launch, and it is usually chosen for market-making convenience rather than for tax throughput.

---

## 7 · D2 — The Collateral Tier

### 7.1 Margin Is Already a Greek Program

This is the tier where the "one Greek source" argument stops being architectural preference and becomes economic necessity, because the margin methodologies **take Greeks as literal inputs**.

**Uncleared, bilateral — ISDA SIMM.** Launched September 2016 under the BCBS–IOSCO margin framework, applying above an aggregate average notional of €8 billion. It is a parametric sensitivities-based VaR model. In ISDA's own summary, **"SIMM uses sensitivities as inputs."** Its structure:

```
  Risk classes:  Interest Rate · Credit Qualifying · Credit Non-Qualifying
                 Equity · Commodity · FX
  Margin legs:   DELTA  +  VEGA  +  CURVATURE
                 (+ inter-curve basis, base correlation, concentration)

  Delta Margin aggregated across buckets within each risk class with
  correlation parameters γ_bc, then across risk classes.

  Instruments with optionality carry vega and curvature margin.
  Instruments without optionality carry neither.
```

Calibrated twice a year. ISDA publishes unit tests so implementations can be checked against a reference — which is exactly the ethos of an attested pipeline, one domain up.

**Cleared, exchange-traded — SPAN 2 / HVaR.** CME's replacement for SPAN moved from **16 scenarios to thousands**, built on a minimum ten-year historical lookback with volatility and correlation scaling, **explicitly treating the implied volatility surface including skew as risk factors**, plus hypothetical stress scenarios, liquidity and concentration add-ons:

```
  Total Portfolio Margin = x·Historical Risk + (1−x)·Stress Risk
                           + Liquidity + Concentration
```

Parameter files grew from a few hundred megabytes to nearly **20 GB**. Note what that number is telling you: margin computation has become a **high-throughput surface revaluation problem**. Firms can no longer estimate their own margin from a small parameter set, which means the ability to reproduce the CCP's number in-house went from routine to a capital investment.

**Margin is now the single most Greek-intensive process in a wealth platform, and it is the one with a legally enforceable deadline.**

### 7.2 Margin-Aware Hedging — The Result Everyone Misses

Because SIMM charges Delta, Vega and Curvature as **separate aggregated legs**, a hedge scored on delta alone is optimising one leg of three.

Take a book that is short 500 thirty-day 5%-out-of-the-money SPX puts — an ordinary yield-enhancement overlay. Now consider three hedges. **All three flatten delta to zero.**

| | Δ$ | Γ$ per 1% | ν$ per vol pt | Θ$ per day |
|:--|--:|--:|--:|--:|
| **Book, unhedged** | 48,642,881 | −14,876,437 | −220,090 | +62,011 |
| **A** — sell SPX futures | **0** | −14,876,437 | −220,090 | +62,011 |
| **B** — sell 139 × 7d ATM calls | **0** | −30,016,860 | −272,355 | +132,963 |
| **C** — sell 500 × 30d 5% calls + futures | **0** | −32,805,776 | −485,346 | +146,540 |

Scored on a SIMM-shaped proxy that aggregates the three legs:

```
  BOOK unhedged         proxy IM  66.07     —
  A  futures            proxy IM  44.71    −32.3%
  B  sell 7d calls      proxy IM  90.11    +36.4%
  C  short call + fut   proxy IM  98.59    +49.2%
```

*(Illustrative weights; the ordering is the point, not the level.)*

Three delta-flat hedges. One reduces initial margin by roughly a third. One raises it by roughly a third. One raises it by half. **A delta-neutral book can be a margin disaster**, and the desk that picks hedge C because it collects premium and flattens delta has traded a P&L improvement for a funding call.

ORRERY's D1 therefore optimises `risk + margin`, not `risk`. That is only possible because D1 and D2 read the same Greek vectors at the same sequence number. Where the hedge engine and the margin engine are separate systems on separate models, the joint objective cannot even be written down — the two sides of it are denominated in different numbers.

### 7.3 The Deadline, Not the Day

Intraday margin calls run on a clock. The relevant question is not evaluations per day; it is whether a cycle completes before the next one starts.

| Book | Positions | Scenarios | Evals / cycle | 1 HELICON lane | 1 CPU core | 64 cores |
|:--|--:|--:|--:|--:|--:|--:|
| $1B | 2,500 | 10,000 | 25,000,000 | **0.062 s** | 4.5 s | 0.07 s |
| $10B | 20,000 | 10,000 | 200,000,000 | **0.500 s** | 36.0 s | 0.56 s |
| $50B | 90,000 | 10,000 | 900,000,000 | **2.250 s** | 162.0 s | 2.53 s |
| $250B | 400,000 | 10,000 | 4,000,000,000 | **10.000 s** | 720.0 s | 11.25 s |

A $250B book at ten thousand scenarios is ten seconds on one lane and twelve minutes on one core. At $1B — the mandate in front of us — it is **62 milliseconds**. Hold that thought too.

---

## 8 · D1 — The Hedging Tier · $200M

### 8.1 The Parallax Residual

D1 does not decide what exposure to hold. D4 decided that, and wrote it as a box. D1's job is narrower and better defined:

```
  hedge_notional  =  measured_exposure(D0, seq=n)  −  intended_exposure(D4 box)
```

The **disagreement between what the slow tier intends and what the fast tier measures** is the hedge. One number, and it does double duty: it is the trading signal *and* it is the audit artifact, because a persistent non-zero residual is exactly what a supervisor wants to see monitored.

This makes D1 a *derived* tier rather than an independent one, which is the correct architecture and the opposite of how hedging desks are usually organised.

### 8.2 Selecting the Contract — Where Greeks Actually Decide

Given a required Δ$ adjustment, the instrument choice is a Greek-space problem, and the short end of the curve makes it brutal.

Same 294-contract notional, sized against a $200M book with SPX at 6,800:

| Tenor / strike | Premium / contract | Γ$ per 1% move² | ν$ per vol pt | Θ$ per day |
|:--|--:|--:|--:|--:|
| 30d, −5% | $2,632 | 43,754 | 129,465 | −36,477 |
| 7d, −2% | $1,902 | 111,941 | 77,285 | −96,141 |
| 1DTE, −0.5% | $1,186 | 364,948 | 35,995 | −319,260 |
| **1DTE, ATM** | $2,529 | **423,381** | 41,758 | **−367,876** |

**Gamma$ is 9.7× larger and Theta$ is 10.1× larger at 1DTE-ATM than at 30d−5%.** The cheap-looking hedge is the expensive one, and the expense arrives as a daily decay charge rather than as a premium at trade date — which is precisely why it escapes the premium-based comparison a trader does in their head.

This is not an edge case. **0DTE contracts reached a record 65% of total SPX volume in Q2 2026**, with SPX 0DTE ADV of 3.1 million contracts (up 48% year over year) and a June record near 3.3 million. Market-wide 0DTE volume exceeded **20 million contracts per day**, up 46.2% year to date. Total US listed options averaged **72.8 million contracts a day in Q2 2026** — the largest quarter ever recorded. The short end is now the market, and the short end is where Greek magnitudes are largest and least intuitive.

The D1 selector therefore scores every candidate on the full vector plus the margin delta:

```
score(candidate) =   w_Δ · |Δ$_resid|
                   + w_Γ · |Γ$_resid|
                   + w_ν · |ν$_resid|
                   + w_Θ ·  Θ$_cost_to_horizon
                   + w_M · ΔIM(candidate)              ← §7.2
                   + w_L ·  spread + impact
   subject to:  candidate ∈ IPS box ∧ IM ≤ envelope
```

`ΔIM` is available because D2 reads the same vectors. Without Law 0, that term is unavailable and the selector optimises four-fifths of the problem.

### 8.3 The Inverse Map Is the Hot Path

Market makers quote in volatility, not price. D1's inner loop runs **backwards** — from an observed premium to the σ that reproduces it — and the forward pricer is a subroutine called two to four times inside it.

That inversion is where the arithmetic turns hostile. Newton's step divides by Vega, and Vega collapses in the wings. At one day to expiry, σ = 20%:

| K/S | Price | Vega | 1/Vega |
|:--|--:|--:|--:|
| 1.00 | 4.245e−01 | 2.087809e+00 | 4.790e−01 |
| 1.05 | 3.580e−07 | 4.364409e−05 | 2.291e+04 |
| 1.10 | 5.768e−21 | **2.468353e−18** | 4.051e+17 |
| 1.20 | 2.556e−69 | 3.908082e−66 | 2.559e+65 |

At K/S = 1.10 the Newton denominator is **six orders of magnitude below one Q24.40 ULP** (9.095e−13). It is not small; it is absent from the number system. The published resolution — branch on log-moneyness into rational initial guesses, transform the objective in the degenerate branches, iterate with a fourth-order Householder step — reaches maximum attainable double precision in as few as two iterations for all admissible inputs, at roughly **180 ns per evaluation** on a modern core.

HELICON's residue harvesting makes this structurally cheap: the Householder step needs `∂C/∂σ` and `∂²C/∂σ²`, which are Vega and Volga, both already sitting in the recombination tree. **The highest-order, fastest-converging inversion is the cheapest one to build on this substrate**, inverting the usual cost ordering.

### 8.4 Mandate Arithmetic

```
  $200,000,000 hedging mandate, SPX at 6,800

  notional per SPX contract        $680,000
  contracts for 1.00 delta          294.1
  30d 5%-OTM put overlay premium    $774,039 total  ($2,632/contract)

  Greek exposures on that overlay:
    Δ$ per 1% move    −286,135
    Γ$ per 1% move²   + 43,754
    ν$ per vol point  +129,465
    Θ$ per day        − 36,477     ← the number that belongs in the IPS box
```

---

## 9 · The Reversal — Compute Was Never the Constraint

Now add up what the four tiers actually demand from D0 across a full trading day.

| Consumer | Load model | Evaluations / day | Share |
|:--|:--|--:|--:|
| **D1 hedging** | 10,000-contract SPX surface × 50 revals/s × 6.5 h | 11,700,000,000 | 94.5% |
| **D4 advisory** | 2,000 households × 45 positions × 5,000 scenarios | 450,000,000 | 3.6% |
| **D2 collateral** | 2,500 positions × 10,000 scenarios × 8 cycles | 200,000,000 | 1.6% |
| **D3 rebalancing** | 350 holdings × 12 lots × 2,000 scenarios × 4 passes | 33,600,000 | 0.3% |
| **TOTAL** | | **12,383,600,000** | |

One HELICON lane at 400 MHz delivers 9.36 × 10¹² evaluations in a 6.5-hour session.

```
  Total daily demand / one lane's session capacity  =  0.132 %

  Seconds of one lane required to clear the entire firm's day:   30.96 s
  Sustained CPU cores required at 180 ns/evaluation:              0.10
```

**The whole firm's daily Greek demand is thirty-one seconds of one pipeline, or a tenth of a CPU core.**

That number should stop the conversation, because it demolishes the premise most of this industry builds on. At a $1 billion mandate, across four business lines, running ten thousand margin scenarios and five thousand household stress paths, **throughput is not scarce.** It is so abundant that arguing about it is a category error.

So what *is* scarce?

**Tail latency under burst.** OPRA's sustained peak reached **53.5 million messages per second in February 2026** (50.9 million in January). During the April 2025 sell-off, one-millisecond bursts exceeded **23.7 million packets per second — over 187 million messages per second.**

| Reval policy at burst | Evaluations / s | Lanes required | CPU cores required |
|:--|--:|--:|--:|
| 1 per message | 187,000,000 | 0.47 | 34 |
| 5 (surface neighbours) | 935,000,000 | 2.34 | 168 |
| 20 (full expiry ladder) | 3,740,000,000 | 9.35 | 673 |

Even here, throughput is met. What is not met is *consistency of service time*. The pipeline delivers **every** result at 372.5 ns — no cache miss, no branch mispredict, no page fault, no interrupt, no garbage collection. A core delivers a median of 180 ns and a tail that is unbounded: a single 10 µs stall costs 56 evaluations, and stalls cluster exactly when the market does. The distribution is the product, not the mean.

**Reconcilability.** The $1.6M daily break in §2 is not a compute problem. Adding cores makes it worse, because more independent implementations produce more independent answers.

**Attestation.** A margin number that cannot be reproduced is a dispute. A recommendation that cannot be substantiated is an enforcement exposure. Neither is solved by FLOPs.

The industry has spent a decade optimising the abundant resource. The scarce ones — determinism, provenance, and a single arithmetic — were never on the roadmap, because they do not appear on a benchmark.

That is the same lesson the transport layer already learned. Aurora to Carteret is 1,185.5 km; the vacuum round trip is **7.909 ms**; the best microwave routes run at roughly **8.0–8.1 ms**. There is **one to two percent** of headroom left in the wire, after $300 million of fibre in 2010 and fifteen years of competition. When the abundant resource is finished, the game moves to the endpoint. Compute is finished too. The game has moved.

---

## 10 · Aggregate Mandate

```
  D4  Advisory       $500,000,000
  D3  Rebalancing    $300,000,000
  D2  Collateral      (serves all)
  D1  Hedging        $200,000,000
  ─────────────────────────────────
      TOTAL        $1,000,000,000

  Blended fee @ 75 / 35 / 50 bps      $5,800,000 / yr   =  58.0 bps
  Tax alpha retained, ETF sleeve       $2,850,000 / yr   (0.95%, net of cost)
                                       ──────────────
  Client-facing value created          $8,650,000 / yr
```

Two observations that follow directly from the arithmetic above and are worth stating plainly:

1. **The tax channel on the $300M sleeve ($2.85M) is comparable to the fee on the $500M sleeve ($3.75M).** The smaller mandate carries the larger economic lever, and it is the one that depends on lot-level optimisation with the in-kind channel as a free variable.
2. **The collateral tier has no mandate of its own and is the reason the other three can be run together.** It is a cost centre on any org chart and the binding constraint on all three revenue lines. Optimising `risk + margin` instead of `risk` is worth roughly a third of initial margin on a book like §7.2's — which, on a leveraged overlay, is a larger number than the management fee.

---

## 11 · Failure Atlas

| # | Failure | Where it bites | Guard |
|:--|:--|:--|:--|
| **F1** | Four systems, four deltas | $1.6M/day of phantom break | Law 0 — one source, four readers |
| **F2** | Slow tier reads a mid-write book | Impossible portfolios in reports | §4 gray-coded `as_of_seq`, snapshot isolation |
| **F3** | Model emits an unretrieved figure | Marketing Rule / §206(4) exposure | §5.2 no-unbound-numeral monitor; block, don't warn |
| **F4** | Model emits an order | Discretion without authority | Law I — slow tiers emit constraints only |
| **F5** | Weight-space optimiser on a non-linear book | Δ$ off by 2×, Γ$ wrong sign | §6.3 Greek-space feasible set |
| **F6** | Delta-only hedge selection | +49% initial margin on a Δ-flat book | §7.2 joint `risk + margin` objective |
| **F7** | Negative or zero underlying | Undefined ln; deterministic garbage at line rate | `model_id` + Bachelier bypass; sign guard at ingress |
| **F8** | Vega below one ULP in the wings | Inversion divides by zero at K/S ≥ 1.10, T = 1d | §8.3 branch-select + transformed objective |
| **F9** | Tax optimisation as a post-process | Lowest-basis lots leave in cash, not in kind | §6.2 in-kind channel inside the optimiser |
| **F10** | Wash-sale blindness across sleeves | 1.08% alpha → 0.82%, or a disallowed loss | Household-level ±30d substantially-identical ledger |
| **F11** | Margin cycle overruns its deadline | Funding call on stale numbers | §7.3 wall-clock budget per cycle, not per day |
| **F12** | Capability claimed but not implemented | The 18 Mar 2024 fact pattern, precisely | Substantiation manifest is the artifact, not the marketing |
| **F13** | Numbers compared across sequence numbers | Silent averaging of incomparable states | Every client-facing figure carries `as_of_seq` |

---

## 12 · Reference Architecture

```
ingress/
├── opra_handler/          burst-tolerant, 53.5M msg/s sustained, 187M peak
├── prop_feeds/            per-venue direct feeds, sequence-gap recovery
└── ref_data/              corp actions, dividends, borrow, holiday calendars

d0_source/                 ← HELICON
├── helicon_top.vhd        400 MHz, 372.5 ns, 1 vector/cycle
├── phi_gate/              tier-A tanh (1.79e−4) | tier-B A&S (7.45e−8)
├── bachelier/             model_id mux; sign guard
├── invert/                Householder-3 σ solver, reuses harvested ν, Volga
└── residual_monitor/      PDE identity + ν/Γ invariant, per-result

spr/                       ← Single Position Record
├── record.rs              immutable, append-only, gray-coded as_of_seq
├── snapshot.rs            copy-on-write, named, reproducible
└── crossing/              ring buffers, two-phase handshakes, synchronisers

d1_hedging/
├── parallax.rs            hedge_notional = measured − intended
├── selector.rs            full-vector + ΔIM scoring, IPS-box constrained
└── router/                venue selection, speed-bump-aware

d2_collateral/
├── simm/                  delta · vega · curvature legs, γ_bc aggregation
│   └── unit_tests/        reference vectors for implementation checking
├── hvar/                  10k+ scenario engine, vol surface incl. skew
├── envelope.rs            emits margin bounds downward to D1, D3
└── replication/           reproduce the CCP number in-house

d3_rebalance/
├── after_tax.rs           lot-level objective, HIFO/specific-ID
├── inkind.rs              redemption basket as a free variable
├── basket_policy/         6c-11 custom-basket written procedures
├── washsale.rs            household-level ±30d substantially-identical
└── concentration.rs       passive-exceedance handling

d4_advisory/
├── numeric_channel/       SPR + D0 + tax ledger + envelope; every value keyed
├── linguistic_channel/    grammar-constrained decoding; NO free numerals
├── binding.rs             {slot} → (record_id, as_of_seq) or BLOCK
├── manifest.rs            substantiation artifact per response
└── ips_box.rs             suitability in (Δ$, Γ$, ν$, Θ$, ρ$) coordinates

attest/
├── residual_log/          per-result PDE attestations, retained
├── unbound_log/           every blocked response, with the offending slot
├── breach_log/            IPS-box exits with sequence numbers
└── replay/                any client-facing figure, reconstructed from seq
```

### 12.1 The One Interface That Matters

```rust
// The only way any tier obtains a sensitivity.
// There is no second implementation. That is the entire point.

pub trait GreekSource {
    fn vector(&self, id: InstrumentId, seq: Seq) -> Result<GreekVector, SourceError>;
    fn implied_vol(&self, id: InstrumentId, px: Price, seq: Seq) -> Result<Vol, SourceError>;
    fn attest(&self, v: &GreekVector) -> Attestation;   // PDE residual + window flag
}

// D1, D2, D3 and D4 depend on this trait and nothing else numeric.
// A tier that constructs its own pricer fails the build.
```

---

## 13 · Seven Predictions

**P1 · Suitability migrates into Greek space by 2029.**
Investment policy statements will carry dollar-Greek bounds alongside or instead of asset-class ranges, because a box in (Δ$, Γ$, ν$, Θ$) is the only representation that the advisor, the optimiser, the hedger and the margin engine can all evaluate.
*Falsifier:* a mature platform in 2029 whose advisory constraints still resolve to asset-class weights and cannot be checked by its hedging engine without translation.

**P2 · Substantiation becomes the examined artifact, not the output.**
Following the March 2024 fact pattern — a firm unable to produce documents to support its claims — supervision will centre on whether a figure can be reconstructed from a named record at a named sequence. Firms that log responses will fail; firms that bind numerals will pass.
*Falsifier:* three years of AI-related adviser examinations that never request per-figure provenance.

**P3 · Margin-aware hedging becomes standard, and it will be sold as alpha.**
Once a desk can price `ΔIM` on the same vectors it prices `Δ$`, the joint objective dominates. On overlay books the margin saving exceeds the management fee. Expect it marketed as "capital-efficient overlay" rather than as margin optimisation, because that is where the fee is.
*Falsifier:* hedge selectors still scoring on delta and premium alone through 2028 at firms with in-house margin replication.

**P4 · The in-kind channel moves inside the optimiser.**
Redemption-basket construction stops being a settlement step and becomes a decision variable, driven by the gap between a ~0.95% net tax alpha and a 25–35 bp management fee. Creation-unit size will start being chosen for tax granularity, not only for market-making convenience.
*Falsifier:* leading active ETF complexes still constructing redemption baskets pro-rata-with-exceptions in 2029.

**P5 · Margin replication becomes a competitive product.**
SPAN 2's move from 16 scenarios to thousands, with parameter files near 20 GB, put in-house margin reproduction out of reach for most participants. That gap is a market. Expect it filled by hardware-backed replication services rather than by larger CPU farms, because the constraint is cycle deadline, not aggregate throughput.
*Falsifier:* a commodity software margin-replication offering achieving CCP-matching accuracy across targeted portfolios by 2028.

**P6 · The published figure of merit stops being latency.**
When one lane clears the firm's entire daily Greek demand in 31 seconds, nanoseconds stop differentiating. Vendors will move to tail-latency percentiles, jitter bounds, and attestation coverage — the properties that are actually scarce.
*Falsifier:* median latency remaining the headline metric on wealth-platform risk infrastructure through 2029.

**P7 · Model-agnostic seeding becomes a procurement requirement.**
April 2020 demonstrated that log-space is a listing convention, not a law. Platforms will be required to show a working non-log path before onboarding commodity or rates exposure, and `model_id` will appear in position records the way currency codes do now.
*Falsifier:* a negative-underlying episode after 2026 in which log-only platforms experience no material outage.

---

## 14 · Notation

| Symbol | Meaning |
|:--|:--|
| **D0–D4** | Clock domains: source, hedging, collateral, rebalancing, advisory |
| **SPR** | Single Position Record — immutable, append-only, sequence-keyed |
| `as_of_seq` | Gray-coded monotone sequence number; two figures with different values are not comparable |
| **IPS box** | Suitability as inequalities in (Δ$, Γ$, ν$, Θ$, ρ$) |
| **Δ$** | Dollar delta per 1% underlying move |
| **Γ$** | Dollar gamma per 1% move squared |
| **ν$** | Dollar vega per one volatility point |
| **Θ$** | Dollar theta per calendar day |
| **ΔIM** | Change in initial margin attributable to a candidate hedge |
| **Parallax residual** | measured exposure (D0) − intended exposure (D4 box) |
| **Unbound numeral** | A figure in a generated response with no (record_id, seq) binding — blocked |
| Q24.40 ULP | 9.094947e−13 |
| HELICON traversal | 372.5 ns at 400 MHz, N=40, radix-2 |
| One lane | 4.00 × 10⁸ Greek vectors / second |

---

## 15 · Lineage

- **c. 100 BCE** — the Antikythera mechanism. At least thirty hand-cut gears; sun, moon, phase, Metonic cycle and eclipse timing from one crank. Recovered 1901; understood decades later. Multi-rate, deterministic, and consistent by construction.
- **1900 / 1973** — Bachelier's arithmetic Brownian motion; Black–Scholes and Merton's lognormal successors. One of these needs a logarithm and one does not, and the difference became operational in a week in April 2020.
- **1956–1971** — Volder's coordinate rotation at Convair; Walther's unification into three geometries. The shift-and-add substrate underneath D0.
- **IRC §852(b)(6)** — in-kind redemption recognises no gain at the fund. The mechanism the entire ETF tax argument rests on.
- **Sept 2016** — ISDA SIMM launches under the BCBS–IOSCO margin framework. Delta, Vega and Curvature as literal margin inputs, above an €8bn aggregate average notional.
- **25 Sept 2019** (eff. 23 Dec 2019) — SEC Rule 6c-11. Custom baskets for transparent active ETFs; daily transparency; six-year records.
- **2020** — *An Empirical Evaluation of Tax-Loss-Harvesting Alpha*, Financial Analysts Journal 76(3), 99–108. CRSP top-500, 1926–2018: 1.08% before costs, 0.82% wash-sale constrained.
- **8 / 21 April 2020** — CME Clearing Advisories 20-152 and 20-171, "Switch to Bachelier Options Pricing Model," effective 22 April. ICE followed. Negative strikes to −50 traded on the June future.
- **2021 →** — CME SPAN 2 / HVaR. 16 scenarios become thousands; ten-year lookback; vol surface including skew as risk factors; parameter files to ~20 GB.
- **18 March 2024** — SEC settles its first AI-related adviser actions. $400,000 total. Advisers Act §206(2), §206(4), Rules 206(4)-1 and 206(4)-7. The finding that mattered: no documents to substantiate.
- **April 2025** — OPRA one-millisecond bursts exceed 23.7 million packets per second, over 187 million messages per second.
- **Jan / Feb 2026** — OPRA peak throughput 50.9 and 53.5 million messages per second.
- **17 March 2026** — SEC Exchange Act §36 relief supplementing 1940 Act ETF share-class relief for multi-class funds.
- **Q2 2026** — 0DTE reaches a record 65% of SPX volume; SPX 0DTE ADV 3.1M contracts; market-wide 0DTE above 20M contracts/day; total US listed options average 72.8M contracts/day, the largest quarter recorded.
- **27 July 2026** — SEC Division of Investment Management no-action letter to ICI on creation baskets during passive concentration exceedances, extended explicitly to actively managed ETFs.

---

*Four consumers. One register. Every client-facing figure carries the sequence number it was derived from, and every sensitivity carries the residual that attests it. The gear train is the product; the pointers are just where you read it.*
