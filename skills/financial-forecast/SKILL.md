---
name: financial-forecast
version: 1.2.2
description: Build a complete FIRE / financial forecast for a household by orchestrating the public planfi MCP. Use whenever someone wants a financial plan, FIRE projection, retirement forecast, net-worth projection, "when can I retire", "am I on track", or wants to model savings / retirement-age / spending trade-offs — e.g. "build me a financial plan, I'm 34 making $180k with $250k invested", "when can I retire if I save $4k/mo?", "am I on track to retire at 55?".
---

# Financial Forecast

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp/free).
All math + the 150-year Shiller market dataset live server-side. This skill only gathers inputs
and calls the tools — it does **not** compute anything locally. Read-only; it never changes the
user's data.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__generate_financial_plan`):
`generate_financial_plan`, `start_plan_intake`, `get_completed_plan`, `wait_for_completion`,
`generate_financial_insights`, `generate_action_plan`, `generate_financial_commentary`,
`run_backtesting`, `get_asset_allocation`, `get_savings_variations`, `solve_goal`, `compare_plans`,
`check_model_completeness`, `explain_plan_state`, plus optional `analyze_*` deep-dives. Use whichever
name your environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp/free
```

> **Try free, then add your key.** The command above adds the **free** connector — `https://ai.planfi.app/mcp/free` (no key needed). Once you create an API key, add a **new** connector with the MCP url — `https://ai.planfi.app/mcp` — and authorize it with your key.

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp/free.)

> **Access — free for personal use.** The planfi MCP is free to try (a small monthly allowance, no key needed). Heavy automated abuse forced us to add limits — but it stays **free for personal use**: email **kam@rateapi.dev** and we'll send you a free API key, no charge. (Companies and commercial use have paid plans.) To use a key, pass it as an `Authorization: Bearer pft_…` header in your MCP client config.


## Step 1 — Gather inputs (only two areas are strictly required)

**REQUIRED — ask if not volunteered:**
1. **Each earner's age + annual salary** (household, 1–4 earners).
2. **Stock / investment portfolio**: `current_value` + `monthly_contribution`.

**Forecast-driving inputs you may not have (retirement age, desired spend, SWR, returns, inflation):**
gather them if the conversation makes it natural, but you don't have to chase them down. Anything you
omit is defaulted server-side and reported back in **`assumed_defaults[]`** (each with `field`,
`assumed_value`, `note`). Surface those so the user can correct any silent assumption (see Step 6) —
the server is the source of truth for what was assumed; don't track defaults yourself.

**ASK ONLY IF VOLUNTEERED / RELEVANT (sensible server defaults exist):**
`cash` (value + monthly + APY 0.04), `real_estate[]` (+ `mortgage`), `debts[]`,
`account_balances` split (taxable / traditional / roth — unlocks RMD, withdrawal-sequencing & Roth
analyses), `retirement_accounts` (401k / IRA / HSA contributions + employer match),
`social_security[]`, `children[]`, `life_events[]`, `income_events[]`, `safe_withdrawal_rate`
(0.04), `inflation_rate` (0.03), `salary_growth_rate` (0.03), stock real `annual_return` (0.07).

**Two input paths:**
- **Conversational (default):** gather the numbers in chat, then call `generate_financial_plan`.
- **Form intake (when the user would rather type it themselves):** call **`start_plan_intake`**
  (optionally pre-fill `partial_data` { earner_age, earner_name, plan_name }) → it returns
  `{ session_id, form_url }`. Share the `form_url`. Then retrieve results with
  **`wait_for_completion({ session_id })`** (server-side long-poll, ~25s; returns `complete` with the
  plan — `markdown` + `share_url` + `summary` — or `pending` + `retry_after_seconds` — call again to
  keep waiting). The single-shot non-blocking alternative is **`get_completed_plan({ session_id })`**,
  which additionally returns a reusable **`plan_id`** for chaining (it may be `null` for older
  sessions — fall back to re-sending the model). Honor `retry_after_seconds`; never blind-loop. The
  intake path's handle is `session_id`; to get a `plan_id` for downstream tools, finish with
  `get_completed_plan`.

**Optional kickoff to decide what to ask:** **`check_model_completeness`** (0–100 score + highest-impact
missing fields) and/or **`explain_plan_state`** (ready / blocked / n-a analyses with `missing_fields`).
Call these BEFORE forecasting when the input is sparse.

> **Engine facts to bake in:** all dollars are **today's (real) dollars**; all decimals are
> **fractions** (7% → `0.07`, 3% → `0.03`); `stocks.annual_return` is a **REAL** rate (default 0.07).

## Step 2 — Build the forecast (PRIMARY PATH, produces `plan_id`)

Call **`generate_financial_plan`** with the gathered model:

```
generate_financial_plan({
  name: "Sarah's FIRE Plan",
  earners: [{ age: 38, annual_salary: 140000, retirement_age: 55 }],
  stocks: { current_value: 320000, monthly_contribution: 3000, annual_return: 0.07 },
  cash:   { current_value: 40000,  monthly_contribution: 500 },
  desired_annual_spend: 70000,
  safe_withdrawal_rate: 0.04
})
```

**CAPTURE the returned `plan_id`** — pass `{ plan_id }` to every downstream tool instead of
re-sending the household model (any field you also pass becomes a shallow override). Present the
summary headline metrics — `fire_achieved_age`, `years_to_fire`, `current_fire_pct`,
`current_net_worth`, `final_net_worth`, `portfolio_needed_for_fire`,
`monte_carlo_success_rate_pct` / `monte_carlo_failure_rate_pct` — plus the `share_url`
(planfi.app) so the user can open the full interactive plan.

## Step 3 — Enrich / explain (standard follow-ons, all via `{ plan_id }`)

- **`generate_financial_insights({ plan_id })`** → prioritized, dollar-quantified insights across
  optimization / risk / strategic / progress. Lead the narrative with these.
- **`generate_action_plan({ plan_id })`** → time-boxed steps (immediate / short-term / long-term).
  Best AFTER insights.
- **`generate_financial_commentary({ plan_id })`** → plain-language narrative (where you stand /
  on track / top moves). LLM-backed, not idempotent.
- **`get_asset_allocation({ plan_id })`** → current vs at-retirement mix + 120-minus-age check.
- **`run_backtesting({ portfolio_value, annual_spend, current_age })`** → Shiller 1871-present
  failure rate + worst/best longevity. Stress-test once FIRE is reached. **NOTE:** this is the ONE
  tool that takes raw numbers, not `plan_id` — pull `portfolio_value` / `annual_spend` / `current_age`
  from the plan summary. Its `annual_return` arg is IGNORED (uses historical real returns).

## Step 4 — Trade-offs & goal solving (optional, only when the user asks)

- **`solve_goal({ plan_id, target_fire_age })`** — "can I retire by 50?": returns ranked,
  easiest-first levers (`savings_rate` / `retire_age` / `spend` / `allocation`). `target_fire_age`
  is required (20–100).
- **`compare_plans({ plan_id, scenarios:[...] })`** — head-to-head "retire 55 vs 60", "save more vs
  spend less" (2–4 named scenarios; each `{ label (req), retirement_age?, desired_annual_spend?,
  monthly_contribution?, stock_annual_return?, safe_withdrawal_rate?, primary_annual_salary?, debts? }`,
  only the changed fields).
- **`get_savings_variations({ plan_id })`** — single-axis sensitivity: what if I save 20 / 50 / 80%
  less.
- **`check_model_completeness({ plan_id })`** — flag missing inputs that would sharpen the forecast.

## Step 5 — Deeper specialist analyses (optional, chained via `{ plan_id }`)

Only invoke when the user's question maps to one: **`analyze_roth_conversion`**,
**`analyze_withdrawal_strategy`** (RMD-aware decumulation), **`analyze_healthcare_bridge`** (pre-65
ACA gap), **`analyze_funding_waterfall`** ("next best dollar" for surplus), **`analyze_refinance`**,
**`optimize_social_security`**, **`analyze_mortgage_prepay`**, **`analyze_debt_payoff`**,
**`analyze_education_account`** (529 / Coverdell college funding gap),
**`analyze_childcare_cost`** (childcare cost + second-income tradeoff),
**`analyze_childcare_tax_offsets`** (layers the dependent-care FSA + Child & Dependent Care Credit /
Child Tax Credit onto projected childcare to show net-of-offset cost and a corrected second-income
tradeoff; pairs with `analyze_childcare_cost`), **`analyze_529_optimization`**
(model a $35k 529→Roth rollover (SECURE 2.0, no MAGI phaseout) + a 5-year gift-tax-averaged
superfunding election — the "what if my kid doesn't need it all" + HNW education/estate move; pairs
with `analyze_education_account` and `analyze_estate_exposure`),
**`analyze_education_credits`** (federal education **tax credits** — AOTC vs Lifetime Learning
Credit per student, MAGI phase-out, refundable/non-refundable split, and the $4k 529 carve-out to
preserve full AOTC; see the dedicated trigger section below), **`analyze_estate_exposure`**
(estate-tax exposure; the superfunding gift removes assets from the taxable estate. Shared engine
`estate-tax-calculator.ts` now models BOTH the 6 state **inheritance** taxes — PA/NJ/KY/IA/MD/NE,
which hit beneficiaries by relationship class (`spouse`/`lineal`/`sibling`/`other`) regardless of
estate size, so a PA/NJ household no longer gets a wrong $0 — AND broadened **estate**-tax-state
coverage of ~12 states (MA/NY/OR/WA + CT/DC/HI/IL/ME/MD/MN/RI/VT; MD double-hits with both). Pass
`beneficiary_relationship` (default `lineal`); `spouse` is exempt everywhere. Example PA/NJ question:
*"We're a PA family with $1M — how much inheritance tax will our kids owe?"* → lineal heirs owe
4.5%, i.e. ~$45,000. The same engine backs the **`relocation-planner`** skill's state-tax arbitrage),
**`analyze_fire_number`** (takes only `desired_annual_spend` + `safe_withdrawal_rate` — no `plan_id`;
elicits either if missing), **`analyze_rent_vs_buy`** (rent-vs-own over a horizon array — net worth
both ways, opportunity cost, and the break-even home-appreciation rate, fully tax- and
inflation-adjusted; accepts `plan_id` to derive the cap-gains rate, or `magi` + `filing_status`).
Most accept `plan_id` plus a few specific fields, but a few (e.g. `analyze_fire_number`) take only
raw inputs — check each tool's schema.
Use **`get_financial_definitions`** when the user asks what a term means.

### "how much education tax credit can I get" / "AOTC vs Lifetime Learning Credit" / "maximize my college tax credit" / "do I qualify for the American Opportunity Credit" / "should I pay tuition out of pocket or from my 529" → `analyze_education_credits`

**Always CALL `analyze_education_credits` for these — do not answer from general knowledge or quote
AOTC/LLC dollar limits or phase-out ranges from memory. When the user gives tuition/qualified-expense
numbers and income, run it and lead with its real output (recommended credit per student, refundable
vs non-refundable split, and the 529 carve-out).** The server is the source of truth for the
$2,500 AOTC / $2,000 LLC caps, the 40%-refundable split, the MAGI phase-out band, the per-return LLC
cap, and the no-double-dipping rule — never recite these from memory.

Trigger condition: the user is paying qualified higher-education expenses (tuition/fees for a
student) and wants to know which federal education credit to claim, how much they'll get, or whether
to keep money out of the 529 to preserve the credit. Pass each student's `qualifiedExpenses` (plus
`isUndergradFirst4Years` / `isGradOrPartTime` / `aotcYearsAlreadyClaimed` / `expensesPaidBy529` when
known) and the household `magi` + `filingStatus` — or a `plan_id` to derive MAGI/filing status. It
returns the recommended credit type **per student** (AOTC vs LLC vs none), the total household
credit, the refundable (40% of AOTC) vs non-refundable split, the phase-out detail, and the
recommended `$4,000` 529 carve-out to preserve full AOTC.

Cross-link: pairs with **`analyze_529_optimization`** (the carve-out coordinates with 529
distributions / rollover) and **`analyze_advanced_taxes`** (for MAGI and marginal-rate context that
drives the phase-out).

### "what's my EFC / SAI" / "FAFSA Student Aid Index estimate" / "how much college financial aid will I get" / "will my kid qualify for need-based aid" / "does a grandparent-owned 529 hurt aid" / "how much does a Roth conversion raise my EFC" / "should I move money out of my kid's name for FAFSA" → `analyze_college_aid_efc`

**Always CALL `analyze_college_aid_efc` for these — do not answer from general knowledge or quote
FAFSA allowances, the 5.64% asset rate, the 22-47% income schedule, the auto-zero/simplified-needs
thresholds, or the student 20%/50% rates from memory.** When the user gives parent income/assets and
student numbers, run it and lead with its real output (the computed SAI, the parent-income /
parent-asset / student components, and the planning-lever sensitivities). The server is the source of
truth for the 2026-27 income-protection and asset-protection allowance tables, the employment-expense
and FICA allowances, the AAI rate schedule, and the auto-zero / simplified-needs cutoffs — never
recite these from memory.

Trigger condition: the user is a parent or grandparent of a college-bound student asking how much
need-based aid eligibility they'll have, or how a planning move changes it — parent- vs
grandparent-owned 529s, holding money as a reportable asset vs realizing base-year income (Roth
conversions, capital gains), or whether retirement-account / home-equity exclusions help. Pass
`parent_income`, `parent_assets`, `household_size`, `number_in_college`, `student_income`,
`student_assets`, `ownership529`, `age_of_older_parent`, and `eligible_means_tested` — or a `plan_id`
to derive the household income/assets. It returns the computed SAI, each contribution component, the
allowances applied, the simplified-needs / auto-zero status, the 529-ownership note, and the marginal
SAI cost of `$10k` of base-year income vs `$10k` of reportable assets.

Cross-link: pairs with **`analyze_529_optimization`** (shift/superfund assets to lower the assessed
SAI; grandparent-owned 529s are no longer counted post-2024) and **`analyze_education_credits`** (the
in-college years where the AOTC/LLC and the 529 carve-out apply).

### "kiddie tax on my kid's dividends/interest" / "gift appreciated stock to my child for 0% capital gains" / "UTMA vs 529 for financial aid" / "how much can I gift my kid tax-free this year" / "custodial account tax on my child" / "should I fund a UTMA or a 529" → `analyze_kiddie_tax`

**Always CALL `analyze_kiddie_tax` for these — do not answer from general knowledge or quote rules of
thumb from memory.** When the user gives the numbers, run it and lead with its real output (the child's
kiddie-tax liability by income tier, the appreciated-stock-gifting family tax saved, the FAFSA
asset-treatment penalty of UTMA vs 529, the annual gift-exclusion headroom, and the recommended funding
structure). The server is the source of truth for the IRC §1(g) kiddie-tax tier thresholds (the
tax-free / child-rate / parent-rate bands), the child's 0% long-term-capital-gains band, the FAFSA
20% student vs 5.64% parent asset assessment rates, and the annual gift-exclusion amount — never recite
these from memory.

Trigger condition: a parent funding a child's college via a custodial (UTMA/UGMA) account, gifting
appreciated stock to a low-income child to harvest gains in the 0% LTCG bracket, weighing UTMA vs 529
for financial-aid impact, or asking how much they can gift tax-free this year. Pass
`childUnearnedIncome` (dividends/interest/cap-gains), `childEarnedIncome`, `childOtherTaxableIncome`,
`appreciatedStockGifted` + `appreciatedStockBasis`, `custodialAccountBalance`, and `giftsAlreadyMade`,
plus the parent's `filingStatus` and `parentTaxableIncome` — or a `plan_id` to derive the parent's
filing status and marginal rate. It returns the kiddie-tax liability split across the three tiers, the
family tax saved by gifting appreciated stock vs the parent selling, the UTMA-vs-529 FAFSA aid penalty,
the remaining gift-exclusion headroom, and a single deterministic funding recommendation.

Cross-link: pairs with **`analyze_529_optimization`** (the 529 alternative to a custodial account, and
superfunding via the gift exclusion) and **`analyze_college_aid_efc`** (the UTMA-as-student-asset 20%
assessment that drives the FAFSA penalty this tool quantifies).

## Step 6 — Present the forecast

- **Lead with the headline:** FIRE age / years-to-FIRE / FIRE% / FIRE number / projected net worth at
  retirement (from the plan summary). Then top insights (Step 3), then the backtesting failure rate as
  the honest risk check. Offer the `share_url`.
- If the user gave a **target retirement age**, frame everything against it ("on track / N years
  short") and surface `solve_goal` levers.
- **Surface `assumed_defaults[]`** from the plan response — read back each assumed `field` +
  `assumed_value` + `note` so the user can correct any silent assumption. The server reports exactly
  what it defaulted; don't enumerate defaults from memory.
- If the **backtesting failure rate is elevated** or the withdrawal rate is aggressive, say so up
  front — do not bury it.

## Next-actions hints

Every tool returns a `next_actions[]` array (each `{ tool, why, prefilled_args }`, with
`prefilled_args` carrying `{ plan_id }` / `{ session_id }`). Follow these server-suggested chains
rather than guessing the next call.

## Recommended call sequence (typical session)

1. (optional) `check_model_completeness` / `explain_plan_state` → decide what to ask for.
2. `generate_financial_plan` → **capture `plan_id`**.
3. `generate_financial_insights` → lead the narrative.
4. `run_backtesting` → honest risk check.
5. `generate_action_plan` → next steps.
6. (on demand) `solve_goal` / `compare_plans` / `get_savings_variations` / `analyze_*`.

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; stock return is real.
- Reuse the `plan_id` across the whole session — never re-send the full model.
- Backtesting uses historical Shiller returns; its `annual_return` arg is ignored.
- This skill is the **FIRE-only deep dive** (savings / retirement-age / spend trade-offs, goal-solving,
  scenario comparison). When the user wants **one comprehensive document** spanning retirement +
  529 college funding + estate-tax exposure + insurance/protection gaps, use the
  **`comprehensive-plan`** skill (its `assemble_comprehensive_plan` orchestrator) — the broad superset.
- If income is from self-employment or a business, the **`self-employed-planner`** skill computes the
  tax-advantaged contribution (Solo 401(k) / SEP / SIMPLE) + QBI deduction to fold into this forecast.
- Not financial advice. Planning estimates only (approximate 2026 brackets/limits where tax tools apply).
