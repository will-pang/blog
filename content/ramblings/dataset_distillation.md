---
title: Thoughts on dataset distillation
date: 2025-02-21
draft: false
---

![Dataset Distillation](ramblings/images/dataset_distillation.png)

Lately, I’ve been reading a bit about dataset distillation, which is the idea of creating a small synthetic dataset from a large real dataset (e.g., chest X-rays) and using the smaller synthetic dataset to train machine learning models (e.g., binary classification of lung cancer using chest X-rays). The premise is quite fascinating: basically, you could chop and dice a large dataset, extract all its important components (cough _gradients_), train a ML model on just the important components, and then get a light-weight model that gets you most of the way.

While there are a plethora of fascinating downstream applications — imagine research accelerating because we are able to share a “compressed” synthetic dataset as opposed to the entire corpus of X-Rays from a database much more easily — I think the upstream impact on core systems, as Yongfeng Zhang has noted, is an equally exciting place to ponder. For instance, if we know that the goal is to collect a smaller but representative enough sample in order to get some desirable performance for a predictive model, how can we build a database system that can do just that (as opposed to current database systems in ML, which is focused on streaming as much data as possible in an optimal way)?

That being said, I think there’s still much we don’t understand about data distillation. How can — in distillation lingo — the student (smaller synthetic model) learn so well from the teacher (larger model), especially with a much smaller set of training data? I think the same question on "learning efficacy" can be asked for prompting techniques, such as Chain-Of-Thoughts and Few-Short Learning. More to ponder…
