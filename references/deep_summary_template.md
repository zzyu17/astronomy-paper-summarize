---
name: deep_summary_template
description: "Output template for the deep summary report — used by report_compiler_agent during Phase 3 assembly"
version: "1.0.0"
---

# Deep Summary Template

The `report_compiler_agent` assembles output from `deep_reader_agent`, `methodology_analyst_agent`, `critical_evaluator_agent`, and `connection_synthesizer_agent` into this structure.

## Template

### Math & Formula Formatting

**All agents must use standard pandoc/LaTeX math notation.** This ensures correct rendering in both Markdown and PDF output.

| Syntax | Usage | Example |
|--------|-------|---------|
| `$...$` | **Inline math** — within paragraph text | `the stellar mass $M_\star = 1.2\,M_\odot$` |
| `$$...$$` | **Display math** — centered, standalone | `$$\chi^2 = \sum_i \frac{(O_i - E_i)^2}{\sigma_i^2}$$` |

**Rules:**
- Always wrap math symbols, Greek letters, subscripts, superscripts, and equations in `$...$` or `$$...$$`
- Use LaTeX math commands: `\alpha`, `\beta`, `\Gamma`, `\times`, `\approx`, `\sim`, `\propto`, `\pm`, `\leq`, `\geq`, `\gg`, `\ll`, `\partial`, `\nabla`, `\int`, `\sum`, `\prod`
- Units: `$\,\mathrm{km\,s^{-1}}$`, `$\,\mathrm{erg\,s^{-1}}$`
- Chemical elements in math mode: `$\mathrm{H}_2\mathrm{O}$`, `$\mathrm{CO}$`
- Subscripts with text: `$T_{\mathrm{eff}}$`, `$\log g$`, `$M_{\mathrm{vir}}$`
- Do NOT use Unicode Greek letters (α, β, γ) — use `$\alpha$`, `$\beta$`, `$\gamma$` for PDF compatibility

---

```markdown
# [Paper Title] — Deep Summary

---

## 1. Structured Paper Summary

### Introduction
[Research background, key questions, and objectives]

### Methods
[Core methodological framework, data sources, sample selection, analysis pipelines, statistical/computational tools]

### Results
[Primary findings and key evidence supporting the conclusions]

### Discussion
[Interpretation of results, connections to prior work, identified limitations]

### Conclusion
[Core takeaways and proposed future research directions]

## 2. Methodological Deep Dive

### Simplified Framework
[What the method does and why it was chosen over alternatives]

### Key Technical Details
[Specific observational designs, data analysis pipelines, statistical methods, robustness checks, critical parameters, datasets, software, code libraries worth referencing]

*Note: Technical details are adapted to detected paper type — observational / theoretical / computational.*

## 3. Critical Evaluation

### Strengths
[Major novel contributions, robust evidence, innovative methodological designs]

### Limitations
[Potential weaknesses in methodology/assumptions, uncertainties, caveats — both explicit and implicit]

### Evidence Robustness
[How reliable and generalizable the evidence is; adequacy of robustness checks; alternative explanations considered]

## 4. Research Connection & Relevance

### Hypothesis Alignment
[How the paper supports, challenges, or complements the user's research hypotheses]

### Methodological Inspiration
[Specific ways the paper's approaches can be adopted, modified, extended, or challenged in the user's research]

### Gap Filling
[Unaddressed research gaps and open questions the user's research can target]

### New Research Directions
1. [Novel research question or follow-up study — concrete and actionable]
2. [Novel research question or follow-up study — concrete and actionable]
3. [Novel research question or follow-up study — concrete and actionable]

## 5. Key Terminology & References

### Key Technical Terms
| Term | Definition & Context |
|------|---------------------|
| **[Term 1]** | [Concise yet clear definition, central to understanding the paper and critical to the user's research] |
| **[Term 2]** | [...] |

### High-Impact References
| # | Reference | Relevance to the User's Research |
|---|-----------|---------------------------|
| 1 | [Author (Year), *Journal*, DOI] | [1-sentence explanation of relevance] |
| 2 | ... | ... |

## 6. Overall Research Implication Summary

[A concise, high-level paragraph summarizing how this paper's insights should influence the design, execution, and/or interpretation of the user's research]

---
```

## Assembly Rules

1. All sections are required
2. Structured Summary (§1) follows Astrobites template: engaging lede → background → methods → results → implications
3. Methodological Deep Dive (§2) adapts detail level to detected paper type (observational/theoretical/computational)
4. Critical Evaluation (§3) maintains constructive tone — critical but aimed at empowering research progress
5. Research Connection (§4) is the most critical section — every insight must be anchored to the user's config
6. Key Terminology (§5) provides concise yet clear definitions; 3-5 high-impact references with 1-sentence relevance each
7. Paper title is sanitized for filesystem safety
