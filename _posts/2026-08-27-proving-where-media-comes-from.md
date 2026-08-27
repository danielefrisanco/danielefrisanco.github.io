---
layout: post
title: "Proving where media comes from is becoming a legal requirement. Here's what's actually on the table."
date: 2026-08-27 11:00:00 +0200
description: "The EU AI Act's Article 50 and California's SB 942 took effect on the same day. A look at what is actually deployed for media provenance — generation-time watermarks, C2PA metadata, fingerprinting — and the gap the industry admits is still open."
tags: [Provenance, C2PA, Watermarking, EU AI Act, Media Integrity]
---

On August 2, 2026, two of the world's largest markets started requiring it on the same day. The EU AI Act's [Article 50](https://artificialintelligenceact.eu/transparency-rules-article-50/) and California's [AI Transparency Act (SB 942)](https://leginfo.legislature.ca.gov/faces/billNavClient.xhtml?bill_id=202320240SB942) both took effect, and both push in the same direction: content needs to carry a machine-readable signal of where it came from.

The stakes are concrete, not abstract. Archives lose the thread on where a scanned object came from after a few migrations between systems. Newsrooms field images with no traceable source, pulled from contributors or old drives, years before anyone asks a question about how they were made. Rights holders lose licensing leverage the moment a copy circulates without attribution. Generative AI adds to this list on both ends: it's one more way a piece of content can end up disconnected from its origin, and it's also a reason a generator itself might want its output provably attributable to it. But it's an addition to the list, not the whole list. None of it is new, and none of it needed AI to become a problem. It's just been easier to ignore at a small scale, and regulators moved once the scale stopped being small.

It's worth being precise about what Article 50 actually asks for. It's a disclosure rule, not a general provenance mandate. If a system generates synthetic audio, image, video, or text, the provider has to mark that output in a machine-readable, detectable way before it reaches the user. Deepfakes and AI-generated content on matters of public interest get an explicit labeling requirement on top. The obligations split between providers (who build the system) and deployers (who use it), and non-compliance runs up to €15M or 3% of global turnover. The EU's Code of Practice, finalized in June, doesn't trust any single technique to do this. It prescribes a layered stack: metadata, watermarking, and fingerprinting as fallback.

California's law works alongside it: providers with over a million monthly users must offer a free public detection tool so people can check whether image, video, or audio content was created or altered by their system. Its sharper edge lands in 2027, when large platforms will be required to stop stripping standards-compliant provenance data on upload.

Most of the current conversation compresses provenance into AI detection, and the two aren't the same question. Detection asks whether content was manipulated. Provenance asks whether a checkable link to the origin survives distribution, whether that origin is a camera, a scanner, a contributor's old hard drive, or a model.

## What's actually being deployed

**Generation-time watermarks.** Google DeepMind's SynthID embeds a signal directly into pixels, audio, or token probabilities at the moment content is created. It's now used well beyond Google: OpenAI, NVIDIA, and ElevenLabs have adopted it since May 2026, and it's watermarked over 100 billion pieces of content. It survives re-encoding well since it lives in the content itself, but it only tells you the content passed through a generator that applies it. It doesn't carry edit history or attribution.

**Metadata credentials.** C2PA, the standard Adobe co-founded in 2021, attaches a cryptographically signed manifest to a file: who made it, what tool, what edits followed. It's rich, probably the richest record of the approaches here, and it's genuinely widely adopted, with Adobe, Microsoft, Google, Meta, and OpenAI all shipping it. Its known weakness, well documented by the people building it, is that the manifest is a sidecar to the file: many transformations along the way (re-encoding, format conversion, upload processing) can detach it from the content it describes.

**Hybrid approaches.** Combining a watermark with a fingerprint isn't new. Digimarc's 2011 patent on linking printed photos back to their source already proposed using a watermark to identify a creator's collection and a fingerprint to disambiguate within it, precisely because pure fingerprint matching against an open-world index is expensive and error-prone.

Trufo and Attestiv run current versions of the same pattern. Trufo pairs C2PA certificate issuance with watermarking; Attestiv pairs blockchain-anchored perceptual fingerprints with forensic tamper scoring. Neither is a solved problem so much as a different set of tradeoffs: Attestiv's own published testing found real content occasionally flagged as suspicious.

**Forensic watermarking for closed distribution.** NAGRAVISION's NexGuard, used across film and TV, shows the same imperceptible-signal idea applied to a narrower case: tracing a leak back to one of a known, bounded set of recipients before public release, rather than proving origin to an open audience after content is already public.

## The gap the industry admits is still open

None of this is a case of the newer approach quietly making the others obsolete. Each does its job well within the problem it was built for. What's notable is that the organizations building this infrastructure say openly that no single layer is sufficient. Microsoft's February 2026 Media Integrity report states it directly: no method, C2PA, watermarking, or fingerprinting, prevents digital deception on its own. The EU's Code of Practice reached the same conclusion independently, which is why it mandates a layered stack instead of picking a winner.

The recurring failure mode across metadata-based approaches is that the sidecar record and the content it describes travel separately, and separation is enough to break the link. A pure watermark survives that better, since the signal lives in the content itself, but on its own it carries little to no context about origin or edit history.

## Where puit.is fits

This is the layer [puit.is](https://puitis.com) is built for: not a replacement for disclosure-focused tools like SynthID, and not a competitor to C2PA's rich metadata model, but an attempt at the specific weak point both of those inherit, staying checkable after the file has been compressed, re-encoded, or moved off-platform.

It embeds an imperceptible identifier directly into the media signal, at or near the source. No metadata rides alongside the file; the signal is the file. A commitment derived from that identifier, never the media itself, gets anchored to a tamper-evident public record, so verification doesn't depend on trusting a single vendor's private database. When a copy resurfaces later, the identifier is extracted from the pixels and checked against that record, returning a verified, partial, or not verified category rather than a flat detection call.

The most concrete evidence so far is a real-world test: 985 out of 1,000 images sent through WhatsApp, one of the more aggressive compression pipelines a photo can go through, still recovered their identifier. It's early-stage work: puit.is has passed the first step of the EU Innovation Council's Accelerator evaluation and is not yet funded through it, and the company is upfront that this is a beta being tested with partners, not a finished product. Like every approach here, it has its own limits: it targets survivability through compression and redistribution, not every way a signal can be lost. It's also deliberately starting where the stakes are highest for getting this right long-term, archives, museums, and rights holders, before extending toward the broader AI-generated content question Article 50 is aimed at.

Does that close the gap the industry has flagged? Partly. It's a real answer to the redistribution problem specifically, tested against a real compression pipeline rather than a lab benchmark, but it's one layer, not the whole stack. The realistic shape of this problem is still a combination: something that survives generation, something that carries context, something that survives redistribution. puit.is is one attempt at that last piece.

*Disclosure: I work at puit.is.*