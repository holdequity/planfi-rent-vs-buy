# Rent vs Buy (Claude Agent Skill)

Decide whether to **rent or buy a home**, right inside Claude. Ask something like _"should I rent
or buy a $525k condo if I can rent for $2,600/mo?"_ or _"when does buying beat renting?"_ and the skill
gathers a few inputs and returns the answer: the **break-even real home-appreciation rate** (the
headline), per-horizon BUY vs RENT net worth (pre- and after-tax), when the mortgage is paid off,
when owning gets cheaper than renting, and a federal tax breakdown.

It's a **thin orchestration layer** over the public **planfi MCP** — all the math lives server-side.
The simulation runs in **nominal** dollars internally (so a fixed-rate mortgage P&I is correctly
held flat in nominal terms and erodes in real terms), then **deflates** every output back to today's
(real) dollars. The skill itself bundles no engine; it just gathers inputs and calls the tool.

## What you need

The skill calls these planfi MCP tools, served from `https://ai.planfi.app/mcp` (public, no auth):

- **`analyze_rent_vs_buy`** — the primary engine. Give it the home, carrying costs, rent, the REAL
  rates (home appreciation, rent growth, market return) plus an explicit inflation rate, your
  horizons, and household tax context — it returns the break-even appreciation rate, per-horizon
  net worth, payoff/crossover years, an opportunity-cost block, and a full tax breakdown.
- Optional chains: **`analyze_property_return`** (dig into the specific property purchase),
  **`forecast_str_market`** (if weighing a short-term rental), **`generate_financial_plan`** (build
  the full household plan and get a `plan_id` that derives MAGI + filing status for the tax rate).

Two things are strictly required: the **core scenario** (purchase price, mortgage, rent) and the
**household tax context** — either a `plan_id` (plan-derived MAGI + filing status take precedence)
or both `magi` and `filing_status` directly. Without tax context the tool returns
`MISSING_HOUSEHOLD_CONTEXT` and asks for it. Everything else has sensible server defaults, and the
skill reads back every default it falls back on so you can correct it.

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
npx skills add holdequity/planfi-rent-vs-buy
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/rent-vs-buy ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/rent-vs-buy .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-rent-vs-buy
/plugin install rent-vs-buy@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r rent-vs-buy.zip rent-vs-buy`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.
   (Your account must have Skills enabled.)

## Example prompts

- "should I rent or buy a $525k condo vs renting at $2,600/mo? I'm single, $150k income"
- "rent vs buy breakeven for a $480k home with 20% down at 6.5%"
- "I already own — owned 4 years, worth $640k, owe $470k. should I have rented instead?"
- "at what home appreciation rate does buying tie renting over 30 years?"
- "is it cheaper to buy or keep renting and invest the difference?"

## How it works (call flow)

1. (optional) `generate_financial_plan { earners, ... }` → full household plan + a **`plan_id`** that
   derives MAGI + filing status server-side.
2. `analyze_rent_vs_buy { purchase_price, mortgage_rate, monthly_rent, …, plan_id }` (or pass `magi`
   + `filing_status` directly) → break-even appreciation rate + per-horizon BUY vs RENT + taxes.
3. (on demand) follow the returned `next_actions[]` — e.g. `analyze_property_return` to dig into the
   specific property, or `generate_financial_plan` to set the decision in the full plan context.

The headline is `breakeven_home_appreciation_real` (both `pre_tax` and `after_tax`): the real
home-appreciation rate at which BUY net worth ties RENT net worth at the longest horizon.

See `SKILL.md` for the full instructions, exact tool params, and output format.

## Notes & honest caveats

- All decimals are fractions (7% → `0.07`); all figures are today's (real, inflation-adjusted)
  dollars. Appreciation / rent-growth / market rates are **REAL**; the sim runs in **nominal**
  internally (grossed up by the explicit `inflation_rate`) and deflates outputs back to real.
- The fixed mortgage P&I is **nominal** and the §121 home-sale exclusion ($500k MFJ / $250k single)
  is **nominal** (not inflation-indexed) — both erode in real terms over long horizons, which is
  precisely why the real break-even appreciation rate is the right headline.
- Tax is **federal only** in v1 (no state tax): LTCG 0/15/20 + 3.8% NIIT on approximate 2026
  brackets. Renter portfolio gains are taxed; the home gain is shielded by the §121 exclusion.
- The skill is explicit about any default it falls back on (it reads back `assumed_defaults[]`), and
  flags that the break-even is sensitive to the appreciation assumption.
- Not financial advice. Planning estimates only.

## License

MIT — see [LICENSE](./LICENSE).
