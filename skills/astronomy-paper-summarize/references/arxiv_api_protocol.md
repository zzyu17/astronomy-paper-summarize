---
name: arxiv_api_protocol
description: "arXiv API usage guide for paper metadata — used by paper_intake_agent as first fallback when ADS is unavailable"
version: "1.0.0"
---

# arXiv API Protocol

## Endpoint

```
https://export.arxiv.org/api/query
```

## Response Format

Atom 1.0 XML feed. A match yields one or more `<entry>` elements; a miss yields a feed with zero `<entry>` elements.

## Rate Limit

~3 seconds between requests (per arXiv terms of use). No polite-pool or higher-tier mechanism.

## Query Patterns

### Pattern 1: arXiv ID Lookup
```
GET ?id_list={arxiv_id}
```
Example: `?id_list=2301.12345`

### Pattern 2: Title Search
```
GET ?search_query=ti:"{title}"&max_results=5
```
Example: `?search_query=ti:"The Neptunian Desert Across Stellar Types"&max_results=5`

## Key Fields per Entry

| XML Path | Description |
|----------|-------------|
| `<entry><title>` | Paper title (arXiv may insert internal whitespace — collapse to single spaces) |
| `<entry><summary>` | Abstract |
| `<entry><published>` | ISO-8601 timestamp (first 4 digits = year) |
| `<entry><author><name>` | Author names |
| `<entry><arxiv:primary_category>` | Primary arXiv category |

## Degradation Handling

| Condition | Action |
|-----------|--------|
| Empty feed (zero `<entry>`) | Treat as miss — fall back to manual metadata entry |
| HTTP 429 (rate limit) | Back off 2 seconds, retry up to 3 times; after exhaustion, fall back to manual |
| HTTP 5xx | Fall back to manual metadata entry immediately |
| Network timeout (30s) | Fall back to manual metadata entry |
| Malformed XML | Fall back to manual metadata entry |

## Integration Note

arXiv API is the **first fallback** after ADS API. It provides metadata and abstract only — the paper's full text comes from the user-provided file.
