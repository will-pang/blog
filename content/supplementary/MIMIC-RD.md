---
title: MIMIC-RD
date: 2026-01-18
draft: true
tags: "llm"
---

[ML4H Paper](https://openreview.net/pdf?id=CrZyNUfWHp)

## Processing MIMIC-IV Notes

1. Download the MIMIC-IV-Note from [PhysioNet](https://www.physionet.org/content/mimic-iv-note/2.2/)(note: you need to complete CITI training before you can use the dataset)
2. Load the note as a Pandas Dataframe
3. Group notes by `id` and join the different texts. In this example, I want the `subject_id` to be the outer object, and the inner object to be the `{charttime: note}`
   ```
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
   ```
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

   ```
   # Pseudocode
      for i, (patient_id, patient_data) in enumerate(tqdm(list(patient_notes.items()), desc="Processing cases")):
       for charttime, note in patient_data.items():
           note['llm_extracted_entities'] = LLMclient(note['note_content'])
           note['entity_context'] = CustomContextExtractor(note['llm_extracted_entities'], note['note_content'], window_size=0)
   ```

## Verifying correctness of extracted entities

1. To grab the context, we want to transform the note text into a list of sentences. We can first split the text by common sentence terminiators:

```
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

```
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

```
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
   # entity_words = set(re.findall(r"\b\w+\b", entity_lower))
   # for i, sentence in enumerate(sentences):
   #     sentence_words = set(re.findall(r"\b\w+\b"), entity_lower)

   #     common_words = entity_words & sentence_words

   #     best_match_i = i # This updates as a better match gets found

   #  return self.get_context_window(sentences, best_match_i, window_size)

```

3. Finally, we put everything together. You probably want to store it something like this:

   ```
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

   ```
   note['entity_context'] = [
      {'entity': 'Squamos cell carcinoma', 'context': 'Squamos cell carcinoma of epiglottis, treated with'},
      {'entity': 'Adenocarcinoma', 'context': 'completed ___ ___, adenocarcinoma of stage IV left lung'}, ...
   ]
   ```

   **Note:** If the context is missing, we can be somewhat confident that the LLM hallucinated its response.

## Benchmarking human annotation

| Metric | Name           | Description                                                          |
| -----: | -------------- | -------------------------------------------------------------------- |
|     TP | True Positive  | The LLM correctly identified a rare disease annotated by a human.    |
|     FP | False Positive | The LLM identified a rare disease that was not annotated by a human. |
|     FN | False Negative | The LLM failed to identify a rare disease annotated by a human.      |

**Note:** True Negatives are undefined in this setting, as the absence of both LLM and human annotations is not a very meaningful outcome to measure.

In annotating rare diseases, precision is probably the most useful metric to track.
