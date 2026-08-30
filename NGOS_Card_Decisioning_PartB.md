# Card Decisioning Scorecard — Part B

**How the CIBIL score (Part A) plus profile data decide which cards a customer qualifies for, in what order, and at what limit.**

Replaces the deck's single `> 750` gate with per-card eligibility thresholds, a ranked recommendation list, and a derived credit limit. Directly answers the open design questions in the Context Pack §6: bands instead of one cutoff, tier-sensitive thresholds, NGOS-derived limits, and auditable decision inputs.

---

## 1  The five cards — tier hierarchy

Cards are ranked by tier. The decisioning engine walks from highest tier down; every card whose gates pass is included in the ranked output.

| Tier | Card | Annual Fee | Target Segment | Headline Perk |
|------|------|-----------|----------------|---------------|
| 5 | **Ultimate** | ₹5,000 | High-income, pristine credit | 3.33% rewards, unlimited lounge, concierge, golf |
| 4 | **Platinum** | ₹2,000 | Established professionals | 5× dining rewards, 8 lounge visits/yr, purchase protection |
| 3 | **EaseMyTrip** | ₹1,500 | Frequent travellers | 20% off flights/hotels, air miles, 4 lounge visits/yr |
| 2 | **Rewards** | ₹750 | Everyday spenders | 5× online rewards, movie & dining offers |
| 1 | **Smart** | ₹0 (LTF) | New-to-credit / value seekers | 2% cashback on online, fuel surcharge waiver |

---

## 2  Eligibility gates — per card

Every gate must pass. A single failure on any gate means the card is **not eligible**.

| Gate | Ultimate | Platinum | EaseMyTrip | Rewards | Smart |
|------|----------|----------|------------|---------|-------|
| Min CIBIL score | 800 | 750 | 725 | 700 | 650 |
| Min monthly income | ₹2,00,000 | ₹1,00,000 | ₹75,000 | ₹50,000 | ₹25,000 |
| Max FOIR | 40% | 45% | 50% | 55% | 60% |
| Min payment status | NEVER_LATE | DPD_1_30 | DPD_1_30 | DPD_31_60 | DPD_61_90 |
| Min credit age (months) | 48 | 36 | 24 | 12 | 6 |

**Universal hard knockouts** (checked before any card gate):

- `is_defaulter = true` → **auto-decline all cards**
- `payment_status = DEFAULT` → **auto-decline all cards**
- Age < 21 or > 60 → **auto-decline all cards** (derived from `date_of_birth`)

### 2.1  Payment status ordering

The gate check compares ranks, not strings. A user's rank must be **≥** the card's minimum rank.

| Status | Rank | Meaning |
|--------|------|---------|
| `NEVER_LATE` | 4 | No delinquency ever |
| `DPD_1_30` | 3 | Worst delinquency was 1–30 days past due |
| `DPD_31_60` | 2 | Worst delinquency was 31–60 days past due |
| `DPD_61_90` | 1 | Worst delinquency was 61–90 days past due |
| `DEFAULT` | 0 | Hard default — universal knockout |

### 2.2  FOIR computation

Reuses the Part A Factor 2 formula exactly:

```
active_emi = SUM(monthly_emi) WHERE months_remaining >= 6
FOIR = active_emi / monthly_income × 100
```

Obligations with fewer than 6 months remaining are excluded — they are about to vanish and should not penalise the applicant.

---

## 3  Comfort score — how well the user fits each eligible card

For every card that passes all gates, a **comfort score** (0–100) measures how far above the minimums the user sits. Higher = more comfortable approval, lower risk.

### 3.1  Four components

| Component | Max pts | Formula |
|-----------|---------|---------|
| CIBIL margin | 35 | `floor((score - card_min_score) / (900 - card_min_score) × 35)` |
| Income margin | 30 | `min(30, floor((income - card_min_income) / card_min_income × 30))` |
| FOIR margin | 20 | `min(20, floor((card_max_foir - user_foir) / card_max_foir × 20))` |
| History bonus | 15 | `NEVER_LATE → 15, DPD_1_30 → 10, DPD_31_60 → 5, DPD_61_90 → 0` |

**Comfort = sum of all four components.**

### 3.2  Comfort bands

| Range | Label | Meaning |
|-------|-------|---------|
| 75–100 | ★★★ Strong Fit | Comfortably exceeds all gates |
| 50–74 | ★★ Good Fit | Meets gates with healthy margin |
| 25–49 | ★ Marginal | Barely qualifies on one or more gates |
| 0–24 | ⚠ Borderline | Approved but high risk of limit strain |

---

## 4  Credit limit derivation

For each eligible card, NGOS derives a credit limit rather than relying on CCMS.

### 4.1  Formula

```
disposable   = monthly_income - active_emi
base_limit   = disposable × card_multiplier
credit_limit = floor(base_limit / 25000) × 25000      -- round down to nearest ₹25,000
credit_limit = clamp(credit_limit, card_floor, card_ceiling)
```

### 4.2  Per-card parameters

| Card | Multiplier | Floor | Ceiling |
|------|-----------|-------|---------|
| Ultimate | 4.0× | ₹3,00,000 | ₹25,00,000 |
| Platinum | 3.0× | ₹2,00,000 | ₹15,00,000 |
| EaseMyTrip | 2.5× | ₹1,50,000 | ₹10,00,000 |
| Rewards | 2.0× | ₹75,000 | ₹5,00,000 |
| Smart | 1.5× | ₹25,000 | ₹3,00,000 |

If `base_limit` falls below the card's floor after rounding, the card is **still eligible** but the limit is set to the floor. The floor represents the minimum viable credit line for that product.

---

## 5  Output structure

The decisioning engine returns:

```
{
  "pan": "ABCPR1234F",
  "cibil_score": 845,
  "foir_pct": 25.00,
  "decision_timestamp": "2026-08-30T14:00:00Z",
  "eligible_cards": [
    {
      "rank": 1,
      "card": "PLATINUM",
      "tier": 4,
      "comfort_score": 57,
      "comfort_band": "GOOD_FIT",
      "credit_limit": 300000,
      "gate_margins": {
        "cibil": "+95 above 750",
        "income": "+40000 above 100000",
        "foir": "25.00% vs 45% cap",
        "payment": "NEVER_LATE (req: DPD_1_30)",
        "credit_age": "134 vs 36 min"
      }
    }
  ],
  "declined_cards": [
    {
      "card": "ULTIMATE",
      "tier": 5,
      "failed_gates": ["income: 140000 < 200000"]
    }
  ]
}
```

This structure satisfies the Context Pack's auditability requirement: every decision input is persisted alongside the outcome.

---

## 6  Source columns — what Part B uses

| Column | Source | Used for |
|--------|-------|----------|
| CIBIL score | Part A output | Eligibility gate, comfort score |
| monthly_income | `bureau.profiles` | Eligibility gate, FOIR denominator, credit limit, comfort score |
| payment_status | `bureau.profiles` | Eligibility gate, comfort score history bonus |
| credit_age_months | `bureau.profiles` | Eligibility gate |
| date_of_birth | `bureau.profiles` | Age knockout (21–60) |
| monthly_emi | `bureau.obligations` | FOIR numerator, credit limit |
| months_remaining | `bureau.obligations` | FOIR filter (≥ 6 months only) |
| is_defaulter | `bureau.profiles` | Universal knockout |

No new columns required beyond Part A.

---

## 7  Dry run A — Rahul, ABCPR1234F (CIBIL 845)

```
PROFILE
  monthly_income    140000    payment_status    NEVER_LATE
  credit_age_months 134       CIBIL score       845
  age               34        is_defaulter      false

OBLIGATIONS (active, months_remaining >= 6)
  HOME_LOAN   emi 35000   months_remaining 96

FOIR = 35000 / 140000 = 25.00%
```

### Eligibility check

```
ULTIMATE   CIBIL 845 >= 800 ✓  income 140000 >= 200000 ✗  FAIL (income)
PLATINUM   CIBIL 845 >= 750 ✓  income 140000 >= 100000 ✓  FOIR 25% <= 45% ✓
           payment NEVER_LATE >= DPD_1_30 ✓  age 134 >= 36 ✓         PASS
EASEMYTRIP CIBIL 845 >= 725 ✓  income 140000 >= 75000 ✓   FOIR 25% <= 50% ✓
           payment NEVER_LATE >= DPD_1_30 ✓  age 134 >= 24 ✓         PASS
REWARDS    CIBIL 845 >= 700 ✓  income 140000 >= 50000 ✓   FOIR 25% <= 55% ✓
           payment NEVER_LATE >= DPD_31_60 ✓ age 134 >= 12 ✓         PASS
SMART      CIBIL 845 >= 650 ✓  income 140000 >= 25000 ✓   FOIR 25% <= 60% ✓
           payment NEVER_LATE >= DPD_61_90 ✓ age 134 >= 6 ✓          PASS
```

### Comfort scores

```
PLATINUM     CIBIL floor((845-750)/(900-750)×35) = floor(22.17) = 22
             Income min(30, floor((140000-100000)/100000×30)) = min(30,12) = 12
             FOIR  min(20, floor((45-25)/45×20)) = min(20,8)   = 8
             History NEVER_LATE                                 = 15
             TOTAL                                              = 57  ★★ Good Fit

EASEMYTRIP   CIBIL floor((845-725)/(900-725)×35) = floor(24.0) = 24
             Income min(30, floor((140000-75000)/75000×30))  = min(30,26) = 26
             FOIR  min(20, floor((50-25)/50×20))  = min(20,10)  = 10
             History NEVER_LATE                                  = 15
             TOTAL                                               = 75  ★★★ Strong Fit

REWARDS      CIBIL floor((845-700)/(900-700)×35) = floor(25.37) = 25
             Income min(30, floor((140000-50000)/50000×30))  = min(30,54) = 30
             FOIR  min(20, floor((55-25)/55×20))  = min(20,10)  = 10
             History NEVER_LATE                                  = 15
             TOTAL                                               = 80  ★★★ Strong Fit

SMART        CIBIL floor((845-650)/(900-650)×35) = floor(27.30) = 27
             Income min(30, floor((140000-25000)/25000×30))  = min(30,138)= 30
             FOIR  min(20, floor((60-25)/60×20))  = min(20,11)  = 11
             History NEVER_LATE                                  = 15
             TOTAL                                               = 83  ★★★ Strong Fit
```

### Credit limits

```
disposable = 140000 - 35000 = 105000

PLATINUM     105000 × 3.0 = 315000  → floor to 25k = 300000  clamp(200000,1500000) = ₹3,00,000
EASEMYTRIP   105000 × 2.5 = 262500  → floor to 25k = 250000  clamp(150000,1000000) = ₹2,50,000
REWARDS      105000 × 2.0 = 210000  → floor to 25k = 200000  clamp(75000,500000)   = ₹2,00,000
SMART        105000 × 1.5 = 157500  → floor to 25k = 150000  clamp(25000,300000)   = ₹1,50,000
```

### Final ranked output

```
Rank  Card          Comfort  Band          Limit
 1    Platinum      57       ★★ Good       ₹3,00,000
 2    EaseMyTrip    75       ★★★ Strong    ₹2,50,000
 3    Rewards       80       ★★★ Strong    ₹2,00,000
 4    Smart         83       ★★★ Strong    ₹1,50,000

Declined: Ultimate (income ₹1,40,000 < ₹2,00,000 required)

Best recommendation: Platinum
```

Rahul scores 845 — excellent credit — but his income (₹1.4L/month) locks him out of Ultimate. Platinum is his highest eligible tier. His comfort score there (57) is solid: he is not stretching, just doesn't have the income ceiling for the top card.

---

## 8  Dry run B — Meena, BXYPM5678K (CIBIL 740)

```
PROFILE
  monthly_income    65000     payment_status    DPD_31_60
  credit_age_months 65        CIBIL score       740
  age               ~32       is_defaulter      false

OBLIGATIONS (active, months_remaining >= 6)
  AUTO_LOAN     emi 9000    months_remaining 36
  PERSONAL_LOAN emi 18000   months_remaining 4   <-- excluded (< 6)

FOIR = 9000 / 65000 = 13.85%
```

### Eligibility check

```
ULTIMATE   income 65000 < 200000 ✗                                    FAIL
PLATINUM   CIBIL 740 < 750 ✗                                          FAIL
EASEMYTRIP income 65000 < 75000 ✗                                     FAIL
REWARDS    CIBIL 740 >= 700 ✓  income 65000 >= 50000 ✓  FOIR 13.85% <= 55% ✓
           payment DPD_31_60 >= DPD_31_60 ✓  age 65 >= 12 ✓           PASS
SMART      CIBIL 740 >= 650 ✓  income 65000 >= 25000 ✓  FOIR 13.85% <= 60% ✓
           payment DPD_31_60 >= DPD_61_90 ✓  age 65 >= 6 ✓            PASS
```

### Comfort scores

```
REWARDS      CIBIL floor((740-700)/(900-700)×35) = floor(7.0)  = 7
             Income min(30, floor((65000-50000)/50000×30)) = min(30,9) = 9
             FOIR  min(20, floor((55-13.85)/55×20)) = min(20,14)  = 14
             History DPD_31_60                                     = 5
             TOTAL                                                 = 35  ★ Marginal

SMART        CIBIL floor((740-650)/(900-650)×35) = floor(12.60) = 12
             Income min(30, floor((65000-25000)/25000×30)) = min(30,48) = 30
             FOIR  min(20, floor((60-13.85)/60×20)) = min(20,15)  = 15
             History DPD_31_60                                     = 5
             TOTAL                                                 = 62  ★★ Good Fit
```

### Credit limits

```
disposable = 65000 - 9000 = 56000

REWARDS    56000 × 2.0 = 112000  → floor to 25k = 100000  clamp(75000,500000) = ₹1,00,000
SMART      56000 × 1.5 = 84000   → floor to 25k = 75000   clamp(25000,300000) = ₹75,000
```

### Final ranked output

```
Rank  Card      Comfort  Band         Limit
 1    Rewards   35       ★ Marginal   ₹1,00,000
 2    Smart     62       ★★ Good      ₹75,000

Declined: Ultimate (income), Platinum (CIBIL), EaseMyTrip (income)

Best recommendation: Rewards (with marginal comfort — consider Smart for lower risk)
```

Under the old single `> 750` gate, Meena was flatly DECLINED. The multi-card system gives her two options: Rewards at a marginal comfort (she is right at the payment-status floor), or Smart as a comfortable fit. The comfort score surfaces the risk: a relationship manager might steer her toward Smart given the DPD_31_60 history.

---

## 9  Dry run C — Priya, CPXYS9012L (CIBIL 860, high income)

```
PROFILE
  monthly_income    350000    payment_status    NEVER_LATE
  credit_age_months 96        CIBIL score       860
  age               38        is_defaulter      false

OBLIGATIONS (active, months_remaining >= 6)
  HOME_LOAN   emi 65000   months_remaining 180
  AUTO_LOAN   emi 22000   months_remaining 48

FOIR = (65000 + 22000) / 350000 = 87000 / 350000 = 24.86%
```

### Eligibility check

```
ULTIMATE   CIBIL 860 >= 800 ✓  income 350000 >= 200000 ✓  FOIR 24.86% <= 40% ✓
           payment NEVER_LATE >= NEVER_LATE ✓  age 96 >= 48 ✓         PASS
PLATINUM   all gates pass                                              PASS
EASEMYTRIP all gates pass                                              PASS
REWARDS    all gates pass                                              PASS
SMART      all gates pass                                              PASS
```

### Comfort scores

```
ULTIMATE     CIBIL floor((860-800)/(900-800)×35) = floor(21.0) = 21
             Income min(30, floor((350000-200000)/200000×30)) = min(30,22) = 22
             FOIR  min(20, floor((40-24.86)/40×20)) = min(20,7)  = 7
             History NEVER_LATE                                   = 15
             TOTAL                                                = 65  ★★ Good Fit

PLATINUM     CIBIL floor((860-750)/(900-750)×35) = floor(25.67) = 25
             Income min(30, floor((350000-100000)/100000×30)) = min(30,75) = 30
             FOIR  min(20, floor((45-24.86)/45×20)) = min(20,8)  = 8
             History NEVER_LATE                                   = 15
             TOTAL                                                = 78  ★★★ Strong Fit

EASEMYTRIP   CIBIL 27  Income 30  FOIR 10  History 15            = 82  ★★★ Strong Fit
REWARDS      CIBIL 28  Income 30  FOIR 10  History 15            = 83  ★★★ Strong Fit
SMART        CIBIL 29  Income 30  FOIR 11  History 15            = 85  ★★★ Strong Fit
```

### Credit limits

```
disposable = 350000 - 87000 = 263000

ULTIMATE     263000 × 4.0 = 1052000  → floor to 25k = 1050000  clamp(300000,2500000) = ₹10,50,000
PLATINUM     263000 × 3.0 = 789000   → floor to 25k = 775000   clamp(200000,1500000) = ₹7,75,000
EASEMYTRIP   263000 × 2.5 = 657500   → floor to 25k = 650000   clamp(150000,1000000) = ₹6,50,000
REWARDS      263000 × 2.0 = 526000   → floor to 25k = 525000   clamp(75000,500000)   = ₹5,00,000
SMART        263000 × 1.5 = 394500   → floor to 25k = 375000   clamp(25000,300000)   = ₹3,00,000
```

### Final ranked output

```
Rank  Card          Comfort  Band          Limit
 1    Ultimate      65       ★★ Good       ₹10,50,000
 2    Platinum      78       ★★★ Strong    ₹7,75,000
 3    EaseMyTrip    82       ★★★ Strong    ₹6,50,000
 4    Rewards       83       ★★★ Strong    ₹5,00,000
 5    Smart         85       ★★★ Strong    ₹3,00,000

Declined: none

Best recommendation: Ultimate
```

Priya qualifies for every card. Ultimate at comfort 65 means she is a solid fit — not scraping in — with a ₹10.5L limit. The Rewards ceiling (₹5L) and Smart ceiling (₹3L) cap her limits on lower tiers, so the higher-tier cards genuinely serve her better.

---

## 10  Edge cases and modelling notes

| Note | Detail |
|------|--------|
| **Comfort ≠ ranking** | Comfort is always highest for the lowest-tier card (the user is most overqualified there). Ranking is by tier, not comfort. Comfort tells the relationship manager *how safe* an approval is, not which card to push. |
| **All cards declined** | If no card passes, the response carries `eligible_cards: []` and the decline reason from the highest-tier card that came closest. The email template uses reason code `DECLINED_ALL`. |
| **Tie on FOIR boundary** | `FOIR = card_max_foir` exactly → gate **passes** (≤, not <). This is deliberate: the boundary is inclusive, and the comfort score will be 0 on the FOIR component, flagging it as tight. |
| **is_defaulter vs DEFAULT** | Both are universal knockouts. If they disagree (noted in Part A §6), treat `is_defaulter = true` OR `payment_status = DEFAULT` as a knockout — either one is sufficient. |
| **Credit limit below floor** | When `base_limit` rounds below the card's floor, the limit is set to the floor. The card is still eligible — the floor is the minimum viable credit line for the product. |
| **Rewards card ceiling hit** | Priya's Rewards limit formula yields ₹5,25,000, capped to the ₹5,00,000 ceiling. This is expected: high-income users hit ceilings on lower-tier cards, reinforcing that the higher-tier card is the better fit. |
| **Manager override** | A manager can override a gate failure and force-approve a card. The override is logged with the manager's ID, reason text, and the original failed-gate detail. Overrides bypass eligibility but not universal knockouts. |

---

## 11  Design decisions log

These choices directly address the Context Pack §6 open questions:

| Context Pack Question | Decision |
|----------------------|----------|
| One cutoff, or bands? | Five per-card cutoffs. No single gate. Bands range from CIBIL 650 (Smart) to 800 (Ultimate). |
| Boundary semantics? | All gates are inclusive (≥ for minimums, ≤ for maximums). A CIBIL score of exactly 750 qualifies for Platinum. |
| Does tier change the threshold? | Yes. Each card has its own threshold table (§2). |
| Who decides the credit limit? | NGOS derives it (§4). Formula: `(income - active_emi) × multiplier`, clamped to card floor/ceiling. |
| Is the decision auditable? | Yes. Output structure (§5) persists all inputs: score, income, FOIR, thresholds in force, gate margins, and the ranked result. |
| Can a manager override? | Yes, with logged justification (§10). Overrides bypass eligibility gates but not universal knockouts. |
