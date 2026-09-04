---
layout: post
title: "Proving where media comes from is now a legal requirement. The hard part is who keeps the record."
date: 2026-08-27 11:00:00 +0200
description: "The EU AI Act's Article 50 and California's SB 942 took effect on the same day. What is actually deployed for media provenance — generation-time watermarks, C2PA credentials, fingerprinting — why the industry says no single layer is enough, and where puit.is fits."
tags: [Provenance, C2PA, Watermarking, EU AI Act, Media Integrity]
---

*Disclosure: I work at [puit.is](https://puitis.com), which builds one of the approaches discussed below. The views here are my own.*

On August 2, 2026, two of the world's largest markets started requiring it on the same day. The EU AI Act's [Article 50](https://artificialintelligenceact.eu/transparency-rules-article-50/) and California's [AI Transparency Act (SB 942)](https://leginfo.legislature.ca.gov/faces/billNavClient.xhtml?bill_id=202320240SB942) both took effect, and both push in the same direction: content needs to carry a machine-readable signal of where it came from.

The stakes are concrete. Archives lose the thread on where a scanned object came from after a few migrations between systems. Newsrooms field images with no traceable source, pulled from contributors or old drives, years before anyone asks a question about how they were made. Rights holders lose licensing leverage the moment a copy circulates without attribution. Generative AI adds to this list on both ends: it's one more way a piece of content can end up disconnected from its origin, and it's also a reason a generator itself might want its output provably attributable to it. But it's one more entry on a list that was already long. None of it is new, and none of it needed AI to become a problem. It's just been easier to ignore at a small scale, and regulators moved once the scale stopped being small.

It's worth being precise about what Article 50 actually asks for. It's a disclosure rule covering synthetic output. If a system generates synthetic audio, image, video, or text, the provider has to mark that output in a machine-readable, detectable way before it reaches the user. Deepfakes and AI-generated content on matters of public interest get an explicit labeling requirement on top. The obligations split between providers (who build the system) and deployers (who use it), and non-compliance runs up to €15M or 3% of global turnover. The EU's Code of Practice, finalized in June, doesn't trust any single technique to do this. It prescribes a layered stack: metadata, watermarking, and fingerprinting as fallback.

The provider/deployer split moves around more than it first appears. Under the Act, a deployer who modifies or rebrands a system can take on provider obligations. Media pipelines are built out of that kind of handoff: a re-encode, a re-crop, a re-caption, a republication under a new masthead. Each handoff can shift who carries the obligation, and reconstructing that afterwards requires a record that survived the handoff.

California's law works alongside it: providers with over a million monthly users must offer a free public detection tool so people can check whether image, video, or audio content was created or altered by their system. Its sharper edge lands in 2027, when large platforms will be required to stop stripping standards-compliant provenance data on upload.

Underneath both, provenance is becoming standardised infrastructure: JPEG Trust became [ISO/IEC 21617-1](https://www.iso.org/standard/86831.html) in January 2025, and ISO 22144 standardises [Content Credentials](https://contentcredentials.org/) on top of the [C2PA](https://c2pa.org/) 2.1 specification.

## Provenance is a lookup problem

Most of the current conversation compresses provenance into AI detection, and the two aren't the same question. Detection asks whether content was manipulated. Provenance asks whether a checkable link to the origin survives distribution, whether that origin is a camera, a scanner, a contributor's old hard drive, or a model.

It helps to think of it as naming. Whatever a file carries — a signed manifest, a watermark, a fingerprint — works as a key. The key resolves to a record held somewhere else, and the record is what actually answers the question. So each approach below can be read through two questions: what does the key name, and where does the key live once the file starts moving?

There are two ways to give a file a name. You can derive one from the content itself, which is what [ISCC](https://www.iso.org/standard/77899.html) (ISO 24138) does: anyone can compute the same code from the same bytes, with no registry and no assignment step, and similar content produces similar codes. Or you can place a name into the content deliberately, which is what watermarking does. Derived names work on files nobody prepared in advance. Embedded names keep working when the file has been separated from any index. Both feed the same kind of lookup.

## What's actually being deployed

**Generation-time watermarks.** [Google DeepMind](https://en.wikipedia.org/wiki/Google_DeepMind)'s [SynthID](https://deepmind.google/technologies/synthid/) embeds a signal directly into pixels, audio, or token probabilities at the moment content is created. It's now used well beyond Google: OpenAI, NVIDIA, and ElevenLabs have adopted it since May 2026, and it's watermarked over 100 billion pieces of content. It survives re-encoding well since it lives in the content itself. What it reports is narrow: the content passed through a generator that applies it. Edit history and attribution sit outside its scope.

**Metadata credentials.** C2PA, the standard Adobe co-founded in 2021, attaches a cryptographically signed manifest recording who made an asset, with what tool, and what edits followed. It carries the richest record of the approaches here, and adoption is broad, with Adobe, Microsoft, Google, Meta, and OpenAI all shipping it.

Where that manifest actually sits matters, because "metadata" makes it sound flimsier than it is. In the normal case the manifest is embedded inside the file: an APP11 segment carrying a [JUMBF](https://www.iso.org/standard/73604.html) box in JPEG, a `caBX` chunk in PNG, a RIFF chunk in WebP, dedicated boxes in MP4 and MOV. It can also live remotely and be referenced by URI, or travel as a separate sidecar file. So the manifest generally does move with the asset.

What ties the manifest to the asset is a hard binding — a cryptographic hash over the file's bytes. That binding is what makes the record trustworthy, and it's also what makes it fragile under ordinary processing. Decoding an image to a bitmap and re-encoding it produces different bytes, so the hash no longer validates, and most encoders rebuild the container without carrying the segment across. Adaptive bitrate video makes this routine: [HLS](https://en.wikipedia.org/wiki/HTTP_Live_Streaming) and [DASH](https://en.wikipedia.org/wiki/Dynamic_Adaptive_Streaming_over_HTTP) re-encode a source into many renditions, each of which is a new file with no byte-level relationship to the original. The specification treats these as asset renditions and addresses them directly, which is why C2PA 2.1 introduced soft bindings — a fingerprint or an embedded watermark that identifies content by what it looks like rather than by its exact bytes, so a manifest can be found again after the bytes have changed.

**Hybrid approaches.** Combining the two naming methods isn't new. [Digimarc](https://en.wikipedia.org/wiki/Digimarc)'s 2011 patent on linking printed photos back to their source used a watermark to identify a creator's collection and a fingerprint to disambiguate within it — an embedded name to narrow the search, a derived name to finish it, because matching fingerprints against an open-world index is expensive and error-prone.

Trufo and Attestiv run current versions of that pattern, pairing C2PA certificate issuance with watermarking and blockchain-anchored fingerprints with tamper scoring respectively. Both carry tradeoffs: Attestiv's own published testing found real content occasionally flagged as suspicious.

**Forensic watermarking for closed distribution.** [NAGRAVISION](https://en.wikipedia.org/wiki/Nagravision)'s NexGuard, used across film and TV, applies the same imperceptible-signal idea to a narrower case: tracing a leak back to one of a known, bounded set of recipients before public release. It is the one deployment here built against a real adversary, and it works by knowing every recipient in advance.

## Both ends of the chain are built

Most of this infrastructure already exists in production.

At capture, signing runs on shipping hardware. [Leica's M11-P](https://leica-camera.com/en-US/photography/content-credentials) was first in late 2023, followed by Nikon's Z9 and Z8 through firmware in 2024, and Sony's Alpha bodies. On phones, the [Pixel 10](https://blog.google/security/pixel-android-trusted-images-c2pa-content-credentials/) signs every JPEG the camera captures, with claim signing keys generated and held in the Titan M2 security chip through Android StrongBox, reaching C2PA Assurance Level 2. Samsung's Galaxy S25 attaches credentials to AI-edited images.

At the other end, display works too. LinkedIn shows a Content Credentials icon that opens a provenance summary. Google's "About this image" reads C2PA data in Search. TikTok and YouTube surface credentials on uploads. Newsrooms sign in daily production: the BBC, Reuters, AFP, AP, the New York Times, the Wall Street Journal, NHK, ARD/ZDF, and [France Télévisions](https://www.francetelevisions.fr/groupe/notre-actualite/france-televisions-adopte-le-protocole-c2pa-47174), which certifies France 2's 1pm and 8pm news editions and publishes them on franceinfo's transparency page.

Between those two ends sits everything else: uploads, transcodes, format conversions, messaging apps, CMS exports, screenshots. Each of those steps rewrites the file, and each rewrite is a point where a byte-bound record has to be deliberately carried across — and, under the AI Act, a point where responsibility may have moved. Some pipelines do carry it. Many don't, which is why California is legislating a preservation duty for large platforms starting in 2027.

## The gap the industry admits is still open

None of this is a case of the newer approach quietly making the others obsolete. Each does its job well within the problem it was built for. What's notable is that the organizations building this infrastructure say openly that no single layer is sufficient. Microsoft's February 2026 Media Integrity report states it directly: no method, C2PA, watermarking, or fingerprinting, prevents digital deception on its own. The EU's Code of Practice reached the same conclusion independently, which is why it mandates a layered stack instead of picking a winner.

There's a further failure that only appears once two layers run together. A [2026 paper on desynchronized provenance and watermarking](https://arxiv.org/pdf/2603.02378) shows that when a manifest and an embedded watermark can be modified independently, you can construct a file where both signals verify and disagree about what happened to it. Two authenticated, contradictory accounts of the same image. Layers that can drift apart need something above them that establishes which came first.

Invisible watermarks can be removed. Diffusion-based regeneration attacks re-synthesize an image without the watermark pattern, and the [winning entry in the NeurIPS 2024 removal challenge](https://arxiv.org/pdf/2508.21072) reported near-total removal at negligible quality cost.

Those attacks describe a different problem from the one most institutions have. An archive migrating a collection between systems has no adversary. Neither does a newsroom exporting a photo through a CMS, or a licensed image forwarded through three messaging apps. Most provenance loss is incidental: compression, format conversion, upload processing, a screenshot. Handling incidental loss well is the job in front of the archives and rights holders working on this today. Adversarial removal is a separate research problem, and mixing the two makes claims harder to evaluate in both directions. It is telling that the one approach above built for a real adversary, NexGuard, buys its strength by knowing every recipient in advance — a condition open distribution never provides.

## The layer that gets the least attention

There's a shift happening in how regulated industries handle auditability, and it transfers cleanly to this problem. The older approach produced documents: a binder, a policy, an attestation that the process was followed. The current expectation is that a system emits its own evidence while it runs — lineage, versions, run records — so that verification reads the system's own output instead of a description written alongside it. Auditability becomes an architectural property.

Media provenance is working through the same shift. A signed manifest is a strong record with an external dependency: it's bound to a specific arrangement of bytes, and every processing step has to preserve that arrangement or carry the record forward. An identifier embedded in the signal itself has a different dependency: it needs the picture to still look like the picture. Both are legitimate designs, and the C2PA specification accommodates both.

Underneath that sits a different question. Nearly all the public argument concerns which signal is most robust. But a provenance system has five parts: the signal in the file, the record it resolves to, the service that answers the lookup, the governance of that service, and the surface that displays the result. Robustness of the signal is one part of five, and it's the part that can be replaced most easily. Changing which party holds the record is a much larger operation, and that decision is usually made once.

So the durable questions are: who runs the record, under which jurisdiction, and for how long. Standards bodies are one of the few places that question gets settled rather than argued, which is what the ISO work above is quietly doing. Archives work on timescales of a century. Software companies rarely last three decades. A verification path that requires one company's live database inherits that company's lifespan, and an archivist evaluating a provenance system will ask about that before asking how robust the watermark is.

## Where puit.is fits

[puit.is](https://puitis.com) works on one part of this: keeping an origin link checkable after a file has been compressed, re-encoded, or moved between platforms.

It embeds an imperceptible identifier into the media signal, at or near the source. A commitment derived from that identifier is anchored to a tamper-evident public record. The media file itself is never published. When a copy resurfaces later, the identifier is read back out of the pixels and checked against the record.

Two design choices are worth pulling out.

The first is what gets published. Only a commitment derived from the identifier reaches the public anchor. The media and its descriptive metadata stay offline. Data minimisation under [GDPR Article 5(1)(c)](https://gdpr-info.eu/art-5-gdpr/) is the relevant obligation there, and whether a given deployment meets it depends on the operator and the processing context.

The second is what a check returns: a recovered identifier, a confidence level, and a measured distance from the stored record. Someone in an archive, a newsroom, or a court has to be accountable for the resulting decision, and knowing what it rests on improves that accountability. Returning the components lets a person inspect them and sign off on the reasoning.

The published benchmark covers 12,331 unique images through 160,303 controlled transformations, with 99.55% perfect identifier recovery (12,276 of 12,331) and 61,655 negative controls. One real-world path has been measured so far: 985 of 1,000 baseline-perfect items recovered after a documented manual [WhatsApp](https://en.wikipedia.org/wiki/WhatsApp) transfer, one of the more aggressive compression pipelines a photo encounters in ordinary use.

The limits are published alongside the results. Recovery confidence degrades under severe or adversarial transformation. Aggressive geometric edits, AI-based removal attacks, and heavy compositing remain active work. The figures apply to the tested configuration and to the single redistribution path measured so far, and none of it is independent external validation.

The public anchor currently uses [Bitcoin](https://en.wikipedia.org/wiki/Bitcoin) and [OpenTimestamps](https://opentimestamps.org/), which provides tamper-evident ordering without qualifying as a trust service under [eIDAS 2](https://en.wikipedia.org/wiki/EIDAS); moving to qualified components is a stated direction with work still to do.

The governance question from the previous section applies here too. puit.is is at [TRL6](https://en.wikipedia.org/wiki/Technology_readiness_level) and runs as a single operator today. Answering that question properly means verification that survives the loss of any one operator, even this one. Independently operated nodes under open governance are what the roadmap places at TRL7. The framing the project works from is archival — [UNESCO's Charter on the Preservation of the Digital Heritage](https://www.unesco.org/en/legal-affairs/charter-preservation-digital-heritage), [NIST IR 8387](https://nvlpubs.nist.gov/nistpubs/ir/2022/NIST.IR.8387.pdf) on digital evidence continuity — matching where it starts: archives, museums, and rights holders, ahead of the broader AI-generated content question Article 50 addresses. It passed step one of the [EU Innovation Council Accelerator](https://eic.ec.europa.eu/eic-funding-opportunities/eic-accelerator_en) evaluation with a score of 3.00 out of 4.00 and was invited to step two in July 2026, with evaluators recording all aspects of TRL5 as convincingly completed and the evidence as supporting the TRL6 claim. The evaluation is ongoing and it is not EIC-funded.

Two structural choices sit behind those numbers. Checks produce artifacts meant to be re-run by someone else — runbooks, run records, per-item results and integrity hashes — so a result can be inspected years later without taking the operator's word for it. And the stack is modular: the watermarking, anchoring and verification layers can each be replaced without rebuilding the trust model around them. That matters given the attack literature above. A watermarking scheme that gets broken is a component to swap, provided the system was not designed around that one component holding.

## Why this is worth getting right

The reason to care about provenance has less to do with fakes improving than with where the burden of proof now sits.

When synthesis was expensive, real content was assumed real, and a fake had to be exposed. Now that synthesis is cheap, anything inconvenient can be waved away as possibly generated. Researchers call this the [liar's dividend](https://www.brennancenter.org/our-work/research-reports/deepfakes-elections-and-shrinking-liars-dividend), and claiming it costs nothing. So the work provenance does is changing: it increasingly serves to keep real material from being dismissed. Courts are already dealing with this. The US Advisory Committee on Evidence Rules has taken up a proposed [Rule 901(c)](https://en.wikipedia.org/wiki/Federal_Rules_of_Evidence) covering potentially fabricated electronic evidence, and the decided cases involving deepfake allegations have come out inconsistently.

That's what makes a durable identifier an archival and evidentiary artifact, and it's why the interesting work sits in the unglamorous layers. Spend time around the institutions that will have to run this — archives, broadcasters, standards bodies, regulators — and the constraints stop being technical. Bit error rates are tractable. The hard parts are coordination, interoperability, governance, and incentives: agreeing whose record counts, working out who funds it once the founding organisations have moved on, and explaining why a platform that profits from frictionless upload would preserve a signal that makes content traceable.

Those parts need agreement between organisations, so no single vendor can ship them. The working shape of this problem remains a combination: a mark applied at the source, a record of context, survival through redistribution, and an answer to who keeps the record. puit.is covers the mark and its survival today, and is working toward the others.
