# Trainer Mode Design

**Date:** 2026-02-16
**Status:** Approved

## Overview

The peon IS your personal trainer. Pavel-style daily exercise mode where your Warcraft peon nags you to do 300 pushups and 300 squats throughout the day. Same orc who tells you "work work" now tells you to drop and give him twenty. ElevenLabs orc voice. Piggybacks on IDE hook events — no daemon, no cron.

The meme: "I replaced my personal trainer with a Warcraft peon."

## Config

New `trainer` section in `config.json`:

```json
{
  "trainer": {
    "enabled": false,
    "exercises": {
      "pushups": 300,
      "squats": 300
    },
    "reminder_interval_minutes": 20,
    "reminder_min_gap_minutes": 5
  }
}
```

- `exercises` — exercise name to daily goal. Default Pavel 300/300, configurable to 100 or 200.
- `reminder_interval_minutes` — how often to nag (clock time, checked on each hook event).
- `reminder_min_gap_minutes` — minimum gap between reminders to prevent spam during rapid task completions.

## State

Trainer state in `.state.json`:

```json
{
  "trainer": {
    "date": "2026-02-16",
    "reps": { "pushups": 75, "squats": 50 },
    "last_reminder_ts": 1739721600
  }
}
```

- Auto-resets when `date` doesn't match today.
- `last_reminder_ts` — unix timestamp for hybrid timing logic.

## Sounds

Orc peon voice lines generated via ElevenLabs (voice ID: `llD6rmWjUTFZjqbsK8zS`). The peon is your drill sergeant — broken English, orcish grunts, deadpan humor.

Trainer sounds live in a dedicated directory, separate from packs:

```
~/.claude/hooks/peon-ping/trainer/
  sounds/
    remind/          # Peon nagging you to do reps
    log/             # Peon acknowledging your work
    complete/        # Peon celebrating daily goal
    slacking/        # Peon disappointed in you
  manifest.json
```

### Voice Line Examples

**trainer.remind** (fired every ~20 min of coding):
- "You sit too long! Peon say do pushups NOW!"
- "Human weak! Need more squats! Peon demand it!"
- "Work work... on MUSCLES! Drop and give peon twenty!"
- "Peon no let you be lazy! Squat time!"
- "Me notice you just sitting there. Floor. Pushups. Now."
- "Something need doing? YES. PUSHUPS."
- "Peon tired of watching human just type type type. MOVE BODY!"
- "You think code review hard? Try three hundred squats!"

**trainer.log** (when you log reps):
- "Hmm, not bad for puny human."
- "Peon approve. Keep going!"
- "Work work! Muscles getting bigger maybe!"
- "Good good! Peon proud... a little."
- "Okie dokie. More later!"

**trainer.complete** (daily goal hit):
- "THREE HUNDRED! Human strong like orc now! ...almost."
- "Peon... Peon impressed. You done good today."
- "GOAL COMPLETE! Peon give you rest. For now."
- "Zug zug! Human finish all reps! Maybe you not so weak after all."

**trainer.slacking** (behind pace, past noon):
- "Peon very disappointed. You barely do anything today."
- "Half day gone and you still weak! MORE REPS!"
- "Human, you falling behind. Peon no like lazy."
- "Me not that kind of orc... but me WILL nag you more."

`manifest.json` maps categories to files:

```json
{
  "trainer.remind": [
    { "file": "sounds/remind/sit_too_long.mp3", "label": "You sit too long! Peon say do pushups NOW!" },
    { "file": "sounds/remind/human_weak.mp3", "label": "Human weak! Need more squats!" },
    { "file": "sounds/remind/work_on_muscles.mp3", "label": "Work work... on MUSCLES!" },
    { "file": "sounds/remind/no_let_lazy.mp3", "label": "Peon no let you be lazy! Squat time!" },
    { "file": "sounds/remind/just_sitting.mp3", "label": "Me notice you just sitting there." },
    { "file": "sounds/remind/something_need_doing.mp3", "label": "Something need doing? YES. PUSHUPS." },
    { "file": "sounds/remind/type_type_type.mp3", "label": "Peon tired of watching human type type type." },
    { "file": "sounds/remind/code_review_hard.mp3", "label": "You think code review hard? Try three hundred squats!" }
  ],
  "trainer.log": [
    { "file": "sounds/log/not_bad.mp3", "label": "Not bad for puny human." },
    { "file": "sounds/log/approve.mp3", "label": "Peon approve. Keep going!" },
    { "file": "sounds/log/work_work.mp3", "label": "Work work! Muscles getting bigger maybe!" },
    { "file": "sounds/log/proud.mp3", "label": "Good good! Peon proud... a little." },
    { "file": "sounds/log/okie_dokie.mp3", "label": "Okie dokie. More later!" }
  ],
  "trainer.complete": [
    { "file": "sounds/complete/three_hundred.mp3", "label": "THREE HUNDRED! Human strong like orc now!" },
    { "file": "sounds/complete/impressed.mp3", "label": "Peon... Peon impressed." },
    { "file": "sounds/complete/goal_complete.mp3", "label": "GOAL COMPLETE! Peon give you rest." },
    { "file": "sounds/complete/zug_zug.mp3", "label": "Zug zug! Human finish all reps!" }
  ],
  "trainer.slacking": [
    { "file": "sounds/slacking/disappointed.mp3", "label": "Peon very disappointed." },
    { "file": "sounds/slacking/half_day.mp3", "label": "Half day gone and you still weak!" },
    { "file": "sounds/slacking/falling_behind.mp3", "label": "Human, you falling behind." },
    { "file": "sounds/slacking/not_that_kind.mp3", "label": "Me not that kind of orc... but me WILL nag you." }
  ]
}
```

## CLI

```bash
peon trainer on                  # enable trainer mode
peon trainer off                 # disable trainer mode
peon trainer status              # show today's progress
peon trainer log 25 pushups      # log 25 pushups
peon trainer log 30 squats       # log 30 squats
peon trainer goal 200            # set both exercises to 200
peon trainer goal pushups 100    # set just pushups to 100
```

`peon trainer status` output example:

```
Peon Trainer -- 2026-02-16

  pushups:  ████████░░░░░░░░  125/300  (42%)
  squats:   ██░░░░░░░░░░░░░░   50/300  (17%)

  Next reminder in ~12 min
```

## Reminder Logic

On every hook event, after normal sound logic:

1. Check `trainer.enabled` — if false, skip.
2. Check `trainer.date` vs today — if different, reset reps to 0.
3. Check if daily goal already hit for all exercises — if so, skip reminders.
4. Calculate time since `last_reminder_ts`.
5. If `>= reminder_interval_minutes` AND `>= reminder_min_gap_minutes`:
   - Pick random sound from `trainer.remind` (or `trainer.slacking` if behind pace).
   - Output trainer sound variables.
   - Update `last_reminder_ts`.
   - Desktop notification with progress summary.

**Pace check:** If current time is past noon and reps below 25% of goal, use `trainer.slacking`.

**Sound priority:** Trainer reminder plays after the normal hook sound with ~1s delay. Skip trainer reminder on `session.start` to avoid stacking.

**On rep log:** Play `trainer.log` sound. If log pushes total to goal, play `trainer.complete` instead.

## Files Touched

- `peon.sh` — trainer logic in Python block + CLI subcommand + delayed trainer sound playback
- `config.json` — add `trainer` section defaults
- `tests/` — new BATS tests for trainer logic
- `README.md` — trainer section in docs

## Not In Scope

- No daemon or background process.
- No generalized exercise framework — Pavel pushups + squats with configurable goal number.
- No web UI or dashboard.
- No trainer-specific mobile notifications (reuses existing mobile infra).
