---
title: To RAG or not to RAG
date: 2025-02-14
draft: false
---

![RAG](ramblings/images/rag-library.png)

Some random thoughts on RAG and healthcare: Lately I’ve been reading a bit of Chip Huyen’s blog, and she gave the analogy that RAG can be thought of as “feature engineering in classical models”. This was a helpful analogy, and reminded me of how even with classical ML models in the healthcare space, we struggle with whether to deploy an algorithmic approach to cull features (think Lasso), or rely on medical expertise to tell us which features they’ve seen clinically to be important. From experience, it’s often a mixture of both. 

Admittedly not a 1:1 analogy, but I wonder if the same holds true in GenAI applications in healthcare, where source attribution is a must (if you’re trying to approve or deny a prior authorization, you have to point to the PDF that contains the rules and regulations). Indeed, there are lot of out-of-the-box tools to play around with (Amazon Bedrock, Azure AI Search) or you could do chunking + embedding and store it into a vector database yourself, but perhaps some heuristic indexing strategy (with domain knowledge) might outperform RAG. Or is it a mix of both strategies?
