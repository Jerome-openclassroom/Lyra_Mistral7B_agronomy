<img src="https://upload.wikimedia.org/wikipedia/en/c/c3/Flag_of_France.svg" width="100px" height="auto" />

![EUstack](https://img.shields.io/badge/🇪🇺%20EUstack-ready-blue)
![SouverainAI](https://img.shields.io/badge/🇫🇷%20SouverainAI-oui-success)
![LLM_Mistral](https://img.shields.io/badge/🧠%20LLM-Mistral_Small-lightblue)


# 🇫🇷 Lyra – Mistral 7B Agronomy (LoRA)

A specialized Mistral 7B model fine-tuned (QLoRA) to perform agronomic diagnosis on tomato plants using three parameters:
- Soil nitrate (NO₃, mg/kg)  
- Soil pH (water method)  
- Green optical density (DO) from leaf scanning  

The model outputs:
- A nuanced nitrogen diagnosis (deficit_N_fort, deficit_N_leger, N_normal, excès_N_leger, excès_N_fort)  
- A computed SPAD-equivalent value (chlorophyll indicator)  
- A structured agronomic explanation  
- Optional detection of stress non-azoté (water stress, diseases, parasitism)  
- Optional detection of chlorose ferrique (pH > 7.8 + DO low + NO₃ high)  
- Optional recommendations for Integrated Pest Management (IPM) when DO is high  

---

# 🌱 Project Overview

This repository contains the full pipeline used to train, evaluate and validate **Lyra_DO_vert_mistral7B_qLoRA**, an open-science agronomy model derived from Mistral-7B-Instruct-v0.3.

The aim is to demonstrate how low-cost leaf scanning (green optical density) combined with soil data can produce a robust nitrogen-nutrition diagnostic model suitable for open-source agronomic tools.

The project is part of a broader effort to build community-accessible AI tools for agricultural diagnostics and environmental monitoring.

---

📁 Arborescence du dépôt — Lyra_Mistral7B_agronomy
```
Lyra_Mistral7B_agronomy/
├── README.md                           # 📘 Main documentation (English version )
├── README_fr.md                        # 📘 French version of the README

├── code/                               # 🧠 Training & inference scripts
│   └── Lyra_DO_vert_7b.py              # Colab/QLoRA script to train the 7B model

├── datasets/                           # 🌱 Datasets for SFT (Supervised Fine-Tuning)
│   ├── train_tomato_nitrogen_DO_pH_1000.jsonl   # Full training dataset (1000 rows)
│   └── eval_tomato_nitrogen_DO_pH_20.jsonl      # Manual evaluation set (20 rows)

├── graphs_statistics/                  # 📊 Dataset analyses and visualizations
│   ├── bilan_analyse_GPT_stat.txt      # Textual dataset analysis by GPT-5.1 (Diagrams)
│   ├── statistic_dataset_nitrogen.png  # Chart: distribution of nitrogen diagnostics
│   └── statistic_DO_vert_barchart.png  # Chart: distribution of DO Green values

```
---

# 🧪 Dataset Construction

### ➤ Training set: 1000 examples  
Structured as Mistral chat format:

```json
{"messages": [{"role": "system","content": ...},{"role": "user","content": ...},{"role": "assistant","content": ...}]}
```

### Inputs  
- **NO₃** from 0 to 200 mg/kg  
- **pH water** from 6.0 to 8.2  
- **DO verte** (green optical density) from **250 to 550**  

### Outputs (assistant)  
Each example contains:
- A nitrogen diagnosis  
- A SPAD estimation computed with  
  **SPAD = 0.178 × DO − 36.454**  
- A structured agronomic explanation  
- Optional special-case markers (stress non-N, chlorose ferrique, lutte intégrée)

### Special cases included explicitly:
- **100 cases of chlorose ferrique**  
  (pH > 7.8 + high NO₃ + low DO)
- **100 cases of stress non-azoté**  
  (low DO + NO₃ medium/high + pH ≤ 7.8)
- **Distribution of DO:**  
  ```
  Mean ~ 383
  Min 250 / Max 550
  Q1 299 / Q3 465
  ```

The dataset is **0% duplicate**, both in strict and semantic checks.

## 📈 Distribution of Nitrogen Diagnostics

![Distribution of nitrogen diagnostics](graphs_statistics/statistic_dataset_nitrogen.png)

This chart shows the distribution of the six nitrogen diagnostic classes in the training dataset.  
A deliberately controlled balance between categories can be observed, with targeted additions for:
- cases of **non-nitrogen stress**  
- cases of **ferric chlorosis** (high pH)

This structuring allowed the model to correctly learn atypical behaviors,  
while maintaining good generalization on normal situations.

## 📉 Distribution of Green Optical Density (DO)

![Distribution of Green Optical Density](graphs_statistics/statistic_DO_vert_barchart.png)

The distribution of green optical density (250–550) follows a wide and continuous range,  
with a mean around 380 and a standard deviation of ~90.  
This variability is essential for the model to learn the physiological relationship  
between **leaf greenness → chlorophyll → SPAD**, as well as the characteristic deviations  
of stress situations (low DO) or excessive vigor (high DO).

---

# 🏋️ Training (QLoRA)

### Model  
`mistralai/Mistral-7B-Instruct-v0.3`

### Method  
- QLoRA (rank 64)  
- No quantization (fp16 load)  
- Batch size 2, grad acc 8  
- 3 epochs  
- Learning rate 2e-4  
- Evaluation each epoch  

### Metrics  
| Epoch | Train Loss | Val Loss | Mean Token Accuracy | Entropy |
|-------|------------|----------|----------------------|---------|
| 1     | 0.1897     | 0.1895   | 0.9387              | 0.2295 |
| 2     | 0.1768     | 0.1773   | 0.9444              | 0.2227 |
| 3     | 0.1727     | 0.1750   | 0.9459              | 0.2205 |

### Interpretation  
- Train / Val loss nearly identical → **excellent generalization**  
- Token accuracy ~94–95% → **strong alignment to domain language**  
- Low entropy ≈0.22 → **model is confident and deterministic**  

This is the behavior of a *specialized expert model*, not a generic LLM.

---

# 🔬 Comparison with Mistral-8B (server-side fine-tuning called 'AI-Studio')

A server-side Mistral-8B fine-tune was performed first.  
Despite being trained on the same dataset, the 8B model:

### Limitations observed:
- Misinterpreted **DO** as *“dissolved oxygen”* (its default meaning)  
- Produced **incorrect SPAD values**  
- Classified DO=289 as “high DO”  
- Gave excessive weight to stylistic patterns  
- Struggled with rare agronomic edge cases  

Cause: server fine-tune is intentionally light — it adjusts *style* more than core semantics.

### Improvements with the 7B QLoRA:
- DO interpreted correctly as **green optical density**  
- **SPAD perfectly aligned** with the formula  
- Stress non-N cases correctly detected  
- Lutte intégrée triggered when DO is high  
- Structure of the agronomic explanation consistently respected  
- Far fewer hallucinations or irrelevant statements  

Conclusion:  
➡️ **QLoRA 7B gives vastly better numerical + conceptual stability than 8B server-FT**  
➡️ **Relationships DO→SPAD→diagnosis are learned correctly**  

---

# 🧪 Inference Tests

### Test 1 — Normal  
DO 354 → SPAD 27  
Diagnosis: N_normal  
→ Perfectly correct.

### Test 2 — Deficit  
DO 275 → SPAD 13  
Diagnosis: deficit_N_leger  
→ Correct direction.

### Test 3 — High NO₃ + High DO  
Expected: excès  
Model: N_normal + lutte intégrée  
→ Learns DO-high patterns correctly; slight under-estimation of nitrogen excess.

### Test 4 — Stress non-azoté  
DO 268, NO₃ high, pH normal  
→ Correct identification: “DO faible non explicable par N → stress hydrique/maladie/parasitisme”.

### Test 5 — Chlorose ferrique  
pH 8.05, NO₃ high, DO low  
→ SPAD correct  
→ Diagnosis currently leaning toward deficit  
→ Chlorosis cue learned but weaker than desired.  
Can be improved with a small reinforcement dataset (50–100 examples).

## 🔬 Validation of the DO → SPAD Relationship (Deeply Learned)

A critical step in the evaluation is to verify that the model has truly  
**internalized the quantitative relationship between green optical density (DO) and the SPAD value**,  
rather than superficially producing plausible numbers.

The theoretical formula used to generate the dataset is:

**SPAD = 0.178 × DO − 36.454**

Source of the model :
https://github.com/Jerome-openclassroom/Lyra_DO_Green_Mesurim?tab=readme-ov-file#-correlation-graphs

We tested several cases, with the following results:

| Green DO | Expected SPAD (theoretical) | Model-predicted SPAD | Difference |
|---------:|----------------------------:|----------------------:|-----------:|
| 354      | 26.6                        | 27                   | +0.4       |
| 275      | 12.5                        | 13                   | +0.5       |
| 510      | 54.3                        | 55                   | +0.7       |
| 268      | 11.3                        | 11                   | -0.3       |
| 271      | 11.8                        | 12                   | +0.2       |

### ✔ Result: Relationship Perfectly Learned

The model’s predictions consistently fall within an interval of  
± **1 SPAD** compared to the theoretical value, which corresponds to simple rounding.

This shows that the model is not merely “rephrasing”  
but has indeed **encoded the DO → SPAD function into its deep layers**.  
In practice, this means it:

- reproduces the physiological slope (0.178)  
- respects the constant offset (−36.454)  
- generalizes correctly to values not explicitly present in the dataset  

👉 **This numerical stability confirms the quality of the SFT and the proper assimilation of the physiological DO → chlorophyll → SPAD relationship.**

---

# 🎯 Conclusion

**Lyra_Mistral7B_agronomy** is a specialized open-source agricultural diagnostic model that:

- Performs deterministic SPAD estimation from DO  
- Produces agronomically consistent diagnoses  
- Recognizes atypical patterns (stress non-N, IPM guidance)  
- Overcomes the limitations of server-side fine-tuning  
- Demonstrates the viability of low-cost sensing (leaf scanner DO)  
- Is fully reproducible (dataset, scripts, adapters all provided)

This repository can serve as a foundation for:
- Agronomic decision-support tools  
- Mobile/IoT diagnostic apps  
- Research on crop nutrition modeling  
- Open-science AI in agriculture  

---

# 📎 Hugging Face Model

https://huggingface.co/jeromex1/lyra_DO_vert_mistral7B_qLoRA

---

