---
layout: post
title: "Europeana's quality tiers, explained: what contentTier and metadataTier actually mean"
date: 2026-08-27 04:00:00 +0200
description: "Two fields sit quietly in every Europeana record: contentTier and metadataTier. What they mean, how they're calculated, and what a given combination tells you about a record's quality — gathered into one place."
tags: [Europeana, Metadata, Linked Open Data, Cultural Heritage]
---

If you've ever browsed the Europeana API response for a record, you've probably noticed two fields sitting quietly in the `europeanaAggregation` block:

```json
"contentTier": "4",
"metadataTier": "B"
```

The Europeana documentation covers these across several scattered Confluence pages. This post brings it together in one place: what the tiers mean, how they're calculated, and what a given combination tells you about a record's quality.

## Two independent scales

Content tier and metadata tier measure completely different things and are calculated independently.

**Content tier** is about the digital object: how good is the media file, and under what terms can it be reused? Scale: 0, 1, 2, 3, 4. Higher is better.

**Metadata tier** is about the descriptive record: how rich, structured, and multilingual is the metadata? Scale: 0, A, B, C. Later letter is better; C is the top.

A record can have any combination — contentTier 4 / metadataTier A (great media, thin metadata) or contentTier 1 / metadataTier C (low-res thumbnail with an exceptionally rich description) are both valid.

## Content tiers: resolution × rights

The content tier algorithm has two inputs: media resolution and rights statement. Importantly, rights only enter the calculation once the media clears the resolution threshold for tier 3. Below that, the rights statement is ignored.

| Tier | Meaning | Image resolution | Rights |
|:----:|---------|------------------|--------|
| 0 | Does not meet publishing criteria | — | — |
| 1 | Minimum viable — thumbnail only | ≥ 0.1 MP | any |
| 2 | Usable in thematic collections | ≥ 0.42 MP (~800×533 px) | any |
| 3 | Reusable by individuals, educators, researchers | ≥ 0.95 MP (~1200×800 px) | restricted-reuse rights |
| 4 | Free reuse platform — top tier | ≥ 0.95 MP | open / public domain rights |

The resolution thresholds shown are for images. Audio, video, 3D, and text types have their own equivalent criteria.

### What "rights" means here

Tier 3 requires one of seven reuse-permitting statements:

- **Creative Commons:** CC BY-NC, CC BY-ND, CC BY-NC-SA, CC BY-NC-ND
- **RightsStatements.org:** NoC-NC, NoC-OKLR, InC-EDU

Tier 4 requires the fully open ones: CC0, Public Domain Mark, CC BY, CC BY-SA.

The logic is intentional: Europeana's tier 4 is explicitly positioned as a free reuse platform. A high-resolution image under an all-rights-reserved statement can't sit at tier 4 regardless of quality, because reuse isn't permitted.

### Reading a content tier

- **Tier 4:** high-resolution media, open rights. The institution is sharing freely.
- **Tier 3:** high-resolution, but with some reuse restrictions. Still very usable.
- **Tier 2:** decent resolution, good for browsing and display, not necessarily for download/reuse.
- **Tier 1:** thumbnail quality. Useful as a placeholder, not much more.
- **Tier 0:** doesn't meet the bar. May be a broken link, or metadata-only with no media.

## Metadata tiers: language × elements × context

The metadata tier is more nuanced. It's scored on three criteria, and the overall tier is the **minimum** across all three — a record only reaches tier C if all three criteria independently clear tier C. One weak criterion drags the whole score down.

### The three criteria

**1. Language tagging.** How much of the metadata carries `xml:lang` attributes? Language-tagged values enable Europeana to display metadata in the user's preferred language and improve multilingual search. Tier C requires ≥ 75% of relevant elements to carry at least one language tag.

**2. Enabling elements.** Beyond the mandatory EDM fields, there's a defined list of enabling elements — optional fields that unlock specific discovery and display scenarios (faceted filtering, timeline browsing, map view, etc.). Tier C requires at least 3 distinct enabling elements covering at least 2 distinct user scenarios.

**3. Contextual classes.** This is the big one. EDM supports four contextual entity types that turn a flat record into a knowledge graph node:

- `edm:Agent` — people and organisations (artists, publishers, workshops)
- `edm:Place` — geographic locations with coordinates
- `edm:TimeSpan` — time periods with structured begin/end dates
- `skos:Concept` — subject classifications

These show up in the record JSON as the `agents[]`, `places[]`, `timespans[]`, and `concepts[]` arrays. A contextual class counts toward the tier when it's fully populated with minimum required fields or when it carries an `owl:sameAs` link to a supported LOD vocabulary (Wikidata, Getty, GeoNames, etc.).

Tier B needs ≥ 1 such contextual class. Tier C needs ≥ 2.

### The tier table

| Tier | Language tags | Enabling elements | Contextual classes |
|:----:|---------------|-------------------|--------------------|
| 0 | below A | — | — |
| A | baseline | minimal | none required |
| B | higher coverage | ≥ 1 discovery scenario | ≥ 1 class populated or LOD-linked |
| C | ≥ 75% of relevant fields | ≥ 3 elements, ≥ 2 scenarios | ≥ 2 classes populated or LOD-linked |

## Reading a real record: contentTier 4 / metadataTier B

This is a common combination for well-digitised collection items from larger institutions.

**contentTier 4** tells you: this institution is sharing a high-resolution image under an open licence. You can download and reuse it.

**metadataTier B** tells you: the record is solid — it has language-tagged metadata and at least one contextual class (most likely a `skos:Concept` for subject classification or an `edm:Place` for location). But it's missing something to reach C: either the language coverage doesn't hit 75%, or there's only one contextual entity rather than two, or the enabling elements don't span two distinct discovery scenarios.

### What B → C would take

Looking at the structure of a metadataTier B record, the most common gaps are:

- The `agents[]` block is absent (no linked author/creator entity). Adding a well-populated agent with `owl:sameAs` pointing to a Wikidata URI would add a second contextual class.
- Language tags are present on titles and descriptions but missing from subject terms or type labels.
- Only one of the three contextual class types is populated.

The path to C runs through linked open data: connecting the record's entities to external vocabularies does double duty — it satisfies the contextual class requirement and tends to pull in better language coverage via the linked vocabulary labels.

## Why the tiers matter

The tiers aren't just labels — Europeana uses them for:

- **Filtering and ranking** in search results. Higher-tier records are more discoverable.
- **Feature eligibility:** tier 4 content can appear in Europeana Stories, exhibits, and reuse campaigns in ways that lower-tier content cannot.
- **Feedback to institutions:** the tier system is designed so data providers can see exactly which axis is limiting their score and what to fix.

If you're building on the Europeana API — ingesting records, analysing collections, or building derivative applications — the tiers are a ready-made quality signal you don't have to compute yourself.

---

**Sources**

- [Content & Metadata Tiers](https://europeana.atlassian.net/wiki/spaces/EF/pages/2059829253/Content+Metadata+Tiers) — Europeana Knowledge Base
- [Tier A–C requirements](https://europeana.atlassian.net/wiki/spaces/EF/pages/1969979393) — Europeana Knowledge Base
- [Metadata Tier C](https://europeana.atlassian.net/wiki/spaces/EF/pages/1967849520/Metadata+Tier+C) — Europeana Knowledge Base
- [Metadata Tier B](https://europeana.atlassian.net/wiki/spaces/EF/pages/1970044936) — Europeana Knowledge Base
- [ContentTier 2–4: Image type](https://europeana.atlassian.net/wiki/spaces/EF/pages/2060386364/ContentTier+2-4+Image+type) — Europeana Knowledge Base
- [Introducing the quality standard for cultural heritage metadata](https://pro.europeana.eu/post/introducing-the-quality-standard-for-cultural-heritage-metadata) — Europeana PRO
