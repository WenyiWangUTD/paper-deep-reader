# Skill 2: Monthly Landscape Analyst

## Purpose

Answer: **What happened this month in criminology?**

This skill provides field-level awareness. It tells you which topics are surging, which methods are gaining traction, and where your research interests intersect with current publication trends. It is not a paper recommendation tool — that is the Scanner's job.

## Trigger

- Monthly: after DOI Harvester completes
- Manual: "landscape report" or "what's trending in the field"

## Frequency

Once per month. Should run after the Harvester so it has the latest data.

---

## Input

```
From database:
  All records where date_first_seen is within the last 30 days

From researcher profile:
  static_interests (for relevance filtering)
  active_projects (for project intersection detection)

From previous landscape reports (if available):
  Topic counts from prior 3 months (for trend detection)
```

---

## Analysis Pipeline

### Step 1: Topic Clustering

```
For each new paper this month:
  Extract topic signals from:
    - keywords (author-provided)
    - OpenAlex concepts (if available)
    - abstract content (AI-extracted themes)

  Cluster papers into topic groups
  Count papers per topic
```

Do NOT use pre-defined topic categories only. Allow new topics to emerge from the data. For example, if 6 papers this month discuss "algorithmic fairness in policing" and that phrase has never appeared before, it should surface as a new cluster.

### Step 2: Temporal Comparison

```
For each topic cluster:
  Compare paper count to same topic in prior 3 months
  Classify trajectory:
    - rising:    this month > average of prior 3 months by > 30%
    - stable:    within ±30% of prior average
    - declining: this month < average of prior 3 months by > 30%
    - new:       zero papers in prior 3 months, > 0 this month
```

### Step 3: Method Trend Detection

```
Scan abstracts for methodology mentions:
  - Named methods: DiD, RDD, RCT, synthetic control, causal forest,
    Bayesian inference, IRT, SEM, multilevel modeling, network analysis,
    machine learning, NLP, spatial analysis, meta-analysis
  - Method novelty: methods appearing for the first time
    or with unusual frequency compared to prior 3 months
```

### Step 4: Cross-Interest Intersection

```
For each paper:
  Check how many of the researcher's 10 interest areas it touches
  Papers touching 2+ areas are "intersection papers"

Report pairs of interests that co-occurred:
  e.g., "4 papers combined victimization + Bayesian methods"
  e.g., "2 papers combined aging + causal inference"

These intersections are high-value signals for a PhD student
working across multiple subfields.
```

### Step 5: Coverage Gap Detection

```
For each of the researcher's 10 interest areas:
  Count how many new papers matched this area this month

If any area has ZERO papers:
  Flag as potential coverage gap
  Suggest possible reasons:
    - Low publication activity in this area this month
    - Journal list may not cover this area well
    - Interest keywords may need refinement
```

### Step 6: Journal Productivity Ranking

```
Rank journals by:
  Number of papers scoring > 7.0 in the Scanner this month

This tells the researcher which journals are currently
producing the most relevant work for them specifically,
not just the most prestigious journals overall.
```

---

## Output Structure

```markdown
Monthly Criminology Landscape — [Month Year]

Overview
  Total new papers this month:  [N]
  Compared to last month:       [+/- N] [↑/↓/→]
  Papers relevant to you (score > 7.0):  [N]

Topic Clusters
  1. [Topic Name] — [N] papers
     Trajectory: [rising / stable / declining / new]
     Representative paper: "[title]" ([journal])
  2. [Topic Name] — [N] papers
     Trajectory: [trajectory]
     ...
  (list top 8-10 clusters)

Methodological Trends
  Rising methods:
    - [Method]: appeared in [N] papers (up from [M] last quarter)
  New appearances:
    - [Method]: first seen this month in [journal]

Cross-Interest Intersections
  Your research areas that co-occurred in papers this month:
    - [Interest A] × [Interest B]: [N] papers
    - [Interest A] × [Interest C]: [N] papers
  (These are potential dissertation-relevant clusters)

Most Productive Journals for You This Month
  1. [Journal] — [N] papers scoring > 7.0
  2. [Journal] — [N] papers
  3. [Journal] — [N] papers

Coverage Gaps
  Interest areas with ZERO new papers this month:
    - [Area]: [possible reason]
  (If none, state "All interest areas had coverage this month.")

Field-Level Observation
  [One paragraph: a qualitative synthesis of what the month's
   publications suggest about where the field is heading.
   Not a list — a narrative observation.]
```

---

## What This Skill Does NOT Do

- Does not recommend individual papers (that is the Scanner)
- Does not produce deep summaries (that is the Deep Reader)
- Does not track your reading progress (that is the Queue Manager)

This skill is purely about field awareness and trend detection.

---

## Token Cost

```
Input:   ~50,000 tokens (100-150 abstracts + prior month data)
Output:  ~3,000-5,000 tokens (report)
Total:   ~55,000 tokens per run
Frequency: 1x/month
Monthly:   ~55,000 tokens
```

---

## Improvement Over Original Design

| Aspect | Original | Revised |
|--------|----------|---------|
| Topics | Static list | Auto-detected clusters + trajectory |
| Temporal | Snapshot only | 3-month comparison |
| Intersections | Not tracked | Cross-interest co-occurrence |
| Gaps | Not detected | Coverage gap alerts |
| Journal ranking | Not included | Ranked by personal relevance |
| Narrative | Not included | Field-level qualitative observation |
