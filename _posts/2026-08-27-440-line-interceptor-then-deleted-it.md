---
layout: post
title: "I Spent a Day Building a 440-Line Interceptor, Then Deleted It"
date: 2026-08-27 03:00:00 +0200
description: "Propagating a trace context from a FastAPI gateway into an asynchronous Temporal workflow turned into a rabbit hole. The fix was deleting the custom interceptor and using the one that already existed."
tags: [OpenTelemetry, Temporal, Observability, Distributed Tracing, Python]
---

Distributed tracing is one of those things that seems straightforward until you cross a process boundary. I recently needed end-to-end tracing across a stack comprising an API gateway (FastAPI), a processing microservice (FastAPI), and a Temporal worker. While HTTP tracing is trivial, moving a trace context from an API request into an asynchronous Temporal workflow — which might execute minutes later on an entirely different node — turned into a classic engineering rabbit hole.

![Architecture diagram: a traced HTTP request leaves the FastAPI API gateway for the Temporal server, where rpc_metadata strips the context (marked with a red cross) while workflow headers successfully propagate the OTEL traceparent onward to the processing microservice worker, which calls reconstruct_trace(); both the gateway and the worker export to a backend UI over OTLP.](/assets/images/posts/temporal-otel-tracing.png)

## The detour into complexity

My first instinct was to leverage our existing SkyWalking infrastructure. I spent the better part of a day building a custom SkyWalking interceptor (a massive 440-line file) designed to manually extract the `sw8` trace context and inject it into Temporal's `rpc_metadata`.

It felt logical: Temporal uses gRPC, and metadata is the standard way to pass headers in gRPC. I pushed the code, ran the tests, and watched the trace context vanish into thin air.

My trace context was being discarded by the very framework I was trying to instrument. I then pivoted to a hybrid "dual-header" approach, trying to pipe both SkyWalking and OpenTelemetry (OTEL) context through workflow headers. It was a fragile, messy architecture that left me with four different debug files committed to the repo — a vivid reminder of the time I'd wasted wrestling with the framework.

## The realization

Frustrated, I stepped back and looked at the ecosystem again. The breakthrough came when I realized two things:

- **Wrong transport.** Temporal `rpc_metadata` is for gRPC transport, not business logic. For cross-process context propagation, you must use workflow headers.
- **Native support.** I was treating SkyWalking as a proprietary silo when I should have been using it as an OTLP-compatible backend. SkyWalking OAP v9+ natively ingests OTEL traces. I didn't need the SkyWalking Python agent; I just needed to export via OTLP.

## The clean solution

I deleted the 440-line custom interceptor and replaced it with the official `temporalio.contrib.opentelemetry.TracingInterceptor`.

This library is designed exactly for this use case. It automatically handles the injection and extraction of `traceparent` headers through Temporal's workflow headers, following the OTEL standard. To handle the "noise" — the influx of internal ASGI calls and Dapr gRPC overhead that made my UI look like a mess — I implemented a `_FilteringExporter` wrapper. This acts as a final gatekeeper, dropping unwanted spans by name before they ever leave the process, ensuring my trace view remains clean and actionable.

## Lessons in tooling

The biggest takeaway here is the danger of "custom-built complexity." I had spent a day trying to reinvent standardized propagation logic that the Temporal and OpenTelemetry teams had already perfected.

When you find yourself writing hundreds of lines of code to force a framework to pass data, stop. You are likely fighting the architecture rather than using it. Modern observability is increasingly standardized around OpenTelemetry; if your stack supports OTLP, lean into it. Often, the best solution isn't the one you build — it's the one you find in the documentation that makes your custom code obsolete.
