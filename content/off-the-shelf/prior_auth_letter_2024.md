---
title: Building a prior auth letter generator (in 2024)
date: 2026-01-17
draft: true
tags: "llm"
---

This is how I did it in 2024, with _commentary from 2026_.

## Problem Statement

Once a doctor/nurse from the insurance company makes a decision on whether to cover a request for medical treatment, a trained letter-writing staff needs to take the decision note written by clinical staff and send it as a letter written in plain english back to the patient.

### Inputs

- Medical Decision from Doctor/Trained Nurse (A type of clinical note)
- Patient's Health Plan Basic Information (i.e., name of insurance)
- An example of how this decision letter should look like, depending on the health plan (fed in the prompt)

### Models Used

- Azure GPT4o
- Medical Named Entity Recognition [[HuggingFace](https://huggingface.co/blaze999/Medical-NER)]

### Evals

- The letter must be at a sixth grade reading level, evaluated using the [Flesch-Kincaid Grade Level](https://readable.com/readability/flesch-reading-ease-flesch-kincaid-grade-level/) algorithm (which basically counts the average sentence length and syllables per word)
- All medical terminologies must be explained
- Must pass the "smell test" (hard to objectively evaluate: it needs to sound like a letter from the health plan!)

## Approach

> _While this was developed before the era of agentic planning/execution, I think we adopted some design principles that we see large providers (i.e., Claude Code) use today!_

We began this project with the simplest implementation: could we just feed the doctor's decision note to GPT-4o with some prompt engineering and return something decently good? After a few iterations of prompt engineering, we found out that the model would usually satisfy one of the three evals requirement, but rarely satisfied all three at the same time (which would just add extra time/work for the letter-writing team!)

To resolve this performance issue, we needed to break down this (inconspicuously complicated) task into smaller chunks: (1) identify all medical terminologies (2) simplify extracted medical terminologies into plain english (3) plug it back into the letter template and have the LLM make sure it was coherent.

> _In hindsight, this is what sub-agents essentially are! We're separating out the tasks so that the model doesn't have to (1) deal with a lot of context all at once (2) try to solve for multiple objectives all at once_

### Objective 1: Identifying all medical terminologies

Our first instinct was to use a NER (named entity recognition) BERT model fined-tuned on medical text to extract medical text entities. We picked a model from HuggingFace (which had a widget that allowed you to play around with a few examples so you can see how the model performs at inference) and fed a few examples, which performed reasonably well as we expected.

While we had no issues with model performance, we quickly realized that this was tricky to implement with other components of our pipeline, which would require us to (1) download the data land run the NER task locally (2) bring it back to Databricks for storage. Ideally, we would like to have done this all within Databricks.

Rather than using a fine-tuned model, we thought that perhaps a non-fined tuned (but way larger) model like gpt-4o would perform the abstraction task reasonably well. Part of the thinking was that we weren't working with extracting obscure diseases (we had some data on which conditions to look out for) so a large model should perform reasonably well at identifying _medically-sounding_ words. Our hypothesis proved to be correct, and we constructed a prompt something like this:

```
prompt =
"""
Tag all medical entities starting with <<< and ending with >>>. Ignore negation & uncertainity.
"""
```

### Objective 2: Simplifying extracted medical terminologies into plain English

To translate medical terminologies into plain English, we put together a relatively simple prompt:

```
# psuedocode
for entity in ListOfExtractedEntities:
	LLMClient(
		prompt =
		"""
		Explain this terminology in plain English. Summarize your explanation in
		20 words or less.
		""")
```

In hindsight, we should have just used some curated a database of medical entities to plain english translations and then use simple regex or RAG-based approach instead of relying on prompting, but we were reasonably satisfied with the consistency of the performance.

> _Or better yet (in 2026), I would just create a tool/skill that directs the LLM to look up a pre-curated database of {medical terminologies to plain english translations}_

### Objective 3: Plug it back into the letter template and have the LLM make sure it was coherent

This was probably the trickiest component, as piecing all the components together meant a large context-window was required and could just result in the same problems we faced when we tried to feed everything into one prompt.

**TODO: Talk about fitting everything together, asking the LLM to make sure everything was coherent, retry logics**
