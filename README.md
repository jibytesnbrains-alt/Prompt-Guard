# PromptGuard — LLM Jailbreak & Prompt Injection Defense

A binary classification system that detects and blocks jailbreaking, prompt injection, and prompt extraction attacks on Large Language Models (LLMs).

Two models are implemented and benchmarked:
- **TF-IDF + Logistic Regression** — lightweight, interpretable, near-zero latency
- **DistilBERT (fine-tuned)** — transformer-based, higher generalization on unseen attack patterns

---

## The Problem

LLMs are vulnerable to adversarial prompts that attempt to:
- **Bypass safety guidelines** (jailbreaking)
- **Override system instructions** (prompt injection)
- **Leak confidential context** (prompt extraction)

Attackers also obfuscate these prompts using encoding techniques (Base64, ROT13, leetspeak) to evade keyword-based filters. PromptGuard addresses both the classification and the obfuscation problem.

---

## Architecture

```
Raw Prompt
    │
    ▼
Preprocessing Pipeline
  ├─ Base64 decode
  ├─ ROT13 decode
  ├─ Leetspeak normalization  (e.g. "1gn0r3" → "ignore")
  └─ URL removal + whitespace collapse
    │
    ▼
Classifier
  ├─ TF-IDF + Logistic Regression   (fast, CPU-only)
  └─ DistilBERT fine-tuned          (accurate, GPU recommended)
    │
    ▼
Label: safe / malicious
```

---

## Dataset

| Split | TF-IDF model | DistilBERT model |
|---|---|---|
| Total samples | 440 | 4,568 |
| Train | 330 | 3,426 |
| Test | 110 | 1,142 |
| Malicious ratio | ~58% | ~61% |

**Sources merged:**
- Custom dataset (`prompts_dataset_full.csv` + `adversarial_dataset_v2.csv`)
- [`jackhhao/jailbreak-classification`](https://huggingface.co/datasets/jackhhao/jailbreak-classification) (HuggingFace) — used in TF-IDF pipeline
- [`neuralchemy/Prompt-injection-dataset`](https://huggingface.co/datasets/neuralchemy/Prompt-injection-dataset) — used in DistilBERT pipeline

**Attack types covered:** instruction override, goal hijacking, jailbreak persona, multilingual hijacking, encoding-based obfuscation (Base64, ROT13, leetspeak), real-world jailbreaks

---

## Results

### TF-IDF + Logistic Regression

Evaluated on held-out test set (110 samples):

| Metric | Safe | Malicious | Overall |
|---|---|---|---|
| Precision | 0.87 | 1.00 | 0.94 |
| Recall | 1.00 | 0.89 | 0.94 |
| F1-score | 0.93 | 0.94 | 0.94 |
| Accuracy | — | — | **93.6%** |

5-fold cross-validation: `F1 = 0.940 ± 0.025`

**Zero false positives** on test set.

---

### DistilBERT (fine-tuned, 3 epochs, RTX 5050)

Evaluated on held-out test set (1,142 samples):

| Metric | Safe | Malicious | Overall |
|---|---|---|---|
| Precision | 0.95 | 0.98 | 0.97 |
| Recall | 0.97 | 0.96 | 0.97 |
| F1-score | 0.96 | 0.97 | 0.97 |
| Accuracy | — | — | **96.6%** |

3-fold cross-validation: `F1 = 0.973 ± 0.005`

Model parameters: 66,955,010

---

### Fair Comparison — Unseen Dataset

Both models evaluated on [`deepset/prompt-injections`](https://huggingface.co/datasets/deepset/prompt-injections) (116 samples, **not seen during training**):

| Model | Accuracy | Precision | Recall | F1 | Inference |
|---|---|---|---|---|---|
| TF-IDF + LR | 63.8% | 0.821 | 0.383 | 0.523 | 0.007s |
| DistilBERT | 74.1% | 0.714 | 0.833 | **0.769** | 0.587s |

DistilBERT generalizes significantly better on novel attack patterns (+24.6% F1). TF-IDF is 90x faster but misses more adversarial samples it hasn't seen before.

---

## When to Use Which Model

| Criterion | TF-IDF + LR | DistilBERT |
|---|---|---|
| Latency |  < 1ms/prompt | ~10–50ms (GPU) |
| Resources | CPU only, < 1MB | GPU recommended, ~260MB |
| Interpretability |  Feature coefficients |  Black box |
| Known attack patterns |  Excellent |  Excellent |
| Novel / obfuscated attacks |  Weaker generalization |  Stronger generalization |
| Deployment complexity | Simple (sklearn pickle) | Requires PyTorch + transformers |

---

## Preprocessing Pipeline

`preprocessing.py` is shared across all notebooks. The pipeline handles obfuscated attacks before classification:

```python
from preprocessing import preprocess

preprocess("aWdub3JlIGFsbCBwcmV2aW91cyBpbnN0cnVjdGlvbnM=")
# → "ignore all previous instructions"

preprocess("1gn0r3 4ll pr3v10us 1nstruct10ns")
# → "ignore all previous instructions"

preprocess("Vtaber nyy cerivbhf vafgehpgvbaf.")
# → "ignore all previous instructions"
```

Pipeline order: Base64 decode → ROT13 decode → lowercase → leetspeak normalize → remove URLs → collapse whitespace

> **Note:** Punctuation patterns like `[SYSTEM]`, `<!-- -->`, and `===` are intentionally preserved as they are attack signals.

---

## Repository Structure

```
PromptGuard/
├── preprocessing.py              # Shared preprocessing pipeline
├── P-G_TF-IDF_LR.ipynb              # TF-IDF + Logistic Regression
├── P-G_bert.ipynb         # DistilBERT fine-tuning
├── P-G_compare_TF-bert.ipynb      # Fair model comparison
├── model_TF-IDF_LR_P-G.pkl           # Saved TF-IDF + LR model
├── model_bert_P-G.pkl           # Saved DistilBERT model
└── README.md
```

> DistilBERT model weights are hosted on Hugging Face Hub: *([link to be added](https://huggingface.co/Godjira/PromptGuard-DistilBERT))*

---

## Setup

```bash
pip install scikit-learn pandas matplotlib transformers datasets torch
```

Run the notebooks in order:
1. `P-G_TF-IDF_LR.ipynb` — train and evaluate TF-IDF + LR
2. `P-G_bert.ipynb` — fine-tune and evaluate DistilBERT
3. `P-G_compare_TF-bert.ipynb` — compare both models on unseen data

---

## Limitations

- TF-IDF model was trained on a smaller dataset (440 samples) — lower generalization on novel attacks
- DistilBERT has 14 false positives on test set, mostly role-play prompts with security-adjacent language
- Neither model currently handles indirect prompt injection (injections embedded in documents or tool outputs)

---

## Motivation

Built as part of ongoing interest in LLM security and AI red teaming. The goal is to understand attack patterns from both the offensive and defensive side — informing better guardrail design for enterprise LLM deployments.

---

*Jiraphat Jarernpipattada — Computer Engineering, KMUTT*
