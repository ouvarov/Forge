# Forge — known bugs / production risks

Issues found during demo prep that **must be fixed before prod cut**, not before May 7 demo.
Demo flow tolerates them; production data integrity does not.

---

## #1 🔴 `forge_measure_outcome` doesn't validate window maturity

**File:** `src/core/measurer.service.ts:52` (method `recordOutcome`)

**Problem:** The function blindly UPDATEs outcome at any time. No check that
`generated_at < NOW() - outcome_window_days`. If anything fires `forge_measure_outcome`
within the 7-day window — webhook, race, premature cron — the outcome is locked early.

**Failure scenario on prod:**
- 09:00 — Customer.io sends push, attempt logged
- 11:00 — user opens app once, doesn't return
- 11:05 — Walhalla webhook fires `lesson_started` → forge_measure_outcome called
- Outcome locked at hour 2 with `reward=0.5` (reactivated, no activation pattern)
- On day 6 user returns and completes 2nd lesson → would have been `reward=1.0`
- We never know — outcome already written

**Impact:** `Activation Pattern Hit-rate` (the slide-04 north-star metric) is
**systematically under-counted** on prod. Curve looks worse than reality.

**Fix:** Add maturity guard. Add `force: boolean` to Zod schema for explicit demo override.
```ts
if (Date.now() < matureAt.getTime() && !force) {
  throw new Error(`Outcome window not matured. Matures at ${matureAt.toISOString()}.`);
}
```

---

## #2 🔴 No idempotency — repeated `forge_measure_outcome` silently overwrites

**File:** `src/core/measurer.service.ts:62-71` (the UPDATE)

**Problem:** No `WHERE outcome_measured_at IS NULL` guard. Two webhook fires
(Customer.io retries, race) on the same attemptId silently overwrite each other.

**Failure scenario:**
- Call 1: `reactivated=true` → row updated
- Call 2 (retry, network hiccup): `reactivated=false` → row OVERWRITTEN, no error

**Impact:** Data corruption invisible to observers. Aggregator returns wrong winners/losers.

**Fix:** Reject second write unless `allowOverwrite: true` passed.
```ts
if (existing.outcomeMeasuredAt && !input.allowOverwrite) {
  throw new Error(`Outcome already measured at ${existing.outcomeMeasuredAt}. Pass allowOverwrite=true to replace.`);
}
```

---

## #3 🟡 No explicit `status` column — state inferred from two NULL checks

**File:** `src/db/schema/engagement-attempts.ts`

**Problem:** "What state is this attempt in" is computed every time from
`outcome_measured_at` + `generated_at` + `outcome_window_days` + `actions.simulated`.
No single column tells you. Easy to mis-classify in queries / admin.

**Fix:** Add enum column:
```sql
status: 'incubating' | 'measurable' | 'measured' | 'demo_simulated'
```
- INSERT in log-attempt → `incubating`
- Cron: `UPDATE … SET status='measurable' WHERE status='incubating' AND generated_at < NOW - 7d`
- `recordOutcome` → `measured` (or `demo_simulated` if `force=true`)

`forge_check_pending_outcomes` becomes `WHERE status='measurable'`. Trivial, explicit.

---

## #4 🔴 `simulateOutcomeNow: true` flag is not env-gated

**File:** `src/mcp/tools/log-attempt.tool.ts:39` (Zod field), `:133` (call site)

**Problem:** DEMO_SIMULATION_v1 flag accepts `true` in any environment. If a buggy
caller (or curious dev) sets it on prod, `DemoSimulatorService` writes random
fake outcomes (`Math.random() < confidence`) into the production
`engagement_attempts` table. Distinguishable from real data only by
`actions.simulated: true` deep in JSONB.

**Fix options:**
- Remove `simulateOutcomeNow` flag + `DemoSimulatorService` entirely (comment
  in the file already requests this: "remove after May 7 demo")
- Or env-gate: throw if `simulateOutcomeNow=true && NODE_ENV !== 'development'`

---

## #5 🔴 `forge_simulate_outcomes` tool is registered in prod

**File:** `src/mcp/mcp.module.ts` (registration), `src/mcp/tools/simulate-outcomes.tool.ts:48`

**Problem:** Tool description literally says "DEMO ONLY — do NOT use in production"
but tool is registered unconditionally. Any MCP client in prod can call it and
fake-outcome 100 attempts in one shot.

**Fix:** Conditional registration:
```ts
if (process.env.NODE_ENV !== 'production') {
  this.mcpServer.registerTool(this.simulateOutcomesTool);
}
```

---

## #6 🟡 No automated measurement for matured attempts

**File:** infrastructure / not yet in code

**Problem:** `forge_check_pending_outcomes` only runs when an external actor
calls it. No cron / scheduled job exists yet. If Customer.io workflow isn't
configured by prod cut → matured attempts pile up as unmeasured forever →
`forge_get_group_stats` returns empty groups → bootstrap curve never builds.

**Fix:** Set up Customer.io HTTP step OR internal NestJS cron that runs hourly:
```
1. forge_check_pending_outcomes (limit 50)
2. For each: Walhalla.user_get → derive freshLastSeenAt
3. forge_measure_outcome
```

PRD.md mentions this as the prod measurement path — needs to actually be wired.

---

## Priority for prod cut

Minimum viable fix list (covers ~80% of risk in ~30 min of code):
- [ ] #1 maturity guard in `recordOutcome`
- [ ] #2 idempotency check
- [ ] #4 env-gate `simulateOutcomeNow`
- [ ] #5 env-gate `forge_simulate_outcomes`
- [ ] #6 wire up Customer.io / cron measurement

Nice-to-have:
- [ ] #3 explicit `status` enum (bigger change, migration involved)

Demo (May 7) is unaffected by all of these — they're hardening for the prod path.