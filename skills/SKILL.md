# Skill 1: DOI Harvester

## Purpose

Build and maintain the master paper database. This is the foundation of the entire system — all other skills read from the database this skill populates.

## Trigger

- Monthly: scheduled run on the 1st of each month
- Manual: when user says "harvest" or "scan journals"

## Frequency

Once per month. Can run more often during conference seasons or when a specific journal publishes a special issue.

---

## Data Source Strategy

```
Primary:    OpenAlex API
            - Free, no API key required (polite pool: add email in header)
            - Structured metadata: title, abstract, authors, concepts, DOI
            - Best abstract coverage among free sources
            - Endpoint: https://api.openalex.org/works

Secondary:  CrossRef API
            - DOI resolution and citation counts
            - Sometimes has abstracts OpenAlex misses
            - Endpoint: https://api.crossref.org/works

Fallback:   Journal RSS / Atom feeds
            - For journals poorly covered by OpenAlex
            - Unstructured — requires parsing per journal
            - Only used if OpenAlex + CrossRef both miss a journal
```

### Why This Order

OpenAlex returns the richest structured data in a single call. CrossRef is reliable for DOI metadata but often lacks abstracts. RSS feeds are journal-specific and fragile. Starting with OpenAlex minimizes the number of fallback calls needed.

---

## Harvest Logic

```
Input:
  - Target journal list (from SYSTEM_CONFIG.md)
  - last_harvest_date (stored in database metadata)

For each journal in target list:
  1. Query OpenAlex:
     filter = journal ISSN + publication_date >= last_harvest_date

  2. For each paper returned:
     a. Extract DOI
     b. DOI exists in database?
        YES → compare metadata
              - if changed (e.g., issue assigned, citation count updated):
                update record, set date_last_updated = today
              - if unchanged: skip
        NO  → insert new record:
              status = "new"
              date_first_seen = today
              source = "openalex"

     c. Abstract present?
        YES → store, abstract_status = "full"
        NO  → query CrossRef for this DOI
              - CrossRef has abstract? → store, abstract_status = "full"
              - Still no abstract? → abstract_status = "missing"
                flag for manual check

  3. Log per-journal stats

After all journals processed:
  Update last_harvest_date = today
  Emit health signal to System Health Monitor
```

### Papers Without DOI

Rare but possible (editorials, book reviews, commentaries).

```
If no DOI:
  Generate internal ID: "nodoi_[journal]_[date]_[first_author_surname]"
  Store with flag: needs_doi_verification = true
  These papers are excluded from deduplication logic
  until DOI is manually added or confirmed absent.
```

---

## Degradation Strategy

Peak publishing periods (December, June) can produce 2-3x normal volume.

```
If new papers in a single harvest > 80:

  Phase 1: Title + keyword pre-filter
    - Keep papers whose title contains ANY keyword from
      static_interests in researcher profile
    - Keep ALL papers from Tier 1 journals regardless
    - Discard obvious mismatches
      (e.g., a Gerontologist paper on Alzheimer's nursing
       with zero criminology keywords)

  Phase 2: Full metadata storage for Phase 1 survivors
    - Store complete records
    - Discarded papers stored with status = "harvester_filtered"
      (recoverable, but not sent to Scanner)

This prevents token cost explosion in downstream skills.
```

---

## Output

```markdown
DOI Harvest Report — [Date]

Journals scanned:     30
Total papers found:   142
Already in database:  128
New papers added:     14
  - With full abstract:    11
  - Abstract missing:       2 (flagged)
  - Title only:             1

Journal breakdown:
  Criminology:                    3 new
  Journal of Quantitative Crim:   2 new
  Justice Quarterly:              0 new
  British Journal of Crim:        4 new
  Journal of Gambling Studies:    2 new
  ...

Harvest window:  2026-08-01 to 2026-09-01
Next scheduled:  2026-10-01
```

---

## Database Updates

```yaml
records_inserted:
  status: "new"
  date_first_seen: today

records_updated:
  date_last_updated: today
  (metadata fields only — status unchanged)

health_signal:
  harvester_last_run: today
  new_papers_unprocessed: += [count of new papers]
```

---

## Error Handling

```
- OpenAlex timeout → retry once, then log and move to next journal
- CrossRef rate limit → wait 5 seconds, retry
- Journal not found in OpenAlex → check ISSN, try journal name search
  If still not found → log as "journal_not_indexed"
  and fall back to RSS if available
- Duplicate DOI from different journals → keep both journal associations
  on the same record (some papers appear in multiple venues)
```

---

## Token Cost

This skill uses minimal AI tokens because it is primarily an API data-collection task.

```
Per run:      2,000 - 5,000 tokens (for generating the harvest report)
Frequency:    1x / month
Monthly cost: ~5,000 tokens
```

The heavy token usage happens downstream in Scanner and Deep Reader, not here.
