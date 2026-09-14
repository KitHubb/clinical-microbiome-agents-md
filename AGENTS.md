
# AGENTS.md — Clinical Microbiome R Analysis Harness

You are a coding agent (Codex, Claude Code, or similar) working with a clinical
microbiome / metagenomics researcher on paper-grade analyses in R, built on the
`phyloseq` ecosystem and `tidyverse`. Your job is to carry out the analysis
pipeline below the way an experienced bioinformatics collaborator would: verify
before you decide, keep code minimal and re-readable months later, and never let
a threshold, filter, or model choice quietly steer the result toward
significance.

Drop this file as `AGENTS.md` at the project root. For study-specific exceptions
(negative control IDs, batch variable names, primary outcome), add them under
"Project-Specific Guidelines" at the bottom — do not edit the sections above it.

---

# Data and pipeline overview

Every project has two input sources:

| Source | Contents | Format |
|---|---|---|
| **Clinical metadata** | Subject demographics, diagnoses, clinical variables (e.g. GDM, BMI, Sex), visit number (V1/V2), SubjectID | xlsx/csv |
| **Wet-lab log** | Collection date, DNA extraction QC, library concentration, technician, sequencing run/batch ID, index | xlsx/csv |

These merge into a fixed three-stage pipeline. Do not skip or merge stages, and
do not start a later stage on anything but a validated (§ Output verification)
artifact from the stage before it.

```
[1] Subject Information      [2] Sample Info → Decontam decision      [3] Base phyloseq analysis
    (moonBook descriptives) →    (depth / batch check → decontam)  →     (main effect / batch effect / confounding)
```

Save each stage's output as an `.rds` under a stage-numbered folder
(`results/01_subject/`, `results/02_sample_decontam/`,
`results/03_phyloseq_base/`). On re-run, load the existing `.rds` instead of
recomputing (see "Caching and file management").

---

# When to ask the user for permission

Proceed without asking when you are implementing an already-approved plan,
doing read-only exploration (EDA, depth distributions, cross-tabs), fixing a bug
in existing code, or producing a re-runnable intermediate artifact.

Stop and ask first when a decision would change what "significant" means for
the paper:

- Fixing a rarefaction depth or an abundance-filtering threshold.
- Choosing a decontam method (frequency / prevalence / combined) or confirming
  whether negative controls exist at all.
- Deciding which clinical variables enter a model as covariates.
- Renaming or restructuring existing analysis objects.

Never pick a plausible-looking default and move on quietly. When there is a
real choice, state the trade-offs in one or two lines, then implement exactly
one primary analysis and one sensitivity analysis — not every option on the
table.

# Think before coding

Before writing code, write out the strategy in plain language: analysis goal →
input (which phyloseq object, which subset) → output (which statistics, which
plots) → verification checklist. If the user asked only for a strategy, do not
output code yet. Keep statistical assumptions and code implementation visibly
separate — e.g., state that PERMANOVA does not assume homogeneity of dispersion
but that you will still check it with `betadisper()`, before writing the
`adonis2()` call.

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

# Goal-driven verification

Treat every stage as an implement-then-verify pair. Do not report a stage
"done" without running its verification. Minimum bar per stage:

| Stage | Success criteria |
|---|---|
| Subject Info | Subject count matches the clinical metadata row count; missingness listed by variable |
| Sample / Decontam | Taxa count before vs. after decontam; method justified by presence/absence of negative controls |
| Base phyloseq analysis | Actual sample count used in each model; results compared with and without batch correction |

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

# Sample Info and the decontam decision

## Depth / library size (must be shown before any threshold is fixed)

```r
depth_df <- ps |> phyloseq::sample_sums() |>
  tibble::enframe(name = "SampleID", value = "depth")
```

Show the depth distribution (summary stats + histogram) first. Do not let code
auto-select a rarefaction depth — expose it as a single constant the user can
edit on one line. For any sample with unusually low depth, cross-check the
wet-lab log for a technical explanation (failed extraction, low-concentration
library, suspected contamination) before treating it as biological.

## Batch info

Merge sequencing run, extraction batch, library-prep date, and technician from
the wet-lab log into `sample_data`. Before any modeling, check sample counts,
depth distribution, and ordination separation per batch. Cross-tabulate batch
× the clinical group of interest — if they are confounded, say so explicitly
to the user rather than silently adjusting for batch as if the confound were
resolvable.

### Batch confounding table (always produce this before any decontam or model)

For every batch variable (extraction batch, library batch, sequencing run),
build one summary table with a row per batch level:

| Batch | n samples | n comparison groups present | n institutions | Sex ratio (M:F) | Notes |
|---|---|---|---|---|---|
| Batch_1 | ... | ... | ... | ... | e.g. "single institution, single group — cannot separate from group effect" |
| Batch_2 | ... | ... | ... | ... | ... |

Build this with `dplyr::count()`/`tidyr::pivot_wider()` from `sample_data`,
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

## Decontam decision

Propose a method from the table below; do not apply one without user
confirmation.

| Condition | Recommended method |
|---|---|
| Negative control present, DNA quant available | `method = "combined"` (frequency + prevalence) |
| Negative control present, no quant | `method = "prevalence"` |
| No negative control, quant available | `method = "frequency"` only — flag the limitation |
| Neither negative control nor quant | decontam not applicable — offer manual prevalence filtering of known reagent-contaminant taxa as the only alternative |

Do not change the default `threshold` (0.1) silently. If you do change it,
report the number of taxa flagged as contaminants and their identity
(genus/family level) so the change is auditable. Compare alpha diversity and
major taxon composition before vs. after decontam so the researcher can judge
whether removal was too aggressive.

## Gate: confirm before running decontam threshold sensitivity

Before running any threshold sweep or cross-tool comparison below, stop and
get explicit confirmation on both of the following. Do not proceed on an
assumed "yes."

1. **Batch balance report reviewed, and a decision on batch-stratified
   decontam.** Present the batch confounding table above, including any
   warnings, and ask whether decontam should be run per batch (e.g. separate
   negative-control pools per sequencing run) or on the pooled dataset.
2. **Scope of the threshold sensitivity report.** Confirm the user wants the
   full sweep below (all 11 thresholds × alpha/beta/taxa) rather than a
   single default-threshold run — it is computationally heavier and produces
   a large comparison output.

## Decontam threshold sensitivity

Use the `decontamSensitivity` package for the sweep — do not hand-roll a loop
over `decontam::isContaminant()` calls, since `decontamSensitivity` already
standardizes the before/after comparison structure.

- **Thresholds to test:** 0.01, 0.05, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8,
  0.9.
- **For each threshold, report before vs. after:**
  - **Alpha diversity** — paired comparison of the chosen index(es) pre- vs.
    post-removal at that threshold.
  - **Beta diversity** — PERMANOVA on the primary analysis model (the same
    formula as the main analysis in "Base phyloseq analysis"), before vs.
    after, so the sensitivity result is directly comparable to the paper's
    main statistic.
  - **Taxa** — restrict to Genus-level taxa with mean relative abundance
    ≥ 1%, and visualize how their abundance shifts across thresholds (e.g. a
    line or heatmap of abundance vs. threshold, one panel per taxon or a
    faceted plot).
- Summarize the full sweep in one table (threshold × metric) plus the
  taxon-level plot, and flag any threshold range where conclusions (direction
  or significance of the main PERMANOVA result) flip — that range is the
  one worth discussing with the user, not the full 11-row table on its own.

## Cross-validation against ScruB / MicrobIEM

Before finalizing the decontam-based result, check whether `ScruB` or
`MicrobIEM` is installed in the environment. If either is available:

- Run it with its **default parameters** (do not tune it) on the same input.
- Compare its contaminant call / decontaminated table against the main
  `decontam` result: overlap in flagged taxa, and the same alpha/beta/taxa
  before-after comparison used in the threshold sensitivity section above.
- Present this as a secondary cross-check next to the main decontam result,
  not as a replacement for it, unless the user asks you to switch the primary
  method.
- If neither tool is installed, say so plainly and skip this step — do not
  attempt to install packages from unapproved sources on your own.

---

# Base phyloseq analysis (main effect / batch effect / confounding)

Run in this order on the decontam-cleaned phyloseq object, every time, before
any manuscript-facing analysis:

1. **Descriptives** — taxa count, sample count, subject count, per-group
   sample counts, depth summary.
2. **Batch effect on its own** — PERMANOVA + `betadisper()` with batch as the
   sole predictor of beta-diversity. If significant, discuss with the user
   whether to include batch as a covariate downstream or apply batch
   correction (e.g. `ConQuR`, `MMUPHin`) — never apply correction silently.
3. **Main effect (clinical variable of interest)**:
   - Crude model: the variable alone.
   - Adjusted model: batch + pre-specified confounders (age, BMI, delivery
     mode, etc.), using `by = "margin"` for the marginal test.
4. **Confounding check** — cross-tabulate/correlate the variable of interest
   against candidate covariates before modeling, to catch collinearity or
   complete confounding.
5. Report the **actual** sample/subject count used in every model (missing
   data can shrink it below the full cohort).

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

# Microbiome preprocessing and QC

Fixed order, every time:

1. Remove empty samples/taxa.
2. Filter low-abundance ASVs/taxa (e.g. total abundance ≤ N).
3. Show the sequencing-depth distribution.
4. Rarefaction depth is a user decision, not an automated one.

Remove singletons (taxa with a total count of 1) at preprocessing and note why
(sequencing-error artifact). Branch preprocessing by metric: count/presence-
based metrics (Bray–Curtis, Jaccard) need rarefaction; CLR-based metrics
(Aitchison) do not and should not reuse a rarefied table. Log taxa/sample
counts before vs. after every filtering step (see "Output verification").

---

# Phyloseq object handling (hard rules)

1. **Never use `subset_samples()` inside a custom function — use
   `prune_samples()`.** NSE (non-standard evaluation) inside `subset_samples()`
   breaks unpredictably when called from within a function.
2. **Check `otu_table()` orientation every time.** Whether taxa are rows or
   columns varies by object; check at the top of every function.
   ```r
   if (!phyloseq::taxa_are_rows(ps)) {
     otu <- t(phyloseq::otu_table(ps))
   } else {
     otu <- phyloseq::otu_table(ps)
   }
   ```
3. **Never convert `otu_table()`/`sample_data()`/`tax_table()` with
   `as.data.frame()` — use `data.frame()` only.** `as.data.frame()` on these S4
   classes frequently drops attributes silently.
4. **Fix the `X`-prefix bug after any matrix → data.frame conversion.** R
   prepends `X` to sample/taxon names that start with a digit. Reverse it right
   after conversion:
   ```r
   df <- df |> dplyr::rename_with(~ stringr::str_replace(.x, "^X", ""))
   ```
5. Do not re-derive rules 1–4 in every function. Write project-level helpers
   once (`get_otu_df(ps)`, `get_sample_df(ps)`) and reuse them — this is the
   one case where factoring out a function is justified (it removes real
   duplication).

---

# Batch effect and contamination — checklist

- [ ] Confirmed whether negative controls (extraction/PCR blanks) exist →
      decontam method decided per the table above
- [ ] Batch confounding table built (n comparison groups / n institutions /
      sex ratio per batch level), with warnings stated for any batch that
      is not separable from group or institution
- [ ] Cross-tabulated batch (extraction date / library-prep date / sequencing
      run / technician) against the group of interest to check confounding
- [ ] Ran PERMANOVA + `betadisper()` on batch alone for beta-diversity
- [ ] If batch is significant and not confounded with group: included batch as
      a covariate in the adjusted model (if confounded: reported the
      limitation instead of adjusting)
- [ ] Batch-correction algorithms (e.g. ConQuR) applied only on explicit
      request, always shown before/after in ordination to catch over-correction
- [ ] Logged taxa count and major taxon composition before/after decontam
- [ ] User confirmed both gate items before any threshold sweep: batch
      balance reviewed + batch-stratified decontam decision, and scope of the
      sensitivity report
- [ ] `decontamSensitivity` sweep run across the 11 thresholds if requested,
      with alpha/beta(PERMANOVA)/Genus-taxa (≥1% mean abundance) comparisons
- [ ] ScruB/MicrobIEM cross-check run and compared to main decontam result, if
      either tool is installed

---

# Study design: cross-sectional vs. longitudinal

Classify the data structure before choosing a statistical strategy.

| Structure | Characteristics | Strategy |
|---|---|---|
| Cross-sectional (single visit, or V1/V2 each analyzed independently) | One sample per subject | Standard PERMANOVA/betadisper; subject-independence assumption holds |
| Longitudinal (same SubjectID across ≥2 visits) | Repeated measures, within-subject correlation | Restrict permutations within SubjectID (`strata =`), or use GLMM/LME. Running plain PERMANOVA risks pseudo-replication |

- Do not test visit-invariant baseline variables (e.g. GDM) in a longitudinal
  model — a variable that doesn't change across visits has nothing for a
  longitudinal test to explain.
- Fix the inclusion rule (e.g. "≥2 visits required") before analysis, and
  report how many subjects it excluded.
- In plots, connect the same SubjectID across visits with a line; if there are
  many subjects, drop shape mapping and rely on the connecting line alone to
  avoid overplotting.

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
- [ ] If there was a choice of method, trade-offs were stated first, and scope
      was narrowed to one primary + one sensitivity analysis
- [ ] Filtering/normalization/model choices were not steered toward a
      significant result — they follow the pre-specified rules above (QC,
      decontam)
- [ ] Feasibility of rarefaction/normalization was checked against the actual
      data before committing to an approach

# Output verification checklist (every stage)

- [ ] Sample count / subject count / taxa count reported
- [ ] Missingness and per-factor-level counts, with **independent subject
      count** kept distinct from sample count
- [ ] Actual sample count used in each statistical model (post-missingness)
- [ ] Taxa/sample counts before vs. after preprocessing
- [ ] Where applicable: decontam before/after comparison, batch-effect
      significance
- [ ] Batch confounding table produced (n groups / n institutions / sex ratio
      per batch), with explicit warnings for any batch that cannot be
      separated from group or institution
- [ ] Where a threshold sweep was run: `decontamSensitivity` output table
      (11 thresholds × alpha/beta/taxa) saved, and any threshold range where
      the main PERMANOVA conclusion flips flagged
- [ ] Where ScruB/MicrobIEM was available: cross-check result saved alongside
      the main decontam result

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

# Appendix: beta-diversity analysis prompt template

Use this structure for any new analysis type (alpha-diversity, differential
abundance, etc.): (1) target phyloseq object and constraints, (2) QC steps,
(3) preprocessing differences by metric, (4) analysis population
(cross-sectional/longitudinal), (5) statistical procedure, (6) visualization
rules, (7) output format.

```
ps_16S already contains only the samples in scope for this analysis. Do not
add an additional clinical-sample filter on your own.

[QC]
- Remove samples/taxa with zero counts
- Remove ASVs with total abundance <= 2
- Show the sequencing-depth distribution; do not auto-fix rarefaction depth
  (expose it as a one-line editable constant)
- Run PERMANOVA + betadisper on BatchID

[Distance]
- Bray-Curtis: rarefied counts
- Jaccard: rarefied presence/absence
- Aitchison: non-rarefied filtered counts, pseudocount 0.5, CLR then
  Euclidean distance

[Analysis population]
- V1 / V2 analyzed independently
- Serial subjects (>=2 visits) analyzed separately (permutations restricted
  within SubjectID; visit-invariant variables like GDM are not tested in the
  longitudinal model)

[Statistics]
- Univariate PERMANOVA per clinical variable; betadisper added for
  categorical variables
- GDM crude / adjusted (by = "margin") models
- BH q-value per distance metric

[Visualization]
- V1/V2: color = GDM, shape = Sex
- Serial: color = visit_group, connect same SubjectID with a line (drop shape
  if many subjects)
- Show explained variance on PCoA axes

[Coding style]
- phyloseq, microbiome, vegan, tidyverse first; crossing() + pmap_dfr() for
  distance x dataset x variable repetition
- seed = 42 fixed, p-values BH-FDR fixed
- Save each stage as .rds; reuse existing .rds on re-analysis
```

---

# Project-Specific Guidelines

<!-- Add study-specific exceptions below this line only, e.g. negative
control sample IDs, the exact batch variable name, the primary outcome
variable. Do not edit the sections above. -->
