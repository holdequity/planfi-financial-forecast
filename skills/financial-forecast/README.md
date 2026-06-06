# Financial Forecast (Claude Agent Skill)

Build a complete FIRE / financial forecast for a household, right inside Claude. Ask something like
_"build me a financial plan — I'm 34 making $180k with $250k invested"_ or _"when can I retire if I
save $4k/mo?"_ and the skill gathers a couple of inputs and returns a full projection: net worth
over time, FIRE age, FIRE number, Monte Carlo success rate, prioritized insights, an action plan,
and what-if scenario comparisons.

It's a **thin orchestration layer** over the public **planfi MCP** — all the math and the 150-year
Shiller market dataset live server-side. The skill itself bundles no engine; it just gathers inputs
and calls the tools.

## What you need

The skill calls these planfi MCP tools, served from `https://ai.planfi.app/mcp` (public, no auth):

- **`generate_financial_plan`** — the primary builder. Give it the household earners + investments
  and it returns the full projection plus a `plan_id` handle, a summary, and a shareable planfi.app
  URL. Everything else chains off the `plan_id` so the model is sent only once.
- **`start_plan_intake` / `get_completed_plan` / `wait_for_completion`** — optional form-based input
  path when the user would rather type their data into a web form (keyed by `session_id`).
- **`generate_financial_insights`** — prioritized, dollar-quantified insights.
- **`generate_action_plan`** — time-boxed next steps.
- **`generate_financial_commentary`** — plain-language narrative.
- **`run_backtesting`** — Monte Carlo failure rate over historical Shiller returns.
- **`solve_goal`** — easiest-first levers to hit a target FIRE age.
- **`compare_plans`** — head-to-head named scenarios (retire 55 vs 60, save more vs spend less).
- **`get_savings_variations`**, **`get_asset_allocation`**, **`check_model_completeness`**,
  **`explain_plan_state`** — sensitivity, allocation, and completeness helpers.
- Optional deep-dives: **`analyze_roth_conversion`**, **`analyze_withdrawal_strategy`**,
  **`analyze_healthcare_bridge`**, **`analyze_funding_waterfall`**, **`analyze_refinance`**,
  **`optimize_social_security`**, and more.

Only two areas are strictly required to forecast: each earner's **age + annual salary**, and your
**stock/investment portfolio** (`current_value` + `monthly_contribution`). Everything else has
sensible server defaults — the skill asks about your target retirement age and desired retirement
spend (and won't silently assume them), and only asks for cash / real estate / debts / accounts if
relevant.

### MCP bootstrap (one time)

If the planfi tools aren't connected yet, run:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

On **claude.ai**: Settings → Connectors → add a custom connector pointing at
`https://ai.planfi.app/mcp` (no auth). The skill also reminds you to do this if the tools are
missing when you invoke it.

## Install

### Quickest — skills.sh CLI (recommended)

```
npx skills add holdequity/planfi-financial-forecast
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/financial-forecast ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/financial-forecast .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-financial-forecast
/plugin install financial-forecast@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r financial-forecast.zip financial-forecast`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.
   (Your account must have Skills enabled.)

## Example prompts

- "build me a financial plan — I'm 34 making $180k with $250k invested, contributing $3k/mo"
- "when can I retire if I save $4k/mo and want $70k/yr in retirement?"
- "am I on track to retire at 55? show me the Monte Carlo success rate"
- "compare retiring at 55 vs 60 for my household"
- "what levers get me to FIRE by 50?"

## How it works (call flow)

1. (optional) `check_model_completeness` / `explain_plan_state` → decide what to ask for.
2. `generate_financial_plan { earners, stocks, ... }` → full projection + **`plan_id`** + share_url.
3. `generate_financial_insights { plan_id }` → lead the narrative.
4. `run_backtesting { portfolio_value, annual_spend, current_age }` → honest risk check.
5. `generate_action_plan { plan_id }` → next steps.
6. (on demand) `solve_goal` / `compare_plans` / `get_savings_variations` / `analyze_*` { plan_id }.

The `plan_id` is reused across the whole session so the household model is sent only once.

See `SKILL.md` for the full instructions, exact tool params, and output format.

## Notes & honest caveats

- All decimals are fractions (7% → `0.07`); all figures are today's (real, inflation-adjusted)
  dollars; the stock `annual_return` is a real rate (default 0.07).
- Backtesting uses historical Shiller returns; its `annual_return` argument is ignored.
- The skill is explicit about any default it falls back on (retirement age 65, spend 50000, SWR
  0.04, 7% real return, 3% inflation) so you can correct it. If the failure rate is elevated, it
  says so up front.
- Not financial advice. Planning estimates only (approximate 2026 brackets/limits where tax tools
  apply).

## License

MIT — see [LICENSE](./LICENSE).
