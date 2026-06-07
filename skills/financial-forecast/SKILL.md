---
name: financial-forecast
version: 1.1.0
description: Build a complete FIRE / financial forecast for a household by orchestrating the public planfi MCP. Use whenever someone wants a financial plan, FIRE projection, retirement forecast, net-worth projection, "when can I retire", "am I on track", or wants to model savings / retirement-age / spending trade-offs — e.g. "build me a financial plan, I'm 34 making $180k with $250k invested", "when can I retire if I save $4k/mo?", "am I on track to retire at 55?".
---

# Financial Forecast

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
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
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

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
**`analyze_fire_number`** (takes only `desired_annual_spend` + `safe_withdrawal_rate` — no `plan_id`;
elicits either if missing), **`analyze_rent_vs_buy`** (rent-vs-own over a horizon array — net worth
both ways, opportunity cost, and the break-even home-appreciation rate, fully tax- and
inflation-adjusted; accepts `plan_id` to derive the cap-gains rate, or `magi` + `filing_status`).
Most accept `plan_id` plus a few specific fields, but a few (e.g. `analyze_fire_number`) take only
raw inputs — check each tool's schema.
Use **`get_financial_definitions`** when the user asks what a term means.

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
- If income is from self-employment or a business, the **`self-employed-planner`** skill computes the
  tax-advantaged contribution (Solo 401(k) / SEP / SIMPLE) + QBI deduction to fold into this forecast.
- Not financial advice. Planning estimates only (approximate 2026 brackets/limits where tax tools apply).
