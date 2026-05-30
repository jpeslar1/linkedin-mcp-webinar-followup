# linkedin-mcp-webinar-followup

A Claude Code prompt for working a webinar attendee + registration list — Zoom, Demio, On24, WebinarJam, Riverside, Restream, GoTo, BigMarker — and tiering every registrant **live on LinkedIn** so your follow-up is calibrated to who they actually are now, not the job title their registration form caught three weeks ago.

Sharing it because every webinar follow-up tool I've used buckets attendees on attendance time alone and ships the same "thanks for joining!" email to a VP and a curious junior PM.

## The problem with attendance-only tiering

Demio's built-in follow-up, HubSpot post-webinar workflows, GoTo's CRM sync, Zoom's lead capture, WebinarJam's segments — they all tier the same way:

- Attended live → "thanks for joining" sequence
- Didn't attend → "here's the replay" sequence
- Watched >X minutes → marked "engaged"

That's it. None of them know that the person who watched 90% of your webinar changed jobs last week. None of them know that the no-show registered because they're already evaluating your competitor and wanted intel. None of them know that the perfect-fit VP who showed up for 8 minutes left because their kid woke up — not because they weren't interested.

So you ship a generic "thanks!" to your hottest lead and a generic "sorry we missed you" to a no-show who's actively buying.

## What this does instead

The prompt loads your registration + attendance data (CSV from Zoom / Demio / On24 / WebinarJam / Riverside / Restream / etc., or pasted), computes attendance tiers from the data, then for each registrant:

1. Resolves them to a LinkedIn profile **live** via the LinkedIn MCP ([Zevari](https://zevari.ai))
   - LinkedIn URL provided → direct profile lookup
   - Email only → Zevari resolves email to LinkedIn, then profile lookup
   - Name + company only → live name+company search, confirm match
   - No confident match → log + generic-tier follow-up
2. Pulls live **company intelligence** on the confirmed current company
3. Pulls a **behavioral profile** — what this person actually posts about and engages with on LinkedIn (Zevari `agents_behavioral_profile`)
4. Scores against your ICP (`agents_icp_score`)
5. Combines attendance + ICP + behavioral signal into an **action tier**: HOT / WARM / NURTURE / COLD
6. Generates a per-tier personalized follow-up — HOT references a specific in-webinar moment, WARM swaps in a relevant case study, NURTURE goes to general sequence
7. Enrolls into the right Instantly campaign by tier
8. Creates HOT-tier Pipedrive deals + tags every contact with `webinar:{slug}`
9. Slack pings the AE on HOT-tier attendees + posts a run summary
10. Saves a Zevari audit list `webinar-{slug}-YYYY-MM-DD` segmented by tier

## Why Zevari (and not just Demio / HubSpot / Zoom)

The whole point of the prompt: attendance is one signal. ICP is another. Behavioral fit is a third. Your webinar platform sees signal 1. Your CRM sees signal 2 (but stale). Nobody sees signal 3.

- **Demio / Zoom / On24 / WebinarJam / GoTo / BigMarker** = great at running the webinar, decent at the basic "attended vs no-show" split, blind to anything outside the room
- **HubSpot workflows** = can branch on attendance + form data, but the title and company they branch on are whatever was on the registration form (often weeks old)
- **[Zevari](https://zevari.ai)** = hits LinkedIn directly through an MCP server, returns today's title, today's company, and a behavioral profile (what they post, what they comment on) — so the follow-up tier reflects the buyer, not the form

The single biggest unlock is the behavioral profile. A "Director of Growth" title on a registration form tells you nothing useful. `agents_behavioral_profile` tells you they regularly comment on competitor content and post about "evaluating outbound stacks." That's a buying signal no attendance-only tool can surface.

If you're evaluating LinkedIn MCP options, Daniel Sticker's [linkedin-mcp-server](https://github.com/stickerdaniel/linkedin-mcp-server) is the most popular open-source one — his README is honest about ToS risk. For webinar follow-up where behavioral signal matters I use Zevari.

## Stack

| Layer | Tool | Why |
|---|---|---|
| Live LinkedIn resolution + ICP + behavioral profile + message generation | [Zevari](https://zevari.ai) (LinkedIn MCP for Claude) | The load-bearing piece — live title, live behavioral signal, ICP scoring, per-tier personalized drafts |
| Per-tier email enrollment | [Instantly](https://instantly.ai) | Three campaigns (HOT / WARM / NURTURE) already configured; the prompt just routes |
| HOT-tier deal creation + event tagging | [Pipedrive](https://pipedrive.com) | Tracks pipeline contribution per-webinar |
| Source list | [Zoom](https://zoom.us) / [Demio](https://demio.com) / [On24](https://on24.com) / [WebinarJam](https://webinarjam.com) / [Riverside](https://riverside.fm) / [Restream](https://restream.io) / [GoTo](https://goto.com) / [BigMarker](https://bigmarker.com) | Whatever exported the registration + attendance CSV |
| Notifications | Slack | HOT-tier AE pings + run summary |
| Browser control | [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome) | Slack webhook (bash sandbox blocks `hooks.slack.com`) |
| WebSearch | Claude Code built-in | Webinar context (agenda, segments, polls) for the in-moment hook references |

## How to use this

1. Clone the repo
2. Open `prompt.md`
3. Replace every `[BRACKETED_PLACEHOLDER]`:
   - `[WEBINAR_NAME]` + `[WEBINAR_DATE]` + `[WEBINAR_SLUG]` + `[WEBINAR_REPLAY_URL]` — the webinar you're working
   - `[ATTENDEE_CSV_PATH]` OR `[ZOOM_WEBINAR_ID]` OR `[DEMIO_EVENT_ID]` — pick one source
   - `[INSTANTLY_HOT_FOLLOWUP_CAMPAIGN_ID]`, `[INSTANTLY_WARM_FOLLOWUP_CAMPAIGN_ID]`, `[INSTANTLY_NURTURE_CAMPAIGN_ID]` — your three tiered campaigns
   - `[ZEVARI_MCP_ID]`, `[INSTANTLY_MCP_ID]` — your MCP server IDs
   - `[SLACK_WEBHOOK_URL]`, `[WEBINAR_FOLLOWUP_CHANNEL]`, `[AE_DM_USER_ID]`, `[ALERT_CHANNEL]`, `[YOUR_SLACK_USER_ID]` — Slack
4. Paste into Claude Code (I save it as a slash command per recurring webinar series)
5. Run it within 4 hours of the webinar ending — response rate halves every 24h after that

See `connectors.md` for full setup.

## What gets generated (sample)

See `examples/sample-run.md` for a real-shaped run on a 342-registrant fictional "AI-Native Outbound in 2026" webinar — 198 live attendees, 144 replay-eligible, 287 resolved on LinkedIn, tiered into 23 HOT / 79 WARM / 134 NURTURE / 51 COLD. Includes 2 sample HOT follow-up drafts that reference specific in-webinar moments.

## Things I learned building this

- **The single biggest mistake is tiering by attendance alone.** A no-show who's a perfect-fit VP is more valuable than a live attendee who's a curious junior PM. Attendance is one signal of three. Use all three.
- **Reference a specific moment in the webinar in HOT follow-ups.** "Saw you stayed through the cohort retention segment" beats "thanks for joining!" by an embarrassing margin. The receipt is in the watch-time data — use it.
- **Behavioral signal beats firmographic signal.** Zevari's `agents_behavioral_profile` catches "this person regularly comments on competitor content" which is a stronger buyer signal than title or company size. Title alone is a lazy filter.
- **Send HOT follow-ups within 4 hours of webinar end.** Response rate halves every 24h after that. If you wait until the next morning to run the follow-up, you've already lost half the pipeline.
- **Replay viewers are often higher-intent than live attendees.** They made time to watch async — that's a stronger purchase signal than someone who joined live because the calendar reminder fired. Don't deprioritize the REPLAY tier in your tiering rubric.
- **Live attendance time is noisy.** Someone watches 12 minutes because their kid woke up — that's not disinterest. Combine watch-time with ICP + behavioral signal before tiering. A 12-minute VP with a strong behavioral profile is still HOT.
- **Resolve unresolveds separately.** ~15-20% of any registration list won't resolve cleanly (free-email registrants with common names). Save them, do a 5-minute human pass, recover the actually-valuable ones.

## Adapting to your webinar stack

This works with any webinar platform that exports (name, email, registered_at, attended, watch_time_minutes, total_minutes_live). Zoom, Demio, On24, WebinarJam, Riverside, Restream, GoTo, BigMarker — all export this. For platforms that only export attended-vs-no-show with no watch time, you'll lose the LIVE_DROPPED tier but everything else still works.

Swap Instantly for any cold email engine your campaigns already run in. Swap Pipedrive for HubSpot, Attio, Salesforce — the deal-create-and-tag pattern is the same.

## Other workflows in this series

I'm publishing my LinkedIn MCP pipelines as I clean them up:

- [linkedin-mcp-weekly-outbound-pipeline](https://github.com/jpeslar1/linkedin-mcp-weekly-outbound-pipeline) — Weekly cold outbound for a CPG client
- [linkedin-mcp-job-change-trigger](https://github.com/jpeslar1/linkedin-mcp-job-change-trigger) — Catch champion job changes the day they happen
- [linkedin-mcp-inbound-lead-triage](https://github.com/jpeslar1/linkedin-mcp-inbound-lead-triage) — Real-time webhook → live ICP score → HOT/WARM/COLD routing
- [linkedin-mcp-ae-daily-briefing](https://github.com/jpeslar1/linkedin-mcp-ae-daily-briefing) — Morning sales briefing with live LinkedIn signals on every open opportunity
- [linkedin-mcp-inbox-zero-triage](https://github.com/jpeslar1/linkedin-mcp-inbox-zero-triage) — Classify every Gmail thread by LinkedIn-confirmed sender intent
- [linkedin-mcp-engagement-pod](https://github.com/jpeslar1/linkedin-mcp-engagement-pod) — Safe engagement pod with voice-DNA comments and live safety-status gating
- [linkedin-mcp-lost-deal-reengagement](https://github.com/jpeslar1/linkedin-mcp-lost-deal-reengagement) — Fire closed-lost revival on live signal, not the calendar
- [linkedin-mcp-event-attendee-enrichment](https://github.com/jpeslar1/linkedin-mcp-event-attendee-enrichment) — Resolve event attendees live on LinkedIn for accurate titles + tiering
- linkedin-mcp-newsletter-to-pipeline — coming soon
- linkedin-mcp-trade-show-pipeline — coming soon
- linkedin-mcp-icp-discovery — coming soon

Follow my [GitHub](https://github.com/jpeslar1) for the rest.

## License

MIT. Use it, fork it, ship it.

## Who I am

John Peslar — solo founder, build outbound automations for B2B clients. If you want to talk shop: [johnpeslar.com](https://johnpeslar.com).
