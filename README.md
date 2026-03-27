# TransparencIA Lexicons

[AT Protocol](https://atproto.com/) Lexicon definitions for **TransparencIA** — Mexican government transparency data on the decentralized web.

These lexicons define the schema for news articles, AI enrichments, and media sources published to the AT Protocol network. They are designed to be indexed by [Hyperindex](https://github.com/hypercerts-org/hyperindex) (or any AT Protocol AppView) and queried via GraphQL.

## Namespace

All lexicons use the `mx.transparencia.*` NSID namespace.

```
mx.transparencia.defs                  — Shared type definitions
mx.transparencia.news.article          — News article record
mx.transparencia.news.enrichment       — AI enrichment (sidecar)
mx.transparencia.news.source           — News source registry
```

### Future namespaces (planned)

```
mx.transparencia.dof.*                 — Diario Oficial de la Federación
mx.transparencia.leg.*                 — Legislative data (convocatorias, legisladores)
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Data Pipeline (existing)                   │
│                                                              │
│  RSS Feeds ──► NewsMex (scrape + enrich) ──► Supabase       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                    Publisher (planned)
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                   AT Protocol PDS                             │
│                                                              │
│  Account: @news.transparencia.mx                             │
│                                                              │
│  Collections:                                                │
│    mx.transparencia.news.source/      (7 records)            │
│    mx.transparencia.news.article/     (16,000+ records)      │
│    mx.transparencia.news.enrichment/  (3,800+ records)       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                    Jetstream (real-time)
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│              transparencia-index (Hyperindex)                 │
│                                                              │
│  /graphql       — Public API                                 │
│  /graphql/ws    — Real-time subscriptions                    │
│  /graphiql      — GraphQL playground                         │
└──────────────────────────────────────────────────────────────┘
```

## Lexicon Overview

### `mx.transparencia.defs`

Shared types reused across lexicons:

| Type | Description |
|------|-------------|
| `#location` | Geographic location with coordinates (lat/lng as strings) |
| `#person` | Named person with role and sentiment analysis |
| `#claim` | Factual claim extracted from content |
| `#economicIndicator` | Economic data point (name, value, direction) |
| `#timelineEntry` | Chronological event with date |

### `mx.transparencia.news.article`

A news article as published by a media outlet. Contains only the raw content — no AI analysis.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | ✅ | Original headline |
| `url` | uri | ✅ | Canonical article URL |
| `source` | strongRef | ✅ | Reference to the source record |
| `author` | string | | Byline |
| `description` | string | | Lead paragraph / RSS description |
| `imageUrl` | uri | | Featured image URL |
| `feedCategory` | string | | RSS feed category |
| `guid` | string | | RSS GUID for deduplication |
| `publishedAt` | datetime | ✅ | When the source published it |
| `createdAt` | datetime | ✅ | When this AT Protocol record was created |

### `mx.transparencia.news.enrichment`

AI-generated structured metadata for an article. Uses the **sidecar pattern** — references the article via `strongRef` so multiple enrichments can coexist (different models, different analysts, updated analyses).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `article` | strongRef | ✅ | Reference to the article being enriched |
| `summary` | string | ✅ | 2-3 sentence AI summary |
| `neutralHeadline` | string | | Bias-stripped rewrite of the headline |
| `politicalOrientation` | string | ✅ | `izquierda` / `centro-izquierda` / `centro` / `centro-derecha` / `derecha` / `neutral` |
| `orientationConfidence` | string | | Confidence score (0-1 as string) |
| `orientationReasoning` | string | | Why this orientation was assigned |
| `emotionalTone` | string | | `alarmista` / `critico` / `neutral` / `optimista` / `celebratorio` / `melancolico` |
| `impactLevel` | integer | ✅ | 1-10 national relevance score |
| `impactReasoning` | string | | Explanation of the impact score |
| `eventType` | string | ✅ | Primary category (12 known values) |
| `readingLevel` | string | | `basico` / `intermedio` / `avanzado` |
| `factCheckability` | integer | | 1-5 verifiability score |
| `claims` | claim[] | | Extracted factual claims |
| `topics` | string[] | | Main topics (max 10) |
| `people` | person[] | | People mentioned with sentiment |
| `organizations` | string[] | | Organizations mentioned |
| `locations` | location[] | | Geographic references |
| `economicIndicators` | economicIndicator[] | | Economic data points |
| `timeline` | timelineEntry[] | | Chronological events |
| `relatedLegislation` | string[] | | Legal references |
| `relatedKeywords` | string[] | | Keywords for clustering |
| `modelUsed` | string | ✅ | AI model identifier |
| `modelVersion` | string | | Model version/checkpoint |
| `createdAt` | datetime | ✅ | When enrichment was generated |

### `mx.transparencia.news.source`

Registry of news outlets being tracked.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✅ | Machine-readable slug |
| `displayName` | string | ✅ | Human-readable name |
| `baseUrl` | uri | ✅ | Outlet website URL |
| `country` | string | ✅ | ISO 3166-1 alpha-2 code |
| `language` | string | | ISO 639-1 language code |
| `cms` | string | | CMS platform |
| `logoUrl` | uri | | Logo image URL |
| `description` | string | | Editorial description |
| `feedUrls` | feedEntry[] | | RSS/Atom feeds monitored |
| `createdAt` | datetime | ✅ | When registered |

## Design Decisions

### Why strings for coordinates and confidence scores?

AT Protocol's data model [does not support floating-point numbers](https://atproto.com/specs/data-model#why-no-floats). Coordinates and decimal values are encoded as strings (e.g., `"19.4326"`) and parsed by consumers.

### Why the sidecar pattern for enrichments?

The [AT Protocol style guide](https://atproto.com/guides/lexicon-style-guide#design-patterns) recommends sidecar records for supplementary data. This means:

- **Multiple enrichments per article** — different AI models or analysts can publish competing enrichments
- **Enrichments can be updated** without breaking references to the original article
- **Third parties can enrich** — a university lab could publish their own `mx.unam.news.enrichment` records referencing the same articles

### Why `knownValues` instead of `enum`?

AT Protocol enums are closed and can never be extended. `knownValues` allows new values to be added over time without breaking existing consumers. For example, new `eventType` categories can be added as coverage expands.

### Why no embeddings in the lexicon?

Vector embeddings are an index optimization, not semantic data. They belong in a sidecar database (e.g., pgvector) rather than in AT Protocol records. The Hyperindex GraphQL API can be extended with custom vector search endpoints.

## Examples

See the [`examples/`](./examples/) directory for sample records:

- [`source.json`](./examples/source.json) — El Universal source registration
- [`article.json`](./examples/article.json) — A news article about Mexico-China trade
- [`enrichment.json`](./examples/enrichment.json) — AI enrichment of the same article

## Usage with Hyperindex

Register these lexicons with a [Hyperindex](https://github.com/hypercerts-org/hyperindex) instance:

```graphql
# Upload lexicons via the admin API
mutation {
  uploadLexicons(files: [...])
}
```

Then query indexed records:

```graphql
query {
  mxTransparenciaNewsEnrichment(
    first: 20
    where: {
      politicalOrientation: { eq: "izquierda" }
      impactLevel: { gte: 7 }
      eventType: { eq: "economia" }
    }
  ) {
    edges {
      node {
        summary
        neutralHeadline
        topics
        people
        locations
        economicIndicators
      }
    }
  }
}
```

## Current Data Sources

| Source | Country | Articles | Status |
|--------|---------|----------|--------|
| El Universal | 🇲🇽 MX | 16,000+ | Active |
| El Financiero | 🇲🇽 MX | Active | Active |
| Infobae México | 🇲🇽 MX | Active | Active |
| SDP Noticias | 🇲🇽 MX | Active | Active |
| El Economista | 🇲🇽 MX | Active | Active |
| Latinus | 🇲🇽 MX | Active | Active |
| Milenio | 🇲🇽 MX | Active | Active |

## Related Repositories

| Repo | Description |
|------|-------------|
| [NewsMex](https://github.com/TransparencIA-MX/NewsMex) | News scraping, enrichment, and clustering pipeline |
| [transparencia-web-v2](https://github.com/TransparencIA-MX/transparencia-web-v2) | Next.js frontend |
| [Hyperindex](https://github.com/hypercerts-org/hyperindex) | AT Protocol AppView indexer (upstream) |

## License

Apache License 2.0
