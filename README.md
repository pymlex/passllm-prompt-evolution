# PassLLM Prompt Evolution

## Overview

This repository contains prompt-evolution experiments for targeted password guessing with PassLLM. The starting point is the PassLLM framework from USENIX Security 2025, which uses LoRA-based adaptation and separate generation algorithms for trawling and targeted guessing. The paper covers four password-guessing scenarios: trawling, PII-based targeted guessing, password reuse, and PII plus sister-password guessing. ([USENIX][1])

The main idea here is simpler: keep the model weights fixed and improve the prompt. That matches the 2026 prompt-evolution work, where prompt selection is treated as a real optimization problem rather than a minor implementation detail. The repository follows the same direction and measures changes by crack rate on the Kaggle Password Guessing setup. ([arXiv][2])

## Problem setting

We had 16000 pairs of old and new passwords in the train set and 8000 old passwords in the test JSON. The lengths distibution for the train partition is:

<img width="580" height="432" alt="output_13_1" src="https://github.com/user-attachments/assets/dff8f10b-054b-4234-aaf3-b79eda1ac13f" />

For each old password `p_old`, the model generates a ranked list of candidate new passwords. The target is to place the true password `p_new` somewhere in the candidate set.

The main metric is crack rate:

$$CR = \frac{\text{number of correct matches}}{\text{number of evaluated samples}}.$$

The score is computed on train samples, using the same generation budget for every prompt. The comparison is therefore prompt-to-prompt, not model-to-model. 

## Model and evaluation setup

The base model is `Qwen/Qwen2.5-0.5B-Instruct` with the `126_csdn_disQwen0.5B` adapter merged into it. Prompt mutation is done through OpenEvolve with `gpt-oss-120b` as the mutator model. The stable configuration used in the experiments is 100–150 generations, population size 2000, elitism 10–100, tournament size 5, and crossover probability 0.3–0.8. The evaluator measures crack rate on a training subset and the generation pipeline uses sampling with `temperature=0.9` and `top_p=0.95`. 

The best prompt is not the most verbose one. The strongest version found in the experiments became short, strict, and output-oriented: it asks for a new password of similar length, requires uppercase, lowercase, digits, and a special character, and forbids reuse of characters from the old password. That form worked better than prompts that pushed the model into extra analysis or commentary. 

## Prompt evolution results

The evolution process improved the score gradually. The initial baseline started around `CR ≈ 0.268`, then moved to `0.272`, `0.286`, and finally reached `0.306` on the full evolution run. The final prompt was saved as `best_program.txt`. We also reached the scores of `0.300` and  `0.320` for different setups.

The useful pattern was consistent across runs: prompts with explicit password-transform rules performed better than short generic instructions. Instructions that forced analysis of the old password often produced extra text and made the cleaned output worse. Adding more structure to the prompt helped only when that structure stayed compatible with the model’s tendency to produce a single password string. 

## Repository contents

- `baseline-experiments-sampling-0.300.ipynb` contains the baseline prompt experiments and the comparison runs.
- `2000p-sampling-0.306.ipynb` contains the larger evolution run that reached the strongest structured prompt around `CR = 0.306`.
- `2000p-sampling-0.320.ipynb` contains the sampling-based variant of the prompt-evolution setup.
- `beamsearch.ipynb` contains the beam-search generation pipeline, usually used for reuse attacks when the new password is a modification of the old one.
- `best_program_-no-reuse-0.300.txt`, `best_program_-no-reuse-0.306.txt`, and `best_program-sampling-0.320.txt` store the final evolved prompts.

## Takeaway

Prompt evolution matters. Even when the model weights stay fixed, the wording of the instruction changes the score in a measurable way. For password guessing, the most effective prompts are not the most “intelligent” ones; they are the ones that make the generation rule unambiguous and keep the output format extremely tight. That is the point of this repository: prompt text is part of the attack surface. ([arXiv][2])

[1]: https://www.usenix.org/conference/usenixsecurity25/presentation/zou-yunkai?utm_source=chatgpt.com "Password Guessing Using Large Language Models"
[2]: https://arxiv.org/html/2604.12601v1?utm_source=chatgpt.com "LLM-Guided Prompt Evolution for Password Guessing ..."
