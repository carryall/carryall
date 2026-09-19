# AI Health & Training Assistant — Architecture

Goal: combine Apple Watch (HealthKit) biometrics, a Runna running plan
(synced to Strava), and CrossFit WODs from a PushPress gym app into one
private store, then use an agent to (a) surface insights and (b) decide
each day whether to run, hit a class, or rest.

Requirements driving this design:
1. PushPress data arrives as **weekly WOD screenshots**.
2. **Personal data stays private** — no third-party health aggregators.
3. **Insights** from the accumulated data.
4. A **scheduling agent** that respects fixed class-day preferences.

## Source reality

| Source | What you get | How it gets in |
|---|---|---|
| Apple Watch / HealthKit | HRV, sleep, resting HR, VO2max, HR zones, workouts | No server API. *Health Auto Export* (iOS) POSTs JSON to **your own** endpoint on a schedule. |
| Strava | Completed runs, incl. everything Runna uploads | Public REST API v3 + OAuth2. Poll every 30 min (avoids exposing a public webhook URL). |
| Runna | Planned sessions (paces, intervals, targets) | No API. Either parse the weekly plan from a screenshot (same pipeline as PushPress) or type the week's key sessions in once. |
| PushPress | Weekly WOD + class schedule | **Screenshot → vision model → structured JSON.** No API access needed. |

Strava is the only unavoidable third party, and your runs are already
there. Everything else stays on hardware you own.

## Privacy model

Draw a hard line: **health data never leaves your network.**

- **Runs at home** on an always-on box — Mac mini, NAS, Raspberry Pi 5,
  or an old laptop. Docker Compose: Postgres + ingest API + job runner
  + Ollama.
- **No wearable aggregators** (Terra, Vital, Spike). They're convenient
  but they hold a copy of your health data — disqualified.
- **Health Auto Export** points at a LAN address (or a Tailscale IP), so
  Watch data goes phone → your box, nothing in between.
- **Remote access** via Tailscale, not a public port. No inbound
  exposure, so also no Strava webhooks — poll instead.
- **Backups** encrypted (restic/borg to a local disk or encrypted
  object storage).

### The one real decision: which model runs the agent

| | Local model (Ollama) | Claude API |
|---|---|---|
| Data leaves your box | Never | Yes — the daily context window |
| Quality of reasoning/narration | Good enough for summaries; weaker on nuanced multi-constraint scheduling | Noticeably better |
| Vision (screenshot → JSON) | Qwen2.5-VL / InternVL work, needs verification step | More reliable |
| Cost | Electricity | Cents per day at this volume |
| Hardware | ~16GB+ RAM for a useful model | None |

**Recommended:** start fully local. A 7–14B model is plenty for the
narration layer, because the analysis itself is computed in code (see
below) — the model only explains numbers and applies scheduling rules.
If you later want sharper reasoning, a middle path is to send only
**derived metrics** (readiness score, load ratio, planned sessions) and
never raw biometrics or identity — a few dozen numbers, no PII.

> Note: if you use a bot for delivery, Telegram/Slack means your daily
> brief transits their servers. Self-hosted **ntfy** or a local web
> dashboard keeps that private too.

## Screenshots → structured data

This applies to both the PushPress WOD and (optionally) the Runna week.

```
iPhone screenshot
  → shared album / Syncthing / iCloud folder watched by the box
  → vision model extracts JSON
  → confidence check + your one-tap confirm
  → wod table
```

Target schema per day:

```json
{
  "date": "2026-09-21",
  "source": "pushpress_screenshot",
  "title": "Fran-ish",
  "blocks": [
    {"type": "strength", "movement": "back squat", "scheme": "5x5 @80%"},
    {"type": "metcon", "movements": ["thruster", "pull-up"],
     "scheme": "21-15-9", "time_domain_min": 8}
  ],
  "load_tags": ["legs_heavy", "pulling", "high_glycolytic"],
  "confidence": 0.86
}
```

Two things make this work in practice:

- **`load_tags` are the point.** The scheduler doesn't need to
  understand "Fran"; it needs to know tomorrow is heavy legs and
  high-intensity, which conflicts with a Runna interval session.
- **Always review before commit.** OCR on a stylised gym app will
  misread reps and weights. A tiny local web page showing
  screenshot-next-to-parsed-JSON with an Approve button removes the
  whole class of silent-bad-data problems. Anything under ~0.8
  confidence gets flagged.

## Insights layer

Critical rule: **compute metrics in code, let the model narrate.** LLMs
are unreliable at arithmetic over time series, and these numbers drive
decisions.

Computed nightly (Python + SQL):

- **Readiness** — HRV vs. your 60-day rolling baseline (z-score),
  resting HR delta, sleep duration/debt over 3 days.
- **Training load** — session TRIMP or Strava's relative effort;
  **acute:chronic workload ratio** (7-day ÷ 28-day). Above ~1.5 is the
  classic spike-injury zone.
- **Running trends** — weekly volume, pace at given HR (aerobic
  efficiency drift), time in zones vs. plan.
- **Strength/CrossFit** — movement-pattern frequency (are you squatting
  4×/week and never hinging?), PR tracking, days since heavy legs.
- **Interference flags** — heavy-leg WOD within 24h of a quality run
  session, or two high-intensity days back to back.

The model then turns this into a few sentences of plain-language
commentary plus a weekly review. Feed it the numbers, never the raw
rows.

## The scheduling agent

Same split: **hard rules in code, judgement in the model.**

**Hard constraints (code — never negotiable):**
- Your fixed preferences, e.g. *"always Wednesday 18:00 class"*,
  *"Saturday is the long run"*, *"Sunday off"*.
- Runna's key sessions (long run + quality session) stay on their days.
- Max N training days/week, minimum 1 full rest day.
- Class must actually exist on the schedule that day.

**Soft signals (model weighs these):**
- Today's readiness score and its trend.
- What the WOD actually is — a skill/gymnastics day next to an interval
  run is fine; a heavy-legs metcon the day before isn't.
- ACWR trend — if you're spiking, it should be biasing toward easy.
- What you skipped recently.

Implementation: generate the week's *feasible* schedules in code (the
constraint solver — this is a small enough search space to brute-force),
then hand the model the feasible set plus context and ask it to pick and
explain. This guarantees it can never propose something that violates a
preference, which is the usual failure mode of a pure-prompt scheduler.

Daily output, roughly:

> **Tuesday — go to the gym.** Today's WOD is gymnastics-skill +
> short AMRAP, low leg load, so it won't compromise Thursday's
> intervals. HRV is back to baseline after two suppressed days and
> you've had 7.5h sleep. Skipping the easy 5k is fine — you're at 42km
> this week vs. a 38km average, and your load ratio is 1.4.

## Build order

1. **Postgres + ingest API** on the box; schema for `biometrics`,
   `activities`, `wods`, `plan`, `preferences`.
2. **Health Auto Export** → LAN endpoint. Verify a week of HRV/sleep
   lands correctly.
3. **Strava OAuth + poller.** Backfill 90 days for baselines — you need
   history before any metric means anything.
4. **Screenshot pipeline** + review page for the WOD.
5. **Metrics jobs** — readiness, ACWR, trends. Sanity-check them against
   days you remember feeling good or wrecked.
6. **Scheduler** — encode your preferences, generate feasible weeks.
7. **Model layer** — narration + daily pick, via Ollama.
8. **Delivery** — local dashboard and/or ntfy push at 06:00.

Steps 1–3 give value on their own (one private store, real trends).
Don't build the agent before the data's trustworthy.

## Open questions

- What always-on hardware do you have, and how much RAM? Decides local
  model size — or whether we do a small always-on box + derived-metrics
  API.
- Which class days are truly fixed vs. preferred-but-movable?
- Is a race on the calendar? Changes how aggressively the scheduler
  should protect Runna's key sessions over gym days.
