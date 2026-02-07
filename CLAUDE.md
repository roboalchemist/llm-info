# llm-info

Structured JSON registry of AI model cards and benchmarks.

## Repo Structure

```
schema/
  model-card.schema.json    # JSON Schema for model cards
  benchmark.schema.json     # JSON Schema for benchmarks
models/                     # One JSON file per model (24 currently)
benchmarks/                 # One JSON file per benchmark (62 currently)
```

## Adding a New Model

1. **Research the model** using official sources in priority order:
   - Official announcement blog post
   - Technical report / arXiv paper
   - System card / model card (if published)
   - HuggingFace model page (for open-weight models)
   - Third-party aggregators: artificialanalysis.ai, llm-stats.com, automatio.ai, vellum.ai

2. **Create `models/{id}.json`** following `schema/model-card.schema.json`. Required fields:
   - `schema_version`: always `"1.0.0"`
   - `id`: lowercase slug (e.g. `claude-opus-4-6`, `gpt-5-2-thinking`, `kimi-k2-5`)
   - `name`: human-readable (e.g. `"Claude Opus 4.6"`)
   - `provider`: `{ "name", "url", "country" }`
   - `release_date`: `YYYY-MM-DD`

3. **Fill in what you can find**. Don't fabricate data. Common fields:
   - `sources`: blog_post, technical_report, system_card, huggingface, api_docs
   - `architecture`: type (dense/moe), family, total_params, context_length, modalities
   - `license`: spdx_id or name, open_weights, commercial_use
   - `pricing`: input_per_mtok, output_per_mtok (USD per million tokens)
   - `benchmarks`: array of benchmark results (see below)
   - `meta`: created_at, updated_at (ISO 8601)

## Adding a New Benchmark

1. **Research the benchmark**:
   - Find the official paper (arXiv), website/leaderboard, and GitHub repo
   - Determine: category, number of problems, primary metric, saturation status
   - Check which labs report it in their model announcements

2. **Create `benchmarks/{id}.json`** following `schema/benchmark.schema.json`. Required fields:
   - `schema_version`: `"1.0.0"`
   - `id`: lowercase slug (e.g. `gpqa-diamond`, `swe-bench-verified`)
   - `name`: display name (e.g. `"GPQA Diamond"`)
   - `category`: one of `coding`, `coding-agent`, `coding-competitive`, `coding-security`, `coding-quality`, `coding-multilingual`, `reasoning`, `knowledge`, `multimodal`, `safety`, `agent`, `long_context`, `multilingual`, `other`

3. **Important metadata**:
   - `tier`: 1 = gold standard (reported by 3+ major labs), 2 = important/emerging, 3 = specialized/niche
   - `saturation`: `saturated` (>95%), `approaching` (>90%), `not_saturated`, `unknown`
   - `contamination_resistance`: `high` (private test set / frequent refresh), `medium`, `low`
   - `adoption`: `universal` (all labs), `high` (most), `medium`, `emerging`, `niche`
   - `reported_by`: array of lab slugs (e.g. `["anthropic", "openai", "google"]`)

## Adding Benchmark Scores to Models

Each model's `benchmarks` array contains entries like:

```json
{
  "name": "GPQA Diamond",
  "category": "reasoning",
  "score": 83.4,
  "score_unit": "percent",
  "methodology": {
    "notes": "0-shot CoT"
  },
  "source_url": "https://automatio.ai/models/claude-sonnet-4-5"
}
```

### Where to Find Scores

Priority order for benchmark data:

1. **Official model announcements** (anthropic.com/news, openai.com/index, blog.google/technology/ai)
2. **Technical reports** (arXiv papers with evaluation sections)
3. **System cards** (detailed eval appendices)
4. **HuggingFace model cards** (especially for Chinese/open models)
5. **Third-party leaderboards**:
   - artificialanalysis.ai - comprehensive multi-benchmark comparisons
   - llm-stats.com - aggregated benchmark data
   - automatio.ai - model comparison pages
   - vellum.ai/blog - benchmark deep-dives per model launch
   - Scale AI leaderboards (for SWE-bench, SEAL)

### Score Entry Rules

- **Always include `source_url`** pointing to where you found the number
- **Use `methodology.notes`** for important context: "0-shot CoT", "with extended thinking", "with tools", "pass@1"
- **Don't mix metric types**: if a benchmark reports Elo (e.g. Chatbot Arena), use `"score_unit": "elo"`, not percent
- **Don't duplicate**: check the model's existing benchmarks array before adding
- **Preserve existing data**: never remove or modify existing benchmark entries when adding new ones

## Validation

Validate JSON files against schemas:

```bash
# Validate all models
for f in models/*.json; do python3 -m json.tool "$f" > /dev/null && echo "OK: $f" || echo "FAIL: $f"; done

# Validate all benchmarks
for f in benchmarks/*.json; do python3 -m json.tool "$f" > /dev/null && echo "OK: $f" || echo "FAIL: $f"; done

# Count benchmarks per model
for f in models/*.json; do
  count=$(python3 -c "import json; print(len(json.load(open('$f'))['benchmarks']))")
  echo "$(basename $f): $count benchmarks"
done
```

## Naming Conventions

- **Model IDs**: `{provider-model-version}` in lowercase, dots become dashes (e.g. `gemini-2-5-pro`, `deepseek-v3-2`)
- **Benchmark IDs**: lowercase slug of the benchmark name (e.g. `swe-bench-verified`, `gpqa-diamond`, `aime-2025`)
- **Benchmark names in model files must match** the `name` field in the benchmark JSON (e.g. `"GPQA Diamond"` not `"GPQA-Diamond"`)

## Workflow for Bulk Research

When backfilling scores across many models:

1. **Research phase**: Run parallel searches per model family (Anthropic, OpenAI, Google, Chinese labs). Check official blog posts, arXiv papers, and aggregator sites.
2. **Update phase**: Edit each model JSON, adding new entries to the `benchmarks` array. Update `meta.updated_at`.
3. **Validate**: Run JSON validation on all modified files.
4. **Commit**: One commit per logical batch (e.g. "Backfill benchmark scores across all 24 model cards").
