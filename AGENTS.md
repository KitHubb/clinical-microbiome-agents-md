# AGENTS.md — Common Clinical Microbiome R Analysis Guidelines

Work with the clinical microbiome / metagenomics researcher on paper-grade
analyses in R. Verify before deciding, keep code minimal and re-readable months
later, and never let a threshold, filter, or model choice quietly steer the
result toward significance.

# Scope and companion documents

These common guidelines apply to both amplicon and shotgun projects.
Before analysis, identify the assay, profiling tool, feature type, abundance
unit, input object/table, and study design.

- For amplicon analysis, also read [amplicon/AGENTS.md](amplicon/AGENTS.md).
  It preserves the existing phyloseq-based pipeline and statistical rules.
- For routine shotgun downstream analysis, also read
  [shotgun/CORE_DOWNSTREAM_ANALYSIS.md](shotgun/CORE_DOWNSTREAM_ANALYSIS.md).
  It covers the core microeco workflow; it does not define read preprocessing,
  assembly, MAG reconstruction, or every possible shotgun analysis.

Keep this file at the project root and retain the companion paths when copying
the guidelines to another project. Add study-specific exceptions under
"Project-Specific Guidelines" at the bottom of the relevant document.

# Shared input sources

| Source | Contents | Format |
|---|---|---|
| **Clinical metadata** | Subject demographics, diagnoses, clinical variables (e.g. GDM, BMI, Sex), visit number (V1/V2), SubjectID | xlsx/csv |
| **Wet-lab log** | Collection date, DNA extraction QC, library concentration, technician, sequencing run/batch ID, index | xlsx/csv |

Validate each stage's artifact before using it in a later stage. The amplicon
companion defines its existing three-stage pipeline and output directories.

# When to ask the user for permission

Proceed without asking when implementing an already-approved plan, doing
read-only exploration (EDA, distributions, cross-tabs), fixing an existing
bug, or producing a re-runnable intermediate artifact.

Stop and ask before fixing filtering or normalization choices, selecting a
contamination-removal method, deciding model covariates, applying batch
correction, or renaming/restructuring existing analysis objects.
Confirm the availability of negative controls before selecting a method.

Never pick a plausible-looking default and move on quietly. When there is a
real choice, state the trade-offs in one or two lines, then implement exactly
one primary analysis and one sensitivity analysis — not every option on the
table. Assay-specific approval gates are in the relevant companion document.

# Think before coding

Before writing code, write the strategy in plain language: analysis goal →
input (which object/table, which subset) → output (which statistics, which
plots) → verification checklist. If the user asked only for a strategy, do not
output code yet. Keep statistical assumptions and code implementation visibly
separate.

# Study design

Classify the data as cross-sectional or longitudinal before choosing a
statistical strategy. Keep independent subject counts distinct from sample
counts, identify repeated measurements, and fix inclusion rules before analysis.
Report how many subjects each rule excludes. Follow the amplicon companion
for its existing model and permutation instructions.

---

# Simplicity and surgical changes

- Do not add options, generalized frameworks, or "flexibility" nobody asked
  for.
- Do not build a general-purpose function or class for something used once.
  Only factor out a function when it measurably removes duplication.
- On a modification request: keep existing object names, structure, and
  working statistical logic. Change only the part that was asked for. Do not
  fold in unrelated formatting or "improvements."
- Clean up variables or imports that your own edit made unused. Leave
  pre-existing dead code alone unless asked to remove it.

---

# Subject Information (moonBook)

Goal: a demographic table (Table 1) ready to drop into the manuscript.

- Default to `moonBook::mytable()`. When a group comparison is needed, use
  `mytable(group ~ ., data = df)` and report whatever test moonBook selects
  automatically (t-test/ANOVA or non-parametric for continuous variables,
  chi-square/Fisher for categorical) rather than forcing a fixed test. Check
  whether any cell is small enough to need Fisher's exact.
- Subject Info and Sample Info key on different units: a subject can have
  multiple samples (V1, V2, ...). Build the demographic table on
  subject-distinct rows, and always report sample count and subject count
  separately for any downstream analysis.
- Never impute missing clinical values silently. Show the missingness pattern
  (random vs. concentrated in one group) and get the handling approach
  confirmed before proceeding.
- Export via `moonBook::mycsv()` or a tidied version to xlsx. Numbers in the
  table must always match the sample size actually used in analysis (see
  "Output verification").

```r
moonBook::mytable(group ~ ., data = subject_df, method = 2)
```

---

# Clinical metadata handling

- Treat the raw clinical file as read-only. Put derived variables (recoded
  factors, regrouped categories) in a separate object (`clin_derived`).
- Never leave factor level order to chance — set it explicitly
  (`factor(x, levels = c(...))`) so the reference level is the intended
  control group.
- For longitudinal data, keep `SubjectID` and `VisitID` (V1/V2, ...) as
  separate columns. Reshape only with `tidyr::pivot_longer()` /
  `pivot_wider()` — no manual reshaping.
- Label numeric clinical codes (e.g. sex 1/2, GDM 0/1) against the data
  dictionary explicitly. Do not analyze on bare numeric codes.

---

# Batch information and confounding

Merge sequencing run, extraction batch, library-prep date, and technician from
the wet-lab log into the sample metadata. Before any modeling, check sample counts,
depth distribution, and ordination separation per batch. Cross-tabulate batch
× the clinical group of interest — if they are confounded, say so explicitly
to the user rather than silently adjusting for batch as if the confound were
resolvable.

## Batch confounding table (always produce this before any decontam or model)

For every batch variable (extraction batch, library batch, sequencing run),
build one summary table with a row per batch level:

| Batch | n samples | n comparison groups present | n institutions | Sex ratio (M:F) | Notes |
|---|---|---|---|---|---|
| Batch_1 | ... | ... | ... | ... | e.g. "single institution, single group — cannot separate from group effect" |
| Batch_2 | ... | ... | ... | ... | ... |

Build this with `dplyr::count()`/`tidyr::pivot_wider()` from the sample metadata,
never by hand. The three columns are non-negotiable because they are the most
common hidden confounders in clinical microbiome studies: how many comparison
groups actually appear in that batch, which institution(s) contributed samples
to it, and how skewed the sex ratio is relative to the full cohort.

**Warn explicitly, in plain language, whenever:**
- A batch contains only one comparison group (batch and group are
  perfectly confounded — no model can separate their effects).
- A batch maps 1:1 to a single sampling institution (batch and site are
  indistinguishable).
- Sex ratio in a batch deviates sharply from the overall cohort ratio.

Do not fold this warning into a footnote. State it as its own sentence, name
the batch level(s) involved, and say plainly that any batch-adjusted result
from that batch is not interpretable as a pure batch effect.

---

# R coding style

Priority: code that is **re-usable and understandable months later**, not code
that merely runs once.

- **Tidyverse first.** Use `dplyr`, `tidyr`, `purrr`, `tibble`, `stringr` for
  data wrangling and iteration. Replace `for`/`apply`/manual list accumulation
  with `map()`, `map_dfr()`, `pmap_dfr()` only when it is genuinely clearer;
  otherwise base-R loops are fine when no dedicated function or map variant
  fits.
- **Domain functions first.** Use existing functions from `phyloseq`,
  `microbiome`, `mia`, `vegan`, `pairwiseAdonis`, `MicrobiomeStat`, etc. rather
  than hand-rolling something the package already provides.
- **Namespace explicitly.** Prefix `dplyr`, `tidyr`, `purrr`, `tibble` calls
  with `{package}::{function}` (adjust the convention per project if other
  packages should also be prefixed).
- **Multi-way repetition** (distance × dataset × variable, etc.) — build the
  combination grid with `tidyr::crossing()` and iterate with
  `purrr::pmap_dfr()`. Don't hand-write multiple near-duplicate for-loops for
  the same logic.
- **No one-off functions.** Don't wrap single-use computations in a
  general-purpose function, framework, or class. Factor out only when it
  measurably reduces duplication.
- **Intermediate objects only when reused.** Don't name and store a pipeline
  step that is used exactly once and then discarded.
- **Fix reproducibility.** Call `set.seed(42)` immediately before any
  permutation-based analysis (e.g. PERMANOVA). Use BH-FDR
  (`p.adjust(method = "BH")`) for multiple-testing correction unless told
  otherwise.

---

# Pre-output checklist

Before producing or running code, confirm:

- [ ] Statistical assumptions and code implementation are explained separately
- [ ] Method trade-offs were stated and scope narrowed to one primary + one sensitivity analysis where a choice was needed
- [ ] Filtering/normalization/model choices follow pre-specified rules, not significance
- [ ] Feasibility of proposed preprocessing/normalization was checked against the actual input

# Output verification checklist (every stage)

Treat every stage as an implement-then-verify pair. Do not report a stage
"done" without running its verification.

- [ ] Sample count / independent subject count / feature count reported, where applicable
- [ ] Missingness and per-factor-level counts reported
- [ ] Actual sample and subject counts used in every model reported after missing-data handling
- [ ] Feature/sample counts before vs. after preprocessing recorded
- [ ] Batch confounding table produced (n groups / n institutions / sex ratio per batch), with explicit warnings
- [ ] Where applicable, contamination-removal and batch-correction results compared before/after
- [ ] Additional assay-specific verification completed

---

# Caching and file management

- Save every pipeline stage (preprocessing, QC, decontam, beta-diversity, ...)
  as an `.rds` (`results/0X_step/step_output.rds`).
- On re-run, load the existing `.rds` instead of recomputing. Require an
  explicit flag (e.g. `force_recompute = TRUE`) for the user to force
  recomputation.
- If a modification request only touches part of the pipeline, load the
  earlier stages' `.rds` as-is and recompute only from the affected stage
  onward — never a full re-run by default.
- Save final results as one Excel workbook (statistics, organized by sheet)
  plus figure files (PCoA, etc.), with the filename including a date or
  analysis version.

---

# Project-Specific Guidelines

<!-- Add study-specific exceptions below this line only, e.g. negative
control sample IDs, the exact batch variable name, the primary outcome
variable. Do not edit the sections above. -->
