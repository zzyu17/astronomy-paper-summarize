---
name: rough_overview_template
description: "Output template for the rough overview report — used by report_compiler_agent during Phase 3 assembly"
version: "1.0.0"
---

# Rough Overview Template

The `report_compiler_agent` assembles output from `rough_skimmer_agent` and `relevance_assessor_agent` into this structure.

## Template

```markdown
# [Paper Title] — Rough Overview

---

## 1. Core Information Summary

| Element | Details |
|---------|---------|
| **Research Question** | [What specific problem or objective does the paper address?] |
| **Methodology** | [High-level: primary data sources, observational tools, analytical methods, theoretical models, simulation techniques] |
| **Core Findings** | 1. [Most significant result 1]<br>2. [Most significant result 2]<br>3. [Most significant result 3] |
| **Stated Limitations** | [Key limitations or open questions explicitly stated by authors; omit if none prominent] |

## 2. Relevance Assessment

| Dimension | Weight | Score | Rationale |
|-----------|--------|-------|-----------|
| **Topic Match** | 35% | [1-10] | [How the paper's topic aligns with the user's core_topic] |
| **Methodology Match** | 25% | [1-10] | [How the paper's methods align with the user's proposed_methodology] |
| **Target Match** | 20% | [1-10] | [How the paper's targets/sample align with the user's research focus] |
| **Impact** | 10% | [1-10] | [Citation count, journal tier, author prominence in subfield] |
| **Timeliness** | 10% | [1-10] | [Recency, connection to active debates, relevance to upcoming surveys] |
| **Weighted Total** | **100%** | **[weighted score]** / 10 | — |
| **Classification** | | **[Must-Read / Maybe / Not-Essential]** | [1-sentence explanation] |

### Project Implications
1. [Key insight 1 for the user's research]
2. [Key insight 2 for the user's research]
3. [Key insight 3 for the user's research]

## 3. Quick Takeaway

> [Single sentence summarizing the most critical piece of information for the user's research]

---
```

## Assembly Rules

1. All fields are required unless marked optional ("omit if none prominent")
2. Jargon is simplified inline — no separate glossary in rough mode
3. Brevity is paramount: bullet points, short sentences
4. Relevance score uses the 5-factor weighted framework from `relevance_assessor_agent`
5. Paper title is sanitized for filesystem safety (remove `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`)
