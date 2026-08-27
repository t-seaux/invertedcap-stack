---
name: update-profile-photo
description: >-
  Refresh a Notion People-DB row's page icon with the person's current LinkedIn profile photo.
  Trigger when the user says "update profile photo for <name>", "refresh <name>'s photo",
  "update the headshot for <name>", "fix <name>'s icon", "<name>'s photo is out of date", or
  otherwise asks to re-pull someone's picture into the People database. Also handles a list of
  names in one request. Does not create People rows — if the person isn't in the DB, report and
  offer add-to-contacts.
---

# Update Profile Photo

Re-pull a person's LinkedIn profile photo and set it as their Notion People-DB page icon.

## Why this exists

Every People row is supposed to carry the person's LinkedIn photo as its page icon — it's what makes
the DB visually scannable (see `add-to-contacts` Step 4). Icons go stale or missing for three reasons:
the person changed their LinkedIn photo, the row was created before the icon rule existed, or the
enrichment run that created it couldn't reach a photo at the time. This skill fixes one row (or a
named handful) on demand.

## Notion Target

- **Database:** 🤝 People
- **Data Source ID:** `1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`
- Page icons are set with a plain REST PATCH — `ntn api -X PATCH /v1/pages/{page_id}` with an
  `icon.external.url`. This is **not** gated by `gate-db-creation.sh` (that hook only intercepts
  `notion-create-pages`), so no bypass marker is needed. Never use `ntn pages update` here — it blanks
  image embeds on round-trip.

## Step 0 — Resolve the person in the People DB

1. Query the data source directly by name — faster and more literal than search:
   ```bash
   ntn api -X POST "/v1/data_sources/1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9/query" \
     -d '{"filter":{"property":"Name","title":{"contains":"<surname or full name>"}},"page_size":10}'
   ```
   Read back `id`, `properties.Name`, `properties.LI.url`, `properties.Company`, and `icon`.
2. If that returns nothing, try a shorter stem (surname only, alternate spelling, maiden/married name)
   before concluding the row is absent. Only then fall back to `notion-search` with
   `content_search_mode: "workspace_search"` and
   `data_source_url: "collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9"`.
   Never conclude "not in the DB" off a single semantic search.
3. **Multiple matches** (two real people with the same name, or a stem that hits several rows) → ask
   Tom which one, listing Name — Company — Role for each. Don't guess; a wrong icon is worse than a
   missing one.
4. **No match at all** → stop. Report that the person isn't in People and offer to run
   `add-to-contacts`. Do not create the row here — Tom curates People deliberately.

## Step 1 — Fetch the photo from ContactOut

ContactOut is the photo source (it's the licensed provider the rest of the stack uses, and it returns a
stable `images.contactout.com/profiles/<hash>` URL that won't expire out from under the Notion icon).
Work down this ladder, stop at the first hit:

1. **`contactout_enrich_linkedin_profile`** on the row's `LI` URL with `profile_only: true` →
   `profile_picture_url`. This is the primary call: it returns the photo even when the person has no
   contact data, and `profile_only` avoids spending email/phone credits.
2. **`contactout_enrich_person`** on the same URL → `profile_picture_url`, for when the profile
   endpoint 404s.
3. **`contactout_search_people`** → the matching profile's picture. A 404 from the two endpoints above
   is not the end of the ladder. **Note the tool takes no name argument** — search on `job_title` +
   `company` (+ `location` to narrow) and scan the returned profiles for the right person. A person
   who has since left that company won't appear, which is itself a signal the row's Company/Role is
   stale.
4. **No `LI` URL on the row** → resolve one first, then re-enter at step 1:
   - `contactout_email_to_linkedin` on the row's Email. Personal Gmail addresses frequently 404 here.
   - `contactout_search_people` on the row's Role + Company per step 3.
   - **WebSearch** `"<full name>" <company> "<role>" linkedin` — this is the rung that works when the
     row's employer is stale, since the web surfaces the person's current profile regardless. Confirm
     the match on corroborating detail (city, prior employer, the role the row records) before
     accepting the URL; never take the first `/in/` hit on name alone.
   If you find a LinkedIn URL this way, **write it back to the `LI` property** in the same PATCH as the
   icon and say so in the report — a People row with no LI is a gap worth closing.
5. Everything missed → leave the existing icon untouched and report which steps were tried. **No emoji
   fallback** for People entries, ever.

**Freshness caveat — state it, don't hide it.** The response carries `updated_at`, ContactOut's crawl
date for that profile. If Tom asked because a photo looks *old* and `updated_at` predates the change he
noticed, the returned picture may be the same stale one. Say that in the report and offer the
Tom-supplied-photo path below rather than writing a no-op and calling it done.

Cache each raw ContactOut payload exactly as `add-to-contacts` Step 1.5 specifies — the network-intel
index reads from that directory, so never skip it.

## Step 2 — Compare, then write

- Compare the new URL against the row's existing `icon.external.url`. Identity is the
  `/profiles/<hash>` segment for ContactOut URLs, or the image-ID + upload-timestamp path segments for
  a `media.licdn.com` URL an older row may already carry — not the query string. Same identity → same
  photo: report "already current" and skip the write.
- Different, or the row has no icon → PATCH it:
  ```bash
  ntn api -X PATCH "/v1/pages/<page_id>" \
    -d '{"icon":{"type":"external","external":{"url":"<photo url>"}}}'
  ```
  The response echoes the new `icon` — confirm it came back before reporting success.
- Touch nothing else. Company, Role, City, and Category are **not** in scope even when the ContactOut
  payload disagrees with the row — ContactOut's current-employer data is frequently stale, and a photo
  refresh is not the place to relitigate it. The only exception is a previously-blank `LI` from Step 1.4.

## Step 3 — Report back

One line per person, with a link to the Notion page:

- `Updated Matt Heiman's photo — [link]`
- `Matt Heiman's photo is already current — no change.`
- `No photo for Matt Heiman — ContactOut has no picture on any of the three endpoints. Icon left as-is.`
- `Nadia Okonjo isn't in the People DB. Want me to add her?`

For a multi-name request, one consolidated list — not a paragraph per person.

## Behavior Rules

**No permission prompts.** Tom's request authorizes the whole flow: the ContactOut MCP calls, the
payload cache write, and the Notion PATCH. Never pause mid-skill except for the genuine ambiguity in
Step 0.3.

**Tom-supplied photo wins.** If Tom hands over an image URL or attaches a picture instead of naming a
source, use it and skip Step 1 entirely. For a local file, upload it through the Notion file-upload API
and set the icon to the resulting file rather than pointing at a filesystem path.

**Scope is one row, or the rows Tom names.** This skill does not sweep. As of 2026-08-26 roughly 2,600
of ~3,900 People rows have no icon; a mass backfill is a separate, credit-spending job Tom has to
green-light explicitly, not something to drift into from a single-name request.
