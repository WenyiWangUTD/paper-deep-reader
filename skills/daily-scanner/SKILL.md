# Skill 3: Daily Relevance Scanner

## Purpose

Answer: **Which papers should Wenyi look at today?**

This is the most frequently used skill. It evaluates unprocessed papers against the researcher's interests and active projects, scores them, and produces a prioritized daily reading list.

## Trigger

- Daily: morning routine
- Manual: "scan new papers" or "what should I read today"

## Frequency

Daily on weekdays. Can skip weekends and catch up Monday.

---

## Input

```
From database:
  All records where status = "new"

From researcher profile:
  static_interests
  learned_weights (adaptive, from Feedback Loop)
  active_projects (with keywords and stage)
```

---

## Scoring System

### Base Weights (Initial)

```yaml
topic_match:     0.35    # does the paper match your interest areas?
method_match:    0.25    # does it use methods you care about?
journal_tier:    0.15    # which tier is the journal in?
novelty:         0.10    # is this topic/method combination unusual?
project_match:   0.15    # does it match an active project?
```

These weights are the starting point. After 60 days of data, the Feedback Loop Engine (Skill 7) begins adjusting them based on actual reading behavior.

### Topic Matching

```
For each paper:
  Compare abstract + keywords against static_interests

  Score = sum of (match_strength × learned_weight) for each area

  match_strength:
    - exact keyword match in title:    1.0
    - keyword match in abstract:       0.7
    - related concept (semantic):      0.4
    - no match:                        0.0

  Weighted by learned_weights from researcher profile
```

### Method Matching

```
Scan abstract for methodology signals:

  High-value methods (for this researcher):
    Bayesian inference, causal inference, DiD, RDD, RCT,
    synthetic control, spatial analysis, experiments,
    machine learning, longitudinal/panel data

  Score:
    - Primary method matches researcher interest: 1.0
    - Secondary method matches: 0.5
    - Novel method not seen before: 0.8 (novelty bonus)
    - No method signal in abstract: 0.0
```

### Journal Tier Scoring

```
Tier 1 (core criminology):       1.0
Tier 2 (topical):                0.7
Tier 3 (methods + niche):        0.5
```

### Novelty Scoring

```
Novelty captures unusual combinations:
  - A method rarely used in criminology:  high novelty
  - A familiar topic with familiar method: low novelty
  - Cross-disciplinary paper:             high novelty

Score:
  - Topic × method combination seen < 3 times in database: 0.8-1.0
  - Seen 3-10 times: 0.4-0.6
  - Seen > 10 times: 0.0-0.2
```

### Project Match Bonus (NEW)

```
For each active_project in researcher profile:
  Compare paper keywords against project keywords

  If match found:
    bonus = base_bonus × stage_multiplier

    stage_multiplier:
      ideation:         0.5  (nice to know)
      literature_review: 1.5  (actively collecting)
      writing:          1.0  (may need to cite)
      revision:         0.8  (might strengthen argument)
      submitted:        0.3  (too late to add, but track)

  Add bonus to total score
```

### Final Score

```
final_score = (topic × 0.35) + (method × 0.25) + (journal × 0.15)
            + (novelty × 0.10) + (project × 0.15) + project_bonus

Normalize to 0-10 scale
```

---

## Adaptive Weight Updates

Every 30 days (triggered by Feedback Loop Engine):

```
For each interest area:
  read_rate = papers_with_this_topic_deep_read / papers_with_this_topic_recommended

  Adjustment:
    read_rate > 0.5:   weight += 0.02 (you consistently read these)
    read_rate 0.1-0.5: no change
    read_rate < 0.1:   weight -= 0.02 (you consistently skip these)

  Constraints:
    No weight below 0.03
    No weight above 0.25
    Normalize to sum = 1.0
```

The Scanner reports weight changes to the user so adjustments are transparent and overridable.

---

## Degradation Strategy

Normal daily volume: 10-15 new papers. Peak volume can reach 30-50.

```
Volume < 20 papers:
  Standard mode — full abstract scoring for all papers

Volume 20-40 papers:
  Stage 1: Title + keyword pre-filter
    - Discard papers matching zero interest areas by title
    - Keep all Tier 1 journal papers regardless
  Stage 2: Full abstract scoring on survivors only

Volume > 40 papers:
  Stage 1: Title-only filter (aggressive)
    - Keep only papers with title keywords matching interests
      OR from Tier 1 journals
  Stage 2: Abstract scoring on top 20 by title relevance
  Stage 3: Remaining papers flagged as "bulk_skipped"
    - Status set to "bulk_skipped" (not "evaluated")
    - Queue Manager includes these in weekend review list
```

---

## Output

```markdown
Daily Scan — [Date]

Papers evaluated: [N]
Your bandwidth: ~10-15 abstracts

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Recommended — Read (score > 8.0):

  1. "[Title]"
     [Authors] | [Journal] | [DOI]
     Score: 9.2
     Matched: victimization, Bayesian, longitudinal
     Project: Chapter 3 — strain theory (writing stage)
     → Action: Deep Read

  2. "[Title]"
     [Authors] | [Journal] | [DOI]
     Score: 8.5
     Matched: causal inference, gambling policy
     → Action: Deep Read

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Recommended — Save (score 7.0-8.0):

  3. "[Title]"
     Score: 7.6
     Matched: aging, life-course
     → Action: Save to Zotero, read when time allows

  4. "[Title]"
     Score: 7.2
     Matched: experiments, causal inference
     → Action: Save

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Monitor (score 5.0-7.0):
  - "[Title]" — 6.8 — spatial + crime mapping
  - "[Title]" — 5.4 — gender + qualitative
  [titles + one-line reason only]

Skipped (score < 5.0): [N] papers

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Scoring Drift Notice (if applicable):
  "Your read-rate for spatial criminology papers has been 5%
   over the past 60 days. Weight reduced: 0.10 → 0.08.
   If this is wrong, say 'reset spatial weight' to restore."
```

---

## Status Updates

```yaml
Papers scored > 7.0:
  status: new → evaluated
  relevance_score: [score]
  topic_scores: {area: score, ...}
  method_scores: {method: score, ...}
  project_match: [project_title if matched]

Papers scored < 7.0:
  status: new → evaluated
  relevance_score: [score]

Bulk-skipped papers (degradation):
  status: new → bulk_skipped

Health signal emitted:
  scanner_last_run: today
  new_papers_unprocessed: updated count
```

---

## Token Cost

```
Standard day (15 papers):
  Input:  15 abstracts × 500 tokens = 7,500
        + researcher profile:          800
        + scoring prompt:              500
  Output: 15 evaluations × 100 tokens = 1,500
  Total: ~10,000 tokens/day

Peak day (40 papers, with degradation):
  Stage 1 title filter: ~2,000 tokens
  Stage 2 (top 20): ~8,000 tokens
  Total: ~10,000-12,000 tokens (degradation keeps cost flat)

Monthly (22 weekdays):
  ~220,000 - 270,000 tokens
```

---

## Interaction with Other Skills

```
Reads from:
  - Database (papers with status = "new")
  - Researcher Profile (interests, weights, projects)

Writes to:
  - Database (status, scores, project_match)
  - System Health Monitor (scanner_last_run)

Triggers:
  - Feedback Loop Engine reads Scanner scores + user reading behavior
  - Queue Manager reads evaluated papers for weekly prioritization
```
