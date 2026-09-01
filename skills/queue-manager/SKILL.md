# Skill 6: Reading Queue Manager

## Purpose

Prevent backlog accumulation and answer: **What should I deep-read this week?**

Without active queue management, a PhD student accumulates hundreds of "I should read this" papers. This skill keeps the queue finite, prioritized, and honest.

## Trigger

- Weekly: every Friday (or Monday, user preference)
- Manual: "show my queue" or "what should I read this week"

## Frequency

Once per week.

---

## Input

```
From database:
  All papers with status in:
    [evaluated, saved, pdf_obtained, manually_added, bulk_skipped]
  Fields needed:
    relevance_score
    date_first_seen (for aging)
    project_match
    status
    paper_type

From researcher profile:
  active_projects (with stages)
  weekly_reading_capacity (default: 5)

From System Health Monitor:
  harvester_last_run
  scanner_last_run
  new_papers_unprocessed
```

---

## Queue Priority Scoring

```
priority_score =
    relevance_score                        (0-10, from Scanner)
  + project_urgency_bonus                  (0-2.0)
  + citation_momentum                      (0-0.5)
  - aging_penalty                          (0-1.0)
```

### Project Urgency Bonus

```
If paper matches an active_project:
  project stage multiplier:
    literature_review:  +2.0  (you need this NOW)
    writing:            +1.5  (might cite it)
    revision:           +1.0  (could strengthen argument)
    ideation:           +0.5  (useful background)
    submitted:          +0.0  (too late)
```

### Citation Momentum

```
If paper's cited_by_count has increased since last check:
  +0.5

Rationale: a paper gaining citations quickly may be
more important than its abstract-based score suggests.
(Requires periodic citation count updates from CrossRef)
```

### Aging Penalty

```
Days since date_first_seen:
  0-14 days:   no penalty
  15-30 days:  -0.3
  31-60 days:  -0.6
  61-90 days:  -0.8
  > 90 days:   -1.0

Papers that sit unread for months are either:
  (a) not actually important — let them sink
  (b) important but you keep avoiding them — surface in backlog alert
```

---

## Backlog Management

This is the mechanism that prevents infinite queue growth.

### Automatic Triage

```
Triggered when backlog (status in [evaluated, saved]) > 60 papers

Process:
  1. Re-score ALL backlog papers against CURRENT profile weights
     (interests may have shifted since first scoring)

  2. Papers that now score < 5.0:
     → auto-archive
     status = "archived"
     archive_reason = "below threshold on rescore"

  3. Papers > 90 days old AND score < 7.0:
     → auto-archive
     archive_reason = "aged out"

  4. Report results to user:
     "Archived [N] papers during backlog triage.
      [M] were low-relevance rescores, [K] were aged out.
      Remaining backlog: [X] papers."

  User can review archived papers anytime.
  Archiving is reversible — status can be changed back.
```

### Bulk-Skipped Paper Review

```
Papers with status = "bulk_skipped" (from Scanner degradation):
  Include in weekend review section of queue output
  Show titles + one-line Scanner reason
  User marks: keep (→ evaluated) or discard (→ archived)
```

---

## Output

```markdown
Weekly Reading Queue — Week of [Date]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

System Health
  Database total:          847 papers
  Harvester last run:      3 days ago ✓
  Scanner last run:        today ✓
  Unprocessed new papers:  6 ✓
  Total backlog:           34 papers (manageable)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

This Week's Deep Reads ([N] papers, matching your capacity of [N])

  1. "[Title]"
     [Authors] | [Journal]
     Priority: 9.4
     Why now: matches Ch.3 (writing stage) + Bayesian + victimization
     Days in queue: 4
     Paper type: empirical_quantitative
     Suggested layer: Layer 1+2

  2. "[Title]"
     [Authors] | [Journal]
     Priority: 8.8
     Why now: causal inference + gambling, relevant to proposal due Oct 15
     Days in queue: 11
     Paper type: methodological
     Suggested layer: Layer 1+2+3 (new method worth understanding deeply)

  3. "[Title]"
     Priority: 8.3
     Why now: life-course + aging intersection, novel dataset
     Days in queue: 7
     Suggested layer: Layer 1+2

  4-5. ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Next in Queue (if you finish early)
  6. "[Title]" — 7.9 — gender + experiments
  7. "[Title]" — 7.6 — spatial + crime mapping
  8. "[Title]" — 7.4 — victimization + qualitative

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Backlog Alert
  [N] papers have been in queue > 30 days.
  Recommended for archive (score < 7.0 on rescore):
    - "[Title]" — original score 7.2, rescore 5.8
    - "[Title]" — original score 6.9, rescore 4.3
    - "[Title]" — original score 7.0, aged 67 days
  → Archive these? (say "archive backlog" to confirm)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Bulk-Skipped Review (from Scanner degradation)
  [N] papers were bulk-skipped during a high-volume day.
  Quick review:
    - "[Title]" — [journal] — [one-line reason]
    - "[Title]" — [journal] — [one-line reason]
  → Keep or discard each? (say "keep 1, discard 2" etc.)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Weekly Stats
  Deep reads completed last week:  [N]
  Papers archived last week:       [N]
  New papers added last week:      [N]
  Queue trend: [growing / shrinking / stable]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Interest Drift (from Feedback Loop, monthly)
  [Only shown when Feedback Loop has updates]
  Topics you're reading more:  victimization, Bayesian
  Topics you're reading less:  spatial, AI
  Weight adjustments applied.
  → Override? (say "reset weights" to restore defaults)
```

---

## Database Updates

```yaml
After queue generation:
  No status changes (queue is read-only — it recommends,
  you decide)

After user confirms archive:
  status → "archived"
  archive_reason: [reason]
  date_last_updated: today

After bulk-skip review:
  kept papers: status → "evaluated"
  discarded papers: status → "archived"

Health signal:
  queue_generated: today
  backlog_count: [current count]
```

---

## Token Cost

```
Per run:       3,000 - 5,000 tokens
Frequency:     1x / week
Monthly total: ~20,000 tokens
```

---

## Interaction with Other Skills

```
Reads from:
  - Database (all non-archived, non-cited papers)
  - Researcher Profile (projects, capacity)
  - System Health Monitor (health status)
  - Feedback Loop Engine (weight changes, drift report)

Writes to:
  - Database (archive status changes)
  - System Health Monitor (queue health)

Triggers:
  - Deep Reader (user picks a paper from the queue)
  - Backlog triage (when backlog > 60)
```
