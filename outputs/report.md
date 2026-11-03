# EduAssist AI - LLMOps Report: Prompt Versioning, Latency and Cost

## 1. LLMOps and how it differs from MLOps
MLOps manages datasets, feature pipelines, training and deployment of models. LLMOps manages the whole LLM application lifecycle, where prompts, retrieved context, the model or API version and output guardrails can each change independently and each affects quality. Cost also shifts from training compute to per-token API charges, and outputs are non-deterministic, so automated checks such as JSON validity replace simple accuracy metrics. For this reason prompts are treated as versioned production artifacts.

## 2. Method
I built an academic question-answering assistant that must return strict JSON with the keys answer, confidence (high, medium or low) and key_points. Using openai/gpt-oss-20b at temperature 0, I measured latency with a wall-clock timer around each API call and read input, output and total tokens from the response usage field. Cost per request is input tokens times the input price plus output tokens times the output price; the monthly projection multiplies the average request cost by an assumed 100,000 requests per month. Three prompts were registered as versions 1 to 3 of one MLflow prompt: V1 is a minimal instruction, V2 adds the explicit schema and rules, and V3 adds a worked example and forbids markdown fences. Each version answered the same 10 questions in its own MLflow run. The valid_json metric requires the raw reply to parse with json.loads; schema_ok additionally checks keys, types and allowed confidence values.

## 3. Results
| Version | valid_json | schema_ok | Avg latency (s) | Avg tokens in / out | Avg cost per request | Projected monthly cost |
|---|---|---|---|---|---|---|
| V1 | 30% | 0% | 0.94 | 94 / 399 | $0.000253 | $25.32 |
| V2 | 100% | 100% | 0.85 | 165 / 213 | $0.000152 | $15.24 |
| V3 | 100% | 100% | 2.27 | 233 / 214 | $0.000163 | $16.31 |

V2 achieved the best quality ranking (100% schema-valid, 100% valid JSON). The cheapest version was V2 and the fastest was V2. Longer prompts raise input tokens and therefore cost, while longer answers raise output tokens and latency, so the cheapest or fastest prompt is not automatically the best one.

## 4. Decision and promotion
Versions were ranked by schema_ok, then valid_json, then cost, then latency, because a malformed reply breaks downstream parsing and costs more than a few extra tokens. V2 (registry version 2) ranked first and was promoted by pointing the production alias at it. Applications load the prompt through the production alias, so a promotion or rollback is one alias change with no code deployment, and every evaluation run stays linked to the exact prompt version it tested.

## 5. Limitations and next steps
The test set has only 10 questions, so differences of one or two replies are not statistically strong, and latency depends on network conditions and provider load. Prices and the monthly volume are assumptions that should be checked against current provider pricing and real traffic. Next steps are a larger and more diverse evaluation set, repeated runs to measure variance, a semantic quality score such as LLM-as-judge or human review of answer correctness, and safety checks before any further promotion.
