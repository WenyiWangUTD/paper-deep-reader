# Research Intelligence System — Skills

DOI-centered research reading system for criminology PhD.

## Architecture

```
                    ┌─────────────────────┐
                    │   SYSTEM_CONFIG     │
                    │  Researcher Profile │
                    │  Journal List       │
                    │  Database Schema    │
                    │  Health Monitor     │
                    └────────┬────────────┘
                             │
          ┌──────────────────┼──────────────────────┐
          │                  │                       │
          ▼                  ▼                       ▼
   ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐
   │ Skill 1     │  │ Skill 2      │  │ Skill 7           │
   │ DOI         │  │ Landscape    │  │ Feedback Loop     │
   │ Harvester   │  │ Analyst      │  │ Engine            │
   │ (monthly)   │  │ (monthly)    │  │ (monthly)         │
   └──────┬──────┘  └──────────────┘  └─────────┬─────────┘
          │                                      │
          │  populates                  adjusts weights
          ▼                                      │
   ┌─────────────┐                               │
   │ DOI         │◄──────────────────────────────┘
   │ Database    │
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐     ┌──────────────┐     ┌─────────────┐
   │ Skill 3     │────▶│ Skill 4      │────▶│ Skill 5     │
   │ Daily       │     │ Deep Reader  │     │ Zotero      │
   │ Scanner     │     │ V2           │     │ Sync        │
   │ (daily)     │     │ (on demand)  │     │ (per paper) │
   └─────────────┘     └──────────────┘     └──────┬──────┘
                                                   │
          ┌────────────────────────────────────────┘
          ▼
   ┌─────────────┐
   │ Skill 6     │
   │ Queue       │
   │ Manager     │
   │ (weekly)    │
   └─────────────┘
```

## Skills

| # | Skill | Frequency | File |
|---|-------|-----------|------|
| — | System Config | shared | `SYSTEM_CONFIG.md` |
| 1 | DOI Harvester | monthly | `doi-harvester/SKILL.md` |
| 2 | Monthly Landscape Analyst | monthly | `landscape-analyst/SKILL.md` |
| 3 | Daily Relevance Scanner | daily | `daily-scanner/SKILL.md` |
| 4 | Deep Reader V2 | on demand | `deep-reader/SKILL.md` |
| 5 | Zotero Sync | per paper + weekly | `zotero-sync/SKILL.md` |
| 6 | Reading Queue Manager | weekly | `queue-manager/SKILL.md` |
| 7 | Feedback Loop Engine | monthly | `feedback-loop/SKILL.md` |

## Build Order

```
Phase 1 (Foundation):     Database + Harvester + Scanner
Phase 2 (Depth):          Deep Reader (Layer 1, then Layer 2)
Phase 3 (Integration):    Zotero Sync + Deep Reader Layer 3
Phase 4 (Intelligence):   Queue Manager + Landscape Analyst + Feedback Loop
Phase 5 (Maturity):       Zotero reverse sync + adaptive scoring + auto-triage
```

## Estimated Token Usage

```
Monthly total: ~670,000 tokens

Breakdown:
  DOI Harvester:         5,000
  Landscape Analyst:    55,000
  Daily Scanner:       270,000
  Deep Reader:         280,000
  Zotero Sync:          30,000
  Queue Manager:        20,000
  Feedback Loop:         4,000
```
