---
name: mademeals-weekly-order
description: >
  Compute Tom's MadeMeals subscription rotation for the upcoming delivery,
  apply the swaps directly on mademeals.co via Playwright (the wc-autoship
  AngularJS plugin), and post the outcome to Slack. **DISABLED 2026-06-17** —
  launchd jobs (`com.tomseo.scheduled.mademeals-weekly-order` and
  `com.tomseo.scheduled.mademeals-daily-check`) are unloaded after recurring
  HTTP 400 errors from the wc-autoship schedule API. Plists remain on disk
  at `~/Library/LaunchAgents/`; re-enable with `launchctl bootstrap gui/$UID`
  once the session/nonce issue is diagnosed.

  Also triggers when Tom says "what's this week's mademeals order", "run
  mademeals rotation", "next week's mademeals", "show me this week's swaps",
  "mademeals pick", or any variant asking what to order this week. Manual
  triggers should pass `--dry-run` to apply.py so the live subscription is
  not mutated outside the Saturday scheduled run.
---

# MadeMeals Weekly Order

Tom is on the mademeals.co **5 Meal Pack** recurring subscription. Each week the order auto-builds from his subscription configuration. This skill rotates which items occupy the 5 meal slots + add-on slots so the order tracks his rotation framework instead of repeating the same items.

End-to-end automated: rotate.py computes the pick → apply.py swaps live items via Playwright → verify → Slack alert with outcome.

---

## Architecture

```
Saturday 9 AM (launchd)
        │
        ▼
   run.sh
        │
        ├─► rotate.py             (pure Python; advances state.json; prints diff markdown)
        │
        ├─► apply.py              (Playwright; reads state.json desired; executes swaps on
        │                          live wc-autoship; verifies via re-fetch; prints outcome)
        │
        ├─► osascript             (on apply rc=0: check off the recurring "mademeals" reminder
        │                          in Apple Reminders "Tasks" list — advances to next Saturday)
        │
        └─► send-alert/send.sh    (combined Slack alert: rotate diff + apply outcome)
```

**Key infra:**

| Parameter | Value |
|---|---|
| Plugin | `wc-autoship` (commercial WooCommerce subscriptions, AngularJS UI) |
| Customer ID | 16376 |
| Schedule ID | 832 |
| Read API | `GET /wp-admin/admin-ajax.php?action=wc_autoship_schedules_get_schedule&customer_id=16376&schedule_id=832` |
| Write API | `POST /wp-admin/admin-ajax.php?action=wc_autoship_schedules_update_composite_items&schedule_id=832` |
| Write body | `{"composite_id": "<slot_id>", "id": "<product_id_or_0_for_remove>", "type": "composite", "order_item_id": "...", "compositeItemMetaKey": "..."}` |
| Auth | Customer session cookie from persistent Playwright profile at `./browser-profile/` |

**Why Playwright instead of raw POST**: the swap payload includes a `compositeItemMetaKey` that Angular generates per render. Driving the page's `<select>` element via `dispatchEvent('change')` lets the plugin's own JS build the payload, so we never have to reconstruct it.

---

## Rotation Framework

Defined in `rotate.py`. Summary:

| Slot | Behavior |
|---|---|
| **Salmon** | Strict alternation: Herb Grilled Salmon ↔ Salmon Cakes |
| **Chicken / Turkey** | Cycle `[Turkey Burgers → Peruvian → BBQ Chicken Thighs → Mediterranean Lemon Chicken]`. Every 6th rotation substitutes Thai Peanut Chicken Breast. |
| **Wildcard** | 4-week cycle: `[Fajita Shrimp BTP → 2 individual dishes → Chimichurri Flank Steak BTP → 2 individual dishes]`. Individual-dish weeks pick 2 from a 10-item pool, excluding the prior 4 picks (= last two dish-weeks). |
| **Carb** | Strict alternation: Roasted Yukon Gold Potatoes ↔ Yellow Rice |
| **Veggie** | Cycle `[Green Beans → Roasted Cauliflower Florets → Mixed Vegetables (Chef's Choice) → Garlic Cauliflower Mash]` |
| **Kids items** | Never change. Preserved from `state.json["current_subscription"]["kids"]`. |

**Individual dish pool (10):** Middle Eastern Chicken, Pesto Chicken, Honey Chipotle Chicken, Tandoori Chicken Thigh, Chicken Tikka Masala, Chicken Rustic Bowl, Korean BBQ Beef Meatballs, Beef Bolognese, Al Pastor Pork, Pulled Pork Carnitas.

All product IDs live in `rotate.py` constants — update there when the menu changes (or when Tom edits the rotation pool).

---

## How apply.py reconciles

1. Load autoship page via Playwright (persistent profile = saved login).
2. Fetch full schedule JSON from the read API.
3. Parse `composite_items` → list of slots: 5 main meal slots (Meal 1..5) + N add-on slots.
4. Infer each main slot's *role* from its current product (e.g., Salmon Cakes → "salmon"); ignore unmappable slots.
5. For each role, compare current product to desired product (from `state.json["current_subscription"]`). On mismatch, drive that slot's `<select>` to the desired product ID.
6. For add-ons: build desired add-on set = kids items + (dish #2 if a 2-dishes wildcard week). Remove any add-on not in the desired set; fill empties with anything desired that's missing.
7. After all changes, reload page and re-fetch schedule. Run `verify()` to compare actual vs desired.
8. **Retry loop**: if verify flags any mismatch (wc-autoship debounces select changes and silently drops some when chained), `reapply_mismatched()` re-issues only the failed slot changes, reload, re-verify. Up to `MAX_RETRY_ATTEMPTS` (3) attempts.
9. Promote each action's status: `submitted` → `done` (verified committed) or `failed` (still mismatched after retries). The alert renders ✅ only for `done`, ❌ for `failed`.

**Wildcard 2-dishes mechanics**:
- Wildcard slot (whichever main slot holds a wildcard-pool item) gets dish #1.
- Dish #2 lives in one of the add-on slots, alongside the two kids items.
- Add-on reconciliation handles the add/remove automatically based on the desired set.

**Slot role inference categories** (in `apply.py`):
- `salmon` → items in `SALMON_PAIR`
- `chicken_turkey` → items in `CHICKEN_TURKEY_CYCLE` + Thai Peanut
- `wildcard` → BTP shrimp/chimichurri OR any individual dish from the pool
- `carb` → items in `CARB_ALTERNATOR`
- `veggie` → items in `VEGGIE_CYCLE`
- `kids` → KIDS MENU portion IDs OR BTP kids variants
- `unknown` → anything else (left alone)

---

## State (`state.json`)

- `current_subscription` — what the next delivery should contain. rotate.py advances this each Saturday to the new pick; apply.py uses it as the desired state.
- `rotation_state.wildcard_cycle_position` — 0..3 pointer into the wildcard cycle.
- `rotation_state.chicken_turkey_cycle_index` — monotonic counter; modulo 4 picks slot, every 6th is Thai Peanut.
- `rotation_state.individual_dish_recent` — FIFO of last 4 individual dish IDs.

State is committed to disk by rotate.py after each non-dry-run invocation.

---

## Files

- `rotate.py` — pure-Python rotation logic + alert composer
- `apply.py` — Playwright-driven live applier (flags: `--dry-run`, `--verify-only`)
- `state.json` — rotation state + canonical desired subscription
- `login.py` — headed Chromium launcher for one-time session login (re-run if session expires)
- `inspect_autoship.py`, `dump_schedule.py`, `test_swap.py`, `test_addremove.py`, `verify_and_clean.py` — diagnostic helpers used to reverse-engineer the plugin; safe to delete once stable
- `browser-profile/` — Playwright persistent profile holding the mademeals login cookie
- `venv/` — Playwright + Chromium

---

## Triggers

**Scheduled (Saturday 9 AM):**
- plist: `~/.claude/local-agents/plists/com.tomseo.scheduled.mademeals-weekly-order.plist`
- runner: `~/.claude/scheduled-tasks/mademeals-weekly-order/run.sh`
- Pipeline: rotate.py → apply.py → send-alert

**Manual (Tom asks in conversation):**
- `python3 rotate.py --dry-run` to preview next week's pick without advancing state.
- `python3 apply.py --dry-run` to preview what would change on the live sub.
- `python3 apply.py --verify-only` to verify live sub matches state.json.

---

## Failure modes

- **Session expired** — Playwright bounces to `/login`. apply.py exits with a session-expired alert. Re-run `python3 login.py` to refresh.
- **Slot role unmappable** — current sub has a product not in any rotation pool category. That slot is reported as "skip" and not modified. Manual fix on autoship page if it's persistent.
- **No empty add-on slot for dish #2** — happens if Tom has filled all add-on slots manually. Reported as "skip". Manual fix.
- **Verify mismatch after apply** — wc-autoship dropped one or more swaps despite the DOM dispatch succeeding. apply.py re-issues just the failed slot changes up to 3 times. Anything still mismatched after the last retry is surfaced in the alert as ❌ on the matching action ("did not persist after retries") AND listed under "Verify mismatches". Tom reviews the autoship page only when both ❌ and the mismatch list appear.
- **Discontinued / off-menu product** — `substitute_discontinued()` runs pre-apply: if the desired wildcard dish (or dish #2 addon) is not in the slot's live `options` list, apply.py auto-picks a fresh replacement from `INDIVIDUAL_DISH_POOL` (excluding `individual_dish_recent`), mutates state.json, and surfaces a 🔁 line in the alert ("X discontinued → Y"). Only applies to pool-backed roles (wildcard dishes); cycle-based roles (salmon, chicken_turkey, carb, veggie) and BTP wildcards (shrimp, chimichurri) require a rotate.py edit and report a ❌ instead. Without this, retries would burn the 3-attempt budget on an impossible swap.
- **mademeals UI changes** — if `wc-autoship` updates breaks the select pattern, apply.py errors are surfaced in the Slack alert and the launchd job continues to fail until fixed. Re-run `inspect_autoship.py` to re-discover the page structure.

---

## Endpoint reference (captured 2026-05-18)

Single endpoint serves both swap and add/remove:

```
POST https://www.mademeals.co/wp-admin/admin-ajax.php?action=wc_autoship_schedules_update_composite_items&schedule_id=832

# Swap product:
{"composite_id":"10277","id":"56367","type":"composite","order_item_id":"255682","compositeItemMetaKey":"1588950700"}

# Remove (set add-on slot to empty):
{"composite_id":"10336","id":"0","type":"composite","order_item_id":"257439","compositeItemMetaKey":"1588950703"}
```

We don't reconstruct this payload directly — Angular generates it when we dispatch `change` on the `<select>` element. See `test_swap.py` and `test_addremove.py` for the reverse-engineering evidence.
