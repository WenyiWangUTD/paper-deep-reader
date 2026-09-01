# Research Intelligence System — Revised Architecture

Wenyi Wang | Criminology PhD  
Revision rationale: address feedback loops, adaptive scoring, layered output, data-source realism, degradation strategy, and cross-skill health monitoring.

---

## System-Level Changes (Before Individual Skills)

### New Component: Adaptive Profile

All skills share a single researcher profile that evolves over time.

```yaml
Researcher Profile (shared, mutable)

  static_interests:
    - life-course criminology
    - victimization and general strain
    - gender and sexuality
    - gambling policy evaluation
    - causal inference
    - spatial/environmental criminology
    - aging and crime
    - experiments
    - AI
    - Bayesian methods

  learned_weights:
    # initialized equally, updated by feedback loop
    life-course: 0.10
    victimization: 0.10
    gender: 0.10
    gambling: 0.10
    causal_inference: 0.10
    spatial: 0.10
    aging: 0.10
    experiments: 0.10
    ai: 0.10
    bayesian: 0.10

  active_projects:
    # add dissertation chapters, papers in progress
    - title: ""
      keywords: []
      stage: ""  # ideation | writing | revision | submitted

  update_trigger:
    - every 30 days, recompute learned_weights from reading behavior
    - when a new project is added/removed
```

### New Component: System Health Monitor

```yaml
Health Check (runs weekly)

  checks:
    - harvester_last_run: date
      alert_if: > 35 days ago
    - new_papers_unprocessed: count
      alert_if: > 50
    - backlog_unread: count
      alert_if: > 100
    - scanner_last_run: date
      alert_if: > 3 days ago
    - deep_read_this_week: count
      warn_if: 0

  output: one-paragraph status + any alerts
```

---

## Skill 1: DOI Harvester (Revised)

### Original Problem

The original design said "scan journals monthly" without specifying where the data comes from. Different sources have different abstract completeness, coverage lag, and rate limits. This is not a detail — it determines whether downstream scoring works.

### Data Source Strategy

```
Primary:    OpenAlex API (free, broad coverage, structured metadata)
Secondary:  CrossRef API (DOI resolution, citation counts)
Fallback:   Journal RSS feeds (for journals poorly covered by OpenAlex)
```

Why OpenAlex first: it returns structured fields (title, abstract, authors, concepts, open_access status, cited_by_count) in a single call. CrossRef sometimes has incomplete abstracts. RSS feeds are unstructured and journal-specific.

### Target Journals (with source notes)

```yaml
Tier 1 (core criminology — always scan):
  - Criminology
  - Criminology & Public Policy
  - Journal of Research in Crime and Delinquency
  - Journal of Quantitative Criminology
  - British Journal of Criminology
  - Justice Quarterly
  - Journal of Experimental Criminology
  - Annual Review of Criminology

Tier 2 (topical — scan monthly):
  - European Journal of Criminology
  - Journal of Criminal Justice
  - Journal of Developmental and Life-Course Criminology
  - Development and Psychopathology
  - Journal of Youth and Adolescence
  - Psychology, Public Policy, and Law
  - The Gerontologist
  - Feminist Criminology
  - Violence Against Women
  - Journal of Interpersonal Violence
  - Victims & Offenders
  - Sexual Abuse

Tier 3 (methods + niche — scan monthly):
  - Crime Science
  - Security Journal
  - Crime Prevention and Community Safety
  - Environment and Planning B
  - Addiction
  - International Gambling Studies
  - Journal of Gambling Studies
  - Sociological Methods & Research
  - Political Analysis
  - Journal of Causal Inference
```

### Harvest Logic (Revised)

```
For each journal:
  1. Query OpenAlex by ISSN + publication_date >= last_harvest_date
  2. For each result:
     a. Extract DOI
     b. Check: DOI exists in database?
        - Yes → update metadata if changed (e.g., issue assigned, citation count)
        - No  → insert new record, status = "new"
     c. Check: abstract present?
        - Yes → store
        - No  → flag as "abstract_missing", try CrossRef fallback
        - Still no → store title + authors only, mark "title_only"
  3. Log harvest stats
```

### Degradation Strategy (NEW)

```
If new papers in a single harvest > 40:
  Phase 1: title + keyword screening only (no abstract scoring)
           keep papers matching any static_interest keyword
           discard obvious mismatches
  Phase 2: run full abstract scoring on Phase 1 survivors

This prevents token blowout during peak publishing seasons.
```

### Output

```
Harvest Report
  Journals scanned:   30
  Papers found:       142
  Already in DB:      128
  New papers added:   14
  Abstract missing:   2 (flagged for manual check)
  Last harvest:       2026-09-01
  Next scheduled:     2026-10-01
```

### Health Integration

```
Emits to System Health Monitor:
  - harvester_last_run = today
  - new_papers_unprocessed += 14
```

---

## Skill 2: Monthly Landscape Analyst (Revised)

### Original Problem

The original version produced a static topic list. It didn't track change over time, so you couldn't see whether "AI in crime prediction" is a new surge or has been steady for six months.

### Input

```
All database records where:
  date_first_seen within last 30 days
```

### Output Structure (Revised)

```markdown
Monthly Criminology Landscape — [Month Year]

Overview
  Total new papers this month: N
  Compared to last month: +/- N (trend arrow)

Topic Clusters (auto-detected, not pre-defined)
  1. [Topic] — N papers
     Trajectory: rising / stable / declining (vs. prior 3 months)
     Representative paper: [title, journal]
  2. ...

Methodological Trends
  Methods appearing for the first time or with unusual frequency
  (e.g., "Causal forests appeared in 4 papers this month,
   up from 0 in the prior quarter")

Cross-Topic Intersections
  Pairs of your research interests that co-occurred in papers
  (e.g., "3 papers combined victimization + Bayesian methods")

Journals Producing Most Relevant Work This Month
  Ranked by number of papers scoring > 7.0 in your Scanner

Gaps Noticed
  Which of your 10 interest areas had ZERO new papers this month?
  (This is a signal — either nothing is happening,
   or your journal list has a coverage hole.)
```

### What Changed from Original

- Added temporal comparison (trend vs. prior months, not just this month's snapshot)
- Added cross-topic intersection detection (directly useful for finding dissertation-relevant work)
- Added gap detection (missing topics may reveal journal coverage problems)
- Removed generic "key papers" list (the Scanner already does this daily — no need to repeat)

---

## Skill 3: Daily Relevance Scanner (Revised)

### Original Problem

Static scoring weights (40/30/20/10) don't adapt to changing interests. No degradation for high-volume days. No learning from reading behavior.

### Input

```
All database records where:
  status = "new"
```

### Scoring System (Revised: Adaptive)

```yaml
Base weights (initial):
  topic_match:     0.35
  method_match:    0.25
  journal_tier:    0.15
  novelty:         0.10
  project_match:   0.15   # NEW: match against active_projects

Adaptive adjustment (every 30 days):
  For each interest area:
    read_rate = papers_deep_read / papers_recommended
    if read_rate > 0.5:  weight += 0.02
    if read_rate < 0.1:  weight -= 0.02
  Normalize so weights sum to 1.0
```

### Project Relevance (NEW)

```
For each active_project in researcher profile:
  Check if paper keywords overlap with project keywords
  If match:
    Add bonus score of 0.5-1.5 depending on project stage
    (papers matching a project in "writing" stage get highest bonus)
```

### Degradation Strategy (NEW)

```
If papers with status = "new" > 20 today:
  Stage 1: title + keyword pre-filter
            discard papers matching zero interest areas
  Stage 2: full abstract scoring on survivors only

If papers with status = "new" > 40 today:
  Stage 1: title-only filter (aggressive)
  Stage 2: abstract scoring on top 20 by title relevance
  Stage 3: flag remainder as "bulk_skipped" for weekend review
```

### Output (Revised)

```markdown
Daily Scan — [Date]

Papers evaluated: 12
Your bandwidth today: ~10-15 abstracts

Recommended (score > 7.0):

  1. [Title]
     Journal: [name] | DOI: [doi]
     Score: 9.2
     Why: victimization + Bayesian + longitudinal
     Project match: Chapter 3 (strain theory)
     → Recommend: Deep Read

  2. [Title]
     Score: 8.5
     Why: causal inference + gambling policy
     → Recommend: Save + Skim

  3. ...

Monitor (score 5.0-7.0):
  [titles only, with one-line reason]

Skipped: 4 papers (below threshold)

Scoring drift notice:
  "Your read-rate for spatial criminology papers has dropped
   to 5% over the past 60 days. Scanner weight reduced from
   0.10 to 0.08. Adjust manually if this is wrong."
```

### Status Update

```
Recommended papers: new → evaluated
Skipped papers: new → evaluated (relevance_score stored)
```

---

## Skill 4: Deep Reader V2 (Revised)

This skill has the most changes.

### Original Problems

1. 12 sections applied uniformly to all papers — too heavy for some, wrong shape for others
2. Section 3 (author thinking reconstruction) risks plausible fabrication
3. Section 12 (novel follow-up) doesn't check whether the idea already exists
4. Flat output — no layered access for different use cases

### Paper Type Detection (NEW — runs before analysis)

```
Classify paper into one of:
  - empirical_quantitative
  - empirical_qualitative
  - mixed_methods
  - theoretical_essay
  - methodological
  - meta_analysis
  - systematic_review

This classification determines which sections are generated.
```

### Section Activation Matrix

```
Section                        | Quant | Qual | Theory | Methods | Meta
-------------------------------|-------|------|--------|---------|-----
0. Executive Summary           |  ✓    |  ✓   |   ✓    |   ✓     |  ✓
1. Research Question            |  ✓    |  ✓   |   ✓    |   ✓     |  ✓
2. Gap Assessment               |  ✓    |  ✓   |   ✓    |   ✓     |  ✓
3. Reconstructed Thinking       |  ✓    |  ✓   |   ✓    |   ✓     |  ✗
4. Method Intuition             |  ✓    |  ✓   |   ✗    |   ✓     |  ✓
5. Method Pipeline + Example    |  ✓    |  ✗   |   ✗    |   ✓     |  ✗
6. Math/Stats Interpreter       |  ✓*   |  ✗   |   ✗    |   ✓*    |  ✓*
7. RQ → Design → Answer         |  ✓    |  ✓   |   ✗    |   ✗     |  ✓
8. Main Takeaways               |  ✓    |  ✓   |   ✓    |   ✓     |  ✓
9. One-Week Replication Plan    |  ✓    |  ✗   |   ✗    |   ✓     |  ✗
10. Most Fragile Claim          |  ✓    |  ✓   |   ✓    |   ✓     |  ✓
11. Adversarial Design          |  ✓    |  ✓   |   ✓    |   ✗     |  ✓
12. Novel Follow-Up             |  ✓    |  ✓   |   ✓    |   ✓     |  ✓

✓* = only if the paper contains formal models/equations
✗  = skipped entirely (not generated with filler)
```

### Layered Output (NEW)

```
Layer 1 — Quick Read (always generated):
  Section 0: Executive Summary
  Section 1: Research Question (condensed)
  Section 8: Main Takeaways
  Section 10: Most Fragile Claim

Layer 2 — Understanding (generated on request or for deep_read):
  Section 2: Gap Assessment
  Section 3: Reconstructed Thinking
  Section 4: Method Intuition
  Section 7: RQ → Design → Answer

Layer 3 — Engagement (generated on request):
  Section 5: Method Pipeline + Example
  Section 6: Math/Stats Interpreter
  Section 9: One-Week Replication Plan
  Section 11: Adversarial Design
  Section 12: Novel Follow-Up
```

Default behavior: always generate Layer 1. Generate Layer 2 automatically for papers scored > 8.0 or tagged to an active project. Generate Layer 3 only when explicitly requested.

This cuts token usage by ~50% on average while preserving full depth when needed.

### Section 3 Revision: Reconstructed Thinking (Constrained)

Original prompt: "Infer the intellectual journey. Do NOT use the authors' stated contribution."

Problem: this invites hallucination. The AI will confidently fabricate a plausible narrative.

Revised prompt:

```markdown
Reconstruct the likely reasoning path that led to this research.

Constraints — you may ONLY draw on:
  1. Papers cited in the introduction and literature review
     (what failures or gaps did the authors explicitly reference?)
  2. The authors' prior publications
     (what were they working on before this paper?)
  3. Stated motivations in the paper
     (practical problems, policy debates, theoretical tensions
      mentioned in the text)
  4. Methodological developments cited
     (new tools or datasets that became available)

Do NOT speculate beyond these anchors.

If you cannot reconstruct the path from available evidence,
say so explicitly rather than inventing a narrative.

Format:
  Known starting points → identified gap → available opportunity
  → resulting research idea
```

### Section 6 Revision: Math/Stats Interpreter (Enhanced)

```markdown
For each formal model in the paper:

  1. Show the equation exactly as written
  2. Plain-language meaning of every term
  3. Intuition: "What is this equation trying to capture?"
  4. Connection: "Why did the authors need this specific model
     instead of a simpler one?"
  5. Worked example with hypothetical numbers
     (use the paper's actual variable names but fake data)

For Bayesian papers specifically:
  - Prior: what assumption, why this prior, how sensitive
  - Likelihood: what data-generating process is assumed
  - Posterior: how to interpret the result
  - Sensitivity: what happens if the prior is changed

For causal inference papers:
  - Identify the causal estimand (ATE, ATT, LATE, etc.)
  - State the identification assumption in plain language
  - Explain what would violate the assumption
  - Assess whether the authors tested for violations

Skip this section entirely if the paper has no formal models.
Do not generate filler like "the authors used regression."
```

### Section 12 Revision: Novel Follow-Up (With Existence Check)

```markdown
Propose one follow-up research idea.

Hard constraints:
  - NOT: different sample, longer time period, another country,
    more control variables
  - MUST target: theoretical gap, methodological limitation,
    measurement limitation, mechanism uncertainty,
    or policy necessity

Required check before proposing:
  1. State the proposed idea in one sentence
  2. Identify 2-3 search queries that would find existing work
     on this idea
  3. Based on your knowledge, does similar work likely exist?
     - If yes: acknowledge and explain how yours differs
     - If uncertain: flag explicitly
       ("This idea may already exist — verify before pursuing")

Output format:
  Core Question:
  Why It Matters:
  Methodological Approach:
  Feasibility (data, timeline, skills needed):
  Existence Risk: [low / medium / high / uncertain]
  Suggested Verification Queries: [for Google Scholar]
  Target Journals: [ranked by fit]
  Publishability Assessment: [with reasoning]
```

### Status Update

```
After Deep Reader runs:
  status: evaluated → deep_read
  deep_review_complete: true
  deep_review_layer: 1 | 2 | 3
  paper_type: [detected type]
```

---

## Skill 5: Zotero Sync (Revised)

### Original Problem

The original design was one-directional (system → Zotero). It didn't handle the common case where you manually add papers to Zotero outside this system.

### Bidirectional Sync (NEW)

```
Direction 1: Database → Zotero
  When status changes to "saved" or "deep_read":
    Check: DOI in Zotero?
    No  → create entry with auto-tags
    Yes → update tags and notes if changed

Direction 2: Zotero → Database (NEW)
  Periodically check Zotero for papers NOT in your DOI database
  (papers you found manually, got from colleagues, etc.)
  For each:
    Extract DOI from Zotero metadata
    Insert into database with status = "manually_added"
    Run Scanner scoring retroactively
```

### Auto-Tagging (Revised)

```yaml
Tag sources (layered, not just topic):
  Topic tags:
    - #Victimization, #LifeCourse, #Bayesian, etc.
    (derived from Scanner topic_match)

  Method tags (NEW):
    - #DiD, #RCT, #SyntheticControl, #BayesianInference,
      #CausalForest, #IVRegression, etc.
    (derived from Deep Reader Section 4/5)

  Status tags (NEW):
    - #ToRead, #DeepRead, #Cited
    (mirrors database status)

  Project tags (NEW):
    - #Ch3_Strain, #Gambling_Paper, etc.
    (derived from active_projects match)
```

### Zotero Note Content (Revised)

```markdown
When Deep Reader has been run, attach to Zotero entry:

  [Layer 1 output — Executive Summary + Takeaways + Fragile Claim]

  If Layer 2/3 exist, add link or reference:
  "Full deep review available in Research Database — [DOI]"

Do NOT paste the full 12-section output into Zotero notes.
It makes Zotero search results unusable.
```

---

## Skill 6: Reading Queue Manager (Revised)

### Original Problem

The original version produced a priority list but didn't explain its reasoning, didn't account for reading capacity variance, and didn't handle the inevitable backlog growth.

### Input

```yaml
From database:
  - all papers with status in [evaluated, saved, pdf_obtained]
  - relevance_score
  - date_first_seen (aging factor)
  - project_match

From researcher profile:
  - active_projects and their stages
  - weekly reading capacity (user-set, default: 5 papers)
```

### Queue Logic (Revised)

```
Priority score = relevance_score
               + project_urgency_bonus (0-2.0)
               + aging_penalty (papers > 30 days old lose 0.5)
               + citation_momentum (if cited_by_count grew recently, +0.5)

Sort by priority score descending.
Take top N where N = weekly_reading_capacity.
```

### Backlog Management (NEW)

```
If backlog (status = evaluated, not yet read) > 60 papers:
  Trigger backlog triage:
    - Re-score all backlog papers against CURRENT profile weights
      (interests may have shifted since first scoring)
    - Papers that now score < 5.0: auto-archive
      (status → archived, with reason: "below threshold on rescore")
    - Papers > 90 days old and score < 7.0: auto-archive
    - Report: "Archived N papers during backlog triage.
               Remaining backlog: M papers."

User can review archived papers anytime, but they leave the queue.
```

### Output (Revised)

```markdown
Weekly Reading Queue — [Week of Date]

System Health:
  Database total: 847 papers
  Harvester last run: 3 days ago ✓
  Unprocessed new papers: 6 ✓
  Backlog: 34 papers (manageable)

This Week's Deep Reads (5 papers):

  1. [Title]
     Journal: [name]
     Priority: 9.4
     Why now: matches Chapter 3 (writing stage) +
              Bayesian methods + victimization
     Days in queue: 4

  2. [Title]
     Priority: 8.8
     Why now: causal inference + gambling policy,
              relevant to funding proposal due Oct 15
     Days in queue: 11

  3-5. ...

Also Consider (next in queue):
  [3 papers with scores, in case you finish early]

Backlog Alert:
  12 papers have been in queue > 30 days.
  Lowest-scored 4 recommended for archive. Confirm?

Interest Drift Report (monthly):
  Topics you're reading more: victimization, Bayesian
  Topics you're reading less: spatial, AI
  Scanner weights adjusted accordingly.
  Override? [list current weights]
```

---

## New Skill 7: Feedback Loop Engine

This skill doesn't exist in the original design. It is the mechanism that makes the other skills adaptive.

### Purpose

Track what you actually read vs. what was recommended, and update the system.

### Signals Collected

```yaml
Positive signals:
  - Paper moved to deep_read → strong interest
  - Paper moved to cited → highest interest
  - Paper opened same day as recommended → high urgency match
  - Paper tagged to active_project → project-relevant

Negative signals:
  - Paper recommended but still "evaluated" after 14 days → weak match
  - Paper archived without reading → topic likely not interesting
  - User manually overrides score → scoring model was wrong

Neutral:
  - Paper saved to Zotero but not deep_read → moderate interest
```

### Update Cycle

```
Every 30 days:
  1. Compute read_rate per interest area
  2. Compute read_rate per method category
  3. Adjust learned_weights in researcher profile
  4. Log weight changes for transparency
  5. Generate one-paragraph "drift report"

Weight adjustment is capped:
  No single interest can drop below 0.03 or rise above 0.25
  This prevents the system from collapsing into a filter bubble
```

### Filter Bubble Prevention (NEW)

```
Rule: even after adaptive adjustment, the Scanner must always
surface at least 1 paper per week from the researcher's
lowest-weighted interest area.

Rationale: a PhD student in year 3 who stops reading spatial
criminology may regret it in year 5 when a reviewer asks
about it. The system should keep a minimal window open.
```

---

## Revised Token Estimates

```
Skill                  | Per run          | Frequency       | Monthly
-----------------------|------------------|-----------------|--------
DOI Harvester          | 2,000-5,000      | 1x/month        | 5,000
Monthly Landscape      | 40,000-55,000    | 1x/month        | 55,000
Daily Scanner          | 6,000-12,000     | daily           | 270,000
  (with degradation)   | max 15,000/day   |                 |
Deep Reader            |                  |                 |
  Layer 1 only         | 4,000-6,000      | 5/week          | 100,000
  Layer 1+2            | 8,000-12,000     | 3/week          | 120,000
  Layer 1+2+3          | 14,000-18,000    | 1/week          | 64,000
Zotero Sync            | 500-1,500        | per paper saved  | 30,000
Queue Manager          | 3,000-5,000      | 1x/week         | 20,000
Feedback Loop          | 2,000-4,000      | 1x/month        | 4,000
-----------------------|------------------|-----------------|--------
Estimated monthly total                                     | ~670,000

Compared to original estimate of 700k-1.5M:
  Lower average due to layered Deep Reader output,
  but with full depth available on demand.
```

---

## Revised Build Order

```
Phase 1 — Foundation (build first, highest ROI):
  ✓ DOI Database schema
  ✓ DOI Harvester (with OpenAlex integration)
  ✓ Daily Relevance Scanner (with static weights initially)

Phase 2 — Depth:
  ✓ Deep Reader V2 (Layer 1 only at first)
  ✓ Deep Reader V2 (add Layer 2)

Phase 3 — Integration:
  ✓ Zotero Sync (database → Zotero direction first)
  ✓ Deep Reader V2 (add Layer 3)

Phase 4 — Intelligence:
  ✓ Reading Queue Manager
  ✓ Monthly Landscape Analyst
  ✓ Feedback Loop Engine (requires ~60 days of data first)

Phase 5 — Maturity:
  ✓ Zotero reverse sync (Zotero → database)
  ✓ Adaptive scoring activation
  ✓ Backlog auto-triage
```

---

## Summary of Key Changes from Original

1. **Adaptive Profile** replaces static scoring weights — the system learns from your behavior
2. **Paper Type Detection** in Deep Reader prevents one-size-fits-all output
3. **Layered Output** (3 tiers) cuts average token cost by ~50% while preserving full depth
4. **Constrained Reconstruction** in Section 3 reduces hallucination risk
5. **Existence Check** in Section 12 prevents proposing already-published ideas
6. **Data Source Strategy** specifies where papers actually come from (OpenAlex + CrossRef + RSS)
7. **Degradation Strategy** handles peak publishing periods without token blowout
8. **Bidirectional Zotero Sync** captures manually-added papers
9. **Backlog Management** prevents infinite queue growth
10. **Filter Bubble Prevention** maintains breadth even as interests narrow
11. **System Health Monitor** catches stale data and skill failures
12. **Feedback Loop Engine** is the new skill that makes everything else adaptive
