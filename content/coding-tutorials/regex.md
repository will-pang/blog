---
title: Regex
date: 2026-01-23
draft: true
---

## Cleaning with `groupby`

1. Normalizing names with `ID`:

```python
df_cleaned = df.copy()

df_cleaned["full_name"] = (
    df_cleaned["full_name"]
    .str.lower()
    .str.strip()
)

grouped_names = (
    df_cleaned
    .groupby("full_name")
    .agg(IDs = ("ID_Col", lambda x: list(x.unique())), ID_count = ("ID_Col", "nunique"))
)
```

2a. Normalizing addresses with `ID`:

```python
df_cleaned = df.copy()

df_cleaned["address"] = (
    df_cleaned["address"]
    .str.lower()
    .str.strip()
    .str.replace(r"[^\w\s]", "", regex=True) # Remove anything that is not a word OR whitespace
    .str.replace(r"\s+", " ", regex=True) # Replace anything that is >1 whitespace character into 1 whitespace character
)

grouped_names = (
    df_cleaned
    .groupby("address")
    .agg(IDs = ("ID_Col", lambda x: list(x.unique())), ID_count = ("ID_Col", "nunique"))
)
```

2b. More sophisitcated normalizing addresses with `ID`:

```python
df_cleaned = df.copy()

df_cleaned["address"] = (
    df_cleaned["address"]
    .str.lower()
    .str.replace(r"\bbldg\b, "building", regex=True)
    .str.replace(r"\bave\b", "avenue", regex=True)
    .str.replace(r"\bst\b", "street", regex=True)
    .str.replace(r"\bste\b", "suite", regex=True)
    .str.replace(r"\brd\b", "road", regex=True)
)

grouped_names = (
    df_cleaned
    .groupby("address")
    .agg(IDs = ("ID_Col", lambda x: list(x.unique())), ID_count = ("ID_Col", "nunique"))
)
```

## Cleaning CMS's ICD-10 Code File

April 1, 2026 Code Descriptions in Tabular Order (ZIP) [[CMS](https://www.cms.gov/medicare/coding-billing/icd-10-codes)]

1. Load file

```python
icd = pd.read_csv(file_path,
                  sep=r"\s{2,}", # Delimiter is 2 spaces
                  header=None,
                  names=["ICD10_Code", "ICD10_Code_Description"],
                  engine="python")

icd_cleaned = icd.copy()
```

2. Normalize ICD-10 codes by removing dots:

```python
dot_remover = re.compile(r"(\d)\.")

icd_cleaned['ICD10_Code'] = (
    icd_cleaned['ICD10_Code'].str.upper()
    .str.replace(dot_remover, "", regex=True)
    # .str.replace(r"(\d)\.", "", regex=True)
)
```

3. In cases where the delimiter is not 2 spaces, we want to rectify it:

```python
mask = icd_cleaned["ICD10_Code_Description"].isna()
icd_cleaned.loc[mask, ["ICD10_Code", "ICD10_Code_Description"]] = (
    icd_cleaned.loc[mask, "ICD10_Code"]
    .str.extract(r"^(?P<code>\S+)\s+(?P<description>.+)$")
    .values
)

icd_cleaned['ICD10_Code_Root'] = icd_cleaned['ICD10_Code'].str[:3]
```
