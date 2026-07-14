---
name: ads_api_protocol
description: "ADS API usage guide for paper metadata enrichment — used by paper_intake_agent during Phase 1"
version: "1.0.0"
---

# ADS API Protocol

The [Astrophysics Data System (ADS) API](https://api.adsabs.harvard.edu/v1/search/query) serves as the **primary** metadata source for paper intake.

## Endpoint

```
https://api.adsabs.harvard.edu/v1/search/query
```

## Authentication

Requires an ADS API token. The user sets this as an environment variable:
```bash
export ADS_API_TOKEN="your-token-here"
```

Get a token at: https://ui.adsabs.harvard.edu/user/settings/token

## Key Fields Returned

| Field | Description |
|-------|-------------|
| `title` | Paper title |
| `abstract` | Full abstract |
| `author` | Author list |
| `keyword` | Author-provided and ADS-assigned keywords |
| `year` | Publication year |
| `citation_count` | Number of citations |
| `arxiv_class` | arXiv category (e.g., astro-ph.EP) |
| `bibcode` | ADS bibliographic code (unique identifier) |
| `doi` | Digital Object Identifier |
| `pub` | Journal name |

## Query Patterns

### Pattern 1: Search by arXiv ID
```
GET ?q=arxiv:{arxiv_id}&fl=title,abstract,author,keyword,year,citation_count,arxiv_class,bibcode,doi,pub
```

### Pattern 2: Search by Title
```
GET ?q=title:"{paper_title}"&fl=title,abstract,author,keyword,year,citation_count,arxiv_class,bibcode,doi,pub
```

### Pattern 3: Search by Bibcode
```
GET ?q=bibcode:{bibcode}&fl=title,abstract,author,keyword,year,citation_count,arxiv_class,bibcode,doi,pub
```

## Rate Limits

- 5,000 queries per day (standard token)
- Higher limits available for institutional tokens

## Special Operators

| Operator | Usage | Description |
|----------|-------|-------------|
| `citations()` | `citations(bibcode:2020ApJ...905L..11B)` | Papers citing the given paper |
| `references()` | `references(bibcode:2020ApJ...905L..11B)` | Papers referenced by the given paper |
| `similar()` | `similar(bibcode:2020ApJ...905L..11B)` | Similar papers based on text analysis |
| `trending()` | `trending(astronomy)` | Currently trending papers |

## Error Handling

| Condition | Action |
|-----------|--------|
| No API token | Skip ADS, fall back to arXiv API |
| HTTP 401/403 | Report authentication error, fall back to arXiv API |
| HTTP 429 (rate limit) | Fall back to arXiv API |
| HTTP 5xx / timeout | Fall back to arXiv API |
| Query returns zero results | Try title search; if still zero, fall back to arXiv API |

## Integration Note

The paper's **full text** always comes from the user-provided file. ADS API provides metadata and abstract enrichment only — it does not replace the need for the full paper text.
