---
name: rent-vs-buy
version: 1.0.0
description: Decide whether to rent or buy a home by orchestrating the public planfi MCP. Use whenever someone asks "should I rent or buy", "is it cheaper to buy a $X house vs renting at $Y/mo", "rent vs buy breakeven", "when does buying beat renting", or wants to compare owning a home against renting + investing the difference over time — with the headline being the real home-appreciation rate at which buying ties renting.
---

# Rent vs Buy

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All math lives server-side — the simulation runs in **nominal** dollars internally (so a fixed-rate
mortgage P&I is correctly held flat in nominal terms and erodes in real terms), then **deflates**
every output back to today's (real) dollars. This skill only gathers inputs and calls the tools — it
does **not** compute anything locally. Read-only; it never changes the user's data.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_rent_vs_buy`):
- **`analyze_rent_vs_buy`** (PRIMARY) — the full rent-vs-buy engine.
- optional: **`analyze_property_return`** (return on a specific property purchase),
  **`forecast_str_market`** (if they're weighing a short-term rental), **`generate_financial_plan`**
  (to derive household tax context and get a `plan_id`).

Use whichever name your environment exposes (bare or `mcp__planfi__`-prefixed); below they are
written bare. If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

## Step 1 — Gather inputs

**Core scenario:**
- **Home**: `purchase_price`, `down_payment_percent`, `closing_costs_percent`, `mortgage_rate`
  (fixed nominal annual), `mortgage_term_years`, `selling_cost_pct`.
- **Carrying costs** (tool assembles these into the all-in non-P&I owning cost): `property_tax_rate`,
  `maintenance_pct`, `hoa_monthly`, `renters_insurance_annual`.
- **Rent side**: `monthly_rent` (today's $), `annual_rent_growth_real`.
- **Rates (all REAL annual, as fractions):** `home_appreciation_real`, `index_return_real`
  (market return for invest-the-difference), and the single explicit `inflation_rate` that grosses
  the real rates up to nominal.
- **Already own it?** `years_already_owned`, plus optional `current_balance` and
  `current_home_value` overrides.
- **Horizons:** `horizons` array of years (default `[5,10,20,30]`).

**Household tax context (REQUIRED to derive the capital-gains rate):** EITHER a `plan_id`
(from `generate_financial_plan` — plan-derived MAGI + filing status TAKE PRECEDENCE), OR both
`magi` and `filing_status` (`single` / `married_joint`) directly. If neither is supplied the tool
returns `MISSING_HOUSEHOLD_CONTEXT` — ask for them.

> **Engine facts to bake in:** all dollars are **today's (real) dollars**; all decimals are
> **fractions** (7% → `0.07`); appreciation / rent-growth / market rates are **REAL**; the sim runs
> in **nominal** then deflates; the fixed mortgage P&I is **nominal** (erodes in real terms); the
> §121 home-sale exclusion ($500k MFJ / $250k single) is modeled **nominal** (not inflation-indexed,
> so it erodes in real terms over long horizons); tax is **federal only** in v1 (LTCG 0/15/20 + 3.8%
> NIIT, 2026 brackets).

## Step 2 — Run the analysis

Call **`analyze_rent_vs_buy`** with the gathered inputs:

```
analyze_rent_vs_buy({
  purchase_price: 525000,
  down_payment_percent: 0.2,
  mortgage_rate: 0.065,
  mortgage_term_years: 30,
  property_tax_rate: 0.011,
  maintenance_pct: 0.01,
  monthly_rent: 2600,
  annual_rent_growth_real: 0.01,
  home_appreciation_real: 0.015,
  index_return_real: 0.07,
  inflation_rate: 0.03,
  horizons: [5, 10, 20, 30],
  magi: 150000,
  filing_status: "single"
})
// already own it? add: years_already_owned, current_home_value, current_balance
```

If you already built a plan, pass `{ plan_id }` instead of `magi` + `filing_status` and let the
server derive the household context (it reports `magi_source: "plan"`).

## Step 3 — Present the results

- **Lead with the headline** — `headline.breakeven_home_appreciation_real` (BOTH `pre_tax` and
  `after_tax`): the real home-appreciation rate at which BUY net worth ties RENT net worth at the
  longest horizon. Then the `verdict` string ("At your X% real appreciation assumption, RENTING/BUYING
  is ahead by $Y at N years.").
- **Per-horizon table** (`per_horizon[]`): for each horizon show BUY vs RENT net worth (pre- and
  after-tax, all REAL dollars), the winner, and the delta (`buy − rent`).
- **Timeline:** `payoff_year` (when the mortgage hits zero) and `crossover_year` (first year owning's
  monthly cost drops below rent, after P&I falls off — or null).
- **Opportunity cost:** `opportunity_cost` block (buy vs rent terminal wealth, multiples, winner).
- **Tax breakdown:** `tax_breakdown` — filing status, MAGI + `magi_source`, renter gain/tax/NIIT,
  home gain, §121 exclusion, home taxable gain/tax/NIIT, and the horizon it's reported at.
- **Risk flags:** surface `risk.flags` (break-even is sensitive to the appreciation assumption;
  §121 exclusion erodes in real terms).
- **Surface `assumed_defaults[]`** — read back each assumed `field` + `assumed_value` + `note` so the
  user can correct any silent assumption (these also ride on `disclosures.assumptions`). The server is
  the source of truth for what it defaulted; don't enumerate defaults from memory.

## Next-actions hints

The tool returns a `next_actions[]` array (each `{ tool, why, prefilled_args }`, with `prefilled_args`
carrying `{ plan_id }` when available). The only server-defined edge out of `analyze_rent_vs_buy` is to
`generate_financial_plan` (fold the rent-or-buy decision into the full FIRE timeline) — follow that
rather than guessing the next call. (`analyze_property_return` chains the other way: it suggests
`analyze_rent_vs_buy` as ITS follow-up, not vice versa.)

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; the sim runs nominal internally
  then deflates; rates are real.
- The fixed mortgage P&I is nominal and the §121 exclusion is nominal — both erode in real terms,
  which is precisely why the real break-even appreciation rate is the right headline.
- Tax is federal only in v1 (no state tax); LTCG 0/15/20 + 3.8% NIIT on 2026 brackets.
- Pass `plan_id` to derive household MAGI + filing status; caller `magi` + `filing_status` are the
  fallback when there's no plan.
- Not financial advice. Planning estimates only (approximate 2026 brackets/limits).
