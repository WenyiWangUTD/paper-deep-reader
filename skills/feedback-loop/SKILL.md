# Skill 7: Feedback Loop Engine

## Purpose

Make the system adaptive. Track what you actually read vs. what was recommended, and update scoring weights so the system improves over time.

This skill did not exist in the original design. Without it, the Scanner's scoring weights are static and will drift away from your actual interests as your PhD progresses.

## Trigger

- Automatic: every 30 days
- Manual: "show my reading behavior" or "update my weights"

## Frequency

Once per month. Requires at least 60 days of data for the first meaningful update.

---

## Signals Collected

### Positive Signals (you care about this)

```yaml
strong_interest:
  - Paper status changed to "deep_read"
    signal_strength: 1.0
  - Paper status changed to "cited"
    signal_strength: 1.5 (highest — you used it in your work)

moderate_interest:
  - Paper saved to Zotero but not yet deep_read
    signal_strength: 0.5
  - Paper opened same day it was recommended
    signal_strength: 0.3 (urgency signal)

project_signal:
  - Paper tagged to active_project by user
    signal_strength: 0.8
```

### Negative Signals (you don't care about this)

```yaml
weak_match:
  - Paper recommended (score > 7.0) but still "evaluated" after 14 days
    signal_strength: -0.3
  - Paper archived without reading
    signal_strength: -0.5

scoring_error:
  - User manually overrides a score (up or down)
    signal_strength: varies
    (treat as direct correction — strongest feedback signal)
```

### Neutral Signals (ambiguous)

```yaml
- Paper saved but not deep_read after 30 days
  → could be "want to read but busy" vs. "not that interesting"
  → do not count as negative until 60 days
```

---

## Weight Update Algorithm

Runs every 30 days.

### Step 1: Compute Read Rates

```
For each interest area (10 areas):
  papers_recommended = count of papers where:
    this area was in topic_scores AND relevance_score > 7.0
    AND paper was recommended in the past 30 days

  papers_engaged = count of papers where:
    this area was in topic_scores AND
    status is in [deep_read, cited, saved]

  read_rate = papers_engaged / papers_recommended
  (if papers_recommended = 0, read_rate = null — no data)
```

### Step 2: Adjust Weights

```
For each interest area:
  if read_rate > 0.5:
    weight += 0.02  (you consistently engage with these papers)
  elif read_rate >= 0.1:
    no change  (normal engagement)
  elif read_rate < 0.1 AND read_rate is not null:
    weight -= 0.02  (you consistently skip these papers)
  elif read_rate is null:
    no change  (not enough data to judge)
```

### Step 3: Apply Constraints

```
Hard floor:   no weight below 0.03
Hard ceiling: no weight above 0.25
Normalization: all weights sum to 1.0

Why floor of 0.03:
  A PhD student who stops reading spatial criminology in year 3
  may regret it in year 5 when a reviewer asks about it.
  The floor keeps a minimal window open.

Why ceiling of 0.25:
  Prevents the system from recommending only one topic.
  Even if you read every victimization paper, the system should
  still show you causal inference and methods papers.
```

### Step 4: Log Changes

```
Store weight change history:
  {
    "date": "2026-09-01",
    "previous_weights": {...},
    "new_weights": {...},
    "changes": [
      {"area": "spatial", "from": 0.10, "to": 0.08, "reason": "read_rate 0.05"},
      {"area": "victimization", "from": 0.10, "to": 0.12, "reason": "read_rate 0.62"}
    ]
  }

All changes are logged and reversible.
```

---

## Filter Bubble Prevention

Even after adaptive adjustment, the system maintains breadth.

### Minimum Exposure Rule

```
Every week, the Scanner MUST surface at least 1 paper
from the researcher's lowest-weighted interest area,
IF such a paper exists in the database with status = "new."

This paper is marked in the Scanner output:
  "Breadth pick: [title] — surfaced to maintain coverage
   in [area], your lowest-weighted interest."
```

### Why This Matters

```
PhD interest drift is real:
  Year 1: broad reading across all areas
  Year 2: narrowing toward dissertation topics
  Year 3-4: tunnel vision on 2-3 areas
  Year 5+: suddenly need breadth again for defense, job talks,
           and post-PhD research agenda

The system should support narrowing without enabling
complete loss of peripheral awareness.
```

---

## Monthly Drift Report

Generated every 30 days and included in the Queue Manager output.

```markdown
Interest Drift Report — [Month Year]

Reading Behavior Summary (past 30 days):
  Papers recommended:  [N]
  Papers deep-read:    [N]
  Papers cited:        [N]
  Papers archived:     [N]

Engagement by Interest Area:
  Area                  | Recommended | Engaged | Rate   | Weight Change
  ----------------------|-------------|---------|--------|-------------
  Victimization         |     12      |    8    | 0.67   | 0.10 → 0.12 ↑
  Bayesian              |      8      |    5    | 0.63   | 0.10 → 0.12 ↑
  Causal inference      |      9      |    4    | 0.44   | no change
  Life-course           |      6      |    2    | 0.33   | no change
  Gambling              |      5      |    2    | 0.40   | no change
  Gender                |      4      |    1    | 0.25   | no change
  Experiments           |      7      |    1    | 0.14   | no change
  Aging                 |      3      |    0    | 0.00   | 0.10 → 0.08 ↓
  Spatial               |      5      |    0    | 0.00   | 0.10 → 0.08 ↓
  AI                    |      4      |    0    | 0.00   | 0.10 → 0.08 ↓

Observations:
  - You are increasingly focused on victimization + Bayesian work.
    This aligns with Chapter 3 of your dissertation.
  - You have not engaged with spatial, aging, or AI papers
    in 60 days. Weights reduced but floor maintained.
  - Scanner will continue surfacing 1 paper/week from your
    lowest-weighted areas.

Override Options:
  - "reset weights" → restore all to 0.10
  - "lock spatial at 0.10" → prevent further reduction
  - "boost aging" → manually increase aging weight
```

---

## Method Engagement Tracking (Secondary)

In addition to topic weights, track method preferences.

```
Same logic as topic weights, but for method categories:
  DiD, RCT, Bayesian, spatial analysis, ML/AI,
  qualitative, meta-analysis, etc.

This does not directly adjust Scanner weights (methods are
already part of method_match scoring), but it informs:
  - Deep Reader: which method sections to emphasize
  - Queue Manager: method diversity in weekly recommendations
```

---

## Database Updates

```yaml
After feedback loop runs:
  researcher_profile.learned_weights: updated
  feedback_history: new entry appended
  date_last_feedback_update: today

No paper status changes — this skill only updates the profile.
```

---

## Token Cost

```
Per run:       2,000 - 4,000 tokens (data aggregation + report)
Frequency:     1x / month
Monthly total: ~4,000 tokens
```

This is the cheapest skill because it operates on metadata, not paper content.

---

## Interaction with Other Skills

```
Reads from:
  - Database (all paper records — scores, statuses, dates)
  - Researcher Profile (current weights)

Writes to:
  - Researcher Profile (updated weights)
  - Feedback history log

Consumed by:
  - Daily Scanner (uses updated weights for scoring)
  - Queue Manager (displays drift report)
  - Deep Reader (uses method engagement data for emphasis)
```

---

## When This Skill Becomes Active

```
Month 1-2:  data collection only (not enough reading behavior)
Month 3:    first weight adjustment (based on 60 days of data)
Month 4+:   monthly updates, increasingly accurate

The system starts with equal weights and learns.
Do not manually tune weights before month 3
unless you have strong prior preferences.
```
