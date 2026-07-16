# Statistical Methodology

How the Scoring Tool trains, selects, and calibrates its models — and where its guarantees end. For the system design, see [ARCHITECTURE.md](ARCHITECTURE.md).

---

## The Prediction Problem

Given historical products with known overall ratings (0–100 scale, published by an independent testing organization) and a set of numeric test attributes per product, predict the rating for products that haven't been rated — with a defensible statement of uncertainty.

Two properties of real product datasets drive the entire design:

1. **They are small.** A product category may have a few dozen to a few hundred rated products. Every design choice must remain honest at n ≈ 50–400.
2. **They contain near-duplicates.** The same product often appears in multiple rows (model-year refreshes, regional variants). Naive cross-validation quietly puts one copy in training and another in validation, and reports beautiful metrics that evaporate on genuinely new products.

## Candidate Models

Three regressors, chosen to span the bias-variance spectrum while remaining defensible on small data:

| Model | Why it's in the lineup |
|---|---|
| Linear Regression | Fully interpretable baseline; if nothing beats it, that itself is a finding |
| Ridge Regression (CV-tuned α) | Controls variance on small, collinear feature sets; α chosen from a log-spaced grid by internal cross-validation |
| Random Forest | Captures non-linear relationships and interactions; robust to outliers and feature scaling |

All three share an identical preprocessing pipeline — mean imputation followed by standardization — fit **inside** each cross-validation fold, so no statistic from a validation fold ever leaks into preprocessing.

## Leakage-Safe Validation

- **k-fold cross-validation** with up to 5 folds, reduced automatically when the dataset is too small to support 5.
- **Grouped by product**: whenever the same product identifier appears in more than one row, all its rows travel together — a product can never sit in both a training and a validation fold. Rows with missing identifiers are treated as singleton groups.
- **Pooled out-of-fold (OOF) metrics**: each labeled row is predicted by the one model that never saw it; R² and MAE are computed once over the pooled OOF predictions rather than averaged per fold. Pooling is the more stable choice at small n, where per-fold R² is noisy and can even be undefined.

## Champion Selection

The champion is the model with the **highest OOF R²**, with **lowest OOF MAE** as tie-break. It is then **refit on all labeled rows** before any inference — the benchmark decides *which* model, the full dataset decides *its parameters*.

**Known bias, stated openly:** selecting the champion on the same OOF metrics that are displayed introduces a small optimistic bias (a winner's-curse effect across three candidates). With three models the effect is modest, but it is real; nested cross-validation would remove it at roughly 5× the training cost. This trade-off is documented in the UI rather than hidden.

## Split-Conformal Prediction Interval

Every prediction ships with a 90% prediction interval computed by split-conformal inference on the champion's out-of-fold residuals:

1. Collect the absolute OOF residuals |yᵢ − ŷᵢ| for all n labeled rows.
2. Take the **finite-sample-corrected** quantile: the ⌈0.9 · (n + 1)⌉-th smallest absolute residual (not the naive 90th percentile — the correction matters exactly where this tool lives, at small n).
3. The interval is ŷ ± that quantile, clipped to the valid rating scale.

Because the residuals are out-of-fold, they measure *generalization* error, not training fit. For exchangeable data, this construction guarantees **at least 90% marginal coverage** — a distribution-free, finite-sample guarantee that an MAE-based heuristic band cannot offer. Empirical coverage on the training data is also computed and displayed, so the guarantee is verifiable rather than asserted.

**Limits of the guarantee, stated openly:**

- The band is **homoscedastic** — one width for all products. It is honest on average but does not widen for individually hard cases; a heteroscedastic variant (e.g. residual-normalized conformal) is the natural upgrade.
- Coverage is guaranteed under **exchangeability**. Predicting a future model year is a distribution shift; the guarantee weakens exactly when extrapolating, which is why the UI flags out-of-training-range inputs.

## Segmentation

Unsupervised K-means over standardized test profiles gives analysts a market-structure view:

- The **number of segments is chosen by silhouette score** across a candidate range, not fixed a priori; the user can override it.
- The **silhouette value is always displayed** — a deliberate honesty mechanism: weak cluster structure shows up as a weak score instead of being presented as insight.
- Segments are visualized on a **PCA projection** with per-segment rating profiles, making it visible whether segments differ in rating or only in test profile.

## Guardrails Summary

| Risk | Mitigation |
|---|---|
| Duplicate products inflating metrics | Product-grouped CV folds |
| Preprocessing leakage | Imputer/scaler fit inside each fold |
| Overconfident point estimates | Split-conformal 90% interval, finite-sample corrected |
| Tiny datasets producing meaningless metrics | Hard minimum of 10 labeled rows, with the reason explained to the user |
| Silent extrapolation | Out-of-training-range inputs flagged in the what-if UI |
| Impossible outputs | Predictions and intervals clipped to the rating scale |
| Cherry-picked training fit shown as accuracy | All displayed metrics are out-of-fold; rows with known ratings display their OOF prediction |
