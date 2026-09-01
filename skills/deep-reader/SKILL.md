# Skill 4: Deep Reader V2 — Research Reverse Engineering

## Purpose

Go beyond summarization. Reverse-engineer a paper to understand how the research was conceived, why the methods were chosen, where the argument is weakest, and how to build upon or challenge it.

This skill is what separates a literature manager from a research training tool.

## Trigger

- Manual: user uploads a PDF and says "deep read" or "analyze this paper"
- Semi-automatic: Queue Manager recommends a paper for deep reading

## Frequency

~5 papers per week (adjustable via researcher profile).

---

## Input

```
Required:
  - PDF of the paper
  - DOI (for database linking)

Optional:
  - Researcher profile (for Section 12: Novel Follow-Up)
  - Active projects (for connection mapping)
```

---

## Step 0: Paper Type Detection

Before generating any sections, classify the paper.

```
Detect paper type:
  - empirical_quantitative
  - empirical_qualitative
  - mixed_methods
  - theoretical_essay
  - methodological
  - meta_analysis
  - systematic_review

Detection signals:
  - Has regression tables / statistical output → quantitative
  - Has interview excerpts / thematic analysis → qualitative
  - Has both → mixed_methods
  - No empirical data, builds framework → theoretical_essay
  - Proposes or evaluates a method → methodological
  - Aggregates prior studies quantitatively → meta_analysis
  - PRISMA diagram or systematic search → systematic_review
```

This classification determines which sections are generated (see Section Activation Matrix below).

---

## Section Activation Matrix

Not all sections apply to all paper types. Generating irrelevant sections produces filler that wastes the researcher's time.

```
Section                          | Quant | Qual | Theory | Methods | Meta
---------------------------------|-------|------|--------|---------|------
 0. Executive Summary            |   ✓   |  ✓   |   ✓    |   ✓     |  ✓
 1. Research Question Analysis   |   ✓   |  ✓   |   ✓    |   ✓     |  ✓
 2. Gap Assessment               |   ✓   |  ✓   |   ✓    |   ✓     |  ✓
 3. Reconstructed Author Thinking|   ✓   |  ✓   |   ✓    |   ✓     |  ✗
 4. Method Intuition             |   ✓   |  ✓   |   ✗    |   ✓     |  ✓
 5. Method Pipeline + Example    |   ✓   |  ✗   |   ✗    |   ✓     |  ✗
 6. Math/Stats Interpreter       |   ✓*  |  ✗   |   ✗    |   ✓*    |  ✓*
 7. RQ → Design → Answer        |   ✓   |  ✓   |   ✗    |   ✗     |  ✓
 8. Main Takeaways               |   ✓   |  ✓   |   ✓    |   ✓     |  ✓
 9. One-Week Replication Plan    |   ✓   |  ✗   |   ✗    |   ✓     |  ✗
10. Most Fragile Claim           |   ✓   |  ✓   |   ✓    |   ✓     |  ✓
11. Adversarial Research Design  |   ✓   |  ✓   |   ✓    |   ✗     |  ✓
12. Novel Follow-Up Research     |   ✓   |  ✓   |   ✓    |   ✓     |  ✓

✓* = only if the paper contains formal models or equations
✗  = skipped entirely — do NOT generate filler content
```

---

## Layered Output System

The 12 sections are grouped into 3 layers. Not every read needs all layers.

```
Layer 1 — Quick Read (always generated):
  Section 0:  Executive Summary
  Section 1:  Research Question Analysis (condensed)
  Section 8:  Main Takeaways
  Section 10: Most Fragile Claim

Layer 2 — Understanding (generated for papers scored > 8.0 or matching active project):
  Section 2:  Gap Assessment
  Section 3:  Reconstructed Author Thinking
  Section 4:  Method Intuition
  Section 7:  RQ → Design → Answer

Layer 3 — Engagement (generated only on explicit request):
  Section 5:  Method Pipeline + Example
  Section 6:  Math/Stats Interpreter
  Section 9:  One-Week Replication Plan
  Section 11: Adversarial Research Design
  Section 12: Novel Follow-Up Research
```

### Default Behavior

```
- Always generate Layer 1
- Auto-generate Layer 2 if:
    relevance_score > 8.0  OR  project_match is not empty
- Generate Layer 3 only when user says:
    "full analysis" or "layer 3" or "I want everything"
```

### Why Layers Matter

```
Without layers:  12 sections × 5 papers/week = 60 sections to read
With layers:     Layer 1 for all 5 = 20 sections
                 Layer 2 for top 3 = 12 more sections
                 Layer 3 for 1 key paper = 5 more sections
                 Total: 37 sections (~40% reduction)
                 Token savings: ~50% on average
```

---

## Section Specifications

### Section 0: Executive Summary

```markdown
Prompt:
  Summarize the paper in exactly:
    - One citation line (APA format)
    - One sentence: what is the contribution
    - Three sentences: summary of argument and findings
    - One sentence: why it matters for [researcher's active projects]

Output example:

  Paper: Smith & Jones (2026)
  Citation: Smith, A., & Jones, B. (2026). Title. Criminology, 64(3), 401-430.

  One-sentence contribution:
  First causal evidence that neighborhood gambling restrictions
  reduce property crime.

  Summary:
  The authors exploit a natural experiment created by staggered
  policy adoption across Australian states. Using a DiD design
  with 10 years of panel data, they find a 12% reduction in
  property offenses within 2km of restricted zones. Effects are
  concentrated in the first 18 months and fade over time.

  Why it matters for you:
  Directly relevant to your gambling policy + causal inference work.
```

### Section 1: Research Question Analysis

```markdown
Prompt:
  Identify the primary research question.
  Explain:
    1. What exactly is the question?
    2. Why is this question important?
    3. What would change if it were answered convincingly?
    4. Who should care? (researchers, policymakers, practitioners)

Formatting:
  Keep each answer to 2-3 sentences maximum.
  Do not pad with generic statements like
  "this is an important topic in criminology."
```

### Section 2: Gap Assessment

```markdown
Prompt:
  What gap did the paper identify?

  Distinguish:
    - Empirical gap: what data or evidence was missing?
    - Theoretical gap: what concept or mechanism was unexplained?
    - Methodological gap: what analytical limitation existed?

  Then evaluate:
    Did the authors actually answer their research question?
    - Fully: the design convincingly addresses the question
    - Partially: answers part of it, leaves part open
    - Weakly: claims exceed what the design can support

  Be specific about what remains unanswered.
```

### Section 3: Reconstructed Author Thinking (CONSTRAINED)

This is the most valuable and most dangerous section. Valuable because understanding how researchers generate ideas is the core PhD skill. Dangerous because AI can fabricate convincing but fictional narratives.

```markdown
Prompt:
  Reconstruct the likely reasoning path that led to this research.

  HARD CONSTRAINTS — you may ONLY draw on:
    1. Papers cited in the introduction and literature review
       (what failures or gaps did the authors explicitly reference?)
    2. The authors' prior publications
       (what were they working on before this paper?
        check the reference list for self-citations)
    3. Stated motivations in the paper
       (practical problems, policy debates, theoretical tensions
        mentioned in the text)
    4. Methodological developments cited
       (new tools, datasets, or techniques that became available)

  Do NOT speculate beyond these four anchors.

  If you cannot reconstruct the path from available evidence,
  say explicitly: "Insufficient evidence to reconstruct
  the authors' reasoning path beyond what is stated in the paper."

  Format:
    Known starting points:
      [what the authors were working on / what literature they cited]
    Identified gap:
      [anchored in specific cited papers or stated problems]
    Available opportunity:
      [new method, new data, new policy event]
    Resulting research idea:
      [how these converge into this specific paper]

  Label each inference with its source:
    (cited: Smith 2020)
    (self-citation: Author's prior work, 2023)
    (stated in paper: p.4)
    (inferred — lower confidence)
```

### Section 4: Method Intuition

```markdown
Prompt:
  Explain the intuition of the method BEFORE any technical detail.

  Use plain language. Assume the reader understands research design
  concepts but has not used this specific method.

  Answer:
    1. What problem does this method solve?
       (what would go wrong with a simpler approach?)
    2. What is the core logic?
       (in one paragraph, no equations)
    3. What assumptions does it require?
       (in plain language)
    4. What is the closest everyday analogy?

  Example for Difference-in-Differences:
    "Imagine two cities. One adopts a policy, one does not.
     Instead of comparing the cities directly — which is unfair
     because they differ in many ways — you compare how each
     city CHANGED over time. The untreated city's change estimates
     what would have happened without the policy. The difference
     between the two changes is your causal estimate.
     This only works if both cities were on similar trajectories
     before the policy."
```

### Section 5: Method Pipeline + Example

```markdown
Prompt:
  Lay out the methodology as a numbered pipeline.
  Then walk through the pipeline with a concrete example
  using the paper's actual variables but simplified hypothetical data.

  Format:
    Pipeline:
      Step 1: [what to do]
      Step 2: [what to do]
      ...

    Worked Example:
      Step 1: [concrete data example]
      Step 2: [concrete calculation]
      ...

  The example should be simple enough to reproduce
  on paper or in a spreadsheet.
```

### Section 6: Math/Stats Interpreter

```markdown
Activation rule:
  ONLY generate this section if the paper contains formal models,
  equations, or statistical frameworks beyond standard regression.
  Do NOT generate for papers that merely report coefficients.
  If unsure, skip.

Prompt (for frequentist papers):
  For each formal model:
    1. Show the equation exactly as written in the paper
    2. Plain-language meaning of every term
    3. Intuition: what is this equation trying to capture?
    4. Why this model instead of a simpler one?
    5. Worked example with hypothetical numbers
       (use the paper's variable names, fake data)

Prompt (for Bayesian papers):
  1. Prior:
     - What assumption does the prior encode?
     - Why did the authors choose this prior?
     - How sensitive are results to prior choice?
  2. Likelihood:
     - What data-generating process is assumed?
  3. Posterior:
     - How to interpret the posterior distribution
     - What does the credible interval mean vs. confidence interval?
  4. Sensitivity:
     - What happens if the prior is changed?
     - Did the authors report sensitivity analyses?

Prompt (for causal inference papers):
  1. Identify the causal estimand:
     ATE / ATT / LATE / CATE / other
  2. State the identification assumption in plain language
  3. Explain what would violate the assumption
  4. Did the authors test for violations? How?
  5. How convincing is the identification strategy? (1-10 with reason)
```

### Section 7: RQ → Design → Answer

```markdown
Prompt:
  Map the paper's logic in three steps.
  Keep each step to 1-3 sentences. Core idea only.

  Format:
    Research Question:
      [the question]
           ↓
    Research Design:
      [how they tried to answer it]
           ↓
    Answer:
      [what they found]

  This section should be readable in under 30 seconds.
```

### Section 8: Main Takeaways

```markdown
Prompt:
  List exactly 5 takeaways. No more, no fewer.
  Each takeaway is ONE sentence.
  No paragraphs. Force concise thinking.

  Format:
    1. [takeaway]
    2. [takeaway]
    3. [takeaway]
    4. [takeaway]
    5. [takeaway]
```

### Section 9: One-Week Replication Plan

```markdown
Prompt:
  Suppose a PhD student has one week to replicate
  the central idea of this paper.

  Not a full replication — a minimum viable replication
  that tests whether the core finding holds.

  Format:
    Day 1: [task — data acquisition]
    Day 2: [task — data cleaning]
    Day 3: [task — descriptive reproduction]
    Day 4: [task — primary model]
    Day 5: [task — robustness check]

    Critical component:
      [the one thing that, if done wrong, collapses the replication]

    Data sources:
      [where to get the data, or closest substitute if original
       data is not public]

    Tools needed:
      [R/Stata/Python packages required]

    Feasibility assessment:
      [Feasible / Partially feasible / Not feasible without
       original data — with explanation]
```

### Section 10: Most Fragile Claim

```markdown
Prompt:
  Identify the single most fragile assumption, claim,
  or empirical finding in this paper.

  Explain:
    1. What is the claim?
    2. Why is it fragile?
    3. What evidence would weaken or overturn it?
    4. Did the authors acknowledge this fragility?

  Be specific. "The causal identification could be questioned"
  is too vague. State WHICH assumption and WHY it might fail.
```

### Section 11: Adversarial Research Design

```markdown
Prompt:
  Suppose you believe the authors' conclusion is wrong.
  Design a study that would challenge or reject their findings.

  Format:
    Alternative hypothesis:
      [what you think might be true instead]

    Proposed design:
      [method that would test the alternative]

    Why this design challenges the original:
      [what weakness in the original does it exploit?]

    Potential result:
      [what finding would undermine the original paper?]

    Feasibility:
      [could a PhD student actually do this?]

  This is a training exercise in critical thinking,
  not a claim that the paper is wrong.
```

### Section 12: Novel Follow-Up Research (WITH EXISTENCE CHECK)

```markdown
Prompt:
  Propose one follow-up research idea.

  Hard constraints — the idea CANNOT be:
    - Different sample
    - Longer time period
    - Another country
    - More control variables
  These are low-value extensions.

  The idea MUST target one of:
    - Theoretical gap (mechanism unexplained)
    - Methodological limitation (design can't answer X)
    - Measurement limitation (key variable poorly measured)
    - Mechanism uncertainty (effect found, pathway unknown)
    - Policy necessity (practical question unanswered)

  REQUIRED EXISTENCE CHECK:
    Before presenting the idea:
    1. State the proposed idea in one sentence
    2. Generate 2-3 Google Scholar search queries that would
       find existing work on this idea
    3. Based on your knowledge, does similar work likely exist?
       - If yes: acknowledge and explain how your proposal differs
       - If uncertain: flag explicitly
         ("This idea may already exist — verify with these
          searches before pursuing")
       - If likely novel: state why you believe so

  Output format:
    Core Question:
      [one sentence]
    Why It Matters:
      [2-3 sentences]
    Methodological Approach:
      [proposed design]
    Feasibility:
      Data: [available? where?]
      Timeline: [realistic estimate]
      Skills needed: [what the researcher needs to know]
    Existence Risk: [low / medium / high / uncertain]
    Verification Queries:
      1. "[search query for Google Scholar]"
      2. "[search query]"
      3. "[search query]"
    Target Journals: [ranked by fit, with reasoning]
    Publishability Assessment: [with reasoning]
```

---

## Database Updates After Deep Read

```yaml
status: evaluated → deep_read
deep_review_complete: true
deep_review_layer: 1 | 2 | 3
paper_type: [detected type]
pdf_available: true
date_last_updated: today
```

---

## Token Cost

```
Layer 1 only:       4,000 - 6,000 tokens
Layer 1 + 2:        8,000 - 12,000 tokens
Layer 1 + 2 + 3:   14,000 - 18,000 tokens

Estimated weekly usage (5 papers):
  5 × Layer 1:              25,000 - 30,000
  3 × Layer 2 (auto):       24,000 - 36,000
  1 × Layer 3 (on request): 14,000 - 18,000
  Weekly total:              63,000 - 84,000
  Monthly total:            ~280,000 tokens
```

---

## Key Differences from Original Design

| Aspect | Original (V1) | Revised (V2) |
|--------|---------------|--------------|
| Paper type | Ignored | Auto-detected, controls section activation |
| Output size | Always 12 sections | 4-12 sections depending on type + layer |
| Section 3 | Unconstrained speculation | Anchored to 4 evidence sources only |
| Section 6 | Generic | Separate prompts for Bayesian / causal / frequentist |
| Section 12 | No existence check | Required verification queries |
| Token cost | ~15,000 per paper always | 4,000-18,000 depending on depth needed |
| Flexibility | All or nothing | Layer 1/2/3 progressive depth |
