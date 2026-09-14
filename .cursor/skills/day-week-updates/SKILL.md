---
name: day-week-updates
description: >-
  Builds day-wise or full-week work update summaries from git history, release
  notes, PRs, and optional DCT/session sources in BST (Bangladesh Standard Time,
  Asia/Dhaka, UTC+6). Use when the user asks for today's update, end-of-day
  update, yesterday's update, last week's update, weekly summary, day-wise
  progress, standup notes, or "what did we ship this week" on any project.
---

# Day / Week Work Updates (BST)

Produce concise, human-readable progress summaries for standups, managers, or personal logs. Works in **any** git project.

## Timezone (mandatory)

- All day boundaries use **BST = Bangladesh Standard Time = `Asia/Dhaka` = UTC+6**.
- Never use British Summer Time or the machine's local TZ unless the user overrides.
- Resolve "today", "yesterday", and "last week" in BST before querying git.

### Resolve dates (run first)

```bash
# Today / yesterday in BST
TZ=Asia/Dhaka date '+%Y-%m-%d'                    # today
TZ=Asia/Dhaka date -d 'yesterday' '+%Y-%m-%d'     # yesterday (GNU date)
# Windows Git Bash fallback if -d fails:
TZ=Asia/Dhaka date -v-1d '+%Y-%m-%d'              # BSD-style; skip if unsupported
```

**Last week** (calendar week Mon–Sun in BST, unless user says otherwise):
- Prefer the previous Mon–Sun relative to "today" in BST.
- If user says "this week", use current Mon → today (BST).

Label every section with weekday + date, e.g. `Monday, 7 Sep 2026`.

## Modes

| User intent | Mode | Range |
|-------------|------|--------|
| "today's update", "EOD update", "end of day" | **Day** | That BST calendar day only |
| "yesterday's update" | **Day** | Previous BST day |
| "last week", "weekly summary", "day wise" | **Week** | Previous Mon–Sun (or stated range), **one section per day** |
| "this week so far" | **Week** | Current Mon → today BST |

If ambiguous, ask once: day vs week. Default to **Week (day-wise)** when they say "day wise" / "day by day".

## Data sources (in order)

1. **Git** (always) — primary source on every project  
2. **Release notes / CHANGELOG** — if present (`docs/**/releases/`, `CHANGELOG*`, `RELEASE*`)  
3. **Merged PRs** — `gh pr list --state merged --search ...` when `gh` works  
4. **DCT / session tools** — only if connected; never invent DCT data if auth fails  

Do not block the summary on DCT. If DCT fails, say so in one line and continue with git.

## Git collection

Prefer meaningful commits; de-noise sync/bot noise.

```bash
# Week (example: 2026-09-07 inclusive through 2026-09-14 exclusive)
git log --since='2026-09-07 00:00:00' --until='2026-09-14 00:00:00' \
  --pretty=format:'%ad|%an|%s' --date=format:'%Y-%m-%d' --all

# Day only (that BST day)
git log --since='2026-09-11 00:00:00' --until='2026-09-12 00:00:00' \
  --pretty=format:'%ad|%an|%s' --date=format:'%Y-%m-%d' --all
```

**Filter defaults** (unless user asks for everything):
- Drop merge commits for the bullet list (`--no-merges`) or omit "Merge pull request / Merge branch" lines
- Drop noise subjects: `Sync Live main`, empty/chore sync bots (keep if user asks for sync activity)
- Keep feature/fix/docs commits; rewrite subjects into plain English bullets

**Author filter:** if user says "my update", filter to their git identity (`git config user.name` / known aliases). Otherwise include the whole team and optionally group by author.

Use `--date=format` with times already aligned: when the repo stores UTC committers, still **bucket days by BST**. Prefer:

```bash
git log --since=... --until=... --pretty=format:'%aI|%an|%s' --all
```

Then convert each `%aI` to `Asia/Dhaka` date for bucketing. If conversion is hard in-shell, use `TZ=Asia/Dhaka git log --date=iso-local` / format that localizes, or state that day buckets use committer dates interpreted in BST.

## Synthesis rules

- Group by **day** (week mode) or single day (day mode).
- 2–6 bullets per day; merge duplicate/related commits into one outcome.
- Prefer outcomes ("Sequential leave approval + live refresh") over raw commit subjects.
- Mention PR numbers / release-note slugs when useful and known.
- Mark empty days: `No notable commits`.
- End week summaries with a one-line **Heaviest days** / highlight if helpful.
- Keep tone direct and concise — standup-ready, not a changelog dump.

## Output templates

### Day mode

```markdown
### [Weekday], [D Mon YYYY] (BST)
- Bullet outcome 1
- Bullet outcome 2

**Sources:** git [, release notes] [, PRs]
```

### Week mode (day-wise)

```markdown
**[Project / repo]** — [Mon D Mon] – [Sun D Mon YYYY] (BST)

### Monday, …
- …

### Tuesday, …
- …

…

**Heaviest days:** …
**Sources:** …
```

## Optional extras (only if asked)

- Filter to one author
- Post to DCT / Slack / ActiveCollab — **confirm before any write**
- Include hours / time-tracking if DCT or timesheets are available
- Compare vs previous week

## Anti-patterns

- Do not invent work that is not in git/notes/PRs/DCT
- Do not use UTC midnight as "the day" without converting to BST
- Do not paste long merge/sync commit lists
- Do not require project-specific paths; adapt to whatever repo is open

## Additional resources

- Output shape examples: [examples.md](examples.md)
