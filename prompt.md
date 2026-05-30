# Webinar Follow-up — Run Within 4 Hours of Webinar End

A Claude Code prompt that takes a webinar registration + attendance list (Zoom, Demio, On24, WebinarJam, Riverside, Restream, GoTo, BigMarker — any platform that exports attended + watch_time_minutes), resolves every registrant **live on LinkedIn** via the LinkedIn MCP (Zevari), combines attendance + ICP + behavioral signal into an action tier, generates per-tier personalized follow-ups, and enrolls them in the right Instantly campaign.

The reason this prompt exists: every webinar follow-up tool (Demio's built-in, HubSpot workflows, Zoom's lead capture, GoTo's CRM sync) tiers attendees on attendance alone — and the title they branch on is whatever was on the registration form weeks ago. A no-show who's a perfect-fit VP gets a generic "sorry we missed you," and a curious junior PM who showed up live gets the same "thanks for joining!" as the buyer in the room. Zevari surfaces live title + behavioral signal so the tier reflects who they actually are.

---

## Overview

You are running follow-up for `[WEBINAR_NAME]` (`[WEBINAR_DATE]`, slug `[WEBINAR_SLUG]`, replay at `[WEBINAR_REPLAY_URL]`).

For each registrant on the list:

1. **Load + normalize** the registration + attendance data
2. **Compute attendance tier** from the data (LIVE_ATTENDED / LIVE_DROPPED / REPLAY / NO_SHOW)
3. **Resolve live to LinkedIn** via Zevari (`linkedin_get_profile`, `inbox_enrich_sender`, or `linkedin_search_profiles`)
4. **Pull live company intelligence + behavioral profile** on the confirmed person
5. **ICP-score** against the rules in `profile_get`
6. **Determine action tier** combining attendance + ICP + behavioral signal — HOT / WARM / NURTURE / COLD
7. **Generate per-tier personalized follow-up** — HOT references an in-webinar moment, WARM swaps in a relevant case study
8. **Enroll** in the right Instantly campaign by tier
9. **Save** to a Zevari list `webinar-[WEBINAR_SLUG]-[YYYY-MM-DD]` segmented by tier
10. **Create HOT-tier deals** in Pipedrive, tag every contact with `webinar:[WEBINAR_SLUG]`
11. **Slack ping the AE** on every HOT-tier attendee + post a run summary

Run this within 4 hours of the webinar ending. Response rate halves every 24h after that. If the list is >200 registrants, expect to hit LinkedIn's ~40/hour cap and pace accordingly (Step 3 explains how).

---

## ERROR & APPROVAL NOTIFICATIONS

If anything stops the task or needs approval, send a notification to `[ALERT_CHANNEL]`. The bash sandbox blocks `hooks.slack.com` — use Chrome:

1. Call `mcp__Claude_in_Chrome__tabs_context_mcp` with `createIfEmpty: true`
2. If the tab is on a `chrome://` URL, navigate to `https://google.com` first
3. Run via `mcp__Claude_in_Chrome__javascript_tool`:

```js
fetch('[SLACK_WEBHOOK_URL]', {
  method: 'POST',
  mode: 'no-cors',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    text: "<@[YOUR_SLACK_USER_ID]> ⚠️ *Webinar Follow-up — Action Needed* — [WEBINAR_NAME]\n\n<error or approval ask>"
  })
}).then(r => ({status: r.status, type: r.type})).catch(e => ({error: e.message}))
```

`{status: 0, type: "opaque"}` = success.

---

## CONNECTORS

- **Zevari (LinkedIn MCP):** `mcp__[ZEVARI_MCP_ID]` — live profile resolution, name+company search, email-to-LinkedIn enrichment, company intelligence, behavioral profile, ICP scoring, message generation, target save
- **Instantly:** `mcp__[INSTANTLY_MCP_ID]` — per-tier campaign enrollment
- **Pipedrive:** `mcp__pipedrive__` — HOT-tier deal creation + webinar tagging
- **Chrome:** `mcp__Claude_in_Chrome__` — Slack webhook
- **WebSearch:** optional — pull webinar agenda / segment titles if you don't already have them

Pick exactly ONE source for the attendee list:
- `[ATTENDEE_CSV_PATH]` — file path to a CSV export from Zoom / Demio / On24 / WebinarJam / Riverside / Restream / GoTo / BigMarker
- `[ZOOM_WEBINAR_ID]` — Zoom webinar ID + API
- `[DEMIO_EVENT_ID]` — Demio event ID + API

See `connectors.md`.

---

## STEP 1 — LOAD ATTENDEE LIST

Use whichever source is configured. If multiple are filled in, prefer in this order: CSV path → Zoom → Demio.

### Required fields (any source)

Every row must have, or be derivable from:

- `name` (or `first_name` + `last_name`)
- `email`
- `registered_at` (ISO timestamp)
- `attended` (bool — did they join live?)
- `watch_time_minutes` (numeric — how long they were actually in the room)
- `total_minutes_live` (numeric — how long the live broadcast ran)

Optional but helpful:
- `company` (self-typed on registration — KEEP but don't trust)
- `title` (self-typed — same)
- `linkedin_url`
- Any poll/Q&A answers (Zevari can use these as behavioral signal)

If columns are non-standard, ask the user before guessing.

### Normalize EVERY row

1. **Trim whitespace** on name, company, email
2. **Lowercase email**
3. **Split full_name** if only one field
4. **Strip company suffixes** for matching (`, Inc.`, ` Inc`, ` LLC`, etc.) but KEEP the original
5. **Dedupe** by lowercase-email, then by (first+last+normalized-company)

Log raw count vs. normalized count.

---

## STEP 2 — COMPUTE ATTENDANCE TIER

For each normalized registrant, compute from the data (do NOT trust any tier the source platform attached):

- **LIVE_ATTENDED** — `attended == true` AND `watch_time_minutes / total_minutes_live >= 0.5`
- **LIVE_DROPPED** — `attended == true` AND `watch_time_minutes / total_minutes_live < 0.5`
- **REPLAY** — `attended == false` AND replay email hasn't been sent yet (default: assume yes — every no-show is REPLAY-eligible)
- **NO_SHOW** — `attended == false` AND replay flow already fired (only set if your platform tracks this; otherwise everyone non-attended is REPLAY)

Save the attendance tier on the record.

**Critical:** Do NOT use attendance tier as the final tier. It's an input to Step 6.

---

## STEP 3 — LIVE RESOLUTION VIA ZEVARI

For each normalized registrant, pick a resolution path based on what data you have:

### Path A — LinkedIn URL provided
Call `linkedin_get_profile` with the URL.
- If it resolves → save the live profile
- If it 404s → fall back to Path C

### Path B — Email only (no LinkedIn URL)
Call `inbox_enrich_sender` with the email.
- If it returns a profile → call `linkedin_get_profile` on the resolved URL
- If it can't resolve (typical for free-email registrants) → fall back to Path C

### Path C — Name + company only
Call `linkedin_search_profiles` with `name` + company-as-keyword filter.
- If exactly 1 confident match → call `linkedin_get_profile`
- If 2-3 candidates → pick the one whose current company best matches the registration. Log the decision.
- If 0 confident matches or >3 ambiguous → log as **unresolved**, do NOT guess. Unresolved go to a generic-tier follow-up only.

### Rate-limit pacing
LinkedIn's safe ceiling via Zevari is ~40 lookups/hour.

- ≤ 40 registrants → run straight through
- 40-200 registrants → run in 40-lookup batches with 60-min pauses, OR start the run earlier
- 200+ registrants → tier the resolution: process LIVE_ATTENDED first (they're hottest decay-wise), then REPLAY, then NO_SHOW

If you hit the cap mid-run, save the queue to `unresolved-pending-[YYYY-MM-DD].csv` and flag in Slack. Do NOT fall back to using the registration title for tiering — that's the anti-pattern this prompt avoids.

### Unresolved
Track every unresolved registrant in a separate `unresolved.csv`. They get the COLD-tier generic follow-up by default (no personalization possible).

---

## STEP 4 — COMPANY INTELLIGENCE + BEHAVIORAL PROFILE

For each resolved registrant, call:

1. `agents_company_intelligence` on their **live confirmed current company** — NOT the registration form company
2. `agents_behavioral_profile` on the profile — what they post about, what they comment on, who they engage with

If `agents_company_intelligence` returns thin data, fall back to `linkedin_get_company` on the company URL.

The behavioral profile is the unlock. It tells you whether this person:
- Posts about your category (strong intent)
- Comments on competitor content (in-market signal)
- Engages with your existing customers' content (warm path)
- Posts about adjacent problems (good wedge)
- Is mostly inactive (firmographic-only signal)

Save the behavioral signal summary on the record.

---

## STEP 5 — ICP SCORE

### Step 5a — Load current ICP
Call `profile_get` to load the active ICP criteria, hard disqualifiers, employee thresholds, industry buckets, company type rules. Never hardcode.

### Step 5b — Score each resolved registrant
Call `agents_icp_score` for each resolved registrant using the company intelligence + their live title.

If `agents_icp_score` returns `server_side_ai_unavailable`, apply the ICP criteria manually from `profile_get` data. Do not pause.

Save the ICP result (PASS / FAIL + numeric score) on each record.

---

## STEP 6 — DETERMINE ACTION TIER

This is the core decision. Combine attendance + ICP + behavioral signal into one of four action tiers.

### HOT
ANY of:
- LIVE_ATTENDED + ICP_PASS + relevant behavioral signal (posts about category, comments on competitor content)
- LIVE_ATTENDED + ICP_PASS + senior title (VP / Head of / Director / Founder)
- LIVE_DROPPED + ICP_PASS + strong behavioral signal (they joined and left for a real reason — kid, meeting, fire drill — but the intent is still there)
- REPLAY + ICP_PASS + senior title + strong behavioral signal (they made async time AND fit AND are in-market — often the highest-intent tier)

→ Enroll in `[INSTANTLY_HOT_FOLLOWUP_CAMPAIGN_ID]`, generate a personalized message via `agents_generate_messages`, create a Pipedrive deal, ping the AE in Slack with the full context.

### WARM
ANY of:
- LIVE_ATTENDED + (ICP_PASS OR strong behavioral signal — but not both)
- REPLAY + ICP_PASS (no behavioral signal — replay viewers are still high-intent baseline)
- LIVE_DROPPED + ICP_PASS (showed up, didn't stay long, but fit is right)

→ Enroll in `[INSTANTLY_WARM_FOLLOWUP_CAMPAIGN_ID]` (3-email sequence: D+0 thank-you + replay link, D+3 related content / case study, D+7 soft pitch). Use `agents_personalize` to swap in a relevant case study based on industry. No Pipedrive deal yet.

### NURTURE
- Anyone with ICP_PASS but low engagement signal (LIVE_DROPPED with no behavioral signal, NO_SHOW with ICP_PASS, REPLAY without ICP confirmation)
- Anyone with strong behavioral signal but ICP_FAIL (might convert later as their company grows)

→ Enroll in `[INSTANTLY_NURTURE_CAMPAIGN_ID]` (general newsletter sequence). No personalization beyond first-name swap.

### COLD
- LIVE_DROPPED + ICP_FAIL + no behavioral signal
- NO_SHOW + ICP_FAIL
- Unresolved registrants (couldn't confirm LinkedIn — default to safe)
- Anyone hitting a hard disqualifier from `profile_get`

→ Send a single generic "thanks for registering, here's the replay" email (via the NURTURE campaign's first email only — exclude from rest of sequence), archive in Zevari via `targets_archive`. Do not pursue further.

Save the action tier on every record.

---

## STEP 7 — GENERATE PER-TIER FOLLOW-UPS

Call `library_context_get` with `skill_slug='craft-outreach'` and `include_examples=true`. Do not draft any messages until you've read this — formatting rules live there.

### HOT-tier drafts (REQUIRED per HOT registrant)

Use `agents_generate_messages` with the full context:
- Webinar name + date
- The specific session / moment they're most likely to have stayed for (derive from watch_time + the webinar agenda — if they stayed through the second half, reference a second-half segment)
- Their live title + company (from Zevari, NOT registration)
- Their behavioral profile summary
- The ICP rationale

#### HOT hook rules

1. **Reference the webinar by name in the first 10 words.**
   - ✓ `"Saw you stayed through the cohort retention segment of ai-native outbound in 2026"`
   - ✗ `"Thanks for joining our webinar!"`

2. **Reference a specific moment if the watch-time data supports it.** This is the unlock — proves you actually looked at the data, beats every templated follow-up.
   - LIVE_ATTENDED (watched >75%) → reference a late-webinar segment
   - LIVE_ATTENDED (watched 50-75%) → reference a mid-webinar segment
   - LIVE_DROPPED → don't reference a specific moment, lean on the behavioral signal instead
   - REPLAY → "saw you grabbed the replay — the [segment] piece is the one that gets the most replies"

3. **Lean on the behavioral signal as the connective tissue.** Don't pitch — use the behavioral profile to make the message feel like it came from someone who actually paid attention.

4. **No LinkedIn references in the email body.** Write as if you noticed them through normal channels.

5. **Soft CTA.** Calendar link OR "want me to send the X playbook we mentioned?" — never "book a demo" in the first message.

### WARM-tier drafts

Use `agents_personalize` to swap in a relevant case study based on their industry (from `agents_company_intelligence`). The D+0 email is thank-you + replay link + industry-matched case study. The campaign sequence handles D+3 and D+7.

### NURTURE-tier drafts

No personalization beyond first-name swap. The campaign sequence is built; just enroll.

### COLD-tier

Single email: replay link + sign-off. No personalization.

---

## STEP 8 — ENROLL IN INSTANTLY (PER TIER)

For HOT → `[INSTANTLY_HOT_FOLLOWUP_CAMPAIGN_ID]`
For WARM → `[INSTANTLY_WARM_FOLLOWUP_CAMPAIGN_ID]`
For NURTURE + COLD → `[INSTANTLY_NURTURE_CAMPAIGN_ID]`

Call `add_leads_to_campaign_or_list_bulk` per tier batch.

Required custom variables:
- `webinar_name`, `webinar_date`, `webinar_slug`, `webinar_replay_url`
- `hook` (HOT only — the in-moment reference)
- `case_study_ref` (WARM only — industry-matched)
- `companyRef`, `industryRef` (LIVE company from Zevari, mapped to your campaign's industry buckets)

Set `skip_if_in_campaign: true`.

---

## STEP 9 — SAVE TO ZEVARI (audit list, segmented by tier)

For all resolved registrants (HOT / WARM / NURTURE / COLD — yes, including COLD for the audit trail):

1. Call `targets_save` for each with their LIVE title + company + LinkedIn URL + action tier
2. Create the parent list `webinar-[WEBINAR_SLUG]-[YYYY-MM-DD]` via `targets_create_list`
3. Optionally create sub-lists per tier: `webinar-[WEBINAR_SLUG]-hot-[YYYY-MM-DD]`, etc.

This is the audit trail. If someone converts six months later, you can trace which webinar.

---

## STEP 10 — PIPEDRIVE: HOT DEALS + ALL-CONTACT TAGGING

### HOT-tier — create deals
For every HOT-tier registrant:

1. Search Pipedrive (`search_contacts` by email, then by name)
2. If contact exists → update title + company to LIVE values, append note `"Attended [WEBINAR_NAME] on [WEBINAR_DATE]. Live-confirmed: [live_title] @ [live_company]. Behavioral: [signal summary]. Tier: HOT."`
3. If not → create contact + organization with LIVE values
4. **Create a new deal** in the post-webinar stage with title `"[live_company] — [WEBINAR_NAME] HOT lead"`. Owner = the AE in `[AE_DM_USER_ID]` (or your routing logic).
5. Tag contact with `webinar:[WEBINAR_SLUG]`

### All other tiers (WARM / NURTURE / COLD) — tag contacts only
For every non-HOT resolved registrant:

1. Search Pipedrive
2. Create or update contact with LIVE values
3. Tag with `webinar:[WEBINAR_SLUG]`
4. **Do NOT create deals** — these come through pipeline if/when they reply to the sequence

---

## STEP 11 — HOT-TIER SLACK ALERTS

HOT-tier registrants deserve a human touch on top of the automated message. Conference / webinar leads decay fast.

For each HOT-tier registrant, post to `[WEBINAR_FOLLOWUP_CHANNEL]` AND DM `[AE_DM_USER_ID]` (via Chrome `javascript_tool`):

```
<@[AE_DM_USER_ID]> 🎯 *HOT from [WEBINAR_NAME]*

[Name] — [LIVE title] @ [LIVE company]
LinkedIn: [URL]
Attendance: [LIVE_ATTENDED / LIVE_DROPPED / REPLAY] ([watch_time]m / [total]m, [%])
ICP score: [N]/10
Behavioral signal: [one-line summary from agents_behavioral_profile]
Why HOT: [one-line rationale]
Hook used: [the in-moment reference from agents_generate_messages]
Pipedrive deal: [link]

→ Recommend manual LinkedIn note within 24h on top of the Instantly sequence.
```

If there are >10 HOT, consolidate into a single ranked summary instead of spamming.

---

## STEP 12 — RUN SUMMARY SLACK

Post to `[WEBINAR_FOLLOWUP_CHANNEL]` via Chrome `javascript_tool`:

```
<@[YOUR_SLACK_USER_ID]> 🎬 *Webinar Follow-up Complete — [WEBINAR_NAME]*

📅 Webinar date: [WEBINAR_DATE]
📥 Registrants loaded: [N]
🧹 After normalization/dedupe: [N]

Attendance breakdown:
🟢 LIVE_ATTENDED (≥50% watched): [N]
🟡 LIVE_DROPPED (<50% watched): [N]
🔵 REPLAY-eligible: [N]
⚪ NO_SHOW: [N]

🔍 Resolved live on LinkedIn: [N]
❓ Unresolved: [N] (in unresolved.csv)

Action tier breakdown:
🎯 HOT (AE pinged, deal created): [N]
✅ WARM (drip enrolled, case-study personalized): [N]
📥 NURTURE (general sequence): [N]
🚫 COLD (single replay email): [N]

📧 Instantly enrollments: HOT [N] / WARM [N] / NURTURE+COLD [N]
📇 Pipedrive deals created (HOT): [N]
📇 Pipedrive contacts tagged: [N]
🏷  Zevari audit list: webinar-[WEBINAR_SLUG]-[YYYY-MM-DD]

Sample HOT hooks:
• [name, company] — [hook]
• [name, company] — [hook]
```

---

## STEP 13 — REFLECTION

After the summary, briefly note:

- **Attendance-only vs final tier delta** — how many people would have been mis-tiered under attendance-only logic. Sample comparison: how many LIVE_ATTENDED ended up COLD because they were ICP_FAIL, and how many NO_SHOW ended up HOT because they were perfect-fit VPs with strong behavioral signal. This is the receipt for the workflow.
- **Title drift** — count of registrants whose LIVE title differs from the registration form title. Same receipt as the event prompt — show this number to anyone who questions tiering on live data instead of form data.
- **Behavioral signal wins** — any HOT-tier registrants who were ICP-borderline but the behavioral profile bumped them up (e.g. comments on competitor content). These are the surprise wins.
- **Unresolved rate** — if >20%, the registration form is too thin. Suggest adding a LinkedIn URL question to next webinar's registration.
- **Replay vs live conversion (next webinar)** — if you can track which tier ends up booking, watch whether REPLAY-tier HOT outperforms LIVE_ATTENDED-tier HOT over time. In my data REPLAY-HOT consistently has higher reply rate.
- **Rate-limit notes** — if the cap was hit, plan next webinar's follow-up to start within 2 hours of the broadcast ending instead of 4.

---

## Schedule

Run within 4 hours of webinar end. Response rate halves every 24h. For multi-session webinar series, run after each session — don't batch by week.

Save as a Claude Code slash command per recurring webinar series, e.g. `/followup-webinar-outbound` with the campaign IDs and replay URL template pre-filled.
