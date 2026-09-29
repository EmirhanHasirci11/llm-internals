# llm-internals

How much more does Turkish cost than English in LLM tokens? Measured on a parallel corpus, so both languages carry the same content.

## Results

OPUS-100 en-tr, test split, 2000 sentence pairs. `tr/en` = total Turkish tokens / total English tokens. Sorted by `tr/en`.

| Model / tokenizer | Source | en tokens | tr tokens | tr/en | Notes |
|---|---|---:|---:|---:|---|
| `dbmdz/bert-base-turkish-cased` | local | 33,578 | 18,814 | 0.560 | Turkish-trained vocabulary |
| `gemini-3.1-pro-preview` | API | 24,302 | 26,099 | 1.074 | `\n`-joined |
| `meta-llama/Llama-3.1-8B-Instruct` | local | 21,567 | 25,141 | 1.166 | vocab 128,256 |
| `Qwen/Qwen3.8-27B` | local | 21,623 | 25,687 | 1.188 | vocab 248,077 |
| `o200k_base` (GPT-4o family) | local | 20,666 | 25,146 | 1.217 | |
| `o200k_base`, `\n`-joined (calibration) | local | 20,734 | 25,196 | 1.215 | same join the API rows use |
| `claude-opus-5-5` | API | 32,239 | 46,089 | 1.430 | `\n`-joined, includes message framing |
| `claude-sonnet-5-5` | API | 32,239 | 46,089 | 1.430 | `\n`-joined, includes message framing |
| `cl100k_base` (GPT-4) | local | 21,578 | 32,115 | 1.488 | |
| `claude-sonnet-4-6` (older tokenizer) | API | 24,835 | 37,546 | 1.512 | `\n`-joined, includes message framing |
| `deepseek-ai/DeepSeek-V3` | local | 21,623 | 33,457 | 1.547 | vocab 128,815 |
| `google/gemma-3-4b-it` | local | not measured | not measured | | gated repo, access not granted to this Hugging Face account |
| GPT-6 | | not measured | not measured | | tokenizer not published |

Local rows are per-sentence sums. API rows are one `\n`-joined request per language.

Notebooks:

- [`tokenization/turkish-fertility.ipynb`](tokenization/turkish-fertility.ipynb): `o200k_base`, `cl100k_base`, `dbmdz/bert-base-turkish-cased`, plus token-level examples.
- [`tokenization/cross-model-comparison.ipynb`](tokenization/cross-model-comparison.ipynb): Qwen, DeepSeek, Llama, Gemma, and Claude and Gemini via their token counting endpoints.

## Method

**Parallel corpus.** Every English sentence has a Turkish translation of the same content. Comparing token totals on the same content isolates the tokenizer's effect from differences in what the texts say.

**Total / total ratio.** The headline number is total Turkish tokens divided by total English tokens over all 2000 pairs. The mean of per-sentence ratios is a worse summary on short sentences: with `o200k_base` the mean per-sentence ratio is 1.280 and the median is 1.231, both above the total/total 1.217, because a 3-token vs 5-token sentence counts as much as a 30-token one.

**Why tokens per word is misleading.** Turkish is agglutinative: suffixes that would be separate English words attach to one Turkish word. The same content took 11,249 words in Turkish and 15,979 in English, 30% fewer. So tokens per word for Turkish (2.235 on `o200k_base`) looks 1.73x worse than English (1.293), but the actual cost for the same content is 1.22x. Tokens per word divides by a unit that means different things in the two languages.

**Local vs API.** Local tokenizers are applied per sentence (no special tokens) and summed. API token counters are called once per language with all 2000 sentences joined by `"\n"`, to keep the number of requests at two per model. The calibration row measures that same join locally with `o200k_base`: 1.215 vs 1.217 per-sentence, so the join moves the ratio by 0.002. Claude's `messages.count_tokens` counts the whole user message, so its numbers include a constant number of message framing tokens per request.

Only token counting and model listing endpoints are used. Model IDs for Claude and Gemini are taken from each provider's `models.list()`, not hard-coded: newest Opus, newest Sonnet, and `claude-sonnet-4-6`; for Gemini, the highest-versioned Pro model that supports `countTokens`.

## Findings

- **A low ratio does not mean cheap Turkish.** Gemini, Llama and Qwen all beat `o200k_base` on `tr/en`, but mostly because English got more expensive, not Turkish cheaper. Against the joined `o200k_base` calibration (same method), Gemini uses 17.2% more English tokens and 3.6% more Turkish tokens. Against per-sentence `o200k_base`, Llama uses 4.4% more English tokens and the same number of Turkish tokens (25,141 vs 25,146), and Qwen uses 4.6% more English and 2.2% more Turkish.
- **Claude's newer tokenizer costs more in both languages.** `claude-opus-5-5` and `claude-sonnet-5-5` return identical counts. Compared with `claude-sonnet-4-6`, they use 29.8% more English tokens and 22.8% more Turkish tokens. The ratio improves (1.512 to 1.430) only because English grew faster.
- **Claude has the highest absolute Turkish count measured.** Against the joined `o200k_base` calibration, the 5.5 models use 55.5% more English tokens and 82.9% more Turkish tokens; `claude-sonnet-4-6` uses 19.8% and 49.0% more.
- `DeepSeek-V3` has the highest `tr/en` (1.547). Its English total equals Qwen's exactly (21,623). This is a coincidence: 264 sentences get different counts and the differences cancel. On Turkish, DeepSeek-V3 needs 30% more tokens than Qwen.
- Moving from `cl100k_base` to `o200k_base` cut Turkish tokens by 22% and English tokens by 4%.
- A Turkish-trained vocabulary (`dbmdz/bert-base-turkish-cased`) reverses the picture: 0.560, with 25% fewer Turkish tokens than `o200k_base`.

## Limitations

- **Short sentences.** OPUS-100 test pairs are short, conversational, subtitle-like lines (about 8 English words on average). Ratios for long formal documents, code, or domain text may differ.
- **Token counts are not prices.** Providers charge different amounts per token, and API counts can include message framing tokens. Neither the ratio nor the absolute counts say which model is cheaper for a given workload.
- **Claude tokenizer change.** Claude models from 4.7 onward use a new tokenizer; `claude-sonnet-4-6` is included as an older-tokenizer reference. Only `claude-opus-5-5`, `claude-sonnet-5-5` and `claude-sonnet-4-6` were measured.
- **One model per family.** One Gemini model (`gemini-3.1-pro-preview`), one Qwen model (`Qwen3.8-27B`) and one Llama model (`Llama-3.1-8B-Instruct`) were measured. Other releases in the same family may use a different tokenizer.
- **GPT-6** is not measured because its tokenizer has not been published.
- **Gemma** is not measured: it is a gated repo and access has not been granted to the Hugging Face account used here.

## Reproduce

```powershell
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt anthropic google-genai python-dotenv
```

Optional, to enable the API and gated rows: create a `.env` file in the repo root (it is git-ignored) with `ANTHROPIC_API_KEY`, `GEMINI_API_KEY` and `HF_TOKEN`. If your Anthropic key is not scoped to a workspace, also set `ANTHROPIC_WORKSPACE_ID`. Empty values are treated as not set. Then run both notebooks top to bottom.
