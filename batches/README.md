# FANTASIA Batch Utilities

This repository contains utilities to:

1. **prepare FASTA datasets for large embedding runs**, and  
2. **execute FANTASIA batch jobs model by model** using a monolithic runner with checkpointing and structured outputs.  

The main scripts documented here are:

- `sort_and_batch_fasta_by_length_v2.py`  
- `run_fantasia_batches_with_checkpoint_REVISED_v6_2_1_monolithic.py`  

Example parameter files documented here are:

- `params_sort_batches_example.txt`  
- `params_fantasia_models_v6_1_test.txt`  

---

# 1. FASTA Preparation — `sort_and_batch_fasta_by_length_v2.py`

## Purpose

This script prepares a large FASTA file for downstream FANTASIA execution by:

- reading the input FASTA in streaming mode,  
- filtering sequences below a configurable minimum length,  
- recording the filtered sequences for traceability,  
- keeping retained sequences ordered from **shortest to longest**,  
- and splitting the retained sequences into large FASTA batches.  

This is useful when very large sequence collections must be processed in a controlled and reproducible way.  

---

## What the script does

`sort_and_batch_fasta_by_length_v2.py` performs the following steps:

1. reads each FASTA record,  
2. computes sequence length,  
3. filters out sequences below `--min-length-kept`,  
4. stores the filtered-out proteins in a dedicated FASTA and CSV manifest,  
5. implicitly sorts the retained sequences from **shortest to longest**,  
6. groups the retained sequences into output FASTA batches,  
7. writes a JSON summary and a CSV batch manifest.  

---

## Main outputs

Inside `--output-dir`, the script typically creates:

- `length_sorted_batch_summary.json`  
- `length_sorted_batch_manifest.csv`  
- `filtered_out_below_min_length.fasta` *(or a custom name)*  
- `filtered_out_below_min_length_manifest.csv` *(or a custom name)*  
- batch FASTA files such as:  
  - `uniref50_sorted_batch_00001.fasta`  
  - `uniref50_sorted_batch_00002.fasta`  
  - ...  

---

## Traceability of filtered sequences

Unless disabled explicitly, the script stores the sequences removed by the length filter in two complementary files:

### Filtered FASTA
Contains the original FASTA entries that were excluded.

### Filtered CSV manifest
Contains one row per removed sequence with fields such as:

- `identifier`  
- `full_header`  
- `sequence_length`  
- `filter_reason`  

This provides traceability of which proteins were excluded and why.

---

## Important parameters

### Input / output

- `--input` → input FASTA file  
- `--output-dir` → output directory for batches and reports  

### Filtering and batching

- `--batch-size` → number of proteins per batch  
- `--min-length-kept` → minimum sequence length kept  
- `--report-threshold` → thresholds to report in the summary  

### Traceability

- `--filtered-fasta-name`  
- `--filtered-manifest-name`  
- `--no-write-filtered-trace`  

### Other

- `--output-prefix`  
- `--report-only`  
- `--keep-temp`  
- `--overwrite`  
- `--args-file`  

---

## Example: direct usage

```bash
python sort_and_batch_fasta_by_length_v2.py \
  --input /home/avelasco/fantasia/data/uniref50_representatives.fasta \
  --output-dir /home/avelasco/fantasia/uniref50_batches_sorted_500k \
  --batch-size 500000 \
  --min-length-kept 51 \
  --output-prefix uniref50_sorted \
  --filtered-fasta-name filtered_out_below_51aa.fasta \
  --filtered-manifest-name filtered_out_below_51aa_manifest.csv \
  --overwrite
```

---

## Example parameter file: `params_sort_batches_example.txt`

Example content:

```txt
# =========================
# Input and output
# =========================
--input /home/avelasco/fantasia/data/uniref50_representatives.fasta
--output-dir /home/avelasco/fantasia/uniref50_batches_sorted_500k

# =========================
# Batching
# =========================
--batch-size 500000
--min-length-kept 51

# =========================
# Output prefix
# =========================
--output-prefix uniref50_sorted

# =========================
# Thresholds to report
# =========================
--report-threshold 30
--report-threshold 50

# =========================
# Traceability files for filtered sequences
# =========================
--filtered-fasta-name filtered_out_below_51aa.fasta
--filtered-manifest-name filtered_out_below_51aa_manifest.csv

# =========================
# General behavior
# =========================
--overwrite
```

Run it using:

```bash
python sort_and_batch_fasta_by_length_v2.py \
  --args-file /home/avelasco/scripts/params_sort_batches_example.txt
```

---

## Report-only mode

If you only want a quick report of sequence-length thresholds without writing batches:

```bash
python sort_and_batch_fasta_by_length_v2.py \
  --input /home/avelasco/fantasia/data/uniref50_representatives.fasta \
  --output-dir /home/avelasco/fantasia/uniref50_length_report \
  --report-only
```

---

# 2. FANTASIA Batch Runner — `run_fantasia_batches_with_checkpoint_REVISED_v6_2_1_monolithic.py`

## Purpose

This monolithic runner executes FANTASIA over a directory of FASTA batches while preserving traceability by:

- **session**,  
- **model**,  
- **batch**,  
- **experiment directory**,  
- and **FANTASIA log destination**.  

It is designed for large embedding workflows where:

- batches have already been prepared,  
- multiple models must be executed sequentially,  
- only one model should be active at a time,  
- and outputs must be organized clearly for later inspection.  

---

## Key features

### Monolithic implementation

This version is fully self-contained in a **single script** and does not depend on wrapper chaining.

### Sequential execution by model and batch

The runner executes:

1. all selected batches for the first selected model,  
2. then all selected batches for the next model,  
3. and so on.  

This avoids having multiple embedding models active at the same time.

### Automatic model enable / disable

Before each run, the runner edits the config so that:

- the selected model is set to `enabled: True`,  
- all other models are set to `enabled: False`.  

### Config compatibility

The runner supports both configuration layouts for model definitions:

#### Root-level layout

```yaml
models:
  ESM:
    enabled: false
```

#### Nested layout under `embedding`

```yaml
embedding:
  models:
    ESM:
      enabled: false
```

This is important because some FANTASIA configurations define models under `embedding.models`.

### Live terminal output

The runner mirrors the output of:

```bash
poetry run fantasia run
```

back to the terminal while the run is still executing.

### Minimal runner log

The runner keeps a small execution log with high-level events such as:

- start time,  
- end time,  
- return code.  

It does **not** duplicate the full FANTASIA internal logs, because FANTASIA already writes its own detailed `info.log` and `debug.log` files.

### Checkpointing

The runner writes a checkpoint JSON containing:

- current stage,  
- current model,  
- current batch,  
- execution plan,  
- run-level status,  
- return codes,  
- final experiment paths,  
- final log paths,  
- and relocation errors.  

### Structured output folders

The runner organizes both experiment folders and FANTASIA log folders by:

- session  
- model  
- batch  

---

## Expected configuration

The runner expects a valid FANTASIA `config.yaml` containing at least:

- `base_directory`  
- `log_path`  
- `input`  
- `prefix`  
- model definitions under either `models` or `embedding.models`  

### Recommended absolute paths

For robust execution, especially on scratch storage or clusters, it is recommended to use absolute paths for:

- `base_directory`  
- `log_path`  
- `constants`  

Example:

```yaml
log_path: /scratch/avelasco/fantasia/logs/
constants: /home/avelasco/FANTASIA/fantasia/constants.yaml
base_directory: /scratch/avelasco/fantasia/
```

---

## How the runner works

For each selected model and batch, the script:

1. updates the input FASTA path in the config,  
2. updates the experiment prefix,  
3. updates the FANTASIA log path,  
4. enables only the selected model,  
5. launches `poetry run fantasia run --config ...`,  
6. streams output live to the terminal,  
7. records run metadata in the checkpoint,  
8. relocates the created experiment folder into the final session/model/batch hierarchy.  

---

## Output structure

### Experiment outputs

Experiment outputs are stored under:

```text
<base_directory>/experiments/<session_name>/<model_name>/<batch_name>_<runstamp>/
```

Example:

```text
/scratch/avelasco/fantasia/experiments/
└── test_uniref50_MF_sorted_batches_20260514132440/
    ├── ESM/
    │   ├── uniref50_MF_sorted_batch_00001_20260514132440/
    │   └── uniref50_MF_sorted_batch_00002_20260514132440/
    └── ESM3c/
        └── uniref50_MF_sorted_batch_00001_20260514132440/
```

### FANTASIA log outputs

FANTASIA logs are redirected to:

```text
<log_path>/<session_name>/<model_name>/<batch_name>_<runstamp>/Logs_<timestamp>/
```

Example:

```text
/scratch/avelasco/fantasia/logs/
└── test_uniref50_MF_sorted_batches_20260514132440/
    └── ESM/
        └── uniref50_MF_sorted_batch_00001_20260514132440/
            └── Logs_20260514133012/
                ├── info.log
                └── debug.log
```

### Minimal runner logs

The script also stores a minimal runner log under the directory given by `--logs-dir`, for example:

```text
/home/avelasco/fantasia/fantasia_batch_runner_logs_v6_1_test/
└── fantasia_runner_Prot-T5__batch_00001.log
```

This log is **not** the same as FANTASIA `info.log` or `debug.log`.

---

## Model selection

The `--model-select` parameter controls which models are executed.

### All models

```txt
--model-select all
```

This runs all models defined in the config.

### Only models enabled in the original config

```txt
--model-select enabled
```

This runs only models whose `enabled` flag is already `True` in the original config.

### Explicit model list

```txt
--model-select "ESM,ESM3c,Ankh3-Large,Prot-T5"
```

This is the recommended way to run **all models except one**.

For example, to run all models except `Prost-T5`:

```txt
--model-select "ESM,ESM3c,Ankh3-Large,Prot-T5"
```

---

## Batch selection

The `--batch-select` parameter controls which discovered batches are processed.

### Supported values

```txt
--batch-select all
--batch-select 1
--batch-select last
--batch-select 1,last
--batch-select 1,2
--batch-select 1-5
--batch-select 3-last
```

Examples:

- `1,2` → first and second batch  
- `3-last` → from batch 3 to the last batch  
- `all` → all discovered batches  

---

## Example parameter file for the runner

The following file is a small test configuration:

- `params_fantasia_models_v6_1_test.txt`  

Example content:

```txt
# =========================
# Main config
# =========================
--config /home/avelasco/fantasia/config.yaml

# =========================
# Batches
# Small test run: first two batches only
# =========================
--batches-dir /home/avelasco/fantasia/uniref50_batches_sorted_500k
--batch-pattern "*_batch_*.fasta"
--batch-select 1,2

# =========================
# Models
# Small test run: Prot-T5 only
# =========================
--model-select Prot-T5

# Base name for the grouped experiment/session
--session-name-base test_uniref50_sorted_batches

# =========================
# Execution
# =========================
--runner-cmd "poetry run fantasia run"

# =========================
# Config keys
# =========================
--input-key input
--prefix-key prefix
--log-path-key log_path

# =========================
# Runner state and minimal runner logs
# =========================
--checkpoint /home/avelasco/fantasia/fantasia_batches_checkpoint_v6_1_test.json
--logs-dir /home/avelasco/fantasia/fantasia_batch_runner_logs_v6_1_test
```

---

## Example usage

### Small test run

```bash
python run_fantasia_batches_with_checkpoint_REVISED_v6_2_1_monolithic.py \
  --args-file /home/avelasco/Downloads/params_fantasia_models_v6_1_test.txt
```

### Run all batches for all models

```txt
--batch-select all
--model-select all
```

### Run all models except `Prost-T5`

```txt
--batch-select all
--model-select "ESM,ESM3c,Ankh3-Large,Prot-T5"
```

### Run batches from 3 to the last one

```txt
--batch-select 3-last
```

---

## Checkpoint inspection

The checkpoint JSON is useful to inspect:

- current execution state,  
- finished runs,  
- failed runs,  
- relocation errors,  
- final experiment and log destinations.  

Example with `jq`:

```bash
jq '{current_stage, current_model, current_batch, session_name, models_plan}' \
  /home/avelasco/fantasia/fantasia_batches_checkpoint_v6_1_test.json
```

List runs:

```bash
jq -r '.runs | keys[]' /home/avelasco/fantasia/fantasia_batches_checkpoint_v6_1_test.json
```

Inspect one run in detail:

```bash
jq '.runs["Prot-T5__batch_00001"]' \
  /home/avelasco/fantasia/fantasia_batches_checkpoint_v6_1_test.json
```

---

## Notes about logging

- The live output shown in the terminal may include warnings emitted directly by the model-loading libraries.  
- Those warnings do not always appear in `info.log` or `debug.log`.  
- FANTASIA detailed logs are written by FANTASIA itself under the configured `log_path`.  
- The runner log is intentionally minimal and does not duplicate the full console stream.  

---

## Practical recommendations

- Run the script from the **root of the FANTASIA project** if your config still uses relative paths such as `./fantasia/constants.yaml`.  
- Prefer a dedicated `config_scratch.yaml` if you want to redirect outputs to `/scratch/avelasco/...`.  
- Start with a short test run (`Prot-T5`, batches `1,2`) before launching all models.  
- Use explicit model selection when excluding specific models.  
- Check the checkpoint file if a flat experiment directory remains in the root `experiments/` folder, since this usually indicates a failed run or relocation problem.  

---

## Suggested file layout

Example:

```text
scripts/
├── sort_and_batch_fasta_by_length_v2.py
├── run_fantasia_batches_with_checkpoint_REVISED_v6_2_1_monolithic.py
├── params_sort_batches_example.txt
├── params_fantasia_models_v6_1_test.txt
└── README.md
```

---

## Final notes

- This README documents the **monolithic** runner workflow and replaces the previous README focused on the older v6 wrapper approach.  
- The runner is intended mainly for **embedding-oriented execution**, especially when `only_embedding: True` is used in the FANTASIA config.  
- For production runs, prefer absolute paths in the YAML and store experiment/log outputs outside the repository itself when using scratch storage.  
