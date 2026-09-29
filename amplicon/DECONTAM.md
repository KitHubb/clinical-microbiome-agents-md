# Decontam Guidelines for Amplicon Analysis

Apply this document when contamination assessment or removal is in scope for
an amplicon analysis. Read and follow [AGENTS.md](AGENTS.md) first, especially
the depth, batch-confounding, output-management, and verification requirements.

# Approval gate and method selection

Confirm whether negative controls exist and obtain user confirmation before
choosing or running a decontam method. Propose one method from the table below;
do not apply it without confirmation.

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

Before running any threshold sweep or cross-tool comparison below, stop and
get explicit confirmation on both of the following. Do not proceed on an
assumed "yes."

1. **Batch balance report reviewed, and a decision on batch-stratified
   decontam.** Present the batch confounding table required by
   [../AGENTS.md](../AGENTS.md), including any warnings, and ask whether
   decontam should be run per batch (e.g. separate negative-control pools per
   sequencing run) or on the pooled dataset.
2. **Scope of the threshold sensitivity report.** Confirm the user wants the
   full sweep below (all 11 thresholds × alpha/beta/taxa) rather than a
   single default-threshold run — it is computationally heavier and produces
   a large comparison output.

# Decontam threshold sensitivity

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
  or significance of the main PERMANOVA result) flip — that range is the one
  worth discussing with the user, not the full 11-row table on its own.

# Cross-validation against ScruB / MicrobIEM

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

# Decontam verification checklist

- [ ] Confirmed whether negative controls (extraction/PCR blanks) exist and
      obtained approval for the method selected from the table above
- [ ] Batch confounding table reviewed, with warnings stated for any batch
      that is not separable from group or institution
- [ ] User decided whether decontam is batch-stratified or pooled
- [ ] Logged taxa count, flagged taxa identity, alpha diversity, and major
      taxon composition before/after decontam
- [ ] User confirmed the scope before any threshold sweep
- [ ] `decontamSensitivity` sweep run across the 11 thresholds if requested,
      with alpha/beta (PERMANOVA)/Genus-taxa (≥1% mean abundance) comparisons
- [ ] Any threshold range where the main PERMANOVA conclusion flips was
      flagged explicitly
- [ ] ScruB/MicrobIEM cross-check run and compared to the main decontam result,
      if either tool is installed
- [ ] Decontam statistics required for reporting are included in the final
      deliverables; no separate intermediate decontam object is retained

# Project-Specific Guidelines

<!-- Add study-specific decontam exceptions below this line only, e.g.
negative-control sample IDs, concentration variable, selected method, approved
threshold, and whether processing is batch-stratified or pooled. -->
