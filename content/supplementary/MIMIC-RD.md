---
title: MIMIC-RD
date: 2026-01-18
private: true
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
           note['entity_context'] = SomeRegex(note['llm_extracted_entities'], note['note_content'], window_size=0)
   ```

3. Then, extract the surrounding context for the extracted entity. You probably want to store it something like this:
   ```
   note['entity_context'] = [
      {'entity': 'Squamos cell carcinoma', 'context': 'Squamos cell carcinoma of epiglottis, treated with'},
      {'entity': 'Adenocarcinoma', 'context': 'completed ___ ___, adenocarcinoma of stage IV left lung'}, ...
   ]
   ```

## Verifying correctness of extracted entities

## Benchmarking human annotation

| Metric | Name           | Description                                                          |
| -----: | -------------- | -------------------------------------------------------------------- |
|     TP | True Positive  | The LLM correctly identified a rare disease annotated by a human.    |
|     FP | False Positive | The LLM identified a rare disease that was not annotated by a human. |
|     FN | False Negative | The LLM failed to identify a rare disease annotated by a human.      |

**Note:** True Negatives are undefined in this setting, as the absence of both LLM and human annotations does not constitute an evaluable outcome.
