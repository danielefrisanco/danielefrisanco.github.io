---
layout: post
title: "Introducing data_redactor — a fast Ruby gem for redacting sensitive data"
date: 2026-08-27 06:00:00 +0200
description: "API keys, IBANs, credit cards and national IDs leak into logs, indices and audit trails. data_redactor is a Ruby gem with a C extension that finds and replaces them — 88 built-in patterns, scanned in a single pass."
tags: [Ruby, Security, Privacy, C Extension, Open Source]
---

When you log a request, debug a production error, or store audit trails, sensitive data leaks in. API keys, IBANs, credit card numbers, social security numbers — they end up in log files, Elasticsearch indices, S3 buckets. [data_redactor](https://github.com/danielefrisanco/data_redactor) is a Ruby gem that finds and replaces them before they land anywhere they shouldn't.

## What it does

One call, 88 built-in patterns:

```ruby
DataRedactor.redact("auth: Bearer sk-ant-api03-XXXX, card: 4111111111111111")
# => "auth: Bearer [REDACTED], card: [REDACTED]"
```

It covers credentials (AWS keys, GitHub PATs, Anthropic/OpenAI tokens, Vault tokens), financial data (IBANs for 18 countries, credit cards), national IDs (SSN, PESEL, AHV, BSN, and more), emails, IPs, phone numbers, and passports — tagged so you can filter by category:

```ruby
DataRedactor.redact(text, only: [:credentials, :financial])
DataRedactor.redact(text, except: [:network])
```

You can also add your own patterns:

```ruby
DataRedactor.add_pattern(name: "employee_id", regex: "EMP-[0-9]{6}")
```

And redact nested structures directly:

```ruby
DataRedactor.redact_deep(params)   # Hash / Array, any depth
DataRedactor.redact_json(body)     # JSON string in, JSON string out
```

Three optional zero-dependency adapters — require only what you need:

```ruby
# Logging - scrub every line
logger.formatter = DataRedactor::Integrations::Logger.new

# Rails - redact params by value, not just key name
Rails.application.config.filter_parameters << DataRedactor::Integrations::Rails.filter

# Rack - scrub response body + sensitive headers
use DataRedactor::Integrations::Rack, scrub: [:body, :headers]
```

All of them take the same filters: `only:`, `except:`, `placeholder:`.

## Why it's fast

The gem ships a C extension. In version 0.10.0 it runs a Thompson NFA → lazy-DFA multi-pattern engine that scans the input once across all patterns simultaneously, with two selective-merge passes for the most common pattern classes (pure-digit patterns and IBANs). No external dependencies — pure C, zero extra libraries.

On a 1 MB log file: 2.1× faster than pure-Ruby `gsub` running the same 88 patterns. On a 168-byte log line: 1.7× faster. Previous versions were 3–5× slower than Ruby — the new engine is a ~9× throughput swing.

## What's coming

The current engine preserves today's per-pattern sequential semantics. The next step is a longest-match-wins resolver, which will handle edge cases where two secrets abut without a separator — and will allow marking a few remaining specs as the new default behaviour. After that: per-call thread re-entrancy (today the GVL serialises concurrent calls), a streaming API for large inputs, and more patterns (assignment-style secrets like `PASSWORD="..."`, more national IDs).

## Get started

```ruby
# Gemfile
gem "data_redactor"
```

- **GitHub:** [github.com/danielefrisanco/data_redactor](https://github.com/danielefrisanco/data_redactor)
- **RubyGems:** [rubygems.org/gems/data_redactor](https://rubygems.org/gems/data_redactor)
