# Skill 5: Zotero Sync

## Purpose

Keep the DOI database and Zotero library in sync — bidirectionally. Papers saved through the system appear in Zotero with proper tags. Papers added manually to Zotero get registered in the database.

## Trigger

- Automatic: when a paper's status changes to "saved" or "deep_read"
- Manual: "sync Zotero" or "add this to Zotero"
- Periodic: weekly reverse sync check

## Frequency

Per-paper (automatic) + weekly reverse sync.

---

## Direction 1: Database → Zotero

This is the primary flow. When you decide to keep a paper, it should appear in Zotero without manual entry.

### Trigger Conditions

```
When status changes to:
  "saved"       → create Zotero entry with basic metadata + tags
  "deep_read"   → create or update entry with Deep Reader notes
  "cited"       → update entry with citation tag
```

### Pre-Check

```
Before creating a Zotero entry:
  1. Query Zotero by DOI
  2. Already exists?
     YES → update tags and notes if changed
           do NOT overwrite user's manual notes
     NO  → create new entry
```

### Entry Creation

```
Zotero item fields:
  Title:          [from database]
  Authors:        [from database]
  Journal:        [from database]
  Date:           [publication_date]
  DOI:            [doi]
  URL:            [url]
  Abstract:       [from database, if available]
```

### Auto-Tagging System

Tags are applied in layers, not just by topic.

```yaml
Topic tags (from Scanner topic_match):
  #Victimization
  #LifeCourse
  #Gender
  #Gambling
  #CausalInference
  #Spatial
  #Aging
  #Experiments
  #AI
  #Bayesian

Method tags (from Deep Reader Section 4/5, if available):
  #DiD
  #RCT
  #SyntheticControl
  #BayesianInference
  #CausalForest
  #IVRegression
  #SpatialAnalysis
  #MLM
  #SEM
  #MetaAnalysis
  #NLP
  #MachineLearning
  #Qualitative
  #MixedMethods

Status tags (mirrors database status):
  #ToRead
  #DeepRead
  #Cited

Project tags (from project_match in Scanner):
  # Derived from active_projects in researcher profile
  # Examples:
  #Ch3_Strain
  #GamblingPaper
  #DissertationLitReview

Paper type tag (from Deep Reader detection):
  #Quantitative
  #Qualitative
  #Theoretical
  #Methodological
  #MetaAnalysis
  #SystematicReview
```

### Zotero Notes Content

What goes into the Zotero note depends on what analysis has been done.

```
If only Scanner has run (no Deep Reader):
  Note content:
    Relevance Score: [score]
    Matched interests: [list]
    Project match: [if any]
    "Full analysis not yet completed."

If Deep Reader Layer 1 has run:
  Note content:
    [Executive Summary]
    [Main Takeaways — 5 bullet points]
    [Most Fragile Claim — 2-3 sentences]

If Deep Reader Layer 2+ has run:
  Note content:
    [Layer 1 content as above]
    ---
    "Extended analysis available (Layer [2/3]).
     Sections covered: [list of generated sections]"

  Do NOT paste the full 12-section output into Zotero.
  Reason: Zotero search indexes note content.
  Long notes make search results noisy and unusable.
  Keep Zotero notes concise; the full analysis lives
  in the database / Deep Reader output files.
```

### Zotero Collection Assignment

```
Assign to collection based on primary topic match:
  - If topic_scores has a clear winner (>= 2x second place):
    assign to that collection
  - If multiple topics are close:
    assign to the highest-scored collection
    add tags for secondary topics

Default collections (create if not existing):
  /Criminology
    /Life-Course
    /Victimization
    /Gender
    /Gambling
    /Spatial
    /Aging
  /Methods
    /Causal Inference
    /Bayesian
    /Experiments
    /AI-ML
  /Dissertation
    /Chapter 1
    /Chapter 2
    /...
  /Inbox  (for manually_added papers pending classification)
```

---

## Direction 2: Zotero → Database (Reverse Sync)

This captures papers you found outside the system — from colleagues, conference talks, Twitter/Bluesky, manual searches, etc.

### How It Works

```
Weekly check:
  1. List all Zotero items added in the past 7 days
  2. For each item:
     a. Extract DOI from Zotero metadata
     b. DOI exists in database?
        YES → no action (already tracked)
        NO  → create database record:
              status = "manually_added"
              source = "zotero_reverse_sync"
              date_first_seen = today
  3. Run Scanner scoring on newly added records
     (retroactive relevance scoring)
```

### Handling Papers Without DOI in Zotero

```
If Zotero item has no DOI:
  Try to extract DOI from:
    - URL field (some URLs contain DOI)
    - Attached PDF metadata
  If DOI found → proceed normally
  If no DOI found → create record with internal ID
    flag: needs_doi_verification = true
    status = "manually_added"
```

---

## Database Updates

```yaml
After Database → Zotero sync:
  zotero_saved: true
  zotero_key: [Zotero item key]
  date_last_updated: today

After Zotero → Database reverse sync:
  New records created with:
    status: "manually_added"
    source: "zotero_reverse_sync"
    (Scanner will evaluate and update status to "evaluated")
```

---

## Error Handling

```
- Zotero API unreachable → queue sync for retry in 1 hour
- DOI not found in Zotero metadata → flag for manual check
- Duplicate Zotero entries for same DOI → warn user,
  do not auto-delete (user may have intentional duplicates
  across collections)
- Tag limit exceeded → Zotero has no hard tag limit,
  but if item has > 15 tags, warn and suggest consolidation
```

---

## Token Cost

```
Per paper sync:    500 - 1,500 tokens (formatting notes + tags)
Weekly reverse sync: 1,000 - 3,000 tokens (metadata comparison)
Monthly total:     ~30,000 tokens
```

This is one of the cheapest skills because it is mostly metadata operations, not content generation.

---

## Interaction with Other Skills

```
Reads from:
  - Database (paper metadata, scores, Deep Reader output)
  - Researcher Profile (active_projects for project tags)
  - Deep Reader (Layer 1 notes for Zotero note content)

Writes to:
  - Database (zotero_saved, zotero_key)
  - Zotero (entries, tags, notes, collections)
  - Database (new records from reverse sync)

Triggers:
  - Scanner runs on reverse-synced papers
  - Queue Manager includes manually_added papers in queue
```
