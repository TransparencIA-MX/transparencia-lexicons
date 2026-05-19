# TransparencIA Lexicons

[AT Protocol](https://atproto.com/) Lexicon definitions for **TransparencIA** — public-interest news and official document analysis on the decentralized web, born in Mexico, designed for the world.

These lexicons define the schema for news articles, official documents, AI enrichments, and sources published to the AT Protocol network. They are designed to be indexed by [Hyperindex](https://github.com/hypercerts-org/hyperindex) (or any AT Protocol AppView) and queried via GraphQL.

The schema is **universal** — it works for any country, any language, and any public-interest domain. The initial data sources are Mexican, but the lexicon is designed to cover global news and official documents from day one.

## Namespace

All lexicons use the `tech.transparencia.*` NSID namespace. The authority domain is `transparencia.tech`.

> **Note:** The namespace reflects the authority domain (`transparencia.tech`), not a content restriction. Articles and enrichments from any country or language are valid.

```
tech.transparencia.defs                              — Shared type definitions
tech.transparencia.document.item                     — Official/institutional document record
tech.transparencia.document.source                   — Document source/publisher registry
tech.transparencia.document.mxdof.note               — DOF nota sidecar (Mexico)
tech.transparencia.document.mxdof.note.enrichment    — AI enrichment for DOF notas
tech.transparencia.news.article                      — News article record
tech.transparencia.news.enrichment                   — AI enrichment (sidecar)
tech.transparencia.news.source                       — News source registry
```

### Future namespaces (planned)

```
tech.transparencia.document.part                     — Document sections, entries, chapters, or notes
tech.transparencia.document.chunk                    — Citable extracted text chunks
tech.transparencia.document.enrichment               — Cross-source AI enrichment sidecar for documents
tech.transparencia.document.mxdof.diario             — Daily DOF gazette edition record
tech.transparencia.leg.*                             — Legislative data (convocatorias, legisladores)
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
  │    tech.transparencia.news.source/      (7 records)          │
  │    tech.transparencia.news.article/     (16,000+ records)    │
  │    tech.transparencia.news.enrichment/  (3,800+ records)     │
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

### `tech.transparencia.defs`

Shared types reused across lexicons:

| Type | Description |
|------|-------------|
| `#location` | Geographic location with country, coordinates (lat/lng as strings) |
| `#person` | Named person with role and sentiment analysis |
| `#claim` | Factual claim extracted from content |
| `#economicIndicator` | Economic data point (name, value, direction) |
| `#timelineEntry` | Chronological event with date |
| `#organization` | Organization/company/brand with ticker, sector, and sentiment |
| `#structuredRef` | Typed reference to an external entity (law, stock, product, team, etc.) |

### `tech.transparencia.document.item`

A canonical document record for official and institutional documents. This is the base layer for DOF publications, UNFCCC documents, reports, laws, agreements, environmental documents, education policy documents, audits, budgets, court rulings, and other public-interest records.

The record stores identity, provenance, classification, dates, issuing bodies, and external identifiers. It intentionally does **not** store full text, sections, chunks, AI analysis, or ingestion pipeline state; those belong in future sidecar records or internal pipeline tables.

Publisher/repository metadata (name, base URL, default license) lives on a separate `tech.transparencia.document.source` registry record, referenced via `strongRef`. Per-document retrieval metadata (canonical URLs, MIME type, checksums, access status) lives in the inline `retrieval` object — which also accepts an optional `blob` (PDF / HTML / plain text / JSON, up to 50 MB) so the document bytes can be preserved on the PDS alongside the metadata.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | ✅ | Official or source-provided title |
| `documentType` | string | ✅ | Open document category such as `official-publication`, `official-gazette-issue`, `decree`, `agreement`, `report`, `submission` |
| `source` | strongRef | ✅ | Reference to the `document.source` record for the publisher/repository (e.g., DOF, UNFCCC) |
| `retrieval` | retrieval | ✅ | Per-document retrieval metadata: URL, canonical URL, PDF URL, MIME type, sha256, file size, access status |
| `publishedAt` | datetime | ✅ | When the source published the document (use midnight UTC when only a date is known) |
| `createdAt` | datetime | ✅ | When this AT Protocol record was created |
| `subtitle` | string | | Secondary title or heading |
| `description` | string | | Source-provided description or short abstract |
| `language` | language | | BCP-47 language code (e.g., `es-MX`, `en`, `pt-BR`) |
| `country` | string | | ISO 3166-1 alpha-2 country code |
| `jurisdiction` | string | | Legal or administrative scope, such as `federal`, `state`, `international` |
| `issuedAt` | datetime | | Signature, approval, adoption, or issuing time |
| `effectiveAt` | datetime | | Legal or administrative effective time |
| `issuingBodies` | organization[] | | Agencies, institutions, repositories, courts, or filing parties. Uses the shared `defs#organization` type; conventional roles: `publisher`, `issuer`, `author`, `adopter`, `filer`, `regulator`, `court`, `legislature`, `repository` |
| `identifiers` | identifier[] | | DOF IDs, UNFCCC symbols, file numbers, case numbers, ISBNs, DOIs, etc. (content hashes live in `retrieval.sha256`) |
| `domains` | string[] | | Broad public-interest domains such as `government`, `environment`, `education`, `climate` |
| `topics` | string[] | | Free-form source categories or tags |
| `updatedAt` | datetime | | Last material update time |

### `tech.transparencia.document.source`

Registry of publishers and repositories of official documents (e.g., DOF, UNFCCC, SCJN, INEGI). One record per source, referenced by `document.item` records via `strongRef` — analogous to how `news.article` references `news.source`.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✅ | Machine-readable slug (e.g., `dof`, `unfccc`, `scjn`) |
| `displayName` | string | ✅ | Human-readable name |
| `sourceType` | string | ✅ | Kind of publishing source: `official-gazette`, `legislative-portal`, `judicial-portal`, `regulatory-agency`, etc. |
| `createdAt` | datetime | ✅ | When registered |
| `baseUrl` | uri | | Public website or repository URL |
| `country` | string | | ISO 3166-1 alpha-2 code (omit for international sources) |
| `jurisdiction` | string | | Legal or administrative scope |
| `language` | language | | Primary language (BCP-47) |
| `publisherName` | string | | Official publishing body, if different from `displayName` |
| `logoUrl` | uri | | Logo or seal URL |
| `description` | string | | Mandate and document scope |
| `license` | string | | Default license/terms for documents from this source |
| `identifier` | string | | Persistent ID (Wikidata QID, ROR, LEI, etc.) |
| `identifierType` | string | | Identifier system: `wikidata`, `ror`, `lei`, `isni`, `internal`, `other` |
| `updatedAt` | datetime | | Last material update time |

### `tech.transparencia.document.mxdof.note`

DOF-specific sidecar for a Diario Oficial de la Federación nota. Mirrors the SIDOF data model — each nota gets one of these plus one `document.item` record (referenced via `strongRef`). The universal `document.item` carries the title, dates, retrieval metadata, and issuing bodies; this sidecar carries gazette-specific fields that wouldn't make sense outside the DOF.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `item` | strongRef | ✅ | Reference to the `document.item` record |
| `codNota` | integer | ✅ | Primary SIDOF identifier (UNIQUE) |
| `codDiario` | integer | ✅ | SIDOF identifier of the parent diario |
| `edition` | string | ✅ | `matutina` / `vespertina` / `extraordinaria` |
| `createdAt` | datetime | ✅ | When the AT Protocol record was created |
| `codSeccion` | string | | Section code (`PRIMERA`, `AVISOS JUDICIALES`, `LICITACIONES`, etc.) |
| `dependencia` | string | | Top-level authority label (verbatim from SIDOF) |
| `organismo` | string | | More specific issuing body (verbatim from SIDOF) |
| `issuingAuthority` | string | | Normalized human-readable issuer name |
| `authorityLevel` | string | | Normalized tier: `federal_executive`, `federal_judicial`, `autonomous_constitutional_body`, etc. |
| `tipoNota` | string | | Raw tipo_nota from SIDOF (often null) |
| `documentClass` | string | | Broad family: `normative`, `judicial`, `procurement`, `intergovernmental`, `planning`, `administrative`, `informational`, `other` |
| `page` / `pageUntil` | integer | | Printed page range within the diario PDF |
| `order` | string | | Ordering value as decimal string |
| `hasHtml` / `hasPdf` / `hasDoc` / `hasImage` | boolean | | Asset availability from SIDOF |
| `contentTextAvailable` | boolean | | Whether the pipeline successfully extracted plain text |
| `pdfStoragePath` | string | | Internal storage path for the diario PDF |
| `rawJson` | string | | Optional stringified SIDOF payload (capped at 50KB) |
| `rawImportedAt` | datetime | | When the SIDOF payload was first ingested |
| `updatedAt` | datetime | | Last material update time |

### `tech.transparencia.document.mxdof.note.enrichment`

AI-generated structured enrichment for a DOF nota. Designed to capture legal-administrative semantics that don't fit a news-style enrichment: type of act, document class, legal effects, obligations, compliance items, related references. Cross-domain fields (`contentDomain`, `eventType`, `region`) keep it joinable with `news.enrichment`.

One nota → many enrichments (different models, languages, analysts). All reference the same `document.mxdof.note` via `strongRef`.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `note` | strongRef | ✅ | Reference to the `document.mxdof.note` being enriched |
| `summary` | string | ✅ | 2–4 sentence plain-language explanation (what is published, who issues it, what it changes, who it affects) |
| `tipoActo` | string | ✅ | Specific act: `decreto`, `acuerdo`, `sentencia`, `edicto`, `licitacion`, `convenio`, `lineamientos`, `manual`, `programa`, etc. |
| `documentClass` | string | ✅ | Broad family (same set as the note record) |
| `sector` | string | ✅ | Affected sector: `salud`, `justicia`, `energia`, `transporte`, `hacienda`, `educacion`, `medio_ambiente`, etc. |
| `impactLevel` | integer | ✅ | 1–10 relevance score |
| `topics` | string[] | ✅ | Up to 15 topical tags |
| `people` | person[] | ✅ | `defs#person` — signatories, officials, litigants |
| `organizationEntities` | organization[] | ✅ | `defs#organization` — issuing/regulated/coordinating bodies |
| `locations` | location[] | ✅ | `defs#location` — territorial scope |
| `relatedReferences` | relatedReference[] | ✅ | Cited / modified / abrogated / referenced laws, decrees, expedientes |
| `structuredRefs` | structuredRef[] | ✅ | `defs#structuredRef` — typed references for graph navigation |
| `contentDomain` | string | ✅ | IPTC top-level (17 known values — same as `news.enrichment`) |
| `eventType` | string | ✅ | Cross-domain event category: `policy-legislation`, `regulatory-action`, `court-ruling-legal`, etc. |
| `modelUsed` | string | ✅ | LLM or analyst profile identifier |
| `language` | string | ✅ | BCP-47 language of this enrichment's text fields |
| `createdAt` | datetime | ✅ | When the enrichment was generated |
| `neutralHeadline` | string | | Plain-language rewrite of the official title |
| `impactReasoning` | string | | Why the impactLevel was assigned |
| `legalEffects` | string[] | | Normalized verbs: `reforma`, `adiciona`, `deroga`, `abroga`, `convoca`, `emplaza`, `notifica`, `establece_plazo`, etc. |
| `obligations` | string[] | | Free-text obligations / duties / deadlines extracted from the text |
| `complianceItems` | complianceItem[] | | Structured tasks: `{ kind, text, responsibleEntity, beneficiary, dueDate, dueDateText, amountText }` |
| `effectiveDate` | datetime | | Normalized vigencia when recoverable |
| `effectiveDateText` | string | | Original textual expression (e.g., "al día siguiente al de su publicación") |
| `targetEntities` | string[] | | Who is directly affected: proveedores, sujetos regulados, gobiernos estatales, etc. |
| `timeline` | timelineEntry[] | | Chronological events; DOF variant requires `startDateText` because many legal dates aren't normalizable |
| `region` | string | | `local` / `state` / `national` / `regional` / `global` |
| `geographicScope` | string | | Free-text spatial scope (e.g., "Estado de México", "República Mexicana") |
| `sourceAuthorityLevel` | string | | Authority tier (mirrored from the note record) |
| `readingLevel` | string | | `basic` / `intermediate` / `advanced` |
| `modelVersion` | string | | Model version/checkpoint |
| `inputTokens` / `outputTokens` | integer | | Token counts for provenance |
| `costUsd` | string | | Inference cost as decimal string |

### `tech.transparencia.news.article`

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
| `language` | string | | BCP-47 language code (e.g., `es`, `en`, `pt-BR`) |

### `tech.transparencia.news.enrichment`

AI-generated structured metadata for an article. Uses the **sidecar pattern** — references the article via `strongRef` so multiple enrichments can coexist (different models, different analysts, updated analyses).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `article` | strongRef | ✅ | Reference to the article being enriched |
| `summary` | string | ✅ | 2-3 sentence AI summary |
| `neutralHeadline` | string | | Bias-stripped rewrite of the headline |
| `politicalOrientation` | string | | `left` / `center-left` / `center` / `center-right` / `right` / `neutral` |
| `orientationConfidence` | string | | Confidence score (0-1 as string) |
| `orientationReasoning` | string | | Why this orientation was assigned |
| `emotionalTone` | string | | `alarmist` / `critical` / `neutral` / `optimistic` / `celebratory` / `melancholic` / `informative` / `satirical` / `inspirational` / `sensationalist` / `technical` / `investigative` |
| `impactLevel` | integer | ✅ | 1-10 relevance score (1=minor local, 10=global crisis) |
| `impactReasoning` | string | | Explanation of the impact score |
| `eventType` | string | ✅ | Cross-domain event category (39 known values) |
| `contentDomain` | string | ✅ | IPTC-aligned broad category (17 known values) |
| `region` | string | | Geographic scope: `local` / `state` / `national` / `regional` / `global` |
| `language` | string | | BCP-47 language of this enrichment's free-text fields |
| `readingLevel` | string | | `basic` / `intermediate` / `advanced` |
| `factCheckability` | integer | | 1-5 verifiability score |
| `clickbaitScore` | integer | | 0-10 headline accuracy (0=pure clickbait, 10=accurate headline) |
| `clickbaitReasoning` | string | | Explanation of the clickbait score |
| `claims` | claim[] | | Extracted factual claims |
| `topics` | string[] | | Main topics (max 10) |
| `people` | person[] | | People mentioned with sentiment |
| `organizationEntities` | organization[] | | Organizations with structured metadata (ticker, sector, sentiment) |
| `locations` | location[] | | Geographic references with country and coordinates |
| `economicIndicators` | economicIndicator[] | | Economic data points |
| `timeline` | timelineEntry[] | | Chronological events |
| `structuredRefs` | structuredRef[] | | Typed references (laws, stocks, products, teams, etc.) |
| `relatedKeywords` | string[] | | Keywords for clustering |
| `modelUsed` | string | ✅ | AI model identifier |
| `modelVersion` | string | | Model version/checkpoint |
| `createdAt` | datetime | ✅ | When enrichment was generated |

### `tech.transparencia.news.source`

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
- **Third parties can enrich** — a university lab could publish their own `tech.unam.news.enrichment` records referencing the same articles

### Why `knownValues` instead of `enum`?

AT Protocol enums are closed and can never be extended. `knownValues` allows new values to be added over time without breaking existing consumers. For example, new `eventType` categories can be added as coverage expands.

### Why two-level classification (`contentDomain` + `eventType`)?

These two fields answer different questions:

- **`contentDomain`** — *What world does this story live in?* (IPTC-aligned broad vertical)
- **`eventType`** — *What happened?* (cross-domain event type)

The combination is more expressive than either alone. For example:

| Story | `contentDomain` | `eventType` |
|-------|-----------------|-------------|
| Beauty product recall | `lifestyle-leisure` | `product-recall` |
| Drug cartel arrest | `crime-law-justice` | `crime-incident` |
| Tesla earnings beat | `economy-business-finance` | `earnings-financial-results` |
| Liga MX match | `sport` | `match-result` |
| Hurricane landfall | `disaster-accident` | `natural-disaster` |

### Why English tokens for `knownValues`?

`knownValues` are **machine tokens, not display strings**. Frontends are responsible for localizing them into the user's language. Using English kebab-case tokens:

- Enables consistent filtering across multilingual enrichments
- Avoids encoding language assumptions into the schema
- Follows the convention of other open standards (IPTC, schema.org)

Old Spanish values (e.g., `izquierda`, `alarmista`) remain valid — `knownValues` are open sets and existing records are not broken. See the [knownValues Migration Guide](#knownvalues-migration-guide) below.

### Why a clickbait score?

Headline accuracy is a key trust metric for news consumers. The `clickbaitScore` field (0-10) measures how well the headline represents the actual article content:

- `0` = pure clickbait (misleading or sensationalist title)
- `10` = headline perfectly represents the article

The 0-10 scale matches the `impactLevel` convention, making both fields easy to reason about together.

### Why IPTC Media Topics alignment?

[IPTC Media Topics](https://iptc.org/standards/media-topics/) is the industry standard taxonomy used by Reuters, AP, AFP, and most major wire services. Aligning `contentDomain` with their 17 top-level categories enables:

- **Interoperability** with existing news aggregation systems
- **Consistent filtering** across sources from different countries
- **Future-proofing** — IPTC is actively maintained and widely adopted

### Why no embeddings in the lexicon?

Vector embeddings are an index optimization, not semantic data. They belong in a sidecar database (e.g., pgvector) rather than in AT Protocol records. The Hyperindex GraphQL API can be extended with custom vector search endpoints.

## Multi-Language Support

The schema is designed to handle multilingual content at every level:

- **`article.language`** — declares the language of the article content (BCP-47, e.g., `es`, `en`, `pt-BR`)
- **`enrichment.language`** — declares the language of the enrichment's free-text fields (summary, reasoning, etc.)
- **Sidecar pattern** — multiple enrichments in different languages can coexist for the same article. An English enrichment and a Spanish enrichment can both reference the same article record.
- **`knownValues` are language-neutral** — machine tokens like `match-result` and `economy-business-finance` are independent of the article or enrichment language. Frontends localize them for display.

**Example:** A Spanish article from El Universal can have:
1. An English enrichment (`language: "en"`) for international indexing
2. A Spanish enrichment (`language: "es"`) for Spanish-language consumers

Both reference the same `article` record via `strongRef`.

## Backward Compatibility

The v2 schema is fully backward-compatible with v1 records:

- **Old Spanish `knownValues`** (e.g., `izquierda`, `alarmista`, `basico`) remain valid — `knownValues` are open sets, not enums
- **`politicalOrientation`** moved from required to optional — non-political articles no longer need a forced value

## knownValues Migration Guide

The following `knownValues` were renamed from Spanish to English in v2. Old values remain valid in existing records.

### `politicalOrientation`

| v1 (Spanish) | v2 (English) |
|--------------|--------------|
| `izquierda` | `left` |
| `centro-izquierda` | `center-left` |
| `centro` | `center` |
| `centro-derecha` | `center-right` |
| `derecha` | `right` |
| `neutral` | `neutral` *(unchanged)* |

### `emotionalTone`

| v1 (Spanish) | v2 (English) |
|--------------|--------------|
| `alarmista` | `alarmist` |
| `critico` | `critical` |
| `neutral` | `neutral` *(unchanged)* |
| `optimista` | `optimistic` |
| `celebratorio` | `celebratory` |
| `melancolico` | `melancholic` |
| `informativo` | `informative` |
| `satirico` | `satirical` |
| `inspiracional` | `inspirational` |
| `sensacionalista` | `sensationalist` |
| `tecnico` | `technical` |
| `investigativo` | `investigative` |

### `readingLevel`

| v1 (Spanish) | v2 (English) |
|--------------|--------------|
| `basico` | `basic` |
| `intermedio` | `intermediate` |
| `avanzado` | `advanced` |

### `eventType`

`eventType` values were already in English kebab-case in v1. New values were added in v2 to cover cross-domain events (sports, finance, science, etc.). See the full list in [`lexicons/tech/transparencia/news/enrichment.json`](./lexicons/tech/transparencia/news/enrichment.json).

## Examples

See the [`examples/`](./examples/) directory for sample records:

- [`source.json`](./examples/source.json) — El Universal source registration
- [`article.json`](./examples/article.json) — Mexican trade news article (Spanish, `language: "es"`)
- [`enrichment.json`](./examples/enrichment.json) — AI enrichment with full v2 fields (English enrichment of a Spanish article)
- [`article-finance.json`](./examples/article-finance.json) — Tesla Q1 2026 earnings (English, `language: "en"`)
- [`enrichment-finance.json`](./examples/enrichment-finance.json) — Finance enrichment with tickers and org sentiment
- [`article-sports.json`](./examples/article-sports.json) — Liga MX Clásico Nacional match (Spanish, `language: "es"`)
- [`enrichment-sports.json`](./examples/enrichment-sports.json) — Sports enrichment with `match-result` event type (Spanish enrichment demonstrating multi-language support)
- [`document-source-dof.json`](./examples/document-source-dof.json) — DOF registry record (publisher metadata)
- [`document-dof.json`](./examples/document-dof.json) — DOF decree document referencing the DOF source via `strongRef`, with full `retrieval` block and multiple `issuingBodies`
- [`document-mxdof-note.json`](./examples/document-mxdof-note.json) — DOF nota sidecar with SIDOF codes, edition, dependencia/organismo, and `strongRef` to the underlying `document.item`
- [`document-mxdof-note-enrichment.json`](./examples/document-mxdof-note-enrichment.json) — Rich enrichment of a real-looking DOF health-sector decree: tipoActo, legalEffects, obligations, complianceItems with deadlines, related references to the Ley General de Salud, geocoded location, and full LLM provenance

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
  techTransparenciaNewsEnrichment(
    first: 20
    where: {
      contentDomain: { eq: "economy-business-finance" }
      impactLevel: { gte: 7 }
      eventType: { eq: "policy-legislation" }
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
        organizationEntities
        structuredRefs
      }
    }
  }
}
```

> **Note:** The GraphQL field names above reflect the v2 schema. If you have existing queries using old Spanish `knownValues` (e.g., `izquierda`), they will continue to work — see [Backward Compatibility](#backward-compatibility).

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

GNU Affero General Public License v3.0 (AGPL-3.0)
