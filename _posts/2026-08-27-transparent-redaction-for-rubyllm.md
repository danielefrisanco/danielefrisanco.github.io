---
layout: post
title: "Stop Leaking Secrets to Your LLM: Transparent Redaction for RubyLLM"
date: 2026-08-27 10:00:00 +0200
description: "Everything you send to a model is free text, and free text leaks. data_redactor now ships an opt-in RubyLLM integration that scrubs PII and secrets from every outbound request — prompts, tool definitions and tool results — with one line at boot."
tags: [Ruby, RubyLLM, Security, Privacy, LLM]
---

If you write Ruby and you've touched an LLM in the last year, you've probably run into [RubyLLM](https://rubyllm.com). It's having a moment — a single, clean interface across Anthropic, OpenAI, Gemini, Bedrock and friends, with first-class tools, streaming, and Rails integration. It's quickly becoming the default way Rubyists talk to models.

But there's a quiet problem that every LLM integration shares, and RubyLLM is no exception: everything you send to a model is free text, and free text leaks.

## The problem nobody puts on the slide

Think about what actually ends up in a prompt:

- A user pastes a support message — and it contains their credit card and email.
- Your system prompt has an internal escalation address or an API key baked in.
- An agent reads a config file or runs a shell command, and feeds the output back to the model on the next turn — now your DB password is in the conversation history.

All of that travels to a third-party provider. It gets logged, it may get retained, and in a tool-using agent loop it gets re-sent again and again. You didn't decide to send your secrets to OpenAI; your code did it for you.

The usual answer is "be careful." That doesn't scale. You want the careful part to be automatic.

## Enter data_redactor

[data_redactor](https://github.com/danielefrisanco/data_redactor) is a Ruby gem with a C extension that scans text for ~89 patterns of PII and secrets — API keys, tokens, credentials, IBANs, national IDs from 15+ countries, emails, phone numbers — and replaces them with `[REDACTED]`. It's fast (that's the whole point of the C core), it has zero runtime dependencies, and it never mutates your input.

You've always been able to use it by hand:

```ruby
require "ruby_llm"
require "data_redactor"

chat = RubyLLM.chat(model: "claude-opus-4-8")
chat.ask(DataRedactor.redact("My card is 4111111111111111 and email is alice@example.com"))
# the model receives: "My card is [REDACTED] and email is [REDACTED]"
```

That works, and for many apps it's exactly enough. But it's a step you have to remember on every call — and it doesn't help with the system prompt, tool definitions, or that file content an agent dragged in three turns ago.

## The new bit: transparent mode

As of the latest release, data_redactor ships an opt-in RubyLLM integration that redacts every outbound request automatically — no per-call wrapping:

```ruby
require "ruby_llm"
require "data_redactor/integrations/ruby_llm"

DataRedactor::Integrations::RubyLLM.install!   # once, at boot

chat = RubyLLM.chat(model: "claude-opus-4-8")
chat.ask("my card is 4111111111111111")        # sent as "my card is [REDACTED]"
```

One line at boot, and from then on the user prompt, the system prompt, your tool definitions, and any file or command output an agent fed back as a tool result are all scrubbed before the request leaves your process. It works the same across every provider RubyLLM supports, because it hooks the single point where RubyLLM has assembled the final request — so you don't have to think about Anthropic's payload shape versus OpenAI's.

You can scope it like the rest of the gem:

```ruby
DataRedactor::Integrations::RubyLLM.install!(only: [:financial, :contact])
```

## Honesty about how it works (and why)

Here's the part I want to be upfront about: right now, this is a monkeypatch.

That's not a design preference — it's the only option available today. RubyLLM doesn't yet expose a public hook to rewrite the outbound request body: its middleware setup is private, its instrumentation hook is read-only, and its tool-result callbacks can't change the value they observe. So to redact transparently, the integration carefully patches RubyLLM's internal request-rendering step.

It's done defensively: it's opt-in (nothing is patched unless you call `install!`), it's version-pinned, and it fails loudly at install time if it's loaded against a RubyLLM version it hasn't been verified against or if the internal method it relies on has moved — so it can never silently stop working and leak your data.

There's an open RubyLLM feature request ([#765](https://github.com/crmne/ruby_llm/issues/765)) for a proper middleware hook. The moment that lands, this integration can drop the patch and plug in cleanly. Until then, the monkeypatch is the pragmatic bridge.

> **Update:** [#765](https://github.com/crmne/ruby_llm/issues/765) — *“Expose `config.faraday_middleware` so observability gems can stop monkey-patching”* — was closed as completed on 12 August 2026. The hook this integration was waiting for now exists upstream, so the monkeypatch described below is on its way out.

And if monkeypatching internals isn't your thing? You don't have to. Skip `install!` entirely and just call `DataRedactor.redact` (or the `#redact` refinement) on the strings you care about before passing them to `ask`. Same engine, zero patching, fully your call.

## One limitation worth knowing

Redaction works on text. It does not reach inside base64-encoded attachments (PDFs, images, audio sent inline) or files referenced by URL — the sensitive bytes are encoded or remote, so the patterns can't see them. If your threat model includes secrets inside uploaded documents, you'll want to handle those before they're attached.

## Try it

```ruby
gem "data_redactor"

require "data_redactor/integrations/ruby_llm"
DataRedactor::Integrations::RubyLLM.install!
```

RubyLLM made talking to every model a one-liner. This makes not leaking your users' data to those models a one-liner too.

- **Gem & docs:** [github.com/danielefrisanco/data_redactor](https://github.com/danielefrisanco/data_redactor)
- **RubyLLM:** [rubyllm.com](https://rubyllm.com)
- **The middleware-hook discussion:** [ruby_llm#765](https://github.com/crmne/ruby_llm/issues/765)
