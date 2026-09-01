# System Configuration — Shared Components

All skills in this system share these two components. Individual skills read from and write to them.

---

## Adaptive Researcher Profile

This profile is the single source of truth for who the researcher is and what they care about. Every skill references it.

```yaml
researcher:
  name: "Wenyi Wang"
  stage: "PhD Student"
  field: "Criminology"

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
  # initialized equally at 0.10 each
  # updated every 30 days by Feedback Loop Engine (Skill 7)
  life_course:      0.10
  victimization:    0.10
  gender:           0.10
  gambling:         0.10
  causal_inference: 0.10
  spatial:          0.10
  aging:            0.10
  experiments:      0.10
  ai:               0.10
  bayesian:         0.10

active_projects:
  # add dissertation chapters, papers in progress
  # stage: ideation | literature_review | writing | revision | submitted
  - title: ""
    keywords: []
    stage: ""

weekly_reading_capacity: 5  # papers per week for deep reading
```

### How Weights Update

Every 30 days, the Feedback Loop Engine (Skill 7) computes:

```
For each interest area:
  read_rate = papers_deep_read / papers_recommended
  if read_rate > 0.5:  weight += 0.02
  if read_rate < 0.1:  weight -= 0.02

Constraints:
  - No weight below 0.03 (prevents filter bubble)
  - No weight above 0.25 (prevents tunnel vision)
  - All weights normalized to sum to 1.0
```

---

## Target Journal List

```yaml
tier_1_core:
  # Always scan. Highest journal_tier score.
  - name: "Criminology"
    issn: "0011-1384"
  - name: "Criminology & Public Policy"
    issn: "1538-6473"
  - name: "Journal of Research in Crime and Delinquency"
    issn: "0022-4278"
  - name: "Journal of Quantitative Criminology"
    issn: "0748-4518"
  - name: "British Journal of Criminology"
    issn: "0007-0955"
  - name: "Justice Quarterly"
    issn: "0741-8825"
  - name: "Journal of Experimental Criminology"
    issn: "1573-3750"
  - name: "Annual Review of Criminology"
    issn: "2572-4568"

tier_2_topical:
  # Scan monthly. Medium journal_tier score.
  - European Journal of Criminology
  - Journal of Criminal Justice
  - Journal of Developmental and Life-Course Criminology
  - Development and Psychopathology
  - Journal of Youth and Adolescence
  - "Psychology, Public Policy, and Law"
  - The Gerontologist
  - Feminist Criminology
  - Violence Against Women
  - Journal of Interpersonal Violence
  - Victims & Offenders
  - Sexual Abuse

tier_3_methods_and_niche:
  # Scan monthly. Lower journal_tier score.
  - Crime Science
  - Security Journal
  - Crime Prevention and Community Safety
  - "Environment and Planning B: Urban Analytics and City Science"
  - Addiction
  - International Gambling Studies
  - Journal of Gambling Studies
  - Sociological Methods & Research
  - Political Analysis
  - Journal of Causal Inference
```

---

## Paper Database Schema

Every paper exists exactly once. DOI is the primary key.

```json
{
  "doi": "",
  "title": "",
  "authors": [],
  "journal": "",
  "journal_tier": 1,
  "publication_date": "",
  "abstract": "",
  "abstract_status": "full | partial | missing",
  "keywords": [],
  "open_access": false,
  "url": "",

  "relevance_score": null,
  "topic_scores": {},
  "method_scores": {},
  "project_match": [],

  "status": "new",
  "paper_type": null,

  "zotero_saved": false,
  "zotero_key": "",
  "pdf_available": false,
  "deep_review_complete": false,
  "deep_review_layer": null,

  "notes": "",
  "date_first_seen": "",
  "date_last_updated": "",
  "source": "openalex | crossref | rss | manual"
}
```

### Status Lifecycle

```
new → evaluated → saved → pdf_obtained → deep_read → cited
                                                    → archived

Special statuses:
  manually_added    (from Zotero reverse sync)
  bulk_skipped      (from Scanner degradation, pending weekend review)
```

---

## System Health Monitor

Runs weekly (triggered by Queue Manager or manually).

```yaml
checks:
  - metric: harvester_last_run
    alert_if: "> 35 days ago"
    message: "DOI Harvester has not run in over a month."

  - metric: new_papers_unprocessed
    alert_if: "> 50"
    message: "Scanner backlog is growing. Consider running Scanner."

  - metric: backlog_total
    alert_if: "> 100"
    message: "Reading backlog exceeds 100. Triage recommended."

  - metric: scanner_last_run
    alert_if: "> 3 days ago"
    message: "Scanner has not run recently."

  - metric: deep_reads_this_week
    warn_if: "0"
    message: "No deep reads this week."

output: one-paragraph status summary + any alerts
```
