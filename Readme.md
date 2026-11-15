# Experimental Artifacts and Evaluation Workflow

This repository contains the code and fixed experimental artifacts for studying how language models respond to misleading information in retrieved context in Retrieval-Augmented Generation (RAG).

The materials support two complementary activities:

1. **Inspecting and verifying the recorded experimental results** using the supplied outputs.
2. **Rerunning the experimental workflow**, subject to access to the external model APIs.

## Overview

RAG systems provide language models with external evidence. Retrieved passages may, however, contain misleading or intentionally contradictory information. The experiments examine two related questions:

- **Detection:** does the model identify misleading information in the retrieved context?
- **Influence:** does that information nevertheless affect the model's final answer?

The evaluation reports two complementary measures of contextual influence:

- **Cordon Rate (CR):** the probability that the model detects the poisoned information and nevertheless produces a poison-aligned answer, conditioned on its no-RAG answer not already being poison-aligned.
- **Leakage Rate (LR):** the probability that the model produces a poison-aligned answer despite an explicit instruction to ignore the retrieved context, when its no-RAG answer is not poison-aligned.

The evaluation also reports Attack Success Rate (ASR), Poison Detection Rate (PDR), and Conditional Cordon Rate (CCR). These measures capture different aspects of model behavior and should be interpreted together.

## Main Results

The main experiment uses **200 SciFact questions**. The table reports the marginal rates from the manuscript. CR and ASR are calculated over the eligible questions for each configuration; PDR and LR use all 200 questions.

| Model | Reasoning | ASR | PDR | CR | CCR | LR |
|---|---:|---:|---:|---:|---:|---:|
| DeepSeek-V4-Flash | Off | 0.233 | 0.965 | 0.205 | 0.211 | 0.235 |
| DeepSeek-V4-Flash | On | 0.298 | 0.665 | 0.075 | 0.107 | 0.140 |
| Qwen3.6-Plus | Off | 0.135 | 0.830 | 0.049 | 0.058 | 0.240 |
| Qwen3.6-Plus | On | 0.279 | 0.500 | 0.006 | 0.011 | 0.170 |

The CR counts and eligible denominators are:

| Model | Reasoning Off | Reasoning On |
|---|---:|---:|
| DeepSeek-V4-Flash | 30/146 | 12/161 |
| Qwen3.6-Plus | 8/163 | 1/172 |

Enabling reasoning is associated with lower CR and LR in both models. PDR also decreases in both models. ASR increases numerically in both models, but the paired analysis finds a statistically significant increase only for Qwen3.6-Plus. For DeepSeek-V4-Flash, the paired ASR change is not statistically significant.

The paired tests use the same questions eligible under both reasoning configurations (135 for DeepSeek-V4-Flash and 157 for Qwen3.6-Plus). After Holm correction across the eight paired tests (LR, CR, PDR, and ASR for each model), the CR and LR reductions for DeepSeek-V4-Flash remain statistically significant. The Qwen3.6-Plus LR reduction is not significant after correction, and its CR comparison is not statistically significant.

These results should not be interpreted as showing that reasoning eliminates susceptibility to poisoned context. In particular, CR can decrease partly because detection itself becomes less frequent; CCR is reported to help describe influence among detected cases, although the detected subsets differ across configurations.

---

## Repository Structure

```text
.
├── src/
│   ├── Poison_generation.ipynb   # Poison-passage generation
│   └── Eval.ipynb                # Evaluation and metric computation
├── poisons/                      # Poison passages used in the experiments
├── evaluation/                   # Model responses and evaluation artifacts
│                                 # in XLSX format
└── README.md
```

### `src/`

Contains the Jupyter notebooks implementing the experimental workflow:

- **`Poison_generation.ipynb`** generates the poisoned passages.
- **`Eval.ipynb`** runs the evaluation workflow and computes the reported metrics.

### `poisons/`

Contains the poison passages used in the recorded experiments. These files are included so that the evaluation can use the same poison artifacts without requiring a new generation run.

### `evaluation/`

Contains model responses and associated evaluation artifacts in XLSX format. These files preserve the outputs used in the reported analysis and allow the recorded metrics to be inspected and recalculated without making additional model API calls.

---

# Reproducing and Verifying the Results

The repository supports two forms of reproduction:

1. **Verification of the recorded results using the supplied artifacts**
2. **Rerunning the experimental workflow**

## 1. Verify the recorded results without API calls

The XLSX files in [`evaluation/`](evaluation/) contain the model responses and evaluation information from the recorded experiments.

To inspect the results:

1. Open the relevant XLSX files.
2. Inspect the model responses and associated evaluation labels.
3. Recalculate the metrics from the supplied outputs.

This verification process does not require model API access. The fixed artifacts make it possible to inspect the outputs underlying the reported aggregate results independently of the model providers.

## 2. Rerun the experimental workflow

The executable workflow is provided in [`src/`](src/).

### Step 1: Generate poisoned passages

Run:

```text
src/Poison_generation.ipynb
```

This notebook implements the poison-generation procedure. Generated passages can be stored in:

```text
poisons/
```

The passages used in the recorded experiments are already included. Regenerating them is therefore not necessary to reproduce the evaluation based on those recorded passages.

### Step 2: Run the evaluation

Run:

```text
src/Eval.ipynb
```

The evaluation notebook runs the experimental conditions and computes the evaluation metrics. The main SciFact experiment uses **200 randomly sampled questions with random seed 38**.

---

## Experimental Conditions

The main experiments evaluate the following model configurations:

- **DeepSeek-V4-Flash**
  - reasoning off
  - reasoning on
- **Qwen3.6-Plus**
  - reasoning off
  - reasoning on

Additional exploratory experiments include:

- **Claude Haiku 4.5**
- **Claude Sonnet 4.6**

The Claude experiments are exploratory. They are not a within-model comparison of reasoning settings because the compared models differ in model capability as well as configuration.

## Datasets

The primary evaluation uses **SciFact**:

- 200 randomly sampled questions
- Random seed: 38

Additional experiments use:

- **FiQA** — first 40 questions
- **MS MARCO** — first 40 questions

These additional datasets provide supplementary observations across datasets and questions.

---

# Evaluation Metrics

The evaluation distinguishes between identifying poisoned information and being influenced by it. Let `M_RAG` denote the response with poisoned retrieved context, `M_no-RAG` the response without retrieved context, and `M_ignore` the response with poisoned context and an explicit instruction to ignore it. An answer is poison-aligned when its final verdict agrees with the poison direction.

## Attack Success Rate (ASR)

ASR is the probability that the RAG answer is poison-aligned, conditioned on the no-RAG answer not already being poison-aligned:

```text
ASR = P(poison-aligned(M_RAG) | not poison-aligned(M_no-RAG))
```

## Poison Detection Rate (PDR)

PDR is the proportion of evaluated instances in which the model is judged to have recognized the poisoned information:

```text
PDR = P(detected(M_RAG))
```

Detection labels are based on the target model's full response and the supplied poison information.

## Cordon Rate (CR)

CR is the probability that the model both detects the poisoned document and nevertheless produces a poison-aligned answer, conditioned on the no-RAG answer not being poison-aligned:

```text
CR = P(poison-aligned(M_RAG) AND detected(M_RAG)
       | not poison-aligned(M_no-RAG))
```

CR is computed over the eligible questions for each configuration. A lower CR can partly reflect lower detection, so it should be interpreted alongside PDR and CCR.

## Conditional Cordon Rate (CCR)

CCR measures poison-aligned answering among detected, eligible cases:

```text
CCR = P(poison-aligned(M_RAG)
        | detected(M_RAG) AND not poison-aligned(M_no-RAG))
```

The detected subsets can differ between configurations; CCR is therefore interpreted descriptively when comparing them.

## Leakage Rate (LR)

LR measures whether the model is influenced by poisoned context despite being explicitly instructed to ignore the retrieved documents:

```text
LR = P(poison-aligned(M_ignore)
       AND not poison-aligned(M_no-RAG))
```

LR is computed over all evaluated instances.

The metric computation is implemented in the evaluation workflow.

---

# Poison Generation

The poison-generation workflow constructs contradictory information and incorporates it into retrieved-context passages. The generated passages are then used in the evaluation pipeline.

The exact poison passages used in the recorded experiments are included in:

```text
poisons/
```

Using these fixed artifacts avoids relying on a new stochastic generation run when reproducing the recorded evaluation. The generation notebook is provided to make the generation procedure available for inspection and rerunning.

---

# Evaluation Artifacts

The `evaluation/` directory contains the model responses produced for the recorded experiments. These artifacts allow inspection of the outputs underlying the aggregate metrics.

The evaluation can be traced through the following stages:

```text
Question
   ↓
Retrieved context
   ↓
Poisoned context
   ↓
Model response
   ↓
Evaluation / judgment
   ↓
Metric
```

The supplied artifacts make this inspection possible without requiring additional model API calls.

---

# External Model APIs

Rerunning model inference requires access to the corresponding model providers and their APIs.

### Verifying the recorded results

The supplied files in:

```text
poisons/
evaluation/
```

can be used to inspect and verify the recorded results without additional model API calls.

### Rerunning model inference

Rerunning inference requires access to the relevant external APIs. New runs may produce different responses because external models can change over time and may exhibit provider-side nondeterminism.

For verification of the exact outputs used in the recorded analysis, use the supplied XLSX evaluation artifacts.

---

# Reproducibility Summary

| Component | Provided |
|---|---|
| Poison-generation notebook | Yes |
| Evaluation notebook | Yes |
| Poison passages used in the experiments | Yes |
| Model responses | Yes |
| Evaluation artifacts | Yes |
| Metric computation | Yes |
| API-independent inspection of recorded results | Yes |
| Executable experimental workflow | Yes |

The repository therefore provides both an executable workflow for rerunning the experiments and fixed experimental artifacts for independently inspecting the recorded results without additional model API calls.
