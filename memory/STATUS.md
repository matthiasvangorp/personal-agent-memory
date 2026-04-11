# Project status

Updated 2026-04-11 by Matthias + Claude resuming for Phase 2 self-gen planning.

## Where we are

**Phase 1 v1 shipped + dashboard tier 1 & 2 done + Project subsystem landed.**
All core infrastructure is in production: Telegram bot (long-poll), Whisper
voice-in, scheduled heartbeat with sleep-gap detection, Apple Health ingest via
HAE webhook, the approval gate, the LogWatcher self-healing loop, the daily
HealthDiary check-in, and a Livewire dashboard at `https://agent.test/dashboard`.

The `Project` subsystem (`app/Project/`, `app/Models/Project.php`,
`ProjectLauncher` skill, migration `2026_04_10_200000_create_projects_tables`)
is our local take on OpenClaw-style agentic work: long-running goal-oriented
projects with their own workspace, heartbeat cadence, and tool loop. One
project = one goal + one workspace dir under `storage/agent/projects/<slug>/`
+ one cron cadence. Each tick is a fresh `AnonymousAgent` session that reads
`HEARTBEAT.md` (source of truth across ticks), does one piece of work, writes
a journal entry, and rewrites `HEARTBEAT.md` for next tick. Tools available
inside a project tick: `WebFetch`, `WorkspaceFile`, `AskUser` (blocks the
project until user replies via Telegram). This replaces the "main agent plans
everything in one megaprompt" anti-pattern with stateless tick loops +
workspace-as-memory, same shape as OpenClaw but stripped down.

## Skills in production (9)

| Skill | Cadence | Approval? | What it does |
|---|---|---|---|
| `SystemSanityCheck` | every 5 min | no | Ollama / Anthropic / disk / heartbeat freshness; Telegrams on failure only |
| `HomeAssistantAnomalyWatcher` | 15 min state + daily 08:00 | no | Bedroom overnight temp >23°C + Gree A/C runtime anomaly (`climate.bedroom`, `climate.living_room`) |
| `HealthInterpreter` | daily 08:00 | no | LLM-driven Apple Health summary from `health_metrics`/`health_workouts`, silent unless notable |
| `HealthDiary` | daily 10:00 | no | Morning check-in via Telegram → Claude routes the reply via PersonalAgent runtime hint → tool writes SQLite + memory/diary/YYYY-MM-DD.md, commits to memory repo |
| `MemoryEditor` | on-demand | yes | Append a bullet to a whitelisted memory file (USER.md, AGENTS.md, travel.md, skills/*.md), commits to memory repo |
| `LogWatcher` | every 30 min | yes | Reads laravel.log, asks Claude for `old_text`/`new_text` fix, validates safety rails, drafts approval, applies patch + commits + pushes to app repo |
| `SendReminder` | on-demand | yes | Throwaway approval-gate demo, no real reminder system |
| `WeatherBriefing` | daily 07:45 | no | Fetches Budapest open-meteo forecast and narrates it via Ollama (`qwen2.5:14b`) into a short Telegram message. First skill routed to Ollama by default via `ModelRouter::SITE_DEFAULTS`. |
| `ProjectLauncher` | on-demand | yes | Main-agent → Project subsystem bridge. Creates a `Project` row, seeds the workspace + `HEARTBEAT.md` with the user's goal, runs the first tick immediately so the project can interview the user via `AskUser`. After it returns, the project lives on its own cron cadence via `next_heartbeat_at`. |

## Architecture summary

- Laravel 13 + Laravel AI SDK (`laravel/ai` ^0.5) + Livewire ^4.2 + Horizon
- Anthropic Claude Sonnet 4.6 default; Ollama configurable per call site via the
  ModelRouter (`storage/agent/model_routing.json`)
- All in Docker: `app` (Apache+PHP 8.3), `redis`, `horizon`, `scheduler`,
  `telegram` (long-poll), `whisper` (faster_whisper, profile `voice`)
- SQLite at `storage/agent/agent.sqlite` (gitignored). Memory markdown at
  `storage/agent/memory/*.md` (separate git repo, pushed to
  `github.com/matthiasvangorp/personal-agent-memory`)
- App code pushed to `github.com/matthiasvangorp/agent` via deploy key
- Whitelisted Telegram user only; LAN-only HTTPS dashboard at `agent.test`

## Two agent modes — Skills vs Projects

- **Skills** (`app/Skills/`) = short, scoped capabilities the main `PersonalAgent`
  can call as tools. Either on-demand (`SendReminder`, `MemoryEditor`) or
  scheduled via `HasHeartbeat` (`WeatherBriefing`, `LogWatcher`). State lives
  in `storage/agent/skills/<slug>/` JSON or SQLite tables. These are small.
- **Projects** (`app/Project/`) = long-running OpenClaw-style agent loops, each
  with its own goal, workspace, heartbeat cadence and tool loop. State lives
  in `storage/agent/projects/<slug>/` as markdown (workspace-as-memory). The
  main agent spawns a project via `ProjectLauncher`; after that, the project
  ticks on its own cadence and can reach back to the user via `AskUser`.
- Rule of thumb: if the work is a one-shot capability or a recurring watcher,
  it's a Skill. If it's a goal that needs to be pursued over hours/days/weeks
  with its own memory and plan, it's a Project.

## Dashboard pages live

Tier 1:
- `/dashboard` — overview (heartbeat, cost MTD+projected, approvals count, skill grid, diary, Apple Health card)
- `/dashboard/models` — per-call-site provider+model swap with Test action
- `/dashboard/approvals` — pending list with LogWatcher diff blocks + MemoryEditor previews, inline approve/reject, 14-day history
- `/dashboard/conversations` + `/conversations/{id}` — list + drill-down with role bubbles, inline tool calls/results, per-turn token counts

Tier 2 (shipped 2026-04-10):
- `/dashboard/activity` — 30d cost bar chart + 48h heartbeat scatter (ticks vs gaps, one lane per skill). Chart.js 4 + date-fns adapter via CDN in layout.blade.php.
- `/dashboard/logs` — skill log tail viewer, efficient reverse-read, 150 lines, 10s auto-refresh
- `/dashboard/memory` — direct textarea editor for USER.md/AGENTS.md/travel.md/skills/*.md. Save = write to disk + git commit + push to memory repo. Bypasses the approval gate on purpose — LAN-only single-user, and every save is already an undo-able git commit. HEARTBEAT.md is not in the whitelist.
- `/dashboard/diary` — monthly calendar of health_diary_entries with energy-colored cells, prev/next nav, click-to-view detail pane. First real entry arrives 2026-04-11 10:00.

## Numbers

- ~2,900 LOC for Phase 1 + tier 2 dashboard; Project subsystem added more on top
- Cost so far: ~$0.37 on the Phase 1 build day, projected ~$1-2/month (well under $20 budget)
- Telegram conversation: 24 messages, 110k input / 2.5k output tokens during build day
- 9 skills, 9 dashboard routes, 11+ migrations, 3 Project tools (`WebFetch`, `WorkspaceFile`, `AskUser`)

## Shared helpers

- `app/Services/Memory/MemoryRepo.php` — whitelist, read/write, commitAndPush for the memory git repo. Used by the `MemoryEditor` skill (append-bullet path) and the `/dashboard/memory` Livewire editor (free-form edit path). Single source of truth for the whitelist.

## Open / pending

- **HealthDiary first real entry** — fires automatically tomorrow 10:00
  Budapest local. Today's smoke-test entry was deleted.
- **HAE automation missing fields** — user has not yet enabled Sleep Analysis
  and Workouts in the Health Auto Export iPhone automation. Until they do,
  HealthInterpreter snapshots will be missing those metrics.
- **Reminder for 2026-04-24** — once ~2 weeks of HealthDiary entries exist,
  propose building a `HealthDiaryReview` skill that cross-references diary
  entries with `health_metrics` (HRV, sleep_analysis) for correlations.
  Note duplicated in AGENTS.md "Pending work".
- **Tailscale auth was broken** at the time of this build. The Telegram bot
  uses long-poll instead of webhook. When Tailscale Funnel is fixed, swapping
  to webhook mode is a small addendum (~20 LOC).
- **Tailwind via CDN** in `dashboard/layout.blade.php` — works fine, browser
  console warns "should not be used in production". ~10 min to swap for Vite.

## What's next (priority order)

1. **(Phase 2 self-generation, ~half day)** Build `meta:create_skill`: takes a
   spec, reads the 9 existing skills as few-shot examples, asks Claude to
   generate a new Skill class, static-validates it, drafts an approval with the
   full generated source, and on approval writes it to disk, runs
   `composer dump-autoload`, commits + pushes to the app repo, and terminates
   Horizon so the container restart picks it up via `SkillServiceProvider`
   auto-discovery. Validate on one small skill (e.g. `RssDigest`) before
   pointing it at anything complex. *Scope note:* this generates Skills, not
   Projects. Projects are still launched manually via `ProjectLauncher`.
2. **(Phase 1.5 IBKR, ~1 day)** Python `ib_insync` sidecar +
   `PortfolioMorningBriefing` + `WheelStrategyTracker` + `PortfolioMomentumWatcher`.
   User runs the wheel strategy.
3. **(USER.md polish)** Sections "How to talk to you", "When to interrupt you",
   "What to prioritise", "What you don't want" still mostly templated. The
   filled-in sections (Identity, family, work) are real. The agent reads this
   every prompt — it makes a real difference.

## How to resume

A fresh session should be able to pick up from this file alone. Start by
reading USER.md, AGENTS.md, this STATUS.md, and the skills under
`memory/skills/`. The 9 skills in `app/Skills/` are the canonical examples
for any new Skill work; `app/Project/ProjectAgent.php` + `ProjectLauncher.php`
are the canonical example for any new Project work. The README in the app
root has the daily commands.
