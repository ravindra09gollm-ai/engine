 README — Pipeline Run Instructions

This guide shows you how to run the end-to-end ingestion → normalization → mapping → flattening .

---

## 0) Prerequisites

- **Python** 3.10+
- **PostgreSQL** (or **MongoDB**) running locally or reachable via URI
- Excel files in a local folder (e.g., `./data/`)
- Python deps:  
  ```bash
  pip install pandas openpyxl psycopg2-binary pymongo python-dotenv openai
  ```

### Environment variables (`.env`)
Create a `.env` file in your project root:
```bash
OPENAI_API_KEY=sk-...
DB_BACKEND=postgres          # or: mongodb
POSTGRES_URI=postgresql://user:password@localhost:5432/yourdb
MONGODB_URI=mongodb://localhost:27017
EXCEL_DIR=./data
```

> Note: Your code should initialize OpenAI as:
> ```python
> from openai import OpenAI
> client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
> ```

---

## 1) Launch a Python session

Use either a script (e.g., `run_pipeline.py`) or an interactive shell (`ipython` / notebook). Make sure all the below functions are **imported** from your project modules.

---

## 2) Create database & ingest Excel

```python
create_database()

import time

start = time.time()
ingest_excel_to_postgres()  # or: ingest_excel_to_mongodb()
print("Ingestion Time:", time.time() - start, "seconds")
```

---

## 3) Load DataFrames

```python
start = time.time()
dfs = load_all_dataframes()
print("loading Time:", time.time() - start, "seconds")
```

---

## 4) Unpack DataFrames by year

```python
# Unpack into both locals() and a canonical dict (your preferred pattern)
dfs = load_all_dataframes()
unpacked_dfs = {}

for key, df in dfs.items():
    if key.startswith("dataframe_"):
        year = key.split("_")[1]
        var_name = f"df_{year}"
        locals()[var_name] = df
        unpacked_dfs[var_name] = df
        print(f"Created variable {var_name} with {len(df)} rows")
```

---

## 5) Extract & clean sentiment fields

```python
for name, df in unpacked_dfs.items():
    extract_sentiment_fields(df)
    print(f"Updated {name}")

# Normalize column names and harmonize sentiment fields across years
clean_and_rename_sentiment_fields(unpacked_dfs)
```

---

## 6) Demographic key mapping (LLM-assisted)

```python
# Collect unique demographic keys
unique_demographic_keys = extract_unique_demographic_keys(unpacked_dfs)

# Step 1: Generate prompt
prompt = generate_demographic_mapping_prompt(unique_demographic_keys)

# Step 2: Query multiple models (includes GPT-4 Turbo)
mapping_responses = {
    "gpt-4":       get_cleaned_key_mapping(prompt, model_name="gpt-4"),
    "gpt-4-turbo": get_cleaned_key_mapping(prompt, model_name="gpt-4-turbo"),
    "gpt-3.5":     get_cleaned_key_mapping(prompt, model_name="gpt-3.5-turbo"),
}

# Step 3: Preview
for model, response in mapping_responses.items():
    print(f"\n{model.upper()} Mapping Suggestion:\n")
    print(response)

# Optional: usage & cost
print_usage_summary()

# Step 4: Choose mapping and apply
demographic_key_mapping = choose_llm_mapping(mapping_responses)
normalize_all_demographics(unpacked_dfs, demographic_key_mapping)
```

---

## 7) Question key normalization

```python
# 1) Extract question keys
question_keys = collect_question_keys_from_unpacked_dfs(unpacked_dfs)

# 2) Get normalization from multiple models
mappings = normalize_question_keys_with_models(question_keys)

# 3) Let user select a mapping
selected_mapping = choose_question_key_mapping(mappings)

# Apply to all yearly DataFrames
normalized_dfs = apply_mapping_to_unpacked_dfs(unpacked_dfs, selected_mapping)
unpacked_dfs = normalized_dfs
```

---

## 8) Explode, label, flatten, and type inference

```python
explode_questions_into_named_ratings(unpacked_dfs)
apply_macro_labels_from_namedquestion(unpacked_dfs, MACRO_THEME_MAP)

flattened_dfs = flatten_demographics_columns(unpacked_dfs)

# Ensure types are inferred after flattening
unpacked_dfs = apply_type_inference_to_unpacked_dfs(unpacked_dfs)
unpacked_dfs = apply_type_inference_to_unpacked_dfs(flattened_dfs)
```

---

## 9) Outputs & sanity checks

- Expect console prints for ingestion and loading times.
- Expect “Created variable df_YYYY…” lines.
- Spot-check:
  - Column types for `year` (categorical vs numeric) are correct.
  - Sentiment/rating fields are numeric where needed.
  - Demographic keys are consistently normalized across years.

---

## 10) Troubleshooting

- **Permission/connection errors**: verify `POSTGRES_URI` / `MONGODB_URI`.
- **Excel parse errors**: confirm files in `EXCEL_DIR` and install `openpyxl`.
- **LLM calls fail**: check `OPENAI_API_KEY`, model names, and network access.
- **Type issues (e.g., `year`)**: ensure your type inference or explicit casts align with downstream grouping/filters.

---

## 11) Recommended defaults

- Use **GPT-4-Turbo** for mappings (balanced accuracy/cost).
- Keep LLM **temperature low** (≈0.2–0.3) for deterministic normalization.
- Optional: add a **1–1.5s retry delay** when batching LLM calls to stay under rate limits.
