# Cookie Cats A/B Test Analysis

Statistical analysis of a mobile game A/B test (Cookie Cats) comparing the
placement of an in-game gate at level 30 versus level 40. The dataset is
the canonical Kaggle release with ~90k players and three outcome metrics.

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1qC5sJCmi7L9MLMy8XocwDATBx2kAXWg8?usp=sharing)

## Dataset

- ~90,000 players, two variants: `gate_30` (control) and `gate_40` (treatment).
- Metrics: `retention_1` (1-day return, binary), `retention_7` (7-day return,
  binary), `sum_gamerounds` (game rounds played, continuous).
- Source: [Kaggle - Mobile Games A/B Testing](https://www.kaggle.com/datasets/yufengsui/mobile-games-ab-testing).

## Methods

| Step | Method |
|---|---|
| Randomization check | Chi-square goodness-of-fit (Sample Ratio Mismatch test) |
| Binary metrics | Two-proportion z-test with Wald confidence intervals |
| Continuous metric | Welch t-test plus Mann-Whitney U (rank-based, robust to outliers) |
| Family-wise correction | Bonferroni and Benjamini-Hochberg |
| Reported metrics | p-value, 95% CI for the difference, effect size |

Why no Shapiro-Wilk: at n ~ 45k per variant the test rejects normality on
trivial deviations and CLT guarantees the sampling distribution of the mean
is normal regardless. We rely on Welch (robust to non-normality at this n)
and Mann-Whitney (fully non-parametric) instead.

> **DISTRIBUTION**

The `sum_gamerounds` distribution is heavily right-skewed with an extreme
outlier in `gate_30` (max = 49,854 vs `gate_40` max = 2,640). This is the
direct reason for preferring Mann-Whitney over a raw-mean t-test.

## Results

| Metric | Test | p-value | Bonferroni p | BH p | Effect size | 95% CI |
|---|---|---|---|---|---|---|
| retention_1 | z-test | 0.0744 | 0.2232 | 0.0744 | -0.59 pp | [-1.24, +0.06] |
| retention_7 | z-test | **0.0016** | **0.0048** | **0.0048** | -0.82 pp | [-1.33, -0.31] |
| sum_gamerounds | Welch t-test | 0.3759 | 0.1506 | 0.0744 | mean -1.16 | - |
| sum_gamerounds | Mann-Whitney U | 0.0502 | - | - | median -1 | - |

Only `retention_7` survives multiple-testing correction. `sum_gamerounds`
shows a notable disagreement between the mean-based and rank-based tests,
driven by a single 49,854-round whale in `gate_30` that inflates the mean
but not the rank distribution.

> **FOREST_PLOT**

The retention_7 confidence interval lies entirely below zero, visually
confirming the statistically significant drop. The retention_1 interval
crosses zero, consistent with the failure to reject H0 for that metric.

## Data quality findings

- **Sample Ratio Mismatch detected.** Variant counts 44,700 / 45,489
  (50.4% / 49.6%), chi-square p = 0.0086. The imbalance is small in
  absolute terms but statistically significant at this n. Analysis
  proceeds with this flagged as a limitation. A production team should
  investigate the randomization layer before acting.
- No missing values in any metric column.
- No duplicate user IDs (one row per player).

## Verdict

**Keep `gate_30`; do not roll out `gate_40`.**

The drop in 7-day retention is the only effect that holds up under
correction, but 7-day retention is the strongest single predictor of
monetisation in mobile games. The 0.31 to 1.33 pp drop at 95% confidence
translates to a 1.6% to 7.0% relative reduction in retained players, with
no compensating gain in 1-day retention or engagement volume.

## Limitations

- Sample Ratio Mismatch as noted above.
- Single experiment window - seasonality not addressed.
- No segmentation by cohort, device, or geography.
- Two-sided tests; a directional prior would change p-value interpretation
  but not the conclusion.
- Three pre-registered metrics; adding more post-hoc would inflate
  false-positive risk and would require re-running corrections.

## Files

```
cookie-cats-ab-test/
├── cookie_cats_analysis.ipynb   # full analysis: EDA, tests, corrections, verdict
├── requirements.txt
└── README.md
```

## How to run

```bash
python -m venv .venv
.venv\Scripts\Activate.ps1            # PowerShell
pip install -r requirements.txt

# Download cookie_cats.csv from Kaggle and place it next to the notebook
jupyter notebook cookie_cats_analysis.ipynb
```
