# Budget

A single-file budget tracker. Tap a circle, type a number, hit Enter.

`index.html` is the whole app — no build step, no dependencies, no server.
Open it in a browser. State lives in `localStorage` under the key `budget-app-v1`.

This README doubles as the **spec for the iOS port**. The rules below are the
part worth getting right; the UI is straightforward.

---

## Data model

```jsonc
{
  "categories": [
    { "id": "c1", "name": "Bar drinks", "max": 200, "color": "#3ecf8e" }
  ],
  "expenses": [
    { "id": "a1b2", "catId": "c1", "amount": 24.5, "note": "tab at Joe's", "ts": 1754250120000 }
  ]
}
```

- `max` is the **monthly** limit for that category.
- `ts` is epoch milliseconds, local-time semantics.
- `note` is optional and may be missing on older records — treat absent as `""`.
- Expenses are **never** auto-deleted. Cycle rollover is derived from `ts`, not
  by clearing data. Past cycles stay until the user deletes them.
- Export/import round-trips this exact shape (`•••` menu).

## The monthly cycle

The cycle runs **the 15th to the 14th**, not calendar months.

- `cycleStart(now)` = the most recent 15th at local midnight, on or before `now`.
- `cycleEnd(now)` = the following 15th at local midnight.
- An expense belongs to the cycle where `cycleStart <= ts < cycleEnd`.

Cycle length varies (28–31 days), which matters for the pace math below.

## Weeks

Weeks are **7-day blocks counted from the cycle start** — they are not
calendar weeks and do not start on Sunday. The final block of a cycle is a
stub of 1–3 days, since no cycle is a whole number of weeks.

```
weekStart = cycleStart + floor(daysBetween(cycleStart, now) / 7) * 7 days
```

> **Do this with calendar-day arithmetic, not by adding `7 * 86400000` ms.**
> Adding fixed milliseconds drifts by an hour across a DST boundary and can
> push a week boundary onto the wrong day. The JS version uses
> `midnight()` / `daysBetween()` / `addDays()` helpers for exactly this reason.
> In Swift, use `Calendar.current` (`date(byAdding:)`, `startOfDay(for:)`).

## Weekly pace — the core rule

The weekly allowance is **not** `monthly / 4`. It re-paces after overspending:
whatever is left at the *start of the current week*, spread over the weeks that
remain in the cycle.

```
weeksRemaining  = max(daysBetween(weekStart, cycleEnd) / 7, 1/7)
leftAtWeekStart = max(cap - spentBeforeThisWeek, 0)
weekBudget      = min(leftAtWeekStart / weeksRemaining, leftAtWeekStart)
weekLeft        = weekBudget - spentThisWeek
```

Worked example (the one this was designed around): a $600 budget over a 28-day
cycle, with $500 spent in week 1. At the start of week 2, `weeksRemaining` is
3.0 and `leftAtWeekStart` is $100 → **$33.33/week** for the rest of the cycle.

Three edge cases that the `min(...)` and `max(...)` above exist to handle:

| Case | Behavior |
|---|---|
| Final stub week (e.g. 3 days left) | Dividing by `3/7` would *inflate* the weekly budget above what's actually left for the month. The `min` caps it at `leftAtWeekStart`. |
| Category fully spent before this week | `weekBudget` is `0`, never negative. |
| Category already past its max | `weekBudget` is `0`, and `weekLeft` goes negative to surface the overage ("$30 over week"). |

This same formula runs twice: once per category (using that category's `max`)
and once overall (using the sum of all `max` values).

**The per-category week figures do not sum to the overall week figure, and that
is intentional.** The overall number pools all remaining money, so overspending
in one category eats the global figure while an untouched category still shows
its own headroom. Read the circles for "can I spend on this," the top card for
"am I okay overall." Don't try to reconcile them in the port.

## UI notes worth preserving

- **Two rings per circle.** Thick outer = month against `max`. Thin inner =
  that category's weekly pace. Both turn red when exceeded.
- **Tap the ring** to log spending. **Tap the count badge** (bottom-right) for
  that category's full transaction list for the current cycle, uncapped.
- **Edit mode** turns ring taps into category edits (name / max / color / delete).
- **Recent** on the main screen is capped at 25 and scoped to the current cycle;
  **History** is everything ever, grouped by cycle, newest first.
- Amount input accepts `12`, `12.50`, `$40`, `1,200`, and sums like `12+8-3`.
- Deleting a category also deletes its expenses (confirmed first).

## iOS port notes

- All amounts are currency — use `Decimal`, not `Double`, for stored values.
  The JS version uses floats and rounds at display time; don't carry that over.
- The `min`/`max` clamps in the pace formula are load-bearing. Port them exactly.
- Timestamps are local-time semantics. Store the instant, but do all cycle and
  week bucketing through `Calendar.current` in the device's timezone.
- Reuse the export JSON as the import path so data can move from the web
  version to the app.
