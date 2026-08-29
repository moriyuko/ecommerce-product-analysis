# E-commerce Product Funnel Analysis

An exploratory and statistical analysis of customer behavior in an online cosmetics store. The project follows users from product view to cart and purchase, then evaluates whether product price and repeated viewing are associated with conversion. It is designed to turn behavioral event data into testable product and merchandising hypotheses—not to make causal claims from observational data.

## Business questions

- Where do customers drop out of the `View → Cart → Purchase` funnel?
- How does funnel performance vary from October 2019 through February 2020?
- Are higher-priced products associated with greater pre-cart or post-cart friction?
- Do repeated product views identify users with stronger subsequent purchase intent?
- Which findings are promising enough to validate with controlled experiments?

## Dataset

The notebook uses Kaggle's [eCommerce Events History in Cosmetics Shop](https://www.kaggle.com/datasets/mkechinov/ecommerce-events-history-in-cosmetics-shop) dataset, credited in the notebook to REES46. The analysis covers five complete source files:

- `2019-Oct.csv`
- `2019-Nov.csv`
- `2019-Dec.csv`
- `2020-Jan.csv`
- `2020-Feb.csv`

Together, the funnel-analysis data contain 16,713,161 view, cart, and purchase rows spanning `2019-10-01 00:00:00 UTC` through `2020-02-29 23:59:59 UTC`. The five raw CSVs are approximately 2.4 GB in total.

### Event schema

Each row is an event with:

- `event_time`: UTC event timestamp
- `event_type`: `view`, `cart`, `remove_from_cart`, or `purchase`
- `product_id` and `category_id`: product and category identifiers
- `category_code`: optional category taxonomy
- `brand`: optional lowercased brand name
- `price`: product price
- `user_id`: persistent user identifier
- `user_session`: temporary session identifier

## Repository structure

```text
.
├── notebook.ipynb    # Executed analysis, visualizations, models, and conclusions
├── requirements.txt  # Python dependencies
├── README.md         # Project overview and reproduction guide
└── data/             # Local raw CSVs; excluded from version control
```

The `.gitignore` excludes `data/`, so the large raw files are not versioned.

## Reproduce the analysis

### 1. Create the environment

On macOS or Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

The requirements include pandas, NumPy, Matplotlib, seaborn, SciPy, statsmodels, KaggleHub, and Jupyter.

### 2. Obtain the data

Create a local data directory:

```bash
mkdir -p data
```

Either run the notebook's **Dataset Installation** cell, which uses `kagglehub.dataset_download()` with the dataset identifier `mkechinov/ecommerce-events-history-in-cosmetics-shop`, or download the dataset from the Kaggle page linked above. KaggleHub may require local Kaggle authentication.

Place the five CSV files directly under `data/` with the exact filenames listed in the Dataset section. The notebook validates file presence, schema, timestamps, event types, and each file's named month before combining them.

### 3. Run the notebook

```bash
jupyter notebook notebook.ipynb
```

Open the notebook and run all cells from top to bottom. The raw files and intermediate DataFrames are large, so allow adequate memory and execution time.

## Analysis methodology

- **Cleaning and audit:** validate the expected columns; parse timestamps as UTC; standardize dtypes; verify month coverage; audit missing metadata, exact duplicates, event counts, and zero/non-positive prices. Exact duplicate rows are retained because identical same-second events may represent legitimate quantity changes or repeated actions; deduplicated counts are used as a sensitivity check.
- **Strict user-product funnel:** count each `(user_id, product_id)` pair once per month. A cart qualifies only after the first view, and a purchase only after a qualifying cart. Stable source order resolves same-timestamp ties.
- **Separate measurement units:** report raw event ratios separately from distinct user-product conversion. A second, presence-based session funnel uses `(user_id, user_session)`; it does not enforce event order and can contain different products at each stage.
- **Uncertainty:** attach 95% Wilson confidence intervals to conversion proportions and use Holm correction for families of hypothesis tests.
- **Price analysis:** fix price, brand, and category at the pair's first qualifying view. Compare five shared, global price bands across months, then repeat with independently calculated within-month bands to test sensitivity to changing price distributions. Invalid or non-positive first-view prices are excluded from price comparisons.
- **Grouped binomial models:** fit grouped binomial GLMs for funnel outcomes and test price-band-by-month interactions without expanding aggregated cells back to millions of rows.
- **Repeated-view landmark design:** assign each user-product pair to its first-view month; count views during a fixed 7-day exposure window; exclude pairs that cart or purchase during exposure; then measure a strict cart-to-purchase funnel during a separate 14-day follow-up. Grouped binomial models adjust for month, global price band, grouped brand, and grouped category.

## Key validated findings

All results below are associations from the executed notebook outputs.

### Monthly funnel pattern

For the strict user-product funnel:

| Month | View → Cart | Cart → Purchase | View → Purchase |
|---|---:|---:|---:|
| 2019-10 | 16.13% | 24.47% | 3.95% |
| 2019-11 | 13.88% | 34.16% | 4.74% |
| 2019-12 | 12.42% | 32.33% | 4.02% |
| 2020-01 | 12.86% | 32.11% | 4.13% |
| 2020-02 | 12.54% | 30.02% | 3.77% |

October generated the most cart progression but the weakest post-cart completion. November produced the highest end-to-end conversion, while February was lowest. The separate session funnel shows the same broad pattern: View → Purchase peaked at 2.83% in November and fell to 2.26% in February.

### High-price penalty

Using shared global bands, the lowest band was `$0.05–$2.38` and the highest was `$12.31–$327.78`.

- High-band View → Cart conversion was lower in every month by 8.44–9.36 percentage points, a relative reduction of approximately 48%–55%.
- Low-band View → Purchase conversion ranged from 5.24% to 6.38%, versus 2.19% to 2.75% for the high band. High-band pairs were therefore approximately 57%–61% less likely to complete the ordered funnel.
- The post-cart difference was smaller: high-band Cart → Purchase conversion was approximately 6%–18% lower.

The negative gradient persisted with within-month price bands and within several major brands, including Runail and Irisk, although category-level patterns were heterogeneous. This is evidence of an association between price and funnel progression, not evidence that price itself caused the difference.

### Repeated views as an intent signal

Across the 5,020,082-pair landmark cohort:

- One-view pairs purchased during follow-up at 0.12% (`5,174 / 4,222,545`).
- Pairs with 2–3 views purchased at 0.25% (`1,764 / 718,466`), an unadjusted risk ratio of 2.00.
- Pairs with 4+ views purchased at 0.54% (`426 / 79,071`), an unadjusted risk ratio of 4.40.

The corresponding unadjusted View → Cart rates were 0.45%, 0.81%, and 1.76%. After adjustment for month, price band, brand, and category, the associations remained:

- **View → Cart:** adjusted odds ratios 1.83 (95% CI 1.78–1.89) for 2–3 views and 3.95 (3.74–4.18) for 4+ views.
- **View → Purchase:** adjusted odds ratios 2.04 (1.93–2.15) for 2–3 views and 4.47 (4.05–4.94) for 4+ views.

Repeated viewing is therefore a useful predictive signal, but it may reflect pre-existing intent rather than create intent.

## Limitations and data-quality caveats

- Brand is missing for a substantial share of events, and `category_code` is almost entirely missing. February also has unusually many zero-price rows.
- Source files contain exact duplicate rows, especially around cart-removal activity. They are retained in the primary data because one-second timestamps cannot distinguish all legitimate repeats; deduplication should remain a sensitivity analysis.
- Monthly funnel and price analyses require stages to occur within the same month, so journeys crossing month boundaries are not captured.
- The landmark design requires a complete 7-day exposure plus 14-day follow-up. February excludes 71.04% of candidate pairs for incomplete coverage and retains only 24.99%, so its landmark result represents early-February cohorts rather than the full month.
- Multiple product pairs can belong to one user, while model intervals are not user-clustered.
- Price, repeated views, and conversion may share confounders such as intent, product quality, promotions, category, brand, and customer mix. Reported effects are observational associations, not causal estimates.
- The price interaction model emitted a divide-by-zero fitting warning; its interaction tests should be treated as supporting, not final, evidence until that warning is resolved.

## Recommended experiments and future work

1. **High-price product-page A/B test:** test richer specifications, benefits, reviews, comparisons, usage guidance, and trust signals; use View → Cart as the primary metric and View → Purchase as a secondary metric.
2. **Repeated-view targeting experiment:** randomize eligible repeat viewers to a reminder or additional product information versus no contact, and measure incremental purchases, revenue, margin, and cost per incremental purchase.
3. **Randomized promotion test:** test discounts, bundles, delivery incentives, or loyalty benefits to separate price sensitivity from uncertainty about value.
4. **Extend the analysis:** use cross-month cohorts, user-clustered uncertainty, survival analysis, continuous/nonlinear price models, and hierarchical brand/category effects; investigate high-view/low-cart products and cart-removal behavior.

The strongest next step is experimentation: the notebook identifies where opportunity may exist, while randomized tests can establish whether the proposed interventions create profitable incremental conversion.
