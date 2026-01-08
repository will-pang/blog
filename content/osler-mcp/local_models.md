---
title: Local Models
date: 2026-01-06
tags:
  - mcp
  - local
---

As proprietary models (e.g., GPT-5.2, Claude Opus 4.5) continue to improve and re-define what is [state-of-the-art](https://gorilla.cs.berkeley.edu/leaderboard.html), it's been equally exciting to see [local models](https://www.linkedin.com/posts/julienchaumond_2026-will-be-the-year-of-local-agents-activity-7411141868303896576-F-VI?utm_source=share&utm_medium=member_desktop&rcm=ACoAABoaxBEBSWALU6nGy5nzir-14PSH_-PQlvQ) improve drastically in capabilities and crush ["older"](https://github.com/idavidrein/gpqa) benchmarks. Admittedly, I haven't played around with open-sourced models since 🦙 Llama was a community favorite (but not [anymore](https://magazine.sebastianraschka.com/p/state-of-llms-2025)), so I was excited to once again hear my laptop fans kick into full gear and (mentally) feel better about the _costly_ capital expenditure of my Macbook Pro (the November 2023 M3 Pro with 18GB RAM, to be exact).

### Setting up the local model

I wanted to be able to set something up quickly for experimentation, so my main requirements were (1) something that I could setup with relatively little pain (2) something that would be compatible with Apple Silicon chips.
After some quick research, I decided to go with [Ollama](http://ollama.com) for its simplicity and docker-like interface, and picked two open models `qwen2.5:7b-ctx32k` and `gpt-oss-20b-ctx32k` to compare against `claude-sonnet-4-5` and `gpt-4-turbo`.

### The importance of applied benchmarks

While the data scientist in me would love to have set up some regression-based analysis, I think it's pretty hard to have an apples to apples comparison between models as each model is optimized towards different use cases (e.g., coding agents vs general Q&A) + comes in different shapes & sizes. I found this [Docker blog post](https://www.docker.com/blog/local-llm-tool-calling-a-practical-evaluation/) on evaluating local LLM tool calling to be pretty insightful, with the main takeaways being: (1) which model can answer the prompt correctly using the provided tools (2) with the least amount of tool invocation?

Here's a simple scoring rubric that I ended up going with:

| Criterion               | Description                                                                                                                                                                             |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pass or Fail Response   | Does the outputs match the ["golden"](https://github.com/will-pang/osler-mcp/tree/WP-support-ollama-models/benchmarks/evals/tuva_project_demo/golden_query) SQL query provided by Tuva? |
| Number of Tools Invoked | How many tools were invoked? In other words, did it take a lot of tokens just to answer a simple question?                                                                              |
| Partial Credit? (Y/N)   | If I was a nice grader who didn't have to mark 800 exams, would I award a partial score?                                                                                                |

Before a question was called, I appended this simple [prompt](https://github.com/will-pang/osler-mcp/blob/WP-support-ollama-models/benchmarks/prompts/tool_policy.md?plain=1) to instruct the LLM:

> Tool dependency order:
>
> 1. `get_database_schema` must be called before any other tool if schema is unknown
> 2. `get_table_info` must be called before `execute_query` when columns are uncertain
> 3. `execute_query` is only allowed after schema and table details are confirmed
>
> Workflow:
>
> - Identify missing information
> - Call the appropriate tool
> - Validate assumptions
> - Then proceed

I then fed the following questions, one at a time:

<details>

<summary>Question Bank</summary>

|                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| What is the prevalence of Tuva Chronic Conditions? Give me a breakout by condition                                                                                                                               |
| What is the count of ED visits by CCSR Category and Body System?                                                                                                                                                 |
| What is the average CMS-HCC Risk Score by Patient Location?                                                                                                                                                      |
| What is the overall readmission rate?                                                                                                                                                                            |
| What is the healthcare quality measure performance? Provide a breakdown by performance rate                                                                                                                      |
| What is the distribution of chronic conditions? I want a breakdown of how many patients have 0 chronic conditions, how many patients have 1 chronic condition, how many patients have 2 chronic conditions, etc. |
| What is the total medicaid spend broken down by member months?                                                                                                                                                   |
| For quality measures, can you give me a breakdown of exclusion reasons and the count by measure?                                                                                                                 |
| What is the 30-day readmission rate by MS-DRG? Give me the MS-DRG description as well                                                                                                                            |

[Github Source](https://github.com/will-pang/osler-mcp/blob/WP-support-ollama-models/benchmarks/evals/tuva_project_demo/qsheet.csv)

</details>

### 🏆 Results

<img src="images/local_models_1.png" alt="Synthetic Data" style="width:70%">

| Model             | Partial Credit Awarded (/Failed Responses) |
| ----------------- | ------------------------------------------ |
| claude-sonnet-4.5 | 1/2                                        |
| gpt-4-turbo       | 4/6                                        |
| gpt-oss-20b       | 0/5                                        |
| gwen2.5-7b        | 0/8                                        |

| Model             | Average Number of Tools Invoked                                 |
| ----------------- | --------------------------------------------------------------- |
| claude-sonnet-4.5 | 7.1                                                             |
| gpt-4-turbo       | 3.8                                                             |
| gpt-oss-20b       | 8.4 (4.8 removing the one outlier where **35** calls were made) |
| gwen2.5-7b        | 3.3                                                             |

### 🤓 Observations
