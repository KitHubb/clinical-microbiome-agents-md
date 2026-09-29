# Shotgun Metagenomics Core Downstream Analysis Guide

Core R workflow for Kraken2/Bracken, MetaPhlAn 4, and HUMAnN outputs. Follow the common [AGENTS.md](../AGENTS.md). Use `microeco` and `microeco::microtable` for the main taxonomic workflow. Use other object classes only when a required method needs them; preserve feature IDs, sample IDs, and abundance units during conversion.

Use [HTML_REPORT_AGENTS.md](../HTML_REPORT_AGENTS.md) for report style, layout, theme, and table UI. This guide covers routine downstream analysis; it does not define read preprocessing, assembly, MAG reconstruction, or every possible shotgun analysis.

# 1. Scope

- Perform R-based downstream analysis of Kraken2, Bracken, MetaPhlAn 4, and HUMAnN outputs.
- Process, statistically analyze, and visualize taxonomic and functional profiles.
- Follow this sequence: validate inputs → join metadata → create analysis objects → preprocess → analyze → visualize → save results.
- Adapt the analysis design to project-specific groups, covariates, paired or repeated measurements, collection sites, time points, and batches.
- Do not apply amplicon-specific ASV, rarefaction, or decontam rules to shotgun profiles without evidence that the rule is valid for the input data.

# 2. Project-specific configuration

Record before analysis:

- Input files and the generating tool and version
- Reference database and version
- Mapping between metadata and sample IDs
- Primary comparison, reference group, and analysis subset
- Subject ID, repeated measurements, paired samples, collection site, time point, and batch variables
- Available sequencing-depth and QC information
- Prespecified covariates and analysis objectives
- Output paths, file names, and the scope of existing results to preserve

Rules:

- Do not hard-code project-specific paths, group names, or covariates in common analysis code.
- Write project-specific values and file paths directly where they are used;
  do not create a config list or path registry solely for indirection.
- Do not infer paired relationships, batches, or covariates that were not provided.
- Record metadata discrepancies separately without modifying the raw metadata.
- Obtain user confirmation before fixing a filtering threshold, normalization method, pseudocount, model covariate, or batch-correction method.

# 3. Coding principles

- Use `dplyr` and `tidyr` as the primary data-wrangling tools.
- Consider a function when the same operation is used at least three times.
- Avoid unnecessary functions and abstraction.
- Prefer official import, analysis, and plotting functions from `microeco` and other domain packages over hand-written replacements.
- Keep transformations between raw tables and analysis objects traceable.
- Distinguish raw data, analysis data, and plotting data.
- Preserve count, relative-abundance, and transformed-abundance representations separately.
- Perform the input checks needed for the analysis; avoid unrelated defensive code.
- Record random seeds for stochastic procedures and record package versions.
- Use explicit namespaces for data-wrangling and analysis functions.

# 4. Input-specific handling

## MetaPhlAn 4

- Accept taxonomic profiles or merged abundance tables.
- Confirm the taxonomic lineage format and the rank to be analyzed.
- Construct an analysis matrix at one rank appropriate to the objective, such as species or genus.
- Do not sum parent and child ranks together; this double counts hierarchical profiles.
- Confirm the relative-abundance unit and column totals, and define how unclassified or unmapped entries are handled.
- Preserve the mapping between cleaned taxonomy labels and original feature IDs.
- Label species richness as `Observed species`.
- Do not describe MetaPhlAn features as ASVs, OTUs, or read counts.
- Do not convert relative abundance into arbitrary integer counts.

## Kraken2 and Bracken

- Distinguish Kraken2 reports from Bracken abundance estimates.
- Distinguish Kraken2 clade counts from taxon-direct counts.
- Record whether Bracken was used and the rank at which abundance was estimated.
- Extract features from one rank when constructing an analysis matrix.
- Do not sum hierarchical clade counts across ranks.
- Distinguish classified reads, unclassified reads, and the denominator used for analysis.
- State whether values are read-assignment counts or estimated abundances.
- When calculating relative abundance, explicitly define whether the denominator is all reads or the reads within the selected analysis scope.

## HUMAnN

- Treat gene-family abundance, pathway abundance, and pathway coverage as separate datasets.
- Confirm the units of raw and normalized abundance.
- Record the feature system, such as UniRef, and its annotation version.
- Separate unstratified totals from taxon-stratified contributions.
- Do not add totals and contributions together.
- Record the handling of special features such as `UNMAPPED` and `UNINTEGRATED`.
- When regrouping gene families, save a mapping between original IDs and the resulting functional IDs.
- Do not treat pathway coverage as counts or abundance.

# 5. Metadata and QC

## Sample linkage

- Check sample IDs for missing values, duplicates, and mismatches.
- Match sample order between the abundance matrix and metadata.
- Preserve sample IDs, subject IDs, and feature IDs explicitly.

## Sample QC

Summarize available fields:

- Raw reads and post-QC reads
- Reads remaining after host removal
- Classified and unclassified read counts or proportions
- Number of detected features and total abundance
- Excluded samples and exclusion reasons
- Distributions by batch, collection site, time point, and processing condition

## Subject information

- Keep the subject-characteristics table separate from the sample table.
- Do not count repeated samples from the same subject as independent subjects.
- Define project-specific rules for inconsistent subject information such as age or sex.
- Distinguish repeated measurements, technical replicates, and biological replicates.
- Decide how technical replicates will be handled before analysis.

## Sequencing depth

- Evaluate sequencing depth using actual read counts or valid QC metrics.
- Do not use the sum of relative abundances as sequencing depth.
- Do not infer read depth when only relative abundance is available.
- Evaluate the relationship between richness and sequencing depth only when valid depth information is available.

# 6. Analysis objects and data preservation

Use `microeco::microtable` as the primary object for taxonomic analysis. Keep the feature matrix, sample metadata, and annotation table available outside the object so that units and transformations remain auditable.

Use another object only when required by a selected method:

- `mia`: `TreeSummarizedExperiment` or a supported `SummarizedExperiment` class
- `phyloseq`: `otu_table`, `tax_table`, and `sample_data`
- Functional analysis: feature matrix, sample metadata, and annotation table

Preserve:

- Original abundance data
- Preprocessed abundance data
- Sample metadata
- Taxonomic or functional annotation
- Filtering, normalization, and transformation settings
- Records of excluded samples and features

After each conversion, check:

- Feature × sample orientation
- Sample and feature order and IDs
- Abundance unit
- Taxonomic rank
- Handling of missing values and zeros
- Whether the conversion changes data precision or feature labels

Avoid repeated object-class conversion.

# 7. Preprocessing and analysis scope

- Set prevalence and abundance filters according to the analysis objective.
- Record filtering criteria and sample and feature counts before and after each filtering step.
- Apply consistent preprocessing rules across comparable subsets.
- When restricting analysis to a kingdom or taxonomic group, state whether the restricted profile is renormalized.
- Distinguish a true zero-detection sample from a sample with missing data.
- Define handling of zero-sum samples for each analysis.
- Choose transformation and pseudocount rules according to the selected method.
- Separate display rules for top taxa and `Others` from statistical filtering.
- Do not select filters or transformations according to which choice produces a significant result.

# 8. Exploratory analysis and batch assessment

- Inspect sample-level abundance distributions and detected-feature counts.
- Tabulate sample counts by group, batch, site, and time point.
- Explore major variation using ordination.
- Add density plots or boxplots of ordination coordinates only when they aid interpretation.
- Assess confounding between the primary group and batch, site, or other design variables.
- Do not treat visual separation alone as proof of a batch effect.
- Choose batch correction or covariate adjustment according to study design and data type; do not apply either silently.
- Consider contamination assessment when negative-control, blank, or DNA concentration information is available.
- Do not classify contamination solely from a fixed list of taxa.

# 9. Alpha diversity

## Index selection

- Select indices appropriate to the analysis objective, such as Observed features, Shannon, and Simpson.
- Label observed richness at the analyzed rank, such as `Observed species` or `Observed genera`.
- Distinguish functional-feature diversity from taxonomic diversity.
- Use phylogenetic diversity only when a valid phylogenetic tree maps correctly to the analyzed features.

## Statistical analysis

- Distinguish independent samples from paired or repeated samples.
- For two independent groups, consider the Wilcoxon rank-sum test or another prespecified appropriate test.
- For two paired groups, consider the Wilcoxon signed-rank test or another paired method.
- For more than two independent groups, consider the Kruskal–Wallis test and an appropriate post-hoc comparison.
- For repeated measurements, use a design-appropriate method such as a mixed-effects model.
- Define covariates and random effects from the prespecified analysis design.
- Use model-appropriate contrasts, such as estimated marginal means, for adjusted comparisons.
- Do not apply an independent-sample test directly to repeated measurements.
- Do not select one repeated sample arbitrarily to manufacture independence.

## Visualization

- Prefer package-provided boxplots, violin plots, or equivalent defaults.
- Keep per-index results and plot-ready data in memory, then consolidate only
  the requested final statistics and figures.
- Preserve index and group order across comparable subsets.
- Show paired relationships when useful and supported by the design.

# 10. Beta diversity

## Distance and ordination

- Select Bray–Curtis, Jaccard, Aitchison, or another distance according to the data representation and analysis objective.
- State the detection threshold for presence/absence analysis.
- For Aitchison analysis, document zero handling and the log-ratio transformation.
- Match the ordination method, such as PCoA or PCA, to the distance or transformation.
- Add group ellipses or marginal plots only when useful.
- Keep ordination coordinates, axis labels, and explained variance available
  in memory for verification; export them only when they are part of the
  requested final deliverable.

## PERMANOVA

- State the analysis unit and distance measure.
- Include the primary comparison and prespecified covariates in the model.
- Use a permutation scheme appropriate for repeated or paired designs.
- Decide whether `strata` is appropriate based on whether the target effect is within-subject or between-subject.
- Do not apply `strata = subject_id` automatically to every repeated-measures analysis.
- Record the number of permutations, test specification, and multiple-testing family.
- Save R², the test statistic, raw p-value, and adjusted p-value.

## Dispersion

- Use `betadisper` or another suitable assessment of dispersion.
- Match permutation restrictions and the sampling unit to the comparison design.
- Distinguish a compositional-location difference from a dispersion difference in interpretation.

# 11. Taxonomic composition

- Define the analysis rank and the taxonomic scope to display.
- Distinguish sample-level composition from group-level composition.
- Apply facets for group, site, time point, or other metadata when needed.
- Preserve facet variables and group order within a comparable analysis.
- Select displayed taxa using a prespecified mean-abundance, prevalence, or top-N rule.
- Combine the remaining displayed taxa as `Others`.
- Store display criteria in the project configuration.

Group-level composition tables:

- Include the same taxa and `Others` category as the figure legend.
- Present relative abundance (%) in a taxa × group table.
- Decide explicitly whether values are sample means or subject-level means.
- Verify that group totals agree with plotted values.
- Consider unequal weighting when subjects contribute different numbers of samples.

# 12. Functional profiling

## Analysis scope

- Gene-family abundance
- Pathway abundance
- Pathway coverage
- Regrouped functional categories when required

## Rules

- Preserve the input unit and normalization for each functional dataset.
- Use unstratified profiles for overall functional comparisons.
- Use stratified profiles for taxon-contribution analysis.
- Keep pathway-abundance and pathway-coverage results separate.
- Construct matrices with comparable samples and feature definitions.

## Visualization

- Abundance plots for selected functions
- Heatmaps
- Ordination of functional profiles
- Taxon-stratified contribution plots
- Coefficient or effect-size plots for functional association results

Keep plotting scales separate from statistical transformations.

# 13. Differential abundance and association

- Verify that the input representation satisfies the selected method''s assumptions.
- Distinguish count-based methods from relative-abundance methods.
- Do not convert relative abundance to arbitrary pseudo-counts for a count-based method.
- Confirm support for the independent, paired, or repeated design and required covariates.
- Record prevalence and abundance filtering criteria.
- State the reference group and contrast direction.
- Save effect sizes or coefficients, uncertainty estimates, and adjusted p-values.
- Select the method according to the data type and research question.
- Do not use a method such as ANCOM-BC2 as the default for every input type.
- Keep taxonomic differential-abundance analysis separate from functional association analysis.

# 14. Profiling-tool comparison

- Use the same samples and comparable taxonomic ranks.
- Reconcile taxonomy names and IDs while retaining database-specific distinctions.
- Align unclassified handling and relative-abundance denominators.
- Preserve differences in how each tool defines abundance.
- Compare shared detected taxa, compositional agreement, diversity, and association results.
- Do not interpret a tool-specific non-detection as confirmed biological absence.
- Analyze taxonomic profiles and HUMAnN functional profiles separately before linking them.
- When comparing with MAG-based results, record reference, mapping, and quantification criteria.

# 15. Analysis outputs

## Statistical result tables

- Use stable machine-readable column names.
- Store exact sample and independent-subject counts.
- Store raw and adjusted p-values in separate columns.
- Record the multiple-testing family.
- Preserve the comparison, reference group, contrast, and effect direction.
- Pass adjusted p-values to the report without adding display-only symbols to the analysis table.

## Plot-ready data

- Do not save a separate plot-data file for every figure. Keep plot-ready data
  in memory and include it only when the requested final workbook requires it.
- Preserve group order, feature order, abundance unit, and taxonomic or functional level.
- Record display-only aggregation, scaling, and `Others` handling.
- Keep plot styling, significance symbols, fonts, colors, themes, and layout in the HTML report guide.
# 16. Saving and verification

Save only final deliverables:

- One consolidated workbook containing final statistical results, sample QC,
  exclusions, software versions, and required annotation mappings
- Final figures
- The final report or final analysis object only when required by the request

Do not persist preprocessing objects, per-index files, plot-ready data, or
other intermediate artifacts solely for caching or audit purposes.

Verify:

- Sample and feature IDs and ordering
- Separation of count, relative abundance, and coverage units
- No double counting of hierarchical taxonomy or stratified functional data
- Correct handling of repeated and paired relationships
- Correct multiple-testing family and contrast direction
- Agreement among figures, tables, and analysis objects
- Preservation of settings required to reproduce the analysis
- Actual sample and independent-subject counts used in every statistical model

# 17. HTML report handoff

Provide:

- Input tool, database, and analysis scope
- Sample and subject counts and exclusion records
- Analysis indices, taxonomic rank, and preprocessing
- Statistical methods, covariates, repeated-measures handling, and multiple-testing correction
- Main results and test statistics
- Data and study-design limitations
- Paths to final figures, tables, and reports

Use [HTML_REPORT_AGENTS.md](../HTML_REPORT_AGENTS.md) for Korean prose, theme, layout, table UI, rendering, and visual checks.

# Project-Specific Settings

<!-- Add project-specific input paths, tool/database versions, feature units,
group definitions, reference levels, subset rules, covariates, repeated-measure
structure, batch variables, filtering decisions, and output paths below this
line. Do not edit the reusable sections above. -->
