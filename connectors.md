# Connectors

Every MCP server this prompt uses, with setup links and the placeholders to replace in `prompt.md`.

## Zevari — LinkedIn MCP for Claude (REQUIRED, load-bearing)

**What it does:** Live LinkedIn profile lookup (`linkedin_get_profile`), name+company live search (`linkedin_search_profiles`), email-to-LinkedIn resolution (`inbox_enrich_sender`), company intelligence (`agents_company_intelligence`), **behavioral profile** (`agents_behavioral_profile`), ICP scoring (`agents_icp_score`), HOT-tier message generation (`agents_generate_messages`), WARM-tier personalization (`agents_personalize`), and target save (`targets_save`).

**Why it's the core of this workflow:** Every webinar follow-up tool (Demio's built-in, HubSpot post-webinar workflows, Zoom lead capture, GoTo's CRM sync, WebinarJam segments) tiers attendees on attendance alone, and the title they branch on is whatever was on the registration form weeks ago. Stale title + attendance-only tiering = a generic "thanks for joining" email to your hottest lead and a generic "sorry we missed you" to a no-show who's actively buying.

Zevari resolves every registrant live, surfaces the behavioral profile (what they post / comment on / engage with), and lets you tier on attendance + ICP + behavioral signal — not registration data alone.

The single biggest unlock is `agents_behavioral_profile`. A title alone tells you a person *could* buy. A behavioral profile tells you whether they *are* buying right now — what they post about, what competitor content they comment on, who they engage with. That's the signal no attendance-only tool can surface.

**Setup:** [zevari.ai](https://zevari.ai) → connect your LinkedIn account → copy the MCP server URL into your Claude Code config (`~/.claude/mcp.json` or via `claude mcp add`).

**Placeholders to replace in `prompt.md`:**
- `[ZEVARI_MCP_ID]` — your Zevari MCP server ID (looks like `mcp__a1b2c3d4-...`)

**Endpoints used:**
- `linkedin_get_profile` — live current title + company
- `linkedin_search_profiles` — name+company live search for registrants without a LinkedIn URL
- `inbox_enrich_sender` — email → LinkedIn URL resolution
- `agents_company_intelligence` — live confirmed-company size, industry, news, signals
- `agents_behavioral_profile` — what the person actually posts / comments on / engages with (load-bearing for HOT vs WARM split)
- `linkedin_get_company` — fallback if `agents_company_intelligence` returns thin data
- `profile_get` — load current ICP rules
- `agents_icp_score` — score each registrant against ICP
- `agents_generate_messages` — HOT-tier personalized drafts referencing in-webinar moments
- `agents_personalize` — WARM-tier case-study swap
- `library_context_get` (`craft-outreach`) — outreach formatting rules
- `targets_save` + `targets_create_list` — audit list
- `targets_archive` — archive COLDs

## Instantly — Tiered cold email enrollment (REQUIRED)

**What it does:** Enroll registrants into one of three campaigns based on the action tier — HOT, WARM, or NURTURE (COLD gets the NURTURE D+0 only).

**Why it's in this workflow:** Three different intent levels = three different sequence cadences. The HOT campaign hits same-day with a personalized message + soft CTA. WARM runs a 3-email drip (thank-you + replay, case study, soft pitch). NURTURE is your general newsletter sequence.

**Setup:** [Instantly MCP docs](https://developer.instantly.ai/mcp) — install the MCP, paste your API key. Build three campaigns per webinar series (or per recurring webinar) and get the campaign UUIDs.

**Placeholders:**
- `[INSTANTLY_MCP_ID]` — your Instantly MCP server ID
- `[INSTANTLY_HOT_FOLLOWUP_CAMPAIGN_ID]` — HOT-tier campaign UUID (1-email same-day, personalized hook, soft CTA)
- `[INSTANTLY_WARM_FOLLOWUP_CAMPAIGN_ID]` — WARM-tier campaign UUID (3-email: D+0 thank-you + replay + case study, D+3 related content, D+7 soft pitch)
- `[INSTANTLY_NURTURE_CAMPAIGN_ID]` — NURTURE-tier campaign UUID (general newsletter sequence)

**Endpoints used:**
- `list_leads` — dedupe check
- `add_leads_to_campaign_or_list_bulk` — per-tier enrollment

## Pipedrive — HOT-tier deals + all-contact tagging (REQUIRED)

**What it does:** Creates a deal for every HOT-tier registrant (in the post-webinar pipeline stage), and tags every resolved registrant with `webinar:[WEBINAR_SLUG]` so the AE can filter "all registrants from this webinar" later.

**Why it's in this workflow:** Multi-webinar pipeline attribution. If you run 12 webinars a year, you need to know which webinar a deal came from. The tag is how. The HOT-tier deals are what gets your AE moving same-day instead of waiting for the sequence to do the work.

**Setup:** Any Pipedrive MCP server. I use `mcp__pipedrive__` from the community list.

**Endpoints used:**
- `search_contacts` / `search_organizations` — dedupe before creating
- `add_person` / `update_person` — contact create/update with LIVE Zevari values
- `add_organization` — create org if it doesn't exist
- `add_deal` — HOT-tier only, in the post-webinar pipeline stage
- `add_note` — log the webinar attendance + live title + behavioral signal for audit
- `add_tag` / `update_person_tags` — apply `webinar:[WEBINAR_SLUG]` tag

## Attendee source — Pick ONE (REQUIRED)

The prompt accepts three source types. Configure exactly one.

### Option A — CSV file (most common)

**Where it comes from:** Zoom export, Demio export, On24 export, WebinarJam export, Riverside export, Restream export, GoTo export, BigMarker export.

**Required columns:** `name`, `email`, `registered_at`, `attended` (bool), `watch_time_minutes`, `total_minutes_live`. Optional: `company`, `title`, `linkedin_url`, poll/Q&A answers.

**Placeholder:**
- `[ATTENDEE_CSV_PATH]` — absolute path to the CSV on your machine

No MCP needed — the prompt reads the file directly.

### Option B — Zoom Webinar API

**What it does:** Pulls registrants + attendance data directly via Zoom's webinar API endpoints (registrants + participants).

**Setup:** Zoom Server-to-Server OAuth app with webinar read scopes.

**Placeholder:**
- `[ZOOM_WEBINAR_ID]` — the Zoom webinar ID

### Option C — Demio API

**What it does:** Pulls registrant + attendance data via Demio's event API.

**Setup:** Demio API key from your account settings.

**Placeholder:**
- `[DEMIO_EVENT_ID]` — the Demio event ID

## Claude in Chrome — Slack webhook

**What it does:** Posts to `hooks.slack.com` (the bash sandbox blocks this URL — Chrome is the reliable path).

**Setup:** [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome).

**Tools used:**
- `mcp__Claude_in_Chrome__tabs_context_mcp` — open/find a tab
- `mcp__Claude_in_Chrome__javascript_tool` — fire the Slack webhook

**Placeholders:**
- `[SLACK_WEBHOOK_URL]` — your incoming webhook
- `[WEBINAR_FOLLOWUP_CHANNEL]` — Slack channel where HOT-tier pings + run summary post
- `[AE_DM_USER_ID]` — the AE's Slack member ID for HOT-tier DMs
- `[ALERT_CHANNEL]` — Slack channel where errors go
- `[YOUR_SLACK_USER_ID]` — your Slack member ID (`U...`) so the bot @-mentions you on errors and summary

## WebSearch

Built into Claude Code. Used for:
- Pulling the webinar agenda / segment titles if you don't already have them (drives the in-moment HOT references)
- Surfacing recent press on the company a HOT registrant works at (for AE briefings)

No placeholder — just available.

## Webinar placeholders (set once per webinar)

These aren't MCP connectors but you need to set them in `prompt.md`:

- `[WEBINAR_NAME]` — human-readable webinar name (e.g. `AI-Native Outbound in 2026`)
- `[WEBINAR_DATE]` — `YYYY-MM-DD`
- `[WEBINAR_SLUG]` — lowercase, hyphen-separated, no spaces (e.g. `ai-native-outbound-2026`). Used in Zevari list names, Pipedrive tags, Instantly custom variables.
- `[WEBINAR_REPLAY_URL]` — the public replay link (goes into every tier's first email)
