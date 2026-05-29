---
name: dataset-analysis
description: Analyze, profile, sanity-check, plot, and produce a professional PDF report for an arbitrary dataset using safe lazy/sampled reads.
argument-hint: "Dataset path/URI plus optional context, output directory, variables, time range, or checks."
allowed-tools: ["find *", "du *", "ls *", "python *", "python3 *", "RIPGREP_CONFIG_PATH=* rg *", "rg *", "aws *", "./bin/aws *", "./bin/s5cmd *", "s5cmd *", "bazel run*", "bazel build*", "pdftoppm *", "pdfinfo *", "python -m pip show *", "write", "read"]
---

# Dataset Analysis

Use this skill when the user asks to analyze, inspect, profile, sanity-check, summarize, or make a report/plot for any dataset. Trigger examples include "analyze this dataset", "dataset whereabouts", "make a dataset report", "sanity check this zarr/parquet/netcdf/csv", or `/skill:dataset-analysis`.

## Goal

Produce a practical exploratory PDF report that answers: "What is in this dataset, where is it, what shape is it, and does it look sane?" The report should be brief but clear, with polished typography and readable figures.

## Inputs to clarify

If not already provided, ask for:

1. Dataset path or URI.
2. Expected format/domain, if known.
3. Output directory preference.
4. Whether large reads are allowed, or only lazy/sampled reads.
5. Any specific variables, time ranges, regions, or checks the user cares about.

If the dataset path and intended output are clear, proceed without asking.

## Workflow

1. Inspect the dataset safely.
   - Prefer lazy/indexed metadata reads.
   - Do not load the full dataset unless clearly small or approved.
   - Support common formats: zarr, netcdf, parquet, csv, json/jsonl, directories of files, images, numpy arrays, and torch checkpoints when relevant.

2. Build an inventory.
   - Format/store type.
   - File count and total size.
   - Dimensions, coordinates, variables/columns.
   - Dtypes, units, attrs/metadata.
   - Time range and cadence, if present.
   - Spatial bounds/resolution/grid, if present.
   - Chunking/compression, if applicable.

3. Run sanity checks.
   - Missing values, NaNs, infs.
   - Duplicate or non-monotonic coordinates/indexes.
   - Irregular time steps.
   - Constant/all-zero/all-NaN variables.
   - Min/max/mean/std on representative samples.
   - Suspicious dtype/shape inconsistencies.
   - Basic outlier checks for recognizable geophysical variables.
   - Any domain-specific checks implied by repo/project context.

4. Generate visual artifacts when useful.
   - One large overview matplotlib figure.
   - Histograms for representative variables.
   - Time coverage/cadence plot.
   - Spatial sample maps for gridded data.
   - Use Cartopy for latitude/longitude plots whenever showing geospatial maps.
   - Missingness/availability heatmap when applicable.
   - Any additional plot that reveals obvious data issues.
   - Optimize figure layout so subplots/subfigures use space efficiently with minimal blank whitespace while preserving readable labels, colorbars, and titles.

5. Write outputs.
   - `report.pdf`: primary human-readable report with text and figures.
   - `report.md`: optional source/companion Markdown report, if useful.
   - `inventory.json`: machine-readable inventory.
   - `checks.json`: machine-readable sanity-check results.
   - `plots/overview.png`: main plot, plus additional plots as needed.

6. Review and iterate on PDF quality.
   - After producing `report.pdf`, open/render it locally and inspect the pages visually.
   - Use tools such as `pdfinfo` to check page count and page sizes, and `pdftoppm -png` to render pages to images for inspection.
   - Look for formatting issues: text overflowing boxes, clipped labels, legends outside figures, unreadable font sizes, inconsistent page sizes, awkward page breaks, excessive whitespace, large blank gaps between subfigures, low-resolution images, or tables running off the page.
   - Fix the report generation/layout and regenerate until the PDF looks professional.
   - Do not claim completion until the rendered PDF has been inspected.

## Report structure

`report.pdf` should include:

1. Executive summary.
2. Dataset location and format.
3. Inventory of dimensions/coordinates/variables.
4. Time/spatial coverage, if applicable.
5. Sanity-check results with severity labels: OK / WARN / FAIL.
6. Clear figures with short captions.
7. Sampling caveats.
8. Recommended next checks.

## Constraints

- Be concise but concrete; report text should be brief but clear.
- Prefer robust scripts over ad hoc shell pipelines for nontrivial inspection and PDF generation.
- Use project-local tooling when available.
- Avoid destructive operations.
- Do not silently ignore read errors; report them as caveats or failures.
- Save all generated artifacts under a timestamped output directory unless the user specifies one.
- The final user-facing artifact is `report.pdf`; include its path in the final response.
- For parquet I/O in this repo, use `engine='fastparquet'` when pandas is used.
