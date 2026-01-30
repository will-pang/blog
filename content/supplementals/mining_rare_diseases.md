---
title: Can LLMs diagnose rare diseases?
date: 2026-01-29
draft: false
tags: "llm"
---

Main Blog Post: [Link](../halfbaked/mining_rare_diseases.md)

**Acknowldegements**:
This psuedo-code implementation is based off of this [ML4H Paper](https://openreview.net/pdf?id=CrZyNUfWHp) and leverages the [PyHealth](https://pyhealth.readthedocs.io/en/latest/) package.

## Processing MIMIC-IV Notes

1. Download the MIMIC-IV-Note from [PhysioNet](https://www.physionet.org/content/mimic-iv-note/2.2/)(note: you need to complete CITI training before you can use the dataset)
2. Load the note as a Pandas Dataframe
3. Group notes by `id` and join the different texts. In this example, I want the `subject_id` to be the outer object, and the inner object to be the `{charttime: note}`
   ```python
   # Pseudocode
   patient_notes = (
    note_df
    .sort_values("charttime")
    .groupby("subject_id")
    .apply(
        lambda x: dict(zip(x["charttime"], x[text]))
     )
    )
   ```
4. Convert to JSON with `id` as the key for better interoperability/better compatibility with LLMs
   ```python
   # Pseudocode
   data = {
    str(patient_id): {
       str(charttime): {"note_content": text}
       for charttime, text in notes.items()
       }
    for patient_id, notes in patient_notes.items()
    }
   ```

## Extracting Entities

1. Set up your prompt. I'm using this one provided by folks at PyHealth:

> Extract all rare diseases and conditions that are NOT negated (i.e., don't include terms that are preceded by 'no', 'not', 'without', etc.) from the text below.
>
> Text: {text}
>
> Return only a Python list of strings, with each term exactly as it appears in the text. Ensure the output is concise without any additional notes, commentary, or meta explanations.

2. Pass each note to the LLM and store the extract entity:

   ```python
   # Pseudocode
      for i, (patient_id, patient_data) in enumerate(tqdm(list(patient_notes.items()), desc="Processing cases")):
       for charttime, note in patient_data.items():
           note['llm_extracted_entities'] = LLMclient(note['note_content'])
           note['entity_context'] = CustomContextExtractor(note['llm_extracted_entities'], note['note_content'], window_size=0)
   ```

## Verifying correctness of extracted entities

1. To grab the context, we want to transform the note text into a list of sentences. We can first split the text by common sentence terminiators:

```python
class CustomContextExtractor:
   def extract_sentences(self, note):
      # First, split by common setnence terminators while preserving them
      sentence_parts = []
      for part in re.split(r"([.!?])", text):
         if part.strip():
            if part in ".?!":
               sentence_parts[-1] += part
         else:
            sentence_parts.append(part.strip())
```

We also want to split by other clinical note deliminiators, such as line breaks and semicolons. To do so:

```python
class CustomContextExtractor:
   def extract_sentences(self, text):
      # First, split by common setnence terminators while preserving them
      sentence_parts = []
      for part in re.split(r"([.!?])", text):
         if part.strip():
            if part in ".?!":
               sentence_parts[-1] += part
         else:
            sentence_parts.append(part.strip())

      # Second, we want to handle other clinical note delimiters like line breaks and semicolons

      sentences = []

      for part in sentence_parts:
         # split by semicolons and newlines
         for subpart in re.split(r"[;\n]", part):
            if subpart.strip():
               sentences.append(subpart.strip())

      return sentences
```

2. Once we've broken up the note into sentences, we want to match the entity to the sentences (i.e., the context). We can look for exact word matches, or use some fuzzy matching:

```python
class CustomContextExtractor:
   def find_entity_context(self, entity: str, sentences: List[str], window_size):
        entity_lower = entity.lower()
        for i, sentence in enumerate(sentences):
            if entity_lower in sentence.lower():
                # Found exact match - include surrounding sentences based on window_size
                return self.get_context_window(sentences, i, window_size)

   def get_context_window(self, sentences: List[str], center_index: int, window_size: int):
        start_index = max(0, center_index - window_size)
        end_index = min(len(sentences) - 1, center_index + window_size)
        context_sentences = sentences[start_index : end_index + 1]

        return " ".join(context_sentences).strip()

   # More sophisitication: Use some sort of fuzzy matching.
   # This requires iterating over each sentence, and then checking
   # if each word in the sentence "fuzzy" matches the entity. If it fuzzy matches above
   # a threshold, then we have found a match and we keep the index (where the sentence came
   # from)

   # ===== Psuedocode =====
   def find_entity_context_with_fuzz_matching(self, entity: str, sentences: List[str], window_size):
      entity_words = set(re.findall(r"\b\w+\b", entity_lower))
      best_score = 0
      best_index = -1
      for i, sentence in enumerate(sentences):
         sentence_words = set(re.findall(r"\b\w+\b"), entity_lower)

         common_words = entity_words & sentence_words

         # Calculate Jaccard similarity as an example
         similarity_score = len(common_words)/ (
               len(entity_words) + len(sentence_words) - len(common_words)
               )

         if score > best_score:
            best_score = score
            best_match_i = i # This updates as a better match gets found

         if best_match_index >=0:
            return self.get_context_window(sentences, best_match_i, window_size)
         return None
```

3. Finally, we put everything together. You probably want to store it something like this:

```python
class CustomContextExtractor:
   def extract_context(self, entities: List[str], text: str, window_size: int = 0):
      sentences = self.extract_sentences(text)
      results = []
      for entity in entities:
            context = self.find_entity_context(entity, sentences, window_size)
            results.append(
               {
                  "entity": entity,
                  "context": context or "",  # Empty string if no context found
               }
            )
```

which will store things like this:

```python
note['entity_context'] = [
   {'entity': 'Squamos cell carcinoma', 'context': 'Squamos cell carcinoma of epiglottis, treated with'},
   {'entity': 'Adenocarcinoma', 'context': 'completed ___ ___, adenocarcinoma of stage IV left lung'}, ...
]
```

**Note:** If the context is missing or too short, we can be somewhat confident that the LLM hallucinated its response.

## Verifying whether extracted entity is a rare disease

To validate whether the extracted entity is indeed a rare disease, we compare it with the Orphanet [database](https://www.orpha.net) using retrieval augmented generation techniques.

1. We download [Orphanet embedding documents](https://github.com/jhnwu3/RDMA) and utilize FAISS for indexing/searching.

```python
class CustomRareDiseaseRAGVerifier:
   def create_index_from_embeddings(self, embeddings_file: np.ndarray):
        self.embedded_documents = np.load(embeddings_file, allow_pickle=True)

        embeddings_list = [
            np.array(doc["embedding"])
            for doc in self.embedded_documents
            if isinstance(doc["embedding"], np.ndarray) and doc["embedding"].size > 0
        ]

        embeddings_array = np.vstack(embeddings_list).astype(np.float32)

        # Create a FAISS index for the embeddings
        dimension = embeddings_array.shape[1]
        self.index = faiss.IndexFlatL2(dimension)
        self.index.add(embeddings_array)
```

2. We then need to create a method to embed our extracted entity (i.e., transform text ito a vector) and a search method to
   query the entity on the indexed Orpahnet embeddings.

   ```python
   class CustomRareDiseaseRAGVerifier:
      def query_text(self, text: str) -> np.ndarray:
        return np.array(list(self.model.embed([text]))[0]).astype(np.float32)

      def search(self, query, k):
         query_vector = self.query_text(query).reshape(1, -1)
         distances, indices = self.index.search(query_vector, k)

         return distances, indices
   ```

3. Finally, we verify whether the extracted entity is a rare disease or not. Here we combine both RAG
   (which will tell us the top matches) and an LLM
   (to see if the top matches make sense and make a judgement call on whether this is a rare disease or not).

   ```python
    def verify_rare_disease(self, term, embeddings_file):
        self.create_index_from_embeddings(embeddings_file)
        distances, indices = self.search(term, k=5)
        context = "\nPotential matches from database:\n" + "\n".join(
            f"{i+1}. {self.embedded_documents[idx].get('name')} (dist: {dist:.4f})"
            for i, (idx, dist) in enumerate(zip(indices[0], distances[0]))
        )

        prompt = f"""Analyze this medical term and determine if it represents a rare disease.

        Term: {term}
        {context}

        A term should ONLY be considered a rare disease if ALL these criteria are met:
        1. It is a disease or syndrome (not just a symptom, finding, or condition)
        2. It is rare (affecting less than 1 in 2000 people)
        3. There is clear evidence in the context or term itself indicating rarity
        4. For variants of common diseases, it must be explicitly marked as a rare variant
        5. The term should align with the type of entries in our rare disease database.
        6. If there is a partial match, i.e cholangitis vs. sclerosing cholangitis. There must be a mention of its descriptor (sclerosing) in the term itself, otherwise it's invalid match.

        Response format:
        First line: "DECISION: true" or "DECISION: false"
        Next lines: Brief explanation of decision"""

        response = self.llm_client.query(prompt, self.system_message).strip().lower()

        return "decision: true" in response.lower()
   ```
