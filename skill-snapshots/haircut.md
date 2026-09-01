---
name: haircut
description: >
  Book a family haircut at Les Enfants Terribles (85 Bergen St, Brooklyn) via the salon's
  Meevo customer portal. TWO profiles: (1) Tom — stylist Hide, Men's/Unisex Short Cut, slot
  ≤4pm, ~monthly; (2) Andy (son) — stylist Jessica, kids' short cut, on WEEKENDS. Trigger on
  "book a haircut", "book Andy a haircut", "schedule my haircut", "need a haircut", or the
  monthly nudge. Pre-stages the cart, surfaces ≤4pm slots, and hands Tom the final tap
  (reCAPTCHA blocks full automation — see below). Never books silently. NOT the
  restaurant-reservation skill.
---

# Haircut Booking

Two profiles — determine WHOSE haircut first (default = Tom):

| Whose | Stylist | Service | Constraint | Cadence |
|---|---|---|---|---|
| **Tom** | **Hide** | Men's/Unisex Short Cut (~$66.99) | slot **≤ 4:00pm** (kid pickup) | ~monthly |
| **Andy** (son) | **Jessica** | kids'/child short cut | **WEEKENDS** (Sat/Sun) | as needed |

Same Meevo flow for both (swap the service + stylist + constraint). If Jessica isn't in the
Meevo employee list for this location, ask Tom. The rest is written for Tom's cut.

## ⚠️ Full automation is IMPOSSIBLE — reCAPTCHA gates it (confirmed 2026-08-31)

Meevo runs **invisible reCAPTCHA (Enterprise)** on the availability + booking calls. An
automated/CDP-driven Chrome scores as a bot, so the availability call
`POST /onlinebooking/api/aob/Scan/NextAvailable` returns **`412 PreconditionFailed:
[reCAPTCHA failed]`** — no slots ever come back, and the React date picker spins forever.
**This is NOT a render bug or a slow load.** A human tap in the same browser passes
reCAPTCHA and loads instantly. So:

- **Do NOT** try to drive the date picker or booking via CDP — it will always hang (412).
- **Do NOT** try to script it with curl — no reCAPTCHA token, same 412.
- The working model is **pre-stage-and-tap**: automation does the no-reCAPTCHA UI steps
  (appointment type → service → specific employee → Hide), then Tom does the ~5-second
  human tap that clears reCAPTCHA (pick date/time → Continue → Confirm).
- Phone fallback: **347-763-2124** if the portal itself is down.

## Fixed facts

- Salon: **Les Enfants Terribles – Cobble Hill**, 85 Bergen St, Brooklyn NY 11201.
- Meevo portal: `https://na1.meevo.com/CustomerPortal/advanced/ob/module-selection?tenantId=201583&locationId=204215`
  (tenantId **201583**, locationId **204215**).
- Booking profile: `launch.sh booking 9224`, drive with
  `AGENT_BROWSER_PORT=9224 node ~/.claude/local-agents/agent-browser/browse.mjs …`.
  Tom is logged in on this profile so Hide knows it's him + it syncs to his calendar; if a
  relaunch drops to "Guest", have Tom sign in as part of his tap.
- **Known API IDs** (from reverse-engineering the OB API — see `meevo_api.mjs` /
  `meevo_capture.mjs` in agent-browser):
  - Service `Men's/Unisex Short Cut` (40 min, $66.99) = `104643c2-082e-4057-84c3-b43900e6d40b`
  - Stylist `Hide` = `35f55e3e-3318-460c-90b0-b43900e6d451`
  - Availability = `POST /onlinebooking/api/aob/Scan/NextAvailable` (reCAPTCHA-gated)
  - Employees for a service = `POST /onlinebooking/api/ob/employee/list`
  - NOT the $94.74 Beard Trim combo, NOT the $81.34 Long cut. Tom keeps it SHORT.
- **HARD constraint: appointment start ≤ 4:00pm.** Never surface or book a later slot.
- **Cadence — CHECK THE CALENDAR FIRST.** Tom goes ~monthly. His haircut auto-syncs to his
  **Inverted work calendar** (`tom@invertedcap.com`) as "Service(s) scheduled at Les Enfants
  Terribles…" (search fullText `Hide` / `Enfants`). Find his LAST cut: if <~3 weeks ago,
  tell him he's not due and confirm before proceeding; else target ~4 weeks out unless he
  names a day. **Standing pref: Tom books the NEXT cut right when he gets the current one**
  (plan ahead + block time) — so proactively tee up the next booking at confirm time.
- **Meevo auto-creates the calendar event** on booking — do NOT add a duplicate "TS Haircut".

## Flow (pre-stage → hand Tom the tap)

1. `launch.sh booking 9224` if not up; open the Meevo portal.
2. Drive the no-reCAPTCHA steps via `browse.mjs click-text`: **Individual Appointment** →
   **Men's/Unisex Short Cut** (the adult one, NOT "Child 5 & Up") → **Specific employee**
   (Hide auto-fills; verify "Remove employee Hide" is present) → leave it on the
   employee/date page. **Stop here — do not click into the date picker (it will 412-hang).**
3. Verify the cart reads `Men's/Unisex Short Cut` + `with Hide`, then hand Tom the tap:
   ```
   ✂️ Haircut staged — Hide, Men's Short Cut ($66.99)
   Booking window's ready. In it: Continue → pick a ≤4pm slot the week of <target> →
   Continue → Confirm. (The picker only loads on your tap — reCAPTCHA blocks me.)
   ```
4. Tom taps through. Afterward, confirm the Meevo-synced event landed on his Inverted
   calendar (don't add a dupe). If he asks, read back the booked date/time.

## Guardrails
- **Never claim it's booked unless Tom confirms he tapped through** (or the synced calendar
  event appears). Automation cannot complete the booking.
- **Never a slot after 4:00pm.** If nothing ≤4pm exists in the window, say so and offer to
  widen the DATE range — never stretch the time.
- Same YES-gate philosophy as purchases: surface options, Tom decides.

## Notes
- The reCAPTCHA wall is the same class of blocker to watch for when reverse-engineering
  reservation platforms. Resy's API is NOT hard-walled this way (scriptable); OpenTable /
  SevenRooms are more locked down — see [[project_appointments]].
- Monthly proactive nudge (future): scheduled check — if ~4 weeks since the last cut on the
  calendar, text Tom "time for a cut? want me to stage Hide's booking?" then run this flow.
