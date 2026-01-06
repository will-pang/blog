---
title: Local Models
date: 2026-01-04
tags:
  - mcp
  - local
---

As proprietary models (e.g., GPT-5.2, Claude Opus 4.5) continue to improve and re-define what is [state-of-the-art](https://gorilla.cs.berkeley.edu/leaderboard.html), it's been equally exciting to see [local models](https://www.linkedin.com/posts/julienchaumond_2026-will-be-the-year-of-local-agents-activity-7411141868303896576-F-VI?utm_source=share&utm_medium=member_desktop&rcm=ACoAABoaxBEBSWALU6nGy5nzir-14PSH_-PQlvQ) improve drastically in capabilities and crush ["older"](https://github.com/idavidrein/gpqa) benchmarks. Admittedly, I haven't played around with open-sourced models since 🦙 Llama was a community favorite (but not [anymore](https://magazine.sebastianraschka.com/p/state-of-llms-2025)), so I was excited to once again hear my laptop fans kick into full gear and (mentally) feel better about the _costly_ capital expenditure of my Macbook Pro (the November 2023 M3 Pro with 18GB RAM, to be exact).

### Setting up the local model

I wanted to be able to set something up quickly for experimentation, so my main requirements were (1) something that I could setup with relatively little pain (2) something that would be compatible with Apple Silicon chips.
After some quick research, I decided to go with [Ollama](http://ollama.com) for its simplicity and docker-like interface.

### All benchmarks are wrong, but some are useful?

While the data scientist in me would love to have set up some regression-based analysis, I think it's pretty hard to have an apples to apples comparison between models as each model is optimized towards different use cases (e.g., coding agents vs general Q&A) + comes in different shapes & sizes. That being said, I found this [Docker blog post](https://www.docker.com/blog/local-llm-tool-calling-a-practical-evaluation/) on evaluating local LLM tool calling to be pretty insightful, with the main takeaways being: (1) which model can answer the prompt correctly using the provided tools (2) with the least amount of tool invocation?

### So why open sourced?

I was having a conversation with a friend earlier this week who wanted to pick my brain on whether to use an open model, or stick with a properity model.
