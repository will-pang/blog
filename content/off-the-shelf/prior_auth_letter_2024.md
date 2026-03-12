---
title: Leverage LLMs in Healthcare Tasks, a commentary
date: 2026-03-12
tags: "llm"
---

This is how I did it with a team in 2024, with _commentary from 2026_.

## Problem Statement

Once a doctor/nurse from the insurance company makes a decision on whether to cover a request for medical treatment, a trained letter-writing staff needs to take the decision note written by clinical staff and send it as a letter written in plain english back to the patient.

### Inputs

- Medical Decision from Doctor/Trained Nurse (A type of clinical note)
- Patient's Health Plan Basic Information (i.e., name of insurance)
- An example of how this decision letter should look like, depending on the health plan (fed in the prompt)

### Models Used

- Azure GPT-4o
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

Once we had all the medical terminologies simplified, we needed to plug it all back into a letter (on behalf of the health plan) and send it back to the patient/provider! This was probably the trickiest component, as piecing all the components together meant a large context-window was required and could just result in the same problems we faced when we tried to feed everything into one prompt (one-shot). We made a separate API call with this prompt:

```
prompt =
"""
This is our draft letter: The patient has dry skin (<<<dermatitis>>>), which does not qualify for skin graft treatment.

Please format it the same way as an example letter from Blue Moss Insurance Plan:
The patient has {condition}, which does not qualify for {treatment} under rule 3(iii) of the Blue Moss health insurance plan.

Additional instructions that you MUST follow:
- Make sure that all medical conditions are explained with the medical terminologies in (parathenses)
- We need this letter to be at a sixth grade reading level as evaluated by the Flesch-Kincaid Grade Level test, which is a blended score that measures the ratio of syllables in words as well as the ratio of words in a sentence
- Remove all <<< >>> symbols
- (A few other edge cases that the model would forget) Always address the patient by their last name (Mr./Ms)
"""
```

While the prompt seemed fool-proof in the beginning (we listed all the instructions!!), in reality we were having a lot of trouble having the model follow the instructions correctly. In particular, the model seemed confused as to why we went through all the trouble of simplying medical terminologies (e.g., putting things in parathenses and `<<< >>>` symbols), and would either end up following the specific instructions (i.e., removing the `<<< >>>`) but rewrite/modify what was within the `<<< >>>` symbols OR ignore the instructions but preserve the work we did in step (1) and (2). tldr; the pipeline went through many automated retry attemps if it failed the eval, and still would not get it right.

One idea we had was to incorporate conversational history, with the idea that in objective (3) it should know the prompts/response from the model in objective (1) and objective (2). We leveraged Langchain's Memory module (v1 Langchain was very clunky), but regardless we were able to see a significant increase in performance (less retries) as the model was able to understand what it previously did and how it relates to the current task.

> _In 2026 terms, this would probably be called (short-term) agentic memory, which is still a very hot topic of research and a challenging problem to tackle_

### Parting thoughts

As someone who has straddled between the extremes of "AI is useless" to "AI is going to take my job away in the next three years", reflecting on the work that I did two years ago when LLM/NLP models was in its nascence and comparing it with the tools available today made me realize a few things:

- A lot of fundamentals are still being built out and will likely stay, so thinking that "models will only get better" != "the experience/knowledge developed today will automatically be obselete a few years from now". _There are still a lot of hard problems to solve!_
- Agents that perform the best rely on good software (e.g., good design/logging/tracing). For now, I see agents as an incredibly powerful orchestrator with "self-healing/correcting" capabilities, but we still need to continue developing the fundamental building blocks which agents leverage
- While the market might say otherwise, I think there's still a huge value in being a programmer and learning the fundamentals well. If you're truly interested in computer systems, I don't think you should be discouraged from pursuing CS/going into the field
