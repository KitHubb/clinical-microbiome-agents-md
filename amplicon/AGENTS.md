# AGENTS.md — Amplicon Analysis with phyloseq

Apply this document only to amplicon analysis. Read and follow the common
[AGENTS.md](../AGENTS.md) first, including Subject Information (moonBook),
clinical metadata handling, batch confounding, coding style, output
verification, and output management. The sections below preserve the original
phyloseq-based workflow; common sections have moved to the parent document.

# Amplicon-specific decisions and statistical explanation

- Fix rarefaction depth and abundance-filtering thresholds only with user confirmation.
- For contamination assessment or removal, follow [DECONTAM.md](DECONTAM.md).
  Confirm whether negative controls exist and obtain approval before choosing
  or running any decontam method.
- Keep statistical assumptions and code implementation visibly separate — e.g.,
  state that PERMANOVA does not assume homogeneity of dispersion but that you
  will still check it with betadisper(), before writing the adonis2() call.

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

Verify each stage before continuing, but keep stage objects in memory rather
than saving separate intermediate `.rds` files. Save only the final workbook,
figures, report, or final analysis object required by the request (see
"R/Rmd working files and output management" in the parent document).

---

# Goal-driven verification

Treat every stage as an implement-then-verify pair. Do not report a stage
"done" without running its verification. Minimum bar per stage:

| Stage | Success criteria |
|---|---|
| Subject Info | Subject count matches the clinical metadata row count; missingness listed by variable |
| Sample / Decontam | Taxa count before vs. after decontam; method justified by presence/absence of negative controls |
| Base phyloseq analysis | Actual sample count used in each model; results compared with and without batch correction |

---

# Sample Info

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

Follow "Batch information and confounding" in [../AGENTS.md](../AGENTS.md).
Merge the wet-lab batch fields into phyloseq sample_data, and build the
required batch confounding table from sample_data before decontam or modeling.
Present its warnings before the approval gate in [DECONTAM.md](DECONTAM.md).

## Decontam

All method-selection rules, approval gates, threshold sensitivity, cross-tool
validation, and decontam-specific verification are in
[DECONTAM.md](DECONTAM.md). Do not run contamination removal until its approval
requirements are satisfied.

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

# Batch effect — checklist

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

For contamination-removal checks, use the dedicated checklist in
[DECONTAM.md](DECONTAM.md).

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

# Output verification checklist (every stage)

- [ ] Sample count / subject count / taxa count reported
- [ ] Missingness and per-factor-level counts, with **independent subject
      count** kept distinct from sample count
- [ ] Actual sample count used in each statistical model (post-missingness)
- [ ] Taxa/sample counts before vs. after preprocessing
- [ ] Where applicable: batch-effect significance; decontam verification per
      [DECONTAM.md](DECONTAM.md)
- [ ] Batch confounding table produced (n groups / n institutions / sex ratio
      per batch), with explicit warnings for any batch that cannot be
      separated from group or institution

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
- Keep stage objects in memory; save only the requested final deliverables
```

---

# Project-Specific Guidelines

<!-- Add study-specific exceptions below this line only, e.g. negative
control sample IDs, the exact batch variable name, the primary outcome
variable. Do not edit the sections above. -->
