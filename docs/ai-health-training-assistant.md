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

No always-on hardware at home, so this runs in the cloud. That's a
real tradeoff, and it's worth being precise about what you still get
and what you give up.

**What cloud can't give you:** cryptographic privacy. On a rented VPS
the provider could in principle read the disk. Automated jobs need
their decryption keys on the same box, so app-level encryption protects
against leaked backups and disk snapshots — not against a determined
host. If that's unacceptable, the only answer is hardware you own.

**What cloud can give you:** everything that actually matters in
practice — no health-data brokers, no ad-tech SDKs, no account linking,
no public attack surface, and nobody training on your data.

### The setup

- **One small VPS you control** — e.g. Hetzner CX22, ~€4/month, 2 vCPU
  / 4 GB. Docker Compose: Postgres + ingest API + job runner. Pick a
  region near you. Everything in one place, no managed-service sprawl.
- **No wearable aggregators** (Terra, Vital, Spike) and **no managed
  DB**. Every extra vendor is another copy of your health data.
- **Tailscale, not a public port.** This is the important one and it
  works exactly the same on a VPS as it would at home: install
  Tailscale on the VPS and your iPhone, bind the ingest API to the
  Tailscale interface only, and close every public port including SSH.
  Health Auto Export posts to a `100.x.x.x` address over an encrypted
  mesh — the endpoint is not on the public internet at all.
  Consequence: no Strava webhooks (they need public inbound), so poll
  every 30 min instead. Fine at this volume.
- **Pseudonymous schema.** No name, email, or DOB anywhere in the DB.
  It's a time series of numbers and dates. A breach yields *somebody's*
  HRV, not yours.
- **Encrypted backups** — restic to object storage with a key you hold
  and never upload.
- **Outbound only**: Strava API and the Claude API. Nothing else.
- **Delivery**: a Tailscale-only web dashboard, or self-hosted ntfy for
  push. A Telegram/Slack bot would route your brief through their
  servers — avoid.

### Using Claude as the model

Since local models are off the table without hardware, the API is the
right call. Two properties make it a reasonable one:

- **Anthropic does not train on API inputs or outputs** under standard
  commercial terms. This is the main thing that separates it from
  consumer apps.
- **Zero data retention** is available on request for eligible
  accounts, which removes even the abuse-monitoring retention window.
  Worth asking about once you're running.

Then reduce what you send anyway:

- **Send derived metrics, not raw data.** The daily call carries a few
  dozen numbers — readiness score, load ratio, this week's sessions,
  the WOD's `load_tags`. Not raw heart-rate streams, not a name, not an
  identifier.
- The **screenshot is the one exception** — it goes up as an image for
  parsing. Crop the gym name and leaderboard out first if you care;
  WOD content is far less sensitive than biometrics either way.

**Model and cost:** use `claude-opus-5` ($5/M input, $25/M output) for
both the screenshot parsing and the daily decision. At one brief a day
plus ~4 screenshots a month this lands around **$2–4/month** — the
reasoning quality is what makes the scheduler worth having, and
downgrading to save a dollar isn't a sensible trade at this volume.

**Total running cost: roughly €4 VPS + $2–4 API ≈ $8/month.**

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

With no fixed class days, the decision is much narrower than a general
weekly planner — and that makes it far more tractable.

**The actual decision space.** You WFH Monday and Tuesday, which are
the only mornings you can make a class. So each of those two days is
one slot with three options:

```
Monday   ∈ {class, run, rest}
Tuesday  ∈ {class, run, rest}
```

Typical weeks are *2 classes* or *1 class + 1 run*. Everything from
Wednesday on is driven by Runna's plan. That's **9 combinations** —
no solver needed, just enumerate and score.

**Hard constraints (code):**
- Class only on Monday/Tuesday (the WFH mornings).
- Runna's key sessions — long run and quality session — stay on their
  scheduled days; the Mon/Tue slots never displace them.
- At least one full rest day in the week.
- The class must actually be on the gym's schedule that morning.

**The core heuristic, and the reason this is worth building.** Your
WOD screenshot arrives for the whole week at once, which means on
Sunday the agent can already see what Monday's and Tuesday's workouts
are — and it knows what Runna has queued for Thursday and the weekend.
So it places the class on the day whose WOD *least* interferes with the
week's key running, and runs on the other:

| Monday WOD | Tuesday WOD | Call |
|---|---|---|
| Heavy squats + running | Gymnastics skill + short AMRAP | Run Mon, class Tue |
| Upper-body / skill | Heavy deadlifts, long metcon | Class Mon, run Tue |
| Both light | Both light | Two classes — bank the easy running volume later |
| Both heavy legs, quality run Thursday | | One class, one easy run — don't fry both days |

**Soft signals the model weighs on top:**
- Readiness (HRV vs. baseline, sleep debt, resting HR) — a bad
  Sunday night turns Monday into the rest option.
- ACWR — if you're spiking above ~1.4, bias toward the easy choice.
- What you actually did last week vs. what was suggested.

**Implementation:** enumerate the 9 combinations in code, filter by
hard constraints, score each against the interference heuristic, then
hand the surviving options plus context to the model to pick and
explain. The model can never propose something structurally invalid,
which is the usual failure mode of a pure-prompt scheduler.

Two outputs:
- **Sunday evening** — the week's plan, once the WOD screenshot lands.
- **Each morning** — a short confirm-or-adjust based on overnight
  readiness.

Roughly:

> **Monday — run, don't go to class.** This morning's WOD is back
> squats at 80% plus a leg-heavy metcon, and Runna has intervals
> Thursday — you'd be going in flat. Tuesday's WOD is handstand work
> and a 7-minute AMRAP, which is much easier to run alongside, so take
> the class then. Easy 6k this morning; HRV is at baseline and you
> slept 7.5h.

## Build order

1. **VPS + Tailscale + Postgres.** Close all public ports first, before
   any data exists. Schema for `biometrics`, `activities`, `wods`,
   `plan`, `preferences`.
2. **Health Auto Export** → Tailscale endpoint. Verify a week of
   HRV/sleep lands correctly.
3. **Strava OAuth + poller.** Backfill 90 days — you need history
   before any baseline or metric means anything.
4. **Screenshot pipeline** + review page for the weekly WOD.
5. **Metrics jobs** — readiness, ACWR, trends. Sanity-check them
   against days you remember feeling good or wrecked.
6. **Scheduler** — enumerate the Mon/Tue combinations, score them.
7. **Claude layer** — weekly plan + morning brief.
8. **Delivery** — Tailscale dashboard and/or ntfy push at 06:00.

Steps 1–3 give value on their own (one private store, real trends).
Don't build the agent before the data's trustworthy.

## Open questions

- What does the rest of your week look like — evening classes, or runs
  only from Wednesday on? Decides whether the agent plans two days or
  seven.
- Is a race on the calendar? Changes how hard the scheduler should
  protect Runna's key sessions over gym days.
- Roughly how many classes per week do you *want*? The heuristic needs
  a target to aim at, otherwise it optimises purely for recovery and
  will quietly talk you out of the gym.
