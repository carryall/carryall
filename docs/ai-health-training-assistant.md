# AI Health & Training Assistant — Architecture Research

Goal: combine data from Apple Watch (HealthKit), a Runna training plan
(synced to Strava), and CrossFit WODs logged in a PushPress-powered gym
app into one place, then use an LLM to generate coaching insights
(readiness, trends, plan adjustments).

## The core problem

None of these three sources talk to each other or expose a single easy
API for a personal project:

| Source | What you get | API reality |
|---|---|---|
| Apple Watch / HealthKit | HR, HRV, sleep, resting HR, VO2max, steps, workouts | No server-side API. Data only leaves the device via an iOS app (Apple's own or third-party) with HealthKit read permission. |
| Runna | Structured weekly running plan (paces, intervals, targets) | No public API. Runna auto-uploads *completed* runs to Strava/Apple Health, but the *planned* structure (paces/targets) is only visible in-app. |
| Strava | Completed activities (including Runna's uploads and Watch-recorded workouts) | Public REST API v3, OAuth2, webhooks for near-real-time sync. Well documented, easy to use. |
| PushPress | Gym class schedule, WOD, member scores | Has an API, but it's aimed at gym owners/franchises, not a public member-facing API. Access typically requires the gym to grant an API key. |

So Strava is the one reliable automated pipe. The other two need a
workaround.

## Three architecture options

### Option A — No-code/low-code (fastest to ship)
- **Apple Watch data**: use an app like *Health Auto Export* to push
  HealthKit metrics on a schedule to a REST endpoint / Google Sheet /
  Airtable.
- **Strava (covers Runna's completed runs)**: create a Strava API app,
  OAuth once, use webhooks to get new activities pushed automatically.
- **PushPress**: ask your gym to enable API/webhook access, or set up a
  Zapier/Make integration if their plan supports it; otherwise log WOD
  scores into a shared sheet manually (lowest effort, most reliable
  fallback).
- **Store**: Google Sheets / Airtable / Notion, or a small Supabase/
  Postgres instance if you want real querying.
- **Orchestration**: n8n or Make.com scenario that fetches/receives from
  each source, normalizes into one schema, and writes to the store.
- **AI layer**: a scheduled job assembles the week's data, sends it to
  the Claude API with a "coach" system prompt, and posts the digest to
  Telegram/Slack/email.

### Option B — Wearable-aggregator API (Terra / Vital / Spike)
These services normalize 20+ wearables (including Apple Health via
their own SDK) and Strava behind one API + webhooks, which removes the
need to build the HealthKit export path yourself. You'd still need a
custom path for PushPress since it's a niche gym platform none of them
cover. Worth it mainly if you want to add more wearables later.

### Option C — Custom app (most control, most work)
- A small companion iOS app (Swift) reads HealthKit directly and posts
  to your own backend (Supabase/Postgres + Cloudflare Workers or
  Vercel functions).
- Strava webhooks feed completed activities into the same backend.
- PushPress: same constraint as above — request API/webhook access
  from the gym, or fall back to manual logging.
- Unified schema: `activities` (date, source, type, metrics JSON) and
  `biometrics` (date, HR/HRV/sleep/VO2max/etc.).
- AI layer: Claude API with tool-calling against your DB, exposed via a
  simple chat frontend or bot, plus a scheduled weekly-digest job.

## Recommended pragmatic path

Given effort vs. payoff, start with a hybrid of A and B:

1. **Strava API app** — OAuth once, backfill history, subscribe to
   webhooks. This alone captures every Runna-driven run automatically.
2. **Health Auto Export → your store** — captures HRV, sleep, resting
   HR, VO2max, steps from the Watch on autopilot.
3. **PushPress** — email/ask the gym admin whether API or webhook
   access can be granted for your account; until then, log WOD scores
   into a shared sheet right after class (5 seconds of manual effort,
   zero engineering).
4. **Normalize** everything into one weekly rollup (running volume/pace
   trends, WOD scores/PRs, sleep & HRV trend, resting HR trend).
5. **Claude integration** — a scheduled script sends the weekly rollup
   plus your goals to the Claude API with a coaching system prompt,
   returning a readiness score, trend commentary, and suggested
   adjustments (e.g., "HRV down 15% this week + 3 hard WODs — consider
   an easy run instead of tomorrow's interval session").
6. **Delivery** — Telegram bot, email digest, or a tiny dashboard.
7. **Later**: correlate HRV/sleep trends against WOD/run intensity for
   overtraining-risk flags; add Runna plan details manually or via
   calendar export if you want the assistant to compare planned vs.
   actual.

## Key blockers to resolve first

- Confirm whether your specific PushPress-powered gym can grant API or
  webhook access — this determines whether WOD logging is automated or
  manual.
- Decide whether Runna's *planned* paces matter to the assistant, or
  whether "what actually happened" (via Strava) is enough — getting the
  plan itself requires manual entry or calendar export since there's no
  API.
