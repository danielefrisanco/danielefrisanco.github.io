---
layout: post
title: "Beyond Fixed-Length: Introducing Semantic Chunker for Ruby"
date: 2026-08-27 01:00:00 +0200
description: "Split every 500 characters and you cut sentences in half. semantic_chunker is the first Ruby gem for true embedding-based semantic splitting — grouping sentences by meaning rather than by character count."
tags: [Ruby, RAG, Embeddings, LLM, Open Source]
---

If you've built a RAG (Retrieval-Augmented Generation) application, you know the "Chunking Dilemma."

Split your text every 500 characters, and you'll inevitably cut a sentence in half. Split by paragraph, and you might get chunks that are too large for your LLM's context window. Most Ruby developers have been stuck with Baran or simple regex splitters — until now.

I'm excited to introduce [semantic_chunker](https://rubygems.org/gems/semantic_chunker), the first Ruby gem designed for true embedding-based semantic splitting.

## The problem: why "naive" chunking fails

Standard chunking is "blind." It doesn't care if the first half of your chunk is about "Ruby Installation" and the second half is about "Cooking Pasta." To a vector database, this creates "noisy" embeddings that degrade the quality of your AI's answers.

## The solution: semantic intelligence

`semantic_chunker` doesn't just look at characters; it looks at meaning. It works by:

- **Sentence segmentation:** using the powerful `pragmatic_segmenter` to find real sentence boundaries.
- **Vector analysis:** generating embeddings for each sentence (via Hugging Face or OpenAI).
- **Centroid comparison:** it groups sentences into a chunk as long as they stay semantically close to the "topic center."
- **Adaptive thresholding:** it automatically detects the "valleys" in meaning to decide where to cut, making it model-agnostic.

## How it compares

While Python developers have had LangChain's SemanticChunker or semchunk, the Ruby ecosystem has been underserved. `semantic_chunker` fills this gap, offering a clean, Ruby-idiomatic way to handle sophisticated data ingestion.

## Quick start

You can get started in seconds. After installing the gem, just configure your provider and go:

```ruby
require 'semantic_chunker'

# 1. Setup your provider (Hugging Face or OpenAI)
SemanticChunker.configure do |config|
  config.provider = SemanticChunker::Adapters::HuggingFaceAdapter.new(
    api_key: ENV["HUGGING_FACE_API_KEY"]
  )
end

# 2. Chunk your text with "Auto" mode
chunker = SemanticChunker::Chunker.new(threshold: :auto)
text = "Your long document text..."
chunks = chunker.chunks_for(text)
# => ["Coherent topic A...", "Coherent topic B..."]
```

## What's next?

We are currently at v0.6.3, and the roadmap is ambitious. Upcoming features include:

- **Local embedding caching:** to save on API costs and speed up processing.
- **Drift protection:** using anchor-sentence comparison to keep chunks strictly on-topic.
- **Metadata support:** for easier integration with vector databases like Pinecone or Chroma.

I am actively looking for feedback! If you have a specific use case or a feature request, please [open an issue on GitHub](https://github.com/danielefrisanco/semantic_chunker).
