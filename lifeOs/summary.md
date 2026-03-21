---
name: summary
description: Query your Life OS session logs and generate a dashboard.md with stats, trends, and insights across climbing and work. Run weekly or on demand to see patterns in your data.
---

# Summary

Read your session logs and generate a fresh dashboard.md with stats,
trends, and insights.

---

## Step 1 — Read the schema

Read `/Users/ruben_gardner/personalProject/docs/SCHEMA.md` to
understand field names, types, and allowed values.

---

## Step 2 — Collect sessions

Scan the following folders recursively for all `.md` files:

```
climbing/
work/sessions/
```

For each file:

1. Parse the YAML frontmatter
2. Store all fields in memory keyed by `date` and `type`
3. Skip any file that has no frontmatter or no `date` field —
   print a warning but continue

Build two collections:

- `climbing_sessions[]`
- `work_sessions[]`

---

## Step 3 — Compute climbing stats

### Volume

- Total sessions all time
- Sessions in last 7 days
- Sessions in last 30 days
- Current streak (consecutive days with a session)
- Longest streak ever

### Grades

- Best grade ever logged (`best_grade` max)
- Best grade in last 30 days
- Best flash grade ever
- Grade progression: group `best_grade` by month, show trend

### Energy and quality

- Average `energy` last 30 days vs all time
- Average `session_quality` last 30 days vs all time
- Distribution of session quality scores (1–5)

### Tags

- Top 10 most used session-level tags all time
- Top 5 route-level tags all time
- Tags trending up in last 30 days vs previous 30 days

### Board breakdown

- Sessions per board (kilter / tension / moon)
- Average quality per board
- Most used angle per board

---

## Step 4 — Compute work stats

### Volume

- Total sessions all time
- Sessions in last 7 days
- Sessions in last 30 days
- Current streak

### Focus and energy

- Average `focus_quality` last 30 days vs all time
- Average `energy` last 30 days vs all time
- Best focus days — day of week with highest average focus

### AI usage trend

- Distribution of `ai_usage` (high/medium/low/none) last 30 days
- Trend: is AI usage going up, down, or flat month over month

### Projects

- All active projects (seen in last 30 days)
- Sessions per project
- Average quality per project
- Last session date per project

### Session types

- Distribution of `session_type` last 30 days

---

## Step 5 — Compute cross-domain insights

These are the most valuable stats. Compute:

### Energy correlation

- Average climbing `session_quality` on days after high-energy
  work sessions (energy ≥ 4) vs low-energy (energy ≤ 2)
- Average work `focus_quality` on days after climbing sessions
  vs rest days

### Weekly load

- For each of the last 8 weeks compute:
  - Climbing sessions count
  - Work sessions count
  - Average energy across both
  - Average session quality across both

### Best week pattern

- Find the single best week (highest combined average quality)
- List what was true that week: energy, volume, tags, notes

### Readiness trend

- Plot `energy` across both domains over last 60 days
- Flag any 7-day window where average energy dropped below 2.5
  — label these "low periods"

---

## Step 6 — Generate insights

Read all the computed stats and write 3–5 natural language insights.
These should be specific and data-driven, not generic.

Good examples:

```
→ Your best climbing sessions follow rest days from work
  (quality 4.2 avg vs 3.1 on same-day work sessions)

→ You climb 0.5 grades harder at 40° than 30° on the Kilter —
  your last 6 sessions at 40° averaged V8, vs V7 at 30°

→ Focus quality has dropped from 3.8 to 2.9 over the last
  3 weeks — your energy scores suggest accumulated fatigue

→ heel-hook is your most logged route tag but you haven't
  tagged a breakthrough in 4 weeks
```

Bad examples (too generic):

```
→ You have been climbing consistently       ← no data
→ Consider resting more                     ← not grounded
→ Great work this month!                    ← meaningless
```

Only write insights you can back with specific numbers from the data.
If the data is too sparse for a meaningful insight, skip it.

---

## Step 7 — Write dashboard.md

Write to `dashboard.md` at the repo root. Overwrite completely on
each run — it is always a generated file, never hand-edited.

Add a generation timestamp at the top so you always know when it
was last run.

```markdown
---
generated: YYYY-MM-DD HH:MM
sessions_scanned: { n }
period: all time + last 30 days
---

# Life OS Dashboard

_Generated {date} — {n} sessions scanned_

---

## Climbing

### Volume

| Period       | Sessions | Streak         |
| ------------ | -------- | -------------- |
| Last 7 days  | 2        | 3 days         |
| Last 30 days | 9        | —              |
| All time     | 47       | 12 days (best) |

### Grades

| Metric           | Value |
| ---------------- | ----- |
| Best grade ever  | V9    |
| Best grade (30d) | V8    |
| Best flash ever  | V7    |

### Grade progression

| Month   | Best grade |
| ------- | ---------- |
| 2026-01 | V7         |
| 2026-02 | V8         |
| 2026-03 | V8         |

### Energy and quality (last 30 days)

| Metric      | Last 30d | All time |
| ----------- | -------- | -------- |
| Avg energy  | 3.4      | 3.2      |
| Avg quality | 3.8      | 3.5      |

### Board breakdown

| Board   | Sessions | Avg quality | Most used angle |
| ------- | -------- | ----------- | --------------- |
| Kilter  | 24       | 3.9         | 40°             |
| Moon    | 18       | 3.6         | 25°             |
| Tension | 5        | 3.4         | 35°             |

### Top tags (last 30 days)

overhang ×12 · crimpy ×8 · heel-hook ×6 · flow ×4 ·
felt-strong ×3

---

## Work

### Volume

| Period       | Sessions |
| ------------ | -------- |
| Last 7 days  | 4        |
| Last 30 days | 16       |
| All time     | 63       |

### Focus and energy (last 30 days)

| Metric     | Last 30d | All time |
| ---------- | -------- | -------- |
| Avg focus  | 3.2      | 3.5      |
| Avg energy | 3.0      | 3.3      |
| Best day   | Tuesday  | —        |

### AI usage (last 30 days)

high ×8 · medium ×5 · low ×2 · none ×1

### Projects (last 30 days)

| Project       | Sessions | Avg quality | Last session |
| ------------- | -------- | ----------- | ------------ |
| project-atlas | 9        | 3.8         | 2026-03-20   |
| life-os       | 7        | 4.2         | 2026-03-21   |

---

## Cross-domain

### Weekly load (last 8 weeks)

| Week of    | Climbing | Work | Avg energy | Avg quality |
| ---------- | -------- | ---- | ---------- | ----------- |
| 2026-03-16 | 3        | 4    | 3.6        | 3.9         |
| 2026-03-09 | 2        | 5    | 3.1        | 3.4         |

...

### Energy trend (last 60 days)

[climbing energy and work energy side by side, one row per week]

---

## Insights

→ {insight 1}
→ {insight 2}
→ {insight 3}

---

_Run `/summary` to regenerate_
```

---

## Step 8 — Confirm

Print:

```
Dashboard written: dashboard.md
Scanned: {n} climbing sessions, {n} work sessions
Period: {earliest date} → {today}
Insights generated: {n}

⚠️  {n} files skipped (no frontmatter) — run `/validate-tags`
    to check for format issues
```

---

## Notes

- dashboard.md is always generated, never hand-edited — treat it
  like a build artifact
- If fewer than 5 sessions exist for a domain, print the stats
  but skip insights for that domain — not enough signal yet
- All averages round to 1 decimal place
- Dates in tables are YYYY-MM-DD
- If a field is missing from a session, exclude that session from
  averages for that field only — do not skip the whole session

```

---

You now have three skills that form a complete loop:
```

/log → capture
/validate-tags → maintain
/summary → reflect
