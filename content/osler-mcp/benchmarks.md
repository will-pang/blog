---
title: On MCP Tools
date: 2026-12-28
tags:
  - mcp
  - benchmarks
---

In my last blog [post](mcp-tuva.md), I explored creating a MCP server to answer questions from a real-life(ish) healthcare database using Tuva Health's [demo setup](https://github.com/tuva-health/demo). As I was adding different MCP tools and testing it via integration of my MCP server in [Cursor](https://cursor.com/docs/context/mcp) to see if I got better responses to increasingly complex SQL queries, I noticed something peculiar: the MCP server was **not** using a tool that I thought would be critical in creating accurate SQL queries/relational joins. Perplexed by this behavior, I realized that I had to do something that I brushed aside while performing rapid prototyping: [benchmarks](https://www.ibm.com/think/topics/llm-benchmarks).

### 🧰 What was in my toolbox?

In my first iteration, I had three tools as I noted in my previous [post](mcp-tuva.md):

- `get_database_schema`: Returns all the schemas as well as the tables under a schema in a file tree format 🌳
- `get_table_info`: Returns the table structure (column names), as well as a sample of the table (e.g., first 5 rows of the table or last 5 rows ot the table)
- `execute_query`: Executes the SQL query

I felt that these tools, while quite powerful by itself (and was able to answer a lot of simpler questions as long as the table names were explicit), would not be sufficient as there was no tool for the LLM to understand the _relationships_ between tables. Since the Tuva Project leverages [dbt](https://thetuvaproject.com/blog/intro-dbt-for-healthcare), I thought I could also utilize dbt's [lineage](https://www.getdbt.com/blog/getting-started-with-data-lineage) as a tool that the LLM might use. After some tinkering 👷 and struggles passing command line outputs to my MCP server:

- `get_model_lineage`: Give a table name, and you can find the associated parent/children tables and its degree(s)/hop(s) of dependencies

### 📊 Building a mini benchmark

Knowing that benchmarks could be a separate [rabbit-hole](https://neurips.cc/virtual/2025/loc/san-diego/128660) by itself, I focused on tackling two questions: (1) Which MCP tools did the LLM agent choose to leverage (2) Did the tool choice differ by model?

For now, I'm only evaluating **OpenAI's GPT-4-turbo**, but I tried to setup a bare-bones framework that would adapt to any model (e.g., abstract methods for normalizing the MCP tools schema to match the speficic model's input format, standardizing the model response etc.).

To build the evaluation dataset with "golden truths", I leveraged [example SQL](https://thetuvaproject.com/data-marts/ahrq-measures) queries that can be found on Tuva's data marts' explainers on their website. This was all compiled into a [CSV](https://github.com/will-pang/osler-mcp/blob/main/benchmarks/evals/tuva_project_demo/input.csv) where I (mentally) sorted which queries would require understanding of table relationships (i.e., by calling `get_model_lineage`).

### 🎯 Results

To my surprise, the LLM rarely invoked the `get_model_lineage` tool, unless explicitly instructed in the prompt. What I mean by this is if you asked a general question, such as:

> What is the prevalence of chronic conditions?

This would likely _not_ invoke any querying of model lineage, but simply rely on `get_database_schema`, `get_table_info` and `execute_query`. To see `get_model_lineage` used, you really needed to "force" the LLM by being very explicit about the tool choice in your prompt, despite (as a human) knowing that this question cannot be answered without some relational understanding of tables.

### 🧠 Concluding Thoughts

While this is far from a rigorous study on tool choice by LLM agents, I had a few quick thoughts:

- The power of a well-curated file system/table schema: The most accurate responses came from table names that were clear and non-repetitive. This was evident by how confident the LLM was in its response simply from reading the database schema and getting a peak into the table columns/structure. For questions that require minimal joins and leaned heavier on aggreation/filtering/window functions, LLMs performed quite well.
- More relational context might **not** helpful: Initially, I was convinced that I could develop clever ways to force the LLM to "think/plan" with relational context in mind. But as I did more testing, it was not clear to me whether more sophisticated prompt engineering and scaffolding would yield meaningful gains, or only lead to [overfitting](https://www.ibm.com/think/topics/overfitting).
- Understanding relational data is still hard: In order for the LLM to effectively navigate through tables that are inherently relational, a graph perspective might be needed (I like to think of this as seeing things in Cartesian system vs Polar coordinates). However, graph databases come with its own set of [challenges](https://arxiv.org/pdf/2504.10950) and will probably still require consistent table pruning before it can be used effectively with LLMs.
