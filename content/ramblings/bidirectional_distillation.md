---
title: Thoughts on bidirectional distillation
date: 2025-02-28
draft: false
---

A few weeks ago, I read a paper by Huan He and team on creating synthetic EHR data, which made me think deeper about the potential impact of generating good synthetic data, especially in the generative AI healthcare space where obtaining any type of data is hard yet necessary to train/fine-tune powerful foundation models.

Perhaps more relevant in the world of scribing and voice AI, being able to simulate a conversation between a doctor and patient using chart notes is extremely valuable. In other words, we want to be able to generate a synthetic full-length conversation from provider notes, and using the synthetic-full length conversation be able to generate synthetic provider notes that matches the original provider notes.

One paper that attempts to tackle this problem is titled “NoteChat: A Dataset of Synthetic Patient-Physican Conversatons Conditioned on Clinical Notes”, where the authors used a cooperative multi-agent framework to generate synthetic patient-physician conversations. The “secret sauce” of the paper was to break up the task into three components: (1) a planning stage, where the LLM is instructed to basically storyboard/do a high-level planning of the conversation (for example, asking about where it hurts, how long you’ve been experiencing the pain etc.) (2) a roleplaying stage, where the LLM essentially fills in the actual dialogues given by the storyboard (3) a polishing stage, where the LLM is tasked with making the conversation more realistic (e.g., making the dialogue more colloquial and more back-and-forth). To my surprise, this entire workflow was generated with GPT-3.5-turbo, and outperformed GPT4 and other models!

The authors also asked medical professionals to evaluate the conversation — one interesting feedback that caught my eye was that the synthetic conversations often “covered to much information from the clinical note compared to real-world conversations”. Although this was not particularly surprising, it is an interesting problem that I think is more challenging in the healthcare space: not only do we need to provide the right context, we need to know WHEN to inject the correct context, as well as HOW MUCH context to provide.
