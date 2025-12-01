# DietCheck – README

This README explains **how we created and managed the manual annotations** for the DietCheck project, including all files, notebooks, and step-by-step instructions for annotators.

The goal is to have a **clean, reproducible, human-annotated dataset** for:

- **Task 1:** Dietary classification (rule-based, FDA thresholds)
- **Task 2:** Claim verification (manual)
- **Task 3:** Ingredient entity tagging (manual BIO tagging)

This document focuses on **Tasks 2 and 3**, where all labels are created manually.

---

## 1. Dataset and Files

### 1.1 Core product dataset

We define a **core dataset of 200 packaged food products**. This is our reference universe for all three tasks.

Typical columns in `core_products.csv`:

- `product_id`
- `name`
- `brand`
- `category`
- `ingredients` (raw ingredient string)
- Numeric nutrition per serving:
  - `sodium_per_serving`
  - `fat_per_serving`
  - `protein_per_serving`
  - `net_carbs_per_serving`
  - (optionally: calories, fiber, sugars, etc.)
- Task 1 rule-based labels (computed from FDA-style thresholds):
  - `low_sodium`
  - `low_fat`
  - `high_protein`
  - `keto_compliant`

> **Important:** Task 1 labels are **computed by script**, but they are **never shown** to annotators during Tasks 2 and 3 to avoid bias.

---

## 2. Overview of Annotation Tasks

### 2.1 Task 1 – Dietary Classification (Rule-Based)

- Uses coded FDA thresholds (e.g., low sodium, low fat, high protein, keto-compliant).
- Labels are generated automatically from numeric facts in `core_products.csv`.
- These labels are used as **targets for Task 1 models** and as **reference** for checking claim conflict, but annotators do **not** see them.

### 2.2 Task 2 – Claim Verification (Manual)

For a subset of **160 products** from the core 200, we manually annotate:

- The **marketing/packaging claim** (if any)
- Whether the claim is **verifiable** from nutrition facts / ingredients
- Whether the claim **conflicts** with the numeric facts
- A short **explanation** for the decision

Columns in the Task 2 annotation template:

- `product_id`
- `name`
- `brand`
- `sodium_per_serving`
- `fat_per_serving`
- `protein_per_serving`
- `net_carbs_per_serving`
- `claim_text` *(manual)*
- `verifiable` *(manual: Yes/No)*
- `conflict` *(manual: Yes/No)*
- `explanation` *(manual, 1–2 sentences)*
- `annotator_id` *(manual: e.g., A, B, C…)*

### 2.3 Task 3 – Ingredient Entity Tagging (Manual BIO)

For a subset of **120 products** from the core 200, we manually tag ingredients using a **BIO scheme** for three risk-relevant entity types:

- **SUGAR**
- **SODIUM**
- **PROTEIN**

We first **pre-tokenize** each ingredient list using a deterministic rule (e.g., split on spaces and punctuation) and then annotate each token.

Typical columns in the Task 3 annotation template:

- `product_id`
- `token_index`
- `token`
- `tag` *(manual: B-SUGAR, I-SUGAR, B-SODIUM, B-PROTEIN, I-PROTEIN, O)*
- `annotator_id`

> Annotators **never edit tokens**; they only assign tags.

---

## 3. Notebooks and Scripts

We use two main notebooks for annotation management:

### 3.1 `04_annotation_templates.ipynb` (or `IMPROVED_04_annotation_templates.ipynb`)

**Purpose:**  
Create reproducible CSV templates for Tasks 2 and 3 (including double-annotation subsets) from `core_products.csv`.

**Key steps inside the notebook:**

1. **Load core dataset**
   - Reads `core_products.csv`.
   - Checks that required columns are present.

2. **Sample products for Task 2**
   - Randomly sample `n = 160` from the core 200 using a fixed seed (e.g., `random_state=42`).
   - Save sampled IDs to `task2_ids.csv` for reproducibility.
   - Build `task2_annotation_template.csv` with:
     - Product metadata + numeric nutrition.
     - **Empty** `claim_text`, `verifiable`, `conflict`, `explanation`, `annotator_id`.

3. **Sample products for Task 3**
   - Randomly sample `n = 120` from the core 200 using a fixed seed (e.g., `random_state=99`).
   - Save sampled IDs to `task3_ids.csv`.
   - Pre-tokenize `ingredients` and build `task3_annotation_template.csv`:
     - One row per `(product_id, token_index, token)`.
     - Empty `tag`, `annotator_id` columns.

4. **Create double-annotation subsets**
   - Task 2: select e.g. 25 product_ids from the 160 to be double-annotated.
   - Task 3: select e.g. 10 product_ids from the 120 to be double-annotated.
   - Save separate CSVs:
     - `task2_double_annotation_ids.csv`
     - `task3_double_annotation_ids.csv`
   - These are later assigned to two annotators each, for κ calculation.

5. **Export templates**
   - Save all templates to disk (for Google Sheets upload) or download via Colab (`files.download`).

---

### 3.2 `05_inter_annotator_agreement.ipynb`

**Purpose:**  
Compute **inter-annotator agreement** (Cohen’s κ) for Task 2 and Task 3 and analyze disagreements.

Typical steps:

1. **Load annotated CSVs**
   - Task 2: two CSVs for the double-annotation subset (e.g., `task2_annotations_A.csv`, `task2_annotations_B.csv`).
   - Task 3: two CSVs for the double-annotation subset.

2. **Align by key columns**
   - Task 2: merge on `product_id`.
   - Task 3: merge on `product_id`, `token_index`.

3. **Compute κ**
   - Task 2:
     - κ for `verifiable`.
     - κ for `conflict`.
   - Task 3:
     - κ over the full BIO tag set (`B-SUGAR`, `I-SUGAR`, `B-SODIUM`, `B-PROTEIN`, `I-PROTEIN`, `O`).

4. **Inspect disagreements**
   - List products/tokens where annotators disagree.
   - Use this to refine guidelines or correct obvious annotation errors.

---

## 4. How to Annotate (For Annotators)

### 4.1 Common Setup (Google Sheets)

For each CSV template (`task2_annotation_template.csv`, `task3_annotation_template.csv`):

1. Go to **Google Sheets**.
2. Create a blank spreadsheet.
3. `File → Import → Upload` and select the CSV.
4. Choose “Insert new sheet(s)” or “Replace current sheet” (depending on preference).
5. Freeze the header row (`View → Freeze → 1 row`).
6. The project lead assigns row ranges to each annotator.

Each annotator is assigned a unique `annotator_id` (e.g., `A`, `B`, `C`).

---

### 4.2 Task 2 Annotation Instructions

For each row in the Task 2 sheet:

**Context columns (read-only):**

- `product_id`
- `name`
- `brand`
- `sodium_per_serving`
- `fat_per_serving`
- `protein_per_serving`
- `net_carbs_per_serving`

**Columns to fill:**

1. `claim_text`
   - Write the **exact** or **paraphrased** dietary claim from the product (e.g., “Low Sodium”, “High Protein”, “Keto Friendly”).
   - If **no relevant claim** is present, write `"No claim"`.

2. `verifiable` (Yes / No)
   - `Yes`: the claim can be checked using the nutrition facts or ingredient list.
   - `No`: vague or non-factual claims (e.g., “Tastes amazing”, “Made with love”).

3. `conflict` (Yes / No)
   - `Yes`: if the numeric facts **contradict** the claim according to the FDA-style rules used in Task 1.
   - `No`: if the facts are consistent with the claim or there is no claim.

4. `explanation`
   - 1–2 short sentences justifying the decision.
   - Example:  
     - `"410mg sodium > 140mg threshold for low sodium; claim 'Low Sodium' is conflicting."`
     - `"No explicit low fat / high protein claim; no conflict."`

5. `annotator_id`
   - Your personal ID (e.g., `A`, `B`, …) for every row you annotated.

> Do **not** invent claims that are not present on the packaging; only record what is actually claimed.

---

### 4.3 Task 3 Annotation Instructions

For each row in the Task 3 sheet:

**Context columns (read-only):**

- `product_id`
- `token_index`
- `token`

**Columns to fill:**

1. `tag`
   - Use one of:
     - `B-SUGAR`, `I-SUGAR`
     - `B-SODIUM`      (typically for “SALT”, “SODIUM”, etc.)
     - `B-PROTEIN`, `I-PROTEIN`
     - `O` (outside, not part of any target entity)
   - Use **BIO rules**:
     - `B-XXX` for the first token of an entity span.
     - `I-XXX` for subsequent tokens of the same entity.
     - `O` for all non-target tokens (including commas, brackets, oils, flavors, etc.).

2. `annotator_id`
   - Your personal ID (e.g., `A`, `B`, …).

> Annotators must **not modify** the tokenization. If a tokenization decision looks odd, leave the token as is and tag it with the most reasonable label.

---

## 5. Double Annotation and Agreement

To measure reliability:

- **Task 2:**
  - A subset of e.g. **25 products** is annotated **independently by two annotators**.
  - These rows appear in separate copies of the sheet or are duplicated with different `annotator_id` values.
- **Task 3:**
  - A subset of e.g. **10 products** (all their tokens) is double-annotated.

The `05_inter_annotator_agreement.ipynb` notebook:

- Loads the double-annotated CSVs.
- Aligns rows by product/token.
- Computes **Cohen’s κ** for:
  - Task 2: `verifiable`, `conflict`.
  - Task 3: BIO tags.
- Prints κ values and example disagreements, which can be used to refine guidelines or, if needed, correct obvious annotation mistakes.

A target of **κ ≥ 0.65** is considered acceptable (substantial agreement); lower values may trigger a guideline revision and re-annotation of the subset.

---

## 6. Reproducibility Notes

- All sampling in `04_annotation_templates.ipynb` uses **fixed random seeds**.
- The following files are produced and version-controlled:
  - `core_products.csv`
  - `task2_ids.csv`, `task3_ids.csv`
  - `task2_annotation_template.csv`, `task3_annotation_template.csv`
  - `task2_double_annotation_ids.csv`, `task3_double_annotation_ids.csv`
- Annotated CSVs are stored as:
  - `task2_annotations_*.csv` (per annotator or merged)
  - `task3_annotations_*.csv` (per annotator or merged)

This ensures that anyone can:

1. Recreate the exact templates.
2. Inspect how labels were produced.
3. Recompute all agreement metrics.

---

## 7. Quick Start Checklist

**For the project lead:**

- [ ] Prepare `core_products.csv` (200 products).
- [ ] Run `04_annotation_templates.ipynb` to generate all templates and double-annotation IDs.
- [ ] Upload Task 2 and Task 3 templates to Google Sheets.
- [ ] Assign row ranges + annotator IDs.
- [ ] Share `ANNOTATION_GUIDELINES.md` with all annotators.

**For annotators:**

- [ ] Read `ANNOTATION_GUIDELINES.md` completely.
- [ ] Confirm your `annotator_id`.
- [ ] Open your assigned Google Sheet and filter to your rows.
- [ ] Fill in all required columns carefully.
- [ ] For double-annotation rows, do not look at anyone else’s labels.

**After annotation:**

- [ ] Export final sheets to CSV.
- [ ] Run `05_inter_annotator_agreement.ipynb` to compute κ.
- [ ] Log κ values and any guideline updates in the repo.
