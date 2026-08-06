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

## Overview

`run_fantasia_batches_with_checkpoint_REVISED_v6_2_1_monolithic.py` is a monolithic orchestration script that runs FANTASIA sequentially across multiple FASTA batches and one or more models defined in `config.yaml`.

The script does not replace FANTASIA. It prepares the FANTASIA configuration for each model-batch combination, launches FANTASIA, tracks progress in a JSON checkpoint, and organizes experiment outputs and logs into a reproducible directory hierarchy.

## Main workflow

The runner performs the following steps:

1. Reads command-line arguments directly or from a text file passed through `--args-file`.
2. Creates a backup of `config.yaml` before modifying the active configuration.
3. Loads the YAML metadata to discover available models and initially enabled models.
4. Discovers FASTA batches using a glob pattern.
5. Applies the requested batch and model selections.
6. Builds an execution plan containing every selected model-batch combination.
7. For each planned run:
   - changes the YAML input value to the current FASTA batch;
   - changes the experiment prefix;
   - redirects the FANTASIA log path;
   - enables only the current model and disables the other known models;
   - updates the checkpoint;
   - launches FANTASIA;
   - optionally mirrors FANTASIA output to the terminal;
   - identifies the experiment directory created by FANTASIA;
   - moves the experiment into the final session/model/batch hierarchy;
   - records completion, failure, and relocation information.
8. Stops if a FANTASIA run returns a non-zero exit code.
9. Records `INTERRUPTED` when the process receives `Ctrl+C`.
10. Restores the original configuration at the end unless `--no-restore-config` is used.

## Requirements

- Python 3
- PyYAML
- FANTASIA installed and available through the configured runner command
- A valid FANTASIA `config.yaml`
- A directory containing FASTA batches
- Read and write permissions for the configuration, checkpoint, log, and experiment directories

Syntax check:

```bash
python3 -m py_compile   run_fantasia_batches_with_checkpoint_REVISED_v6_2_1_monolithic.py
```

## Recommended execution

Run the script with the supplied parameter file:

```bash
python3 run_fantasia_batches_with_checkpoint_REVISED_v6_2_1_monolithic.py   --args-file params_fantasia_models_v6_1_test.txt
```

Arguments placed after `--args-file` are also included. Arguments from the text file are loaded first, followed by the remaining terminal arguments.

Dry-run example:

```bash
python3 run_fantasia_batches_with_checkpoint_REVISED_v6_2_1_monolithic.py   --args-file params_fantasia_models_v6_1_test.txt   --dry-run
```

## Parameters in the supplied TXT file

### `--config`

```text
--config /home/avelasco/FANTASIA/fantasia/config.yaml
```

Path to the main FANTASIA YAML configuration. The runner creates a backup using the suffix:

```text
.bak_before_batches
```

For the supplied path, the backup is:

```text
/home/avelasco/FANTASIA/fantasia/config.yaml.bak_before_batches
```

The backup is created only when it does not already exist. The active configuration is modified during execution and is restored at the end by default.

### `--batches-dir`

```text
--batches-dir /home/avelasco/FANTASIA/data_sample/uniref50_MF_batches_sorted_500k
```

Directory containing the FASTA batch files.

### `--batch-pattern`

```text
--batch-pattern "*_batch_*.fasta"
```

Glob pattern used inside `--batches-dir`. Only regular files matching the pattern are retained. Matching files are sorted lexicographically by resolved path.

Use zero-padded numbering, such as `batch_00001`, so lexical and numeric orders are consistent.

### `--batch-select`

```text
--batch-select 3-last
```

Selects every discovered batch from position 3 through the last position, inclusive.

Supported forms include:

```text
all
1
first
last
1,last
1,2,5
1-5
3-last
last-first
```

Positions are one-based. Repeated positions are removed while preserving their first occurrence.

### `--model-select`

```text
--model-select "ESM,ESM3c,Ankh3-Large"
```

Selects the listed models. Names must exactly match keys under `models` or `embedding.models` in the YAML.

Special values:

```text
all      All discovered models
enabled  Only models enabled in the original backup configuration
```

For each run, the script sets `enabled: True` only for the current model and sets the other discovered models to `enabled: False`.

### `--session-name-base`

```text
--session-name-base test_uniref50_MF_sorted_batches
```

Base name for the grouped session. The runner sanitizes it and appends a timestamp:

```text
test_uniref50_MF_sorted_batches_YYYYMMDDHHMMSS
```

If omitted, the batch directory name is used.

### `--runner-cmd`

```text
--runner-cmd "poetry run fantasia run"
```

Base command used to launch FANTASIA. The script appends the active configuration path automatically:

```bash
poetry run fantasia run   --config /home/avelasco/FANTASIA/fantasia/config.yaml
```

### `--input-key`

```text
--input-key input
```

Name of the YAML key containing the input FASTA path. Before every run, its value is replaced with the current batch path.

If the key occurs multiple times, the script stops unless `--input-key-occurrence N` selects a specific occurrence.

### `--prefix-key`

```text
--prefix-key prefix
```

YAML key used for the internal experiment prefix. The generated prefix combines the model name, batch name, and a timestamp.

Disable prefix modification with:

```text
--prefix-key ""
```

### `--log-path-key`

```text
--log-path-key log_path
```

YAML key containing the FANTASIA logging directory. The runner redirects it to a session/model/batch-specific location.

Disable log-path modification with:

```text
--log-path-key ""
```

### `--checkpoint`

```text
--checkpoint /home/avelasco/fantasia/fantasia_batches_checkpoint_v6_1_test.json
```

JSON state file containing:

- creation and update timestamps;
- session metadata;
- selected models and execution plan;
- current stage, model, and batch;
- per-run status;
- effective command;
- YAML changes;
- output paths;
- return codes;
- relocation errors;
- event history.

Writes are atomic: the runner writes a temporary file first and then replaces the checkpoint.

When restarted with a compatible checkpoint, runs already marked `done` are skipped.

### `--logs-dir`

```text
--logs-dir /home/avelasco/fantasia/fantasia_batch_runner_logs_v6_1_test
```

Directory for the runner's minimal logs. These logs primarily hold start time, end time, return code, and dry-run commands. Full FANTASIA output remains in FANTASIA's own `info.log` and `debug.log` files under the redirected `log_path`.

## Optional parameters not present in the TXT

### `--args-file`

Loads arguments from a shell-like text file. Quotes, line breaks, and `#` comments are supported.

### `--input-key-occurrence N`

Selects which occurrence of the input key to modify when it appears more than once.

### `--prefix-key-occurrence N`

Selects which occurrence of the prefix key to modify.

### `--log-path-key-occurrence N`

Selects which occurrence of the log-path key to modify.

### `--max-batches N`

Restricts discovery to the first `N` batches before applying `--batch-select`.

For example, `--max-batches 10 --batch-select 3-last` processes positions 3 through 10 of the restricted set.

### `--no-live-output`

Prevents FANTASIA output from being mirrored to the terminal. FANTASIA still writes its normal logs.

### `--dry-run`

Prepares the configuration, plan, checkpoint, and runner logs without launching FANTASIA or moving a real experiment directory.

### `--no-restore-config`

Prevents restoration of the original configuration. Without this option, the backup is restored in the runner's `finally` block.

## Execution order

The plan is model-major:

```text
for each selected model:
    for each selected batch:
        run FANTASIA
```

With the supplied parameters:

```text
ESM         × batches 3 through last
ESM3c       × batches 3 through last
Ankh3-Large × batches 3 through last
```

All selected ESM batches run first, followed by ESM3c and then Ankh3-Large.

## Checkpoint and restart behavior

Each run is identified as:

```text
<model>__batch_<NNNNN>
```

Per-run statuses:

```text
preparing
running
done
failed
```

Global stages include:

```text
INIT
PREPARE_<MODEL>_<BATCH>
RUNNING_<MODEL>_<BATCH>
FAILED_<MODEL>_<BATCH>
INTERRUPTED
ALL_DONE
```

A restart skips entries marked `done`. Use a new checkpoint if batch selection, directory contents, or ordering changes, because run labels use model name and discovered batch position rather than a content hash.

## YAML preservation strategy

The runner attempts to preserve the YAML byte for byte except for these values:

- `input`;
- `prefix`;
- `log_path`;
- model `enabled` flags.

Scalar keys are edited with regular expressions that preserve indentation and inline comments. The runner aborts if a key is ambiguous and no explicit occurrence is supplied.

## Error handling

- A non-zero FANTASIA return code marks the run `failed` and stops the plan.
- A relocation failure is recorded in `relocation_error`. In the current implementation, a zero FANTASIA return code still marks the run `done` even when relocation failed, so this field must be reviewed.
- `Ctrl+C` records `INTERRUPTED` and returns exit code 130.
- The configuration is restored in `finally` unless restoration is disabled.

## Operational recommendations

1. Start with `--dry-run`.
2. Confirm that the batch pattern discovers exactly the intended files.
3. Use zero-padded batch numbers.
4. Do not reuse checkpoints across incompatible plans.
5. Verify exact model names against the YAML.
6. Keep an independent copy of the configuration backup.
7. Run long jobs inside `screen` or `tmux`.
8. Review both `return_code` and `relocation_error` in the checkpoint.

## Example with `screen`

```bash
screen -S fantasia_batches

python3 run_fantasia_batches_with_checkpoint_REVISED_v6_2_1_monolithic.py   --args-file params_fantasia_models_v6_1_test.txt
```

Detach without stopping:

```text
Ctrl+A, D
```

Reconnect:

```bash
screen -r fantasia_batches
```
---

## Output structure

### Experiments

The YAML must define `base_directory`. The runner assumes that FANTASIA creates experiments under:

```text
<base_directory>/experiments
```

The runner then moves each experiment to:

```text
<base_directory>/experiments/
  <session_name>/
    <model_name>/
      <batch_stem>_<timestamp>/
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

### Full FANTASIA logs

The runner uses the original `log_path` as the root and redirects logs to:

```text
<log_path>/
  <session_name>/
    <model_name>/
      <batch_stem>_<timestamp>/
```

### Minimal runner logs

```text
<logs-dir>/fantasia_runner_<model>__batch_<NNNNN>.log
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
