# Khmer Sentence Simplification

Rewriting complex, formal Khmer sentences (royal and literary vocabulary from folk tales) into simple, everyday Khmer.

| | Example |
|---|---|
| **Complex** | ព្រះរាជកុមារ ថ្វាយបង្គំលាព្រះមាតារួចស្រេច ក៏ម្នីម្នាតម្រង់ទៅ ដែនពារាណសី ដែលជាទីឃុំឃាំងព្រះបិតាព្រះអង្គ ។ |
| **Simple** | ព្រះរាជកុមារលាម្ដាយរួច ក៏ប្រញាប់ទៅដែនពារាណសី ជាកន្លែងដែលគេឃុំឪពុករបស់គាត់ ។ |

## Repository contents

| File | What it is |
|---|---|
| `khmer_simplification.ipynb` | The full pipeline (Google Colab): data checks, baselines, training, decoding, evaluation, error analysis and an interactive "try your own sentence" box. |
| `complex_to_simple_pairs_1000.csv` | The parallel dataset: 1,030 complex → simple sentence pairs from 115 stories. Columns: `source` (story ID), `complex`, `simple`. |
| `rating_sheet.xlsx` | Human evaluation: 60 test sentences rated by Khmer speakers on meaning kept, simplicity and fluency (1–5), plus an overall acceptable Y/N. |

## Method

The pipeline has four parts:

1. **Royal-word glossary (pre-pass).** About 55 royal or literary words are replaced with everyday words before the model sees the sentence, for example ព្រះបិតា → ឪពុក, ព្រះមាតា → ម្ដាយ, ព្រះរាជា → ស្ដេច, ទតឃើញ → ឃើញ. The longest match wins, and protected words such as ព្រះអង្គម្ចាស់ are left as they are. Each rule was kept only if it did not make the outputs on the training data less similar to the references.
2. **Three fine-tuned mT5-small models.** `google/mt5-small` was fine-tuned three times with input prefix `simplify: `:
   - seed 42 with learning rate 5e-4
   - seeds 43 and 44 with learning rate 3e-4 and label smoothing 0.1

   Training used warm-up and early stopping, and pairs containing royal words appear twice in the training data.
3. **Guarded decoding.** Each model returns 5 beam-search candidates. A candidate loses points if it:
   - is too short (under 65% of the input length)
   - drops content words the training data normally keeps, which catches wrong-word rewrites such as "sword" turning into "mother"
   - still contains royal words

   On top of that, a **fluency score** from pretrained `xlm-roberta-base` (pseudo-log-likelihood) helps reject garbled Khmer words. A **three-model vote** gives a bonus to candidates the models agree on.
4. **Output clean-up.** Remaining royal words are replaced and repeated words are removed.

The decoding setting (one model or the three-model vote, and how much weight fluency gets) was chosen on the validation set only. The test set was scored once.

### Data split

The data is split **by story**, so validation and test sentences come from stories the model never saw:

| Train | Validation | Test |
|---|---|---|
| 821 | 110 | 99 |

## Results

### Automatic metrics (test set, 99 sentences)

| System | chrF | BLEU | SARI-1 | Copy rate ↓ | Royal words left ↓ |
|---|---|---|---|---|---|
| Copy input | 65.2 | 60.4 | 26.9 | 1.00 | 0.80 |
| Glossary only | 68.8 | 66.3 | 49.5 | 0.45 | 0.58 |
| Glossary + learned lexicon | 70.0 | 68.1 | 60.8 | 0.20 | 0.18 |
| **Final (3-model vote, fluency w=10)** | **73.0** | **73.6** | **71.0** | **0.04** | **0.02** |

With only one reference per sentence, chrF and BLEU are inflated by copying and barely penalize a wrong or garbled word. That is why the notebook also reports SARI-1, copy rate, royal words left, too-short outputs and missing content words, and why the human rating is the main result.

### Human evaluation (`rating_sheet.xlsx`)

These 60 test sentences were rated by Khmer speakers. They were rated on the output of the seed-42 model with guarded decoding (before the fluency score and clean-up were added):

| Measure | Result |
|---|---|
| **Acceptable** | **42 / 60 = 70.0%** (95% Wilson CI 57.5–80.1%) |
| Meaning kept (1–5) | mean 4.17 |
| Simpler (1–5) | mean 3.23 |
| Fluent (1–5) | mean 3.13 |

The model keeps the meaning well. Simplicity and fluency are the main areas to improve.

## How to run

1. Open `khmer_simplification.ipynb` in [Google Colab](https://colab.research.google.com/) and select a **GPU runtime** (it was developed on a T4).
2. Run the cells in order. When asked, upload `complex_to_simple_pairs_1000.csv`.
3. Trained models are saved to Google Drive (`MyDrive/khmer_simplifier_models_v4`). If Colab disconnects, re-running loads them instead of training again.
4. Use section 12 of the notebook to simplify your own sentences. The model was trained on folk-tale style sentences of roughly 25–120 characters.

Main libraries: `transformers`, `datasets`, `sentencepiece`, `sacrebleu`, `khmer-nltk`, `scikit-learn`.

## Limitations

- **The data is not fully verified.** The sentence pairs and the glossary were written with help from an AI assistant, and some of them have not yet been checked by a native speaker.
- The training set is small (821 pairs) and covers one domain: folk tales.
- The glossary covers only about 55 royal words. Names and rare words are left to the model.
- Long sentences may lose a clause.
- The model weights are not included in this repository. Run the notebook to train them.
