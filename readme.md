# MINNOW: Multi-Instance Network of Narrow Open-Weight Models

**MINNOW** is a small ablation study that tests whether a Fugu/Fusion-style orchestration pattern can scale down to smaller open-weight models.

Instead of using a single larger model, MINNOW coordinates multiple narrow 4-bit GGUF models through lightweight scaffolds such as:

- direct answering
- direct answering with a finalizer
- solver → verifier → finalizer
- adaptive scaffold selection

The current experiment is focused only on **GSM8K mathematical reasoning**. It is **not production-ready**, **not optimized**, and **not validated for real-world deployment**. The goal is to understand whether small-model orchestration can approach larger-model reasoning performance under constrained deployment conditions.

---

## Latest Result First: 100-Question GSM8K Ablation

The latest run evaluates all small-model configurations on the same **100 GSM8K test questions** and compares them with a separate **14B direct baseline** on the same 100-question setup.

### Summary


| Configuration                   | Accuracy | Correct / 100 | Avg time / question | Total time   | Est. cost / answer    | 100-question cost   | Avg peak VRAM |
| ------------------------------- | -------- | ------------- | ------------------- | ------------ | --------------------- | ------------------- | ------------- |
| Small direct                    | 78%      | 78            | 12.9s               | 21.5 min     | $0.00126 / ₹0.119     | $0.126 / ₹11.94     | ~9.05 GB*     |
| Small direct + finalizer        | 88%      | 88            | 14.0s               | 23.3 min     | $0.00136 / ₹0.129     | $0.136 / ₹12.95     | ~9.08 GB*     |
| Solver → Verifier → Finalizer   | 90%      | 90            | 19.2s               | 32.0 min     | $0.00186 / ₹0.177     | $0.186 / ₹17.73     | ~9.11 GB      |
| **MINNOW adaptive scaffold v2** | **93%**  | **93**        | **21.9s**           | **36.6 min** | **$0.00213 / ₹0.203** | **$0.213 / ₹20.29** | **~9.12 GB**  |
| 14B direct baseline             | 95%      | 95            | 63.1s               | 105.2 min    | $0.00614 / ₹0.584     | $0.614 / ₹58.37     | ~9.46 GB      |


 The small direct VRAM number is inflated because the full small-model pool was kept resident in memory during the resident experiment. A true single-small-model direct setup would only need one 4-bit GGUF model, closer to roughly **~2.9 GB VRAM**.

### Main Takeaway

The 14B model still has the highest accuracy: **95%**.

MINNOW adaptive scaffold v2 reaches **93%**, within 2 percentage points of the 14B baseline, while being approximately:

- **2.9× faster** on the 100-question run
- **65% cheaper per answer** under the same GPU cost assumption
- more modular, since the system is composed of separate smaller models

This does **not** prove that small-model orchestration is generally better than larger models. It suggests that, for verifiable reasoning tasks like GSM8K, a resident network of narrow open-weight models can approach larger-model performance while improving runtime and estimated inference cost.

---

## Motivation

This project started from interest in systems such as **Sakana Fugu Ultra** and **OpenRouter Fusion API**, both of which point toward a broader trend: using orchestration over multiple models rather than relying only on one monolithic model.

The question behind MINNOW is:

> Can the same model-orchestration idea scale down to small open-weight models?

More specifically:

> Can multiple narrow 4-bit GGUF models, loaded locally or semi-resident in GPU memory, approach a larger 14B model on reasoning while being cheaper to run?

---

## What MINNOW Is

MINNOW is a test-time orchestration experiment.

It uses multiple small GGUF models in different roles:


| Role         | Model used in latest script                | Purpose                                |
| ------------ | ------------------------------------------ | -------------------------------------- |
| Solver       | `unsloth/Phi-4-mini-reasoning-GGUF`        | Primary mathematical reasoning         |
| Verifier     | `unsloth/Qwen3-4B-Instruct-2507-GGUF`      | Checks or recomputes the solver answer |
| Finalizer    | `unsloth/Phi-4-mini-instruct-GGUF`         | Produces a clean `FINAL_ANSWER`        |
| Router       | `unsloth/Qwen3-4B-Instruct-2507-GGUF`      | Chooses a scaffold / workflow          |
| 14B baseline | `bartowski/microsoft_Phi-4-reasoning-GGUF` | Larger single-model comparison         |


All models are used in **4-bit GGUF** form, loaded through `llama-cpp-python`.

---

## What MINNOW Is Not

MINNOW is not a trained orchestrator yet.

It is not a direct reproduction of Fugu Ultra.

It is not a production inference stack.

It is not benchmarked across broad real-world tasks yet.

It is currently a **small, controlled ablation study** designed to test whether the architectural idea is promising enough to explore further.

---

## Architectures Tested

### 1. Small Direct

A single small reasoning model answers the GSM8K question directly.

```text
question → Phi-4-mini-reasoning → numeric answer
```

### 2. Small Direct + Finalizer

The small reasoning model produces a solution, then a small instruction model extracts or formats the final answer.

```text
question → Phi-4-mini-reasoning → Phi-4-mini-instruct finalizer → FINAL_ANSWER
```

This ablation tests how much improvement comes from cleaner answer extraction rather than better reasoning.

### 3. Solver → Verifier → Finalizer

The primary scaffold separates reasoning, checking, and final formatting.

```text
question
  → solver model
  → verifier model
  → finalizer model
  → FINAL_ANSWER
```

The verifier is part of the inference system. It is **not** used as the benchmark judge.

### 4. Adaptive Scaffold v2

The router first proposes a workflow, then heuristic safety checks decide whether the system can use direct-finalized inference or should fall back to solver-verifier-finalizer.

```text
question
  → router
  → direct_finalized OR solver_verifier
  → guardrail override if needed
  → final answer
```

In the latest run, the adaptive scaffold was intentionally conservative. The router proposed `direct_finalized` on some examples, but guardrails often escalated to solver-verifier when the question contained multi-step or arithmetic-risk signals.

### 5. 14B Direct Baseline

A larger 14B model answers directly.

```text
question → Phi-4-reasoning-14B → numeric answer
```

This is used as the larger single-model baseline.

---

## Experiment History

### Smoke Test: 10 GSM8K Questions

Initial run with lower token caps and lazy loading.


| Configuration                 | Accuracy |
| ----------------------------- | -------- |
| Small direct                  | 20%      |
| Solver → Verifier → Finalizer | 70%      |
| Adaptive scaffold             | 60%      |


This run mainly revealed that the direct model was being heavily hurt by truncation and answer-formatting issues.

### 100-Question Lazy-Loading Run

With the same early scaffold but 100 questions:


| Configuration                 | Accuracy |
| ----------------------------- | -------- |
| Small direct                  | 32%      |
| Solver → Verifier → Finalizer | 86%      |
| Adaptive scaffold             | 67%      |
| 14B direct baseline           | 85%      |


This was promising, but still confounded by token caps and weak direct-answer extraction.

### Latest Resident + Higher Token Cap Run

The latest run increased token caps, added `small_direct_finalized`, and tested resident loading of the small model pool.


| Configuration                   | Accuracy |
| ------------------------------- | -------- |
| Small direct                    | 78%      |
| Small direct + finalizer        | 88%      |
| Solver → Verifier → Finalizer   | 90%      |
| **MINNOW adaptive scaffold v2** | **93%**  |
| 14B direct baseline             | 95%      |


This is the primary result of the repo.

---

## Cost Assumptions

The cost numbers are estimated from measured wall-clock time using:

```text
GPU hourly cost = $0.35/hour
USD to INR = ~₹95.10 per $1
```

The cost estimate is not a cloud-provider bill. It is a normalized proxy to compare runtime cost under the same assumed GPU hourly rate.

Formula:

```text
estimated_cost = wall_time_seconds / 3600 * GPU_HOURLY_COST_USD
```

---

## Why GSM8K?

GSM8K was chosen as the first benchmark because:

- it is simple to run locally
- it has deterministic numeric gold answer
- it directly tests arithmetic reasoning
- solver-verifier-finalizer scaffolds are naturally suited to verifiable reasoning tasks

This benchmark does not prove real-world deployment readiness. It is a controlled first test of the orchestration idea.

---

## Evaluation Method

The evaluation pipeline is:

```text
model output
  → parse FINAL_ANSWER if available
  → fall back to numeric extraction if needed
  → compare extracted number to GSM8K gold answer
```

The final score is deterministic numeric matching.

Important distinction:

- The verifier model is an internal inference component.
- The verifier is not the judge of the benchmark.
- The benchmark score does not use LLM-as-a-judge.

---

## Running the Experiment

### 1. Install dependencies

For Colab or a CUDA environment, install a prebuilt `llama-cpp-python` CUDA wheel where possible.

```bash
pip install -U pip setuptools wheel
pip install -U datasets huggingface_hub pandas tqdm psutil pynvml
pip install -U --prefer-binary --only-binary=:all: llama-cpp-python \
  --extra-index-url https://abetlen.github.io/llama-cpp-python/whl/cu124
```

If CUDA 12.4 wheels do not work in your runtime, try the matching wheel index for your CUDA version, for example `cu121`.

### 2. Run the script

```bash
python fugu_lite_gsm8k_gguf_resident_adaptive_v2.py
```

The script will:

1. download the required GGUF files from Hugging Face
2. load the GSM8K test split
3. sample 100 questions using the configured seed
4. run the selected ablations
5. save per-mode summaries and detailed JSONL outputs

### 3. Important configuration values

Inside the script:

```python
N_SAMPLES = 100
SEED = 42
N_CTX = 2048
TEMPERATURE = 0.0
MAX_TOKENS_DIRECT = 768
MAX_TOKENS_SOLVER = 768
MAX_TOKENS_VERIFIER = 512
MAX_TOKENS_FINALIZER = 64
MAX_TOKENS_ROUTER = 160
MAX_TOKENS_LARGE_BASELINE = 768
PRELOAD_SMALL_MODELS_IN_GPU = True
GPU_HOURLY_COST_USD = 0.35
```

To run the larger baseline, enable:

```python
RUN_LARGE_BASELINE = True
```

If the 14B model fails to load after resident small models, release resident models or run the 14B baseline in a fresh runtime.

---

## Output Files

The experiment produces files such as:

```text
aggregate_summary.csv
small_direct_summary.csv
small_direct_details.jsonl
small_direct_finalized_summary.csv
small_direct_finalized_details.jsonl
solver_verifier_summary.csv
solver_verifier_details.jsonl
adaptive_scaffold_v2_summary.csv
adaptive_scaffold_v2_details.jsonl
large_14b_direct_summary.csv
large_14b_direct_details.jsonl
```

The summary CSVs contain per-question metrics such as correctness, wall time, model calls, token counts, peak VRAM, and chosen scaffold.

The JSONL files contain detailed outputs and are useful for error analysis.

---

## Key Observations

### 1. Token caps matter

Earlier low-token runs made the direct small model look much weaker than it actually was. Increasing the token caps improved small direct accuracy from **32%** to **78%**.

### 2. Finalization matters

Adding a finalizer improved small direct accuracy from **78%** to **88%**. This shows that answer formatting and extraction reliability are important confounders in reasoning benchmarks.

### 3. Verification helps, but less than finalization

The solver-verifier-finalizer scaffold improved from **88%** to **90%** over direct-finalized inference.

### 4. Adaptive v2 is conservative but effective

Adaptive scaffold v2 reached **93%**, but it should not yet be interpreted as a fully learned router. It is a prompt-based router with heuristic guardrails.

### 5. The 14B model still wins on accuracy

The 14B baseline reached **95%**. MINNOW did not beat it outright, but came close with lower runtime and estimated cost.

---

## Limitations

- Only 100 GSM8K questions were used in the latest result.
- Only one benchmark family has been tested.
- The adaptive router is not trained.
- The system is not optimized for production serving.
- The 14B baseline may be further optimizable with better stopping behavior and prompting.
- The current cost estimates are proxy estimates based on wall time, not real cloud billing.
- Results may vary by GPU, llama.cpp build, CUDA runtime, quantization file, and prompt template.

---

## Next Steps

Planned or recommended follow-up experiments:

- run multiple 100-question samples with different seeds
- run the full GSM8K test set
- add an optimized 14B direct baseline
- compare against a 14B direct + finalizer baseline
- test MCQ benchmarks such as ARC-Challenge or MMLU-Pro as parsing controls
- test code or tool-use benchmarks where scaffolding may matter more
- replace heuristic routing with a learned router
- add self-consistency baselines to separate orchestration gains from pure test-time compute gains

---

## Project Status

MINNOW is currently a research prototype and ablation study.

The current result is best summarized as:

> On a 100-question GSM8K subset, MINNOW adaptive scaffold v2 reached 93% accuracy compared with 95% for a 14B direct baseline, while reducing average runtime from 63.1s to 21.9s per question and estimated cost from $0.614 to $0.213 for the 100-question run.

This suggests that small-model orchestration may be a promising direction for constrained, verifiable reasoning workloads.
