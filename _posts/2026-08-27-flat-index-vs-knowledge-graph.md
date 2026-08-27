---
layout: post
title: "Flat index vs knowledge graph: understanding Europeana's two APIs"
date: 2026-08-27 05:00:00 +0200
description: "Europeana's search endpoint and record endpoint don't return the same schema with fields omitted — they return a flat index document versus a knowledge graph. What that difference means when you build on the API."
tags: [Europeana, API Design, Linked Open Data, Cultural Heritage]
---

A common expectation when working with REST APIs is that list endpoints and detail endpoints share roughly the same schema — the list gives you a subset of fields, the detail gives you the full object. A Spotify search returns a track with a handful of fields; the full track endpoint returns the same structure with more of them populated. You can often write code that handles both with the same types.

The Europeana API works differently. Its search endpoint and its record endpoint return different data models — not the same schema with fields omitted, but a flat index document versus a knowledge graph. This post explains the difference, why it exists, and what it means when building on the API.

## Background: what Europeana stores

Europeana is a cultural heritage aggregator — it ingests metadata from thousands of European museums, libraries, and archives and makes it searchable through a single API. The underlying data model is EDM, the Europeana Data Model, which is graph-based.

A single record in EDM is not a flat document. It is a small knowledge graph: a central cultural heritage object connected to entity nodes — agents (people, organisations), places, time spans, concepts — each fully described with multilingual labels, structured attributes, and `owl:sameAs` links to external vocabularies like Wikidata, Getty, and GeoNames.

A record for a 17th-century Dutch painting might contain:

- an `edm:Agent` node for the artist, with birth/death dates, profession, and a Wikidata URI
- an `edm:Place` node for where it was made, with GPS coordinates and labels in multiple languages
- multiple `skos:Concept` nodes for subject tags, each with full concept hierarchies
- an `edm:TimeSpan` node for the period, with structured begin/end dates
- web resource metadata: MIME type, pixel dimensions, colour palette
- quality annotations and rights statements

This is what makes EDM useful for linked open data applications — each record is a node in a broader semantic graph, not just a row in a catalogue.

## The two endpoints

Europeana exposes two main ways to retrieve record data:

- **The Search API** (`/record/v2/search.json`) — paginated search over the full collection. Returns an `items[]` array.
- **The Record API** (`/record/v2/{id}.json`) — fetch a single record by ID. Returns an object containing the full EDM graph.

Same underlying data, different response shapes.

## What the Search API returns

The search result is a flat, denormalized document. There are no nested EDM objects — every entity is collapsed into top-level key-value pairs:

```json
{
  "id": "/15508/D_II_11_46",
  "title": ["Madonna with Child"],
  "dcTitleLangAware": { "nl": ["Madonna met Kind"], "fr": ["Madone à l'Enfant"] },
  "edmAgentLabel": [{ "def": "Joos van Cleve" }],
  "edmAgentLabelLangAware": { "en": ["Joos van Cleve"] },
  "edmPlaceLatitude": ["51.0"],
  "edmPlaceLongitude": ["4.3"],
  "edmPlaceLabel": [{ "def": "Antwerp" }],
  "edmConceptPrefLabelLangAware": { "en": ["portrait"], "nl": ["portret"] },
  "edmTimespanLabel": [{ "def": "1500-1550" }],
  "edmIsShownAt": "https://...",
  "rights": ["http://creativecommons.org/publicdomain/zero/1.0/"],
  "type": "IMAGE",
  "europeanaCompleteness": 10
}
```

A `profile=rich` search result has around 45 flat keys, regardless of how many contextual entities the underlying record contains. An agent with biographical fields — birth date, death date, profession, place of activity, Wikidata link — is represented as a single `edmAgentLabel` string. A concept with hierarchies, exact matches, and labels across EU languages becomes `edmConceptPrefLabelLangAware`.

The minimal, standard, and rich profiles control how many of those ~45 flat keys are populated, but the shape is consistent across all profiles.

## What the Record API returns

The Record API returns `.object` — the full EDM graph, serialised as nested arrays of typed objects:

```json
{
  "object": {
    "about": "/15508/D_II_11_46",
    "type": "IMAGE",
    "proxies": [ ... ],
    "aggregations": [ ... ],
    "europeanaAggregation": { ... },
    "providedCHOs": [ ... ],
    "agents": [ ... ],
    "places": [ ... ],
    "timespans": [ ... ],
    "concepts": [ ... ],
    "organizations": [ ... ],
    "edmDatasetName": [ ... ],
    "europeanaCompleteness": 10,
    "timestamp_created": "...",
    "timestamp_update": "..."
  }
}
```

Each array contains full entity objects. `agents[]` carries every RDA-GR2 field. `concepts[]` carries the full SKOS structure with broader/narrower/exactMatch and prefLabels in every EU language. `places[]` carries coordinates, altitude, isPartOf hierarchies, and `owl:sameAs` links to GeoNames.

A leaf-path count on real records across different media types illustrates the scale of the difference:

| Record type | Full record leaf paths | Search result keys |
|-------------|-----------------------:|-------------------:|
| IMAGE | 218 | ~45 |
| TEXT  | 241 | ~45 |
| VIDEO | 131 | ~45 |

The gap in raw key count is 3–5×, and larger in semantic content: the 45 search keys are strings and arrays of strings, while the record's nested objects contain structured data with typed fields, language maps, and cross-references.

## Why the gap exists

The two endpoints reflect two different internal layers of the system.

The search API exposes the search index — a flat document store optimised for querying, faceting, and result-list rendering. Systems like Elasticsearch and Solr work this way internally: data is indexed in a denormalised projection for search, and the original is stored separately for retrieval. The search result format is designed for display, not for graph traversal.

The record API exposes the graph store — the original EDM data with all its linked entities intact.

Many APIs abstract over this distinction, presenting a single unified schema at both endpoints with the list version simply returning fewer fields. The Europeana API surfaces the split directly, which reflects the underlying architecture but requires some awareness from the developer side.

## Practical implications

Building on the search API is appropriate for result lists, search UIs, faceted browsing, and any use case that only needs display-essentials: title, thumbnail, provider, rights, type, basic place and agent labels. It is paginated and efficient for large result sets.

Building on the record API is necessary when you need the full graph: agent biographies, concept hierarchies, temporal structured data, `owl:sameAs` links to external vocabularies, web resource technical metadata, or quality annotations.

A common issue when starting with the Europeana API is using the search results as a canonical record representation and then finding that agent, concept, and temporal data is missing or reduced to a label. That data exists in the system — it is just not part of the search response format. The reverse is also possible: calling the record API in a loop to build result lists, at unnecessary per-record latency.

If you need both (a paginated list with full record detail), the standard approach is to use the search API with `profile=minimal` to harvest IDs efficiently via cursor pagination, then fetch full records individually via the record API.

## Broader context

The list-vs-detail distinction is common in API design — most APIs return less data in list endpoints than in detail endpoints. What distinguishes the Europeana case is that the difference is not just in field count but in data model: the search result is a flat projection, the record is a graph. This is a more significant structural difference than developers typically encounter in REST APIs, where list and detail endpoints usually share the same schema.

The pattern itself — maintaining a flat search index alongside a richer source representation — is standard in search infrastructure. What is less common is surfacing both directly as separate API endpoints without abstracting the distinction.

When working with any API that exposes both search and detail endpoints, it is worth checking early whether the two responses share a schema or represent the same data in different forms. The difference has direct implications for what you can build on each endpoint.

---

*Verified against the live Europeana API (v2) in May 2026. Record and search responses sampled across IMAGE, TEXT, VIDEO, and 3D record types. Leaf path counts computed with `jq`.*

**API documentation:**

- [Search API](https://europeana.atlassian.net/wiki/spaces/EF/pages/2385739812/Search+API+Documentation) — Europeana Knowledge Base
- [Record API](https://europeana.atlassian.net/wiki/spaces/EF/pages/2385674279/Record+API+Documentation) — Europeana Knowledge Base
- [Search API field list](https://github.com/europeana/labs-preview/blob/master/api/search.md) — europeana/labs-preview
- [EDM Object Templates](https://github.com/europeana/corelib/wiki/EDMObjectTemplatesEuropeana) — europeana/corelib Wiki
