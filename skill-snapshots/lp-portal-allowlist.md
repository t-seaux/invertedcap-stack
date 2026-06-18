---
name: lp-portal-allowlist
description: >-
  Add or remove people on the LP / investor-portal email allowlist that gates portal.invertedcap.com
  (the OTP login). The investor portal, LP portal, and SOI page are ALL served by the same lp-portal
  Cloudflare Worker and gate on this ONE allowlist — there is no separate SOI allowlist. Trigger:
  "add <email or domain> to the investor portal / LP portal / SOI allowlist", "give <person> portal
  access", "allow <domain> on the portal", "remove <email> from the portal allowlist", or similar.
  Edits ALLOWED_DOMAINS / ALLOWED_EMAILS in ~/code/lp-portal/worker/index.js and redeploys.
---

# LP Portal Allowlist

`portal.invertedcap.com` (investor portal / LP portal / SOI) is gated by an email-OTP login. Only
addresses on the allowlist can request a code. The allowlist lives in code — there is no admin UI and
no separate SOI list; all three names refer to this same gate.

## Source of truth
`~/code/lp-portal/worker/index.js` — three collections near the top of the file:
- `ALLOWED_DOMAINS` — a JS array of bare domains. Anyone `@<domain>` gets in. Use for a firm / org
  domain where every address should have access.
- `ALLOWED_EMAILS` — a JS `Set` of individual addresses. Use for people on shared domains
  (gmail.com, comcast.net, outlook.com, yahoo.com, etc.) where you must NOT open the whole domain.
- `TEMP_EMAILS` — a JS `Map` of `email → ISO expiry date (YYYY-MM-DD)`. Access lapses end-of-day UTC
  on the expiry date — the Worker enforces this in `isAllowed()`, no manual cleanup required.

The Worker lowercases + trims the submitted email before matching, so always store entries lowercase.

## Steps (add)
1. Parse the address(es) / domain(s) Tom gave. Handle multiple in one request.
2. Classify each:
   - A bare domain (`acme.com`) or "anyone @acme.com" / "the whole firm" → `ALLOWED_DOMAINS`.
   - A specific address (`jane@gmail.com`) → `ALLOWED_EMAILS`.
   - "Temp / temporary / time-limited access" → `TEMP_EMAILS` (see below).
   - Given a full email on what looks like a dedicated firm domain, default to adding the **individual
     email** unless Tom says "the whole domain" / "everyone at". When unsure which, ask.
3. Read index.js. Check the entry isn't already present (dedupe — skip silently if it is, and say so).
4. Insert the new entry into the correct collection, lowercase, matching the existing formatting and
   trailing-comma style. Keep entries grouped/readable.
5. Redeploy: `cd ~/code/lp-portal/worker && bash deploy.sh`
6. Confirm to Tom exactly what was added (and to which list) and that the deploy succeeded with the new
   Version ID. For TEMP_EMAILS entries, also state the expiry date. Do NOT send a test login code
   unless Tom asks.

## Steps (temp access)
When Tom says "give X temp / temporary / time-limited access" (or similar):
- Default window is **1 week** unless Tom specifies otherwise ("2 weeks", "through Friday", "until
  2026-07-01", "for the day", etc.). Compute the expiry from today's date.
- Add to `TEMP_EMAILS` as `['email@example.com', 'YYYY-MM-DD']`. Always lowercase the email, always
  ISO-format the date.
- If the email is already in `ALLOWED_EMAILS`, they already have permanent access — flag that and ask
  whether to downgrade to temp instead of duplicating.
- Deploy normally. Confirmation message must include the expiry date so Tom can verify the window.

## Steps (remove)
Same flow in reverse: delete the entry from `ALLOWED_DOMAINS`, `ALLOWED_EMAILS`, or `TEMP_EMAILS`,
redeploy, confirm. Any existing session for that person stays valid until it expires (24h) — mention
this only if relevant. Expired `TEMP_EMAILS` entries are harmless (auto-denied) but can be pruned
opportunistically.

## Notes
- This is an explicitly-invoked maintenance task: do NOT ask permission for the edit + `bash deploy.sh`.
- Deploy uses `bash deploy.sh` from `~/code/lp-portal/worker` (not clasp / wrangler directly).
- If Tom gives a name without an email, ask for the address — never guess an LP's email.
- Related: [[lp_portal_fixes_deploy_default]], [[soi_notion_generator]].
