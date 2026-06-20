# CVSBench Evaluation Scripts

This folder contains public-facing evaluation scripts for CVSBench-style tasks, including:

- single-image and multi-image VQA / MCQ evaluation
- cross-view grounding / bbox evaluation
- category-aware result summarization
- two-image ablation evaluation for extra visual inputs

The scripts are adapted from internal research code and cleaned so they can be shared in a public GitHub repository.

## Files

- `eval.py`
  Main evaluation entrypoint. Supports both VQA-style tasks and bbox / grounding tasks.
- `eval_double_category.py`
  Two-image evaluation script for FOV extra-input experiments such as depth, z-image, or nanobanana.
- `summarize_results.py`
  Recomputes and formats summary metrics from prediction files, with category-first grouping.
- `eval_config.example.json`
  Public example config with placeholder model settings.
- `requirements.txt`
  Minimal Python dependencies for these scripts.

## Supported task types in `eval.py`

The main evaluator dispatches by `dataset["type"]`:

- `mcq_vqa`
  Generic MCQ VQA format from older scripts.
- `g2s`
  Ground-to-satellite VQA.
- `s2s`
  Satellite-view VQA. In this codebase this is also used for many `s2g` question files.
- `s2g`
  Accepted in some helper paths and image pickers.
- `ge_view`
  FOV `gs_view` matching tasks.
- `gs_grounding`
  Cross-view grounding / bounding-box localization.
- `bbox_5level`
  Older multi-level bbox format.
- `arrow_5level`, `arrow_mcq`
  Older arrow-based tasks kept for compatibility.

## Input assumptions

### 1. VQA / MCQ

For `g2s`, `s2s`, `s2g`, and `ge_view`, each jsonl sample should contain the question, answer, options, and enough image fields for the script to resolve the correct image path.

The script stores the following category metadata directly into prediction files when present:

- `category`
- `category_l1`
- `category_l2`

This makes later summarization more reliable.

### 2. Grounding / bbox

For `gs_grounding`, the public version assumes a two-image setup whenever possible:

- image 1: source image
- image 2: target image
- output: bbox in the second image

For level-3 style grounding tasks, the first image may already contain the source bbox.

The script also supports these fields when available:

- `source_image`
- `target_image`
- `source_bbox`
- `target_bbox`
- `answer`
- `answer_bbox`
- `gt_bbox`
- `region_hint`

Combined-image fallback logic is still kept for backward compatibility, but the recommended public format is separate source and target images.

## Running `eval.py`

### Option A: OpenAI-compatible API

1. Copy `eval_config.example.json` to your own config file.
2. Set your model information in the config.
3. Run:

```bash
python eval.py --config eval_config.json
```

You can also use environment variables:

```bash
export OPENAI_API_KEY=your_key
export OPENAI_BASE_URL=http://localhost:8000/v1
export EVAL_MODEL=your-model-name
python eval.py --config eval_config.json
```

This works for remote APIs or locally deployed OpenAI-compatible servers such as vLLM or similar serving stacks.

### Option B: Local Transformers model

The current public local path supports:

- `LOCAL_MODEL_FAMILY=qwen3vl`
- `LOCAL_MODEL_FAMILY=gemma3`

Example:

```bash
export LOCAL_TRANSFORMERS=1
export LOCAL_MODEL_FAMILY=qwen3vl
export LOCAL_MODEL_PATH=/path/to/your/local/model
python eval.py --config eval_config.json
```

If you want to use another local model family, add a new adapter in `eval.py` or serve it behind an OpenAI-compatible API.

## Running `eval_double_category.py`

This script is for two-image evaluation with an extra modality or generated view.

Supported `--extra-kind` values:

- `depth`
- `zimage`
- `nanobanana`

Example:

```bash
python eval_double_category.py \
  --base-dir . \
  --extra-kind nanobanana \
  --local-model-path /path/to/Qwen3-VL \
  --g2s-path fov/g2s/Ground2Sat_VQA_test.jsonl \
  --s2g-path fov/s2g/Sat2Ground_VQA_test.jsonl
```

This script writes:

- per-split prediction files
- missing-pair diagnostics
- per-category tables
- `summary.json`

All paths are exposed as CLI arguments so the script can be moved to a different machine without editing absolute paths.

## Summarizing results

After evaluation, run:

```bash
python summarize_results.py --root outputs
```

The summarizer now prefers category fields saved directly in predictions:

1. `category_l1` / `category_l2`
2. `category`
3. dataset qid mapping
4. option-signature fallback

This is intentional so that public prediction files remain self-contained.

## Output structure

By default, results are written under:

```text
outputs/
  model_name/
    dataset_name/
      predictions.jsonl
      metrics.json
    summary.json
```

`eval_double_category.py` also writes category tables and generated split files.

## What to edit before GitHub release

Before publishing, you should still check:

- dataset paths in your config
- whether your released jsonl files use `s2s` or `s2g` as the satellite-to-ground type label
- whether you want to keep backward-compatible old task types in `eval.py`
- whether you want to add support for more local model families

## Notes

- These scripts do not assume a specific commercial API provider.
- Private endpoints, secrets, and internal notification hooks have been removed.
- The recommended public setup is either:
  - an OpenAI-compatible model server
  - or a local Hugging Face model with a supported adapter
