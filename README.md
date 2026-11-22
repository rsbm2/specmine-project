# SpecMine Research Project

## Research question

This project investigates whether the quality of software specifications is associated with subsequent code-related pull request (PR) activity. The analysis focuses on eligible specifications for which an initial implementation PR can be identified, then measures later merged PR activity involving code paths changed by that initial implementation.

The analysis uses two observation windows:

- **30 days:** primary analysis window.
- **90 days:** robustness analysis window.

The outcome is **follow-up code PR activity**, not a direct measure of bug fixes. Follow-up PRs may include fixes, feature work, refactoring, or other changes.

## Repository / notebook

The main analysis is in:

- `specmine_clean_30_90_day_analysis.ipynb` — cleaned analysis notebook with filtering, implementation identification, follow-up outcome construction, descriptive analysis, sample-window comparisons, negative binomial regressions, and exports.

Run the notebook cells in order, from top to bottom.

## Data requirements

Place the following Parquet files in a `data/` directory alongside the notebook:

```text
data/
├── spec_files.parquet
├── spec_content_features.parquet
├── pull_requests.parquet
└── pr_files.parquet
```

The notebook uses these tables:

| File | Role |
|---|---|
| `spec_files.parquet` | Specification file metadata, including repository, path, and spec tool. |
| `spec_content_features.parquet` | Language, content, placeholder, and substantive-feature measures used to filter and score specs. |
| `pull_requests.parquet` | PR metadata, merge status/date, and code-change indicators. |
| `pr_files.parquet` | Files changed by each PR, including spec/code classifications. |

These datasets are not included in this repository/notebook distribution. Access to the collected data is required to reproduce the analysis.

## Eligibility criteria

A specification is considered eligible if it:

1. Is identified as English.
2. Has at least 100 words, 10 lines, and 2 headings.
3. Does not contain detected lorem ipsum or unfilled placeholders.
4. Contains at least 2 of the 8 substantive features below.

The eight binary substantive features are:

- Requirements
- Acceptance criteria
- User scenarios
- Success criteria
- Purpose
- Overview
- Edge cases
- Dependencies

The specification quality score is the sum of these eight features (0–8). Because eligibility requires at least two substantive features, eligible specs have scores of at least 2.

## Initial implementation and outcome construction

An **initial implementation PR** is defined as the earliest merged PR in the collected data that modifies an eligible specification and code.

Follow-up PRs are counted if they:

- are merged after the initial implementation PR;
- modify code; and
- touch at least one code file/path changed by the initial implementation PR.

PR records are matched using the composite key `tool`, `repo_name`, and `pr_number`.

Counts are computed cumulatively for 30- and 90-day windows: a PR counted in the 30-day window is also included in the 90-day window.

A specification without an identifiable initial implementation or without initial code paths to match is not assigned a follow-up count. A zero follow-up count means no matching follow-up PR was identified in the available collected data during the relevant window; it does not prove that the repository had no subsequent activity.

## Observation cutoff and sample restrictions

The confirmed data collection cutoff used by the notebook is **July 14, 2026 (UTC)**. To ensure each observation has the full opportunity for follow-up, each model sample is restricted to specs whose initial implementation date is at least the relevant window length before this cutoff:

- 30-day window: initial implementation on or before the cutoff minus 30 days.
- 90-day window: initial implementation on or before the cutoff minus 90 days.

The notebook reports retained and excluded counts and compares retained vs. excluded observations on available characteristics. These restrictions reduce right-censoring, but can introduce sample-selection differences because the longer window necessarily retains older implementations.

Historical sample flow (from the data version previously analyzed; rerun the notebook to refresh these counts):

| Stage | Specifications |
|---|---:|
| Original spec records | 28,583 |
| English | 27,936 |
| Passed content / placeholder criteria | 24,240 |
| Eligible (2+ substantive features) | 6,336 |
| Identifiable initial implementation | 3,729 |

The 2,607 eligible specs without an identified initial implementation are **not necessarily unimplemented**. Their implementation may be absent from the collected PR data, not linkable under the identification rule, separated from spec changes, or not yet have occurred. Downstream analyses are limited to specs with identifiable implementation events.

## Statistical analysis

### Descriptive analysis

For each follow-up window, the notebook reports:

- sample size and follow-up PR count summaries by quality score;
- Spearman rank correlation between quality score and follow-up activity;
- distribution of follow-up PR counts.

### Negative binomial regression

The follow-up outcome is a count and was found to be overdispersed in the analyzed data, so negative binomial regression is used. Standard errors are clustered by repository.

Three model specifications are fit within each window using the same complete-case sample for comparability:

- **M0 — Quality only:** quality score.
- **M1 — Adjusted:** quality score, log specification word count, and specification tool (OpenSpec reference category).
- **M2 — Implementation-size sensitivity:** M1 plus the log number of code files changed by the initial implementation PR.

Tools with fewer than 30 observations in a window are grouped into `Other` rather than dropped. M2 is treated as a sensitivity model because implementation size may be downstream of specification quality.

Interpret the quality-score coefficient using the incidence rate ratio (IRR):

- `IRR = exp(coefficient)`
- Percent change in expected follow-up PR count per one-point quality increase: `(IRR - 1) * 100`

Report effect estimates with confidence intervals and p-values. The analysis is observational and does not establish causality.

## Running the analysis

1. Install Python and Jupyter (or use an environment that supports Jupyter notebooks).
2. Install the notebook dependencies:

   ```bash
   pip install numpy pandas matplotlib scipy statsmodels pyarrow jupyter
   ```

3. Place the required Parquet files in `data/` as described above.
4. Open `specmine_clean_30_90_day_analysis.ipynb`.
5. Run all cells from top to bottom.

The notebook prints the sample sizes, descriptive results, and model summaries.

## Outputs

The notebook saves the following files in the working directory:

- `eligible_spec_followups.parquet` — eligible specifications with quality features, implementation metadata, and 30-/90-day follow-up counts.
- `quality_followup_model_comparison_30_90d.csv` — model comparison summary for the two windows.

## Limitations

- **PR data coverage:** the collected PR data is specification-associated and may not include every PR or code change in a repository.
- **Implementation identification:** requiring a merged PR that changes both a spec and code may miss implementations linked across separate PRs or through other workflows.
- **Follow-up measure:** the outcome represents code-related follow-up PR activity, not specifically defects or bug fixes.
- **Sample selection:** eligible specs without identifiable implementations and specs without complete observation windows are excluded from downstream analyses. These exclusions may be systematic.
- **Repository and tool heterogeneity:** projects and tools differ in development practices, PR conventions, and data coverage.
- **Observational design:** estimates describe associations and should not be interpreted as causal effects.
- **File coverage:** PR-file records may be incomplete or capped for some PRs, which can affect path matching and counts.

## Reproducibility notes

The notebook uses a fixed data cutoff of July 14, 2026 (UTC). Reproducing the reported results requires the same or a clearly documented version of the underlying Parquet datasets and their collection coverage. Refreshing the datasets may change sample sizes and estimates.
