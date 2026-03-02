# Salary & Negotiation for AI Engineers

Market reality: **AI engineers are paid well but must negotiate.**

Companies anchor low. If you accept their first offer, you leave $20K-$100K on the table.

---

## Current Market Rates (2025)

By level and location:

| Level | SF Bay | NYC | Remote | UK London | Germany | India |
|-------|--------|-----|--------|-----------|---------|-------|
| **Junior (0-2yr)** | $180K | $150K | $140K | £90K | €75K | ₹25L |
| **Mid (2-5yr)** | $250K | $200K | $180K | £130K | €110K | ₹40L |
| **Senior (5yr+)** | $350K+ | $280K+ | $220K+ | £180K+ | €150K+ | ₹60L+ |

(Total comp = salary + bonus + equity)

### What's Included in "Total Comp"

- **Salary:** Base pay (direct, guaranteed)
- **Bonus:** 15-30% of salary (if company performs)
- **Stock/Equity:** Company shares vesting over 4 years (usually)
- **Sign-on:** If from competitor ("golden handcuffs" for switching)
- **Benefits:** Health, 401K match, unlimited PTO (worth ~$20K/year)

**Example (Mid-level SoFo):**
```
Salary: $180K
Bonus: $30K (15-20% = typical)
Equity: $80K/year (stock grant / 4 years)
Sign-on: $50K (if switching)
Benefits: $25K (healthcare, 401K, perks)
------
Total 1st year: $365K ← This is what you negotiate
```

---

## Evaluating an Offer Beyond Salary

**Scoring Matrix:**

| Factor | Weight | Good | Medium | Bad |
|--------|--------|------|--------|-----|
| **Compute Budget** | 10% | GPU access, train big models | CPU only, shared cloud | No clear budget |
| **Data Access** | 10% | Real production data | Anonymized, old data | No data, blocked |
| **Team Quality** | 20% | 3+ senior AI engineers | 1 mid + juniors | Only you, or non-technical |
| **Learning Pace** | 15% | New tech constantly, talks | Stable, occasional learning | Stagnant |
| **Equity Position** | 15% | Pre-IPO, likely to 10x+ | Public company, steady | Post-IPO dilution |
| **Remote Flexibility** | 10% | Full remote, async-first | 3 days office | 5 days in office |
| **Title Growth** | 10% | Clear path to principal | Stuck at mid | Blocked |
| **Salary/Equity** | 10% | $300K+ or 0.1%+ | $200K or 0.05% | $150K or 0.01% |

Score each factor. Aim for total >75%.

**Example:**
- High-paying but small team + no equity = risky (75% salary, 0% upside)
- Moderate pay but killer team + equity = better long-term (50% salary, 50% growth)

---

## How to Negotiate Without Burning Bridges

### Step 1: Get Offer in Writing

Never verbally accept. Always say: "Let me review the written offer."

### Step 2: Do Your Research

- **Levels.fyi:** see actual offers from companies ← most reliable
- **Glassdoor:** salary ranges
- **Blind:** anonymous posts (heavily AI nerd heavy, ranges can be inflated)
- **Your network:** ask other engineers (many share)

Expected range for your level: $XXK - $XXK

### Step 3: Respond to Offer

**Email template:**

```
Hi [Hiring Manager],

Thank you for the offer. I'm excited about the role and team.

Before I move forward, I wanted to discuss the compensation. Based on market 
research (Levels.fyi, conversations with others at your level), the range for 
my experience is typically $XXK - $XXK base.

My current offer is $100K, which is below market. Can we discuss adjusting to $150K?

Separately, I noticed the equity grant is 0.02%. I'd like to understand the 
vesting schedule and how that compares to [Peer Company] at 0.05%.

Happy to discuss further. I'm enthusiastic about joining.

Thanks,
[Your Name]
```

### Step 4: Get Competing Offers (If Possible)

If you have 2+ offers: **use the other one as leverage.**

```
"I have competing offers at $250K and 0.05% equity. I prefer your company 
because of the team. Can you match the competing offer?"
```

**Be honest.** If you don't actually have another offer, don't make one up. Recruiters will call your bluff.

### Step 5: What's Actually Negotiable?

| Item | Negotiable? |
|------|---|
| Salary | ✅ Usually 10-20% movement |
| Sign-on bonus | ✅ Easier to move than salary |
| Equity | ✅ Modest movement (0.02% → 0.03%) |
| Start date | ✅ Usually flexible |
| Remote flexibility | ✅ If you have leverage |
| PTO | ❌ Rarely negotiates (usually "unlimited") |
| Title | ⚠️ Possible but rarely matters |
| Relocation package | ✅ But lower priority (most remote now) |

### Step 6: When They Say "That's Our Max"

**It rarely is.**

Response:
```
"I understand that's your initial ceiling. Given my background in [X], 
I think I'm a strong fit for this role. What if we structure it: 
- $150K salary 
- $30K sign-on (deferred risks if I don't work out)
- 0.03% equity

Does that work better?"
```

Offering flexibility (deferred pay, performance bonuses) sometimes unblocks.

---

## Red Flags in Job Offers

❌ **"We're an AI company but no GPU compute."**
→ How do they train models? Outsourcing? Sketchy.

❌ **"Equity is 0.005%"**
→ At typical valuations, worth <$500/year. Borderline insult. Push back.

❌ **"We'll train you on AI" (but hiring "Senior AI Engineer")**
→ Role description doesn't match. Will you be training or building?

❌ **"No written comp agreement, we'll email details"**
→ Never do this. Always get offer in writing.

❌ **"Equity fully vests in 1 year"**
→ Unusual. Usually 4 years (1 year cliff). Maybe they're unstable?

❌ **"We own your AI code outside of work"**
→ Aggressive IP assignment. Push back (most companies allow your side projects).

---

## Green Flags

✅ **"GPU budget per engineer is $10K/quarter"**
→ Clear compute investment. They respect deep learning.

✅ **"We use MLflow, DVC, W&B internally"**
→ Mature ML ops. Not flying blind.

✅ **"Last 3 promotions happened within 18 months"**
→ Clear growth path.

✅ **"Here's our model evaluation infrastructure"**
→ They understand importance of measurement. Professional.

✅ **"Salary is $X, bonus is $Y, equity is $Z/year, cliff is 1yr, vest over 4yrs"**
→ Complete transparency. This is professional.

✅ **"You can do side projects as long as no IP conflict"**
→ Trusts employees, wants them to grow.

---

## Equity: Understanding Vesting & Valuation

### Vesting Schedule (Standard)

```
Grant: 0.1% equity (16,000 shares if company has 16M shares)
Standard schedule:
- 1 year cliff: 0 shares for 12 months, then 4K shares (25%)
- Then 3 years: 4K additional shares per year

Year 1: 4K vested
Year 2: 8K vested
Year 3: 12K vested
Year 4: 16K vested

If you leave after 2 years: only 8K shares. Cliff means you get $0 if you leave before 1 year.
```

### Equity Value Estimation

```
Shares vesting per year: 4K
Share price per strike: $1 (most early-stage)
Company valuation: $100M (example)
Per-share value: $100M / 16M shares = $6.25

Annual equity value: 4K shares * $6.25 = $25K/year

BUT: It's paper value until IPO/acquisition.
```

**Harsh truth:** Most startups fail → equity = $0.

Negotiate with assumption: "This equity is a lottery ticket. Salary is the real comp."

### When Does Equity Matter?

- Pre-IPO unicorns (Stripe, Databricks, etc.): could be $1M+ if they go public
- Series C/D startups: likely, but 40% chance failure → $0
- Post-IPO companies: worth something, but diluted

---

## Salaries by Company Type

| Company Type | Salary | Equity | Total Year 1 |
|--------------|--------|--------|--------------|
| **FAANG** (Google, Meta, Apple, Amazon, Netflix) | $180-250K | 0.05-0.1% | $250-350K |
| **High-growth startup** (Stripe, Databricks, Anthropic) | $200-300K | 0.05-0.2% | $250-400K |
| **Series B-C AI startup** | $150-200K | 0.1-0.5% | $200-300K |
| **Seed-stage startup** | $100-150K | 0.5-2% | $100-200K |
| **Scaleup** (unicorn+) | $220-280K | lottery | $230-400K |
| **Consulting** (Boston Consulting, McKinsey) | $150-200K | low/none | $150-250K |

---

## The Negotiation Timeline

```
Day 1: Receive offer
Day 1-2: Do research (Levels.fyi, network, competitors)
Day 2-3: Counter-offer email
Day 3-5: Back-and-forth
Day 5-6: Final offer + acceptance
Day 6-30: Notice period at current job + prep
```

**Never rush this.** Take your time. A $20K difference is 2 months of your life before it "breaks even."

---

## Red Flags in Negotiation

If company **demands** immediate decision:
- "It's a hot offer, someone else wants it"
- "We need an answer today"

**These are negotiation tactics.** Reasonable companies give 1-2 weeks to decide.

Counter: "I'm very interested, but I need a week to review and discuss with mentors. I'll have a decision by [date]."

Most will extend.

---

## Post-Offer: Final Checks

Before you sign:

1. **Verify equity terms:** What are strike price, vesting, cliff, expiration?
2. **IP agreement:** Do you own side projects?
3. **Non-compete:** Does it restrict your next job?
4. **Relocation:** Who pays? How much help?
5. **Start benefits:** Health insurance start date (gap coverage)?
6. **Sign-on tax:** Is it structured to minimize taxes?

Read the fine print. Ask lawyer if something seems off.

---

## Final Negotiation Tips

✅ **Do:**
- Research market rates (Levels.fyi is gold)
- Get competing offers if possible
- Be polite but firm ("This is market, I want to join your team")
- Offer flexibility if stuck (performance bonus, deferred pay)
- Get everything in writing
- Take time to decide (don't rush)

❌ **Don't:**
- Demand absurd amounts (respect market reality)
- Threaten ("take it or lose me") — they will
- Lie about other offers
- Negotiate after accepting (burns bridges)
- Accept first offer without counter
- Negotiate solely on salary (benefits matter too)

---

## After the Job

Once you've negotiated and accepted:

**You're done.** Don't renegotiate again unless 2+ years have passed (title change, major regrade).

Focus on: **doing excellent work, shipping stuff, getting promoted.**

That's how you get true leverage for next negotiation.

---

## 21. Next: Final Milestone

You've prepared technically and financially.

Now: **final portfolio audit and job search.**

→ **[Phase 5 Milestone →](./milestone.md)**
