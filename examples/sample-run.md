# Sample run

A real-shaped run of the pipeline (sanitized — fictional webinar, fictional people) from a recent follow-up pass.

## Setup

- Webinar: **AI-Native Outbound in 2026** (fictional)
- Date: 2026-05-23
- Slug: `ai-native-outbound-2026`
- Replay URL: `https://example.com/replay/ai-native-outbound-2026`
- Source: Demio export → CSV → 342 rows (live broadcast ran 58 minutes)
- All connectors healthy
- Ran the follow-up at T+3h post-webinar

## Step 1 — load + normalize

- Raw CSV rows: 342
- After whitespace trim + email lowercase: 342
- After (first+last+normalized-company) dedupe: 338 (caught 4 dupes — same person registered twice with different company strings)
- Net into Step 2: **338 registrants**

Common dirt:
- 22 rows trailing whitespace in `name`
- 11 mixed-case emails
- 18 company suffix variants
- 47 free-email registrants with self-typed companies

## Step 2 — attendance tier

Computed from `attended` + `watch_time_minutes / total_minutes_live`:

| Attendance tier | Count | Definition |
|---|---|---|
| **LIVE_ATTENDED** (≥50% watched) | 198 | Stayed for at least 29 minutes |
| **LIVE_DROPPED** (<50% watched) | 36 | Joined live, left early |
| **REPLAY-eligible** | 104 | Registered, didn't attend live |
| **NO_SHOW** | 0 | Replay flow auto-fires on Demio |

(Note: 198 + 36 + 104 = 338. All non-attendees treated as REPLAY-eligible since Demio fires the replay automatically.)

## Step 3 — live LinkedIn resolution

Resolution path distribution:

| Path | Description | Count | Resolved |
|---|---|---|---|
| A | LinkedIn URL provided → `linkedin_get_profile` | 124 | 122 |
| B | Email only → `inbox_enrich_sender` → `linkedin_get_profile` | 167 | 132 |
| C | Name + company only → `linkedin_search_profiles` | 47 | 33 |
| — | Unresolved (no confident match) | 51 | 0 |

**Net resolved: 287 / 338 (85%)**

51 unresolved — mostly free-email registrants with very common names. Saved to `unresolved.csv`. They default to COLD (single replay email, no further sequence).

**Rate-limit pacing:** Hit the ~40/hour cap after batch 1. Paced 7 batches across the afternoon. Total Step 3 wall time: 6.5 hours. (Next webinar I'm starting at T+1h instead of T+3h.)

## Step 4 — company intelligence + behavioral profile

For each of the 287 resolved, pulled `agents_company_intelligence` on the LIVE confirmed company AND `agents_behavioral_profile`.

**Behavioral signal distribution:**

| Signal | Count |
|---|---|
| Posts about category / adjacent problems | 38 |
| Comments on competitor content | 27 |
| Engages with our existing customers' posts | 19 |
| Mostly inactive on LinkedIn | 114 |
| Posts about unrelated topics | 89 |

The 27 "comments on competitor content" registrants are the unlock. Pure attendance + ICP scoring would have missed them. 9 of those 27 ended up in HOT tier on this combined signal alone.

**Title drift:** 49 of 287 resolved registrants (17%) had a LIVE title different from the registration form title. 6 of those moved from a non-buyer title to a buyer title in the 2-3 weeks since registering. Those 6 are HOT-tier surprises that an attendance-only tool would have ignored.

## Step 5 — ICP score

Of the 287 resolved:
- ICP_PASS: 156
- ICP_FAIL: 131

## Step 6 — action tier breakdown

| Action tier | Count | Rationale mix |
|---|---|---|
| **HOT** | 23 | LIVE_ATTENDED + ICP_PASS + (senior title OR strong behavioral) → 14; REPLAY + ICP_PASS + senior title + strong behavioral → 6; LIVE_DROPPED + ICP_PASS + strong behavioral → 3 |
| **WARM** | 79 | LIVE_ATTENDED + ICP_PASS (no behavioral) → 38; REPLAY + ICP_PASS → 29; LIVE_DROPPED + ICP_PASS → 12 |
| **NURTURE** | 134 | NO_SHOW + ICP_PASS → 0 (none); LIVE_DROPPED + ICP_FAIL with behavioral → 41; REPLAY + ICP_FAIL with behavioral → 28; LIVE_ATTENDED + ICP_FAIL → 65 |
| **COLD** | 51 | All unresolveds + hard DQ hits |

Total: 23 + 79 + 134 + 51 = 287 (resolved) + 51 (unresolved counted as COLD) = 338 ✓

**Attendance-only-tier vs final-tier delta (the receipt):**
- 11 LIVE_ATTENDED registrants ended up COLD because they were ICP_FAIL with no behavioral signal. An attendance-only tool would have shipped them a "thanks for joining!" personalized follow-up. Wasted send.
- 6 REPLAY-eligible registrants ended up HOT (perfect-fit VPs with strong behavioral signal who couldn't make it live). An attendance-only tool would have put them in the generic "sorry we missed you" replay flow.
- 3 LIVE_DROPPED (under 50% watched) ended up HOT. Pure attendance tools would have flagged them as low-intent — but their behavioral profiles told the real story.

That's 20 mis-tierings on a 287-resolved list, in BOTH directions.

## Step 7 — sample HOT follow-up drafts

Two of the 23 HOT drafts:

```
TO: maya.chen@northwind-growth.example
FROM: john@example.com
SUBJECT: cohort retention segment

Hey Maya —

Saw you stayed through the cohort retention segment of ai-native outbound
in 2026 - that's the part most of the LIVE_ATTENDED folks dropped at.

Noticed you've been posting about evaluating outbound stacks for the
northwind-growth team this quarter. The signal-based routing piece we
covered (the third pillar, around the 38-min mark) is the one I'd dig into
first if you're sizing the build vs buy question.

If it's useful, happy to send the actual playbook we walked through
on the slide deck — it's got the routing rules + the prompt we use for
the tiering call. No pitch, just the artifact.

— John

(Hook references: 38-min mark = cohort retention segment, watched 51 of 58 minutes.
Behavioral signal: 4 posts in the last 30 days about outbound stack evaluation.
LIVE title: VP Revenue Operations @ Northwind Growth. Registration form said
"Director of Sales Ops" — promoted 18 days before the webinar.)
```

```
TO: devon.brooks@altimark.example
FROM: john@example.com
SUBJECT: replay caught my eye

Hey Devon —

Saw you grabbed the replay of ai-native outbound in 2026 over the weekend
- the segment on signal-based routing is the one that gets the most
replies from people in the altimark-shaped stage.

The reason I'm reaching out: caught a comment of yours on a [competitor]
post about their sequence-builder a couple weeks back. The thread is
exactly the question we covered around the 22-min mark - the one about
whether the buyer signal lives in the sequence step or in the routing
layer.

If you want the deck-and-prompt walk-through (no pitch), reply here and
I'll send. 15-min Loom if you want me to talk through it.

— John

(Hook references: replay viewer, 22-min mark = signal-vs-routing segment.
Behavioral signal: commented on competitor outbound post 9 days before webinar.
LIVE title: Head of Growth @ Altimark. ICP_PASS + senior title + strong behavioral
signal → HOT despite REPLAY tier.)
```

Both followed the rules: webinar name in first 10 words, lowercase after first letter, no LinkedIn references, in-moment specificity, soft CTA.

## Step 8-10 — enrollment + audit + CRM

- **Instantly enrollments:**
  - `AI-Native Outbound 2026 — HOT Follow-up`: 23 (1-email same-day, personalized)
  - `AI-Native Outbound 2026 — WARM Follow-up`: 79 (3-email drip)
  - `AI-Native Outbound 2026 — NURTURE`: 185 (134 NURTURE + 51 COLD, COLDs excluded from steps 2-3)
- **Zevari audit list:** `webinar-ai-native-outbound-2026-2026-05-23` (287 resolved, segmented by tier)
- **COLD archived:** 51 via `targets_archive`
- **Pipedrive deals (HOT):** 23 created in the post-webinar pipeline stage, owner = AE
- **Pipedrive contacts tagged:** 287 contacts tagged `webinar:ai-native-outbound-2026`. CRM titles updated to LIVE Zevari titles. Behavioral signal logged in note.

## Step 11 — HOT-tier AE Slack pings

23 HOT-tier registrants got individual pings to `#webinar-followups` AND DMs to the AE. Sample format:

```
🎯 HOT from AI-Native Outbound in 2026

Devon Brooks — Head of Growth @ Altimark
LinkedIn: linkedin.com/in/devon-brooks-[id]
Attendance: REPLAY (watched 47m / 58m of replay, 81%)
ICP score: 9/10
Behavioral signal: Commented on competitor sequence-builder post 9d before webinar.
                   3 posts in last 30d about outbound stack evaluation.
Why HOT: REPLAY + ICP_PASS + senior title + strong behavioral signal.
         Replay viewers in the senior-title + behavioral bucket convert
         at ~2x our LIVE_ATTENDED baseline.
Hook used: "Saw you grabbed the replay of ai-native outbound in 2026..."
Pipedrive deal: [link]

→ Recommend manual LinkedIn note within 24h on top of the Instantly sequence.
```

## Step 12 — run summary

```
🎬 Webinar Follow-up Complete — AI-Native Outbound in 2026

📅 Webinar date: 2026-05-23
📥 Registrants loaded: 342
🧹 After normalization/dedupe: 338

Attendance breakdown:
🟢 LIVE_ATTENDED (≥50% watched): 198
🟡 LIVE_DROPPED (<50% watched): 36
🔵 REPLAY-eligible: 104
⚪ NO_SHOW: 0

🔍 Resolved live on LinkedIn: 287
❓ Unresolved: 51 (in unresolved.csv)

Action tier breakdown:
🎯 HOT (AE pinged, deal created): 23
✅ WARM (drip enrolled, case-study personalized): 79
📥 NURTURE (general sequence): 134
🚫 COLD (single replay email): 51

📧 Instantly enrollments: HOT 23 / WARM 79 / NURTURE+COLD 185
📇 Pipedrive deals created (HOT): 23
📇 Pipedrive contacts tagged: 287
🏷  Zevari audit list: webinar-ai-native-outbound-2026-2026-05-23

Sample HOT hooks:
• Maya Chen, Northwind Growth — Saw you stayed through the cohort retention segment of ai-native outbound in 2026
• Devon Brooks, Altimark — Saw you grabbed the replay of ai-native outbound in 2026 over the weekend
• Priya Nair, Loomstack — Saw you stayed through the signal-routing demo of ai-native outbound in 2026
```

## Step 13 — reflection notes

- **Attendance-vs-final-tier delta: 20 mis-tierings (7% of resolved).** 11 LIVE_ATTENDED that ended up COLD (would have wasted a personalized send) and 6 REPLAY + 3 LIVE_DROPPED that ended up HOT (would have been buried in generic flow). The 6 REPLAY-HOTs are the clean win — those are leads an attendance-only tool actively suppresses.
- **Title drift 17%** — 49 LIVE titles differed from registration form. 6 of those moved into buyer titles in the registration window. Showed the number to the marketing lead who pushed for "just use HubSpot's branching" — settled the argument.
- **Behavioral signal wins:** 9 of 23 HOT-tier registrants got bumped to HOT primarily because of behavioral signal (competitor post comments, category posts). These would have been WARM at best under pure attendance + ICP. The Devon Brooks case (REPLAY + competitor comment) is the prototypical example.
- **Unresolved rate 15%** — acceptable. Suggested adding a LinkedIn URL field to the next webinar's registration to cut this to <10%.
- **Replay-HOT vs LIVE_ATTENDED-HOT** — too early to measure on this run alone, but tagging the deals so I can compare reply / book / close rates by tier across the next 3-4 webinars.
- **Rate cap:** Hit it again. Need to start at T+1h, not T+3h. The 6.5h Step 3 wall time pushed the HOT sends past the 4h-from-end response-rate cliff for 4 of the 23 HOTs.
- **Total run time:** ~7 hours wall clock (mostly Step 3 LinkedIn pacing), ~30 min of active Claude time.
