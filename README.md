# Predicting a Stock's Daily Direction from the First Two Hours of Trading

**2nd place overall (Excellence Award), 1st KRX Financial Big Data Competition, Korea Exchange (2022) · Team Micro-A.N.T** · [한국어](README.ko.md)

[Final ranking (DACON)](https://dacon.io/competitions/official/235914/leaderboard) · Coverage: [Etoday](https://www.etoday.co.kr/news/view/2173163), [Maeil Business News](https://www.mk.co.kr/news/stock/10457335)

Using minute-level Korea Exchange (KRX) data from 09:00 to 11:00, we predicted whether each KOSPI stock would **close above its opening price** that day. We engineered features from foreign investor flows, order-book microstructure (spreads, depth, order imbalance, cancellation rates) and intraday trend indicators, then compared six models from logistic regression to CatBoost and a neural network.

After the competition, I re-validated the result. It held up on a later time period, but **the extra accuracy over a simple rule came from a look-ahead leak**. This repository contains both the original submission and the re-validation.

## Key findings

| | Accuracy |
|---|---|
| Competition result (CatBoost, random 70/30 split) | 0.830 |
| Baseline rule: "up if the 11:00 price is above the open" (2022) | 0.800 |
| CatBoost, features available at 11:00, trained 2019–21, tested on 2022 | 0.801 |
| Same, plus the competition's trading value / market cap feature | 0.842 |
| Same, with that feature built from the previous day's close | 0.801 |

1. **Out-of-time validation passed.** Training on 2019–2021 and testing on 2022 changed every model's accuracy by about 0.1 percentage points or less.
2. **No model beat a one-line rule.** Stocks that rise in the first two hours usually finish above their open, so the rule alone is right 80% of the time. CatBoost disagreed with it on only 2.4% of stock-days and was right on about half of those.
3. **The competition score included a leak.** One feature divided morning trading value by the daily `market_cap`, which turned out to be **that day's closing price × shares outstanding** in every row. The close is set at 15:30, long after the 11:00 prediction. Adding this feature lifts 2022 accuracy by about 4 points; rebuilding it with the previous day's close removes the gain.

Full analysis: [`notebooks/01_validation_and_leakage.ipynb`](notebooks/01_validation_and_leakage.ipynb)

## Lessons

- Compare against a trivial baseline first. A 3-point gain over the baseline would have looked suspicious, while 0.83 against an implied 0.50 looked impressive.
- Validate on a later period, not only on shuffled data.
- Check the timestamp of every input against the moment of prediction, especially values merged in from end-of-day files.

## Original competition results

As submitted (random 70/30 split, `random_state=9`). These figures include the leak described above.

| Model | Accuracy | F1 | Hyperparameters |
|---|---|---|---|
| Logistic Regression | 0.7716 | 0.7320 | C=0.8, L1 penalty, liblinear |
| Decision Tree | 0.7929 | 0.7577 | max_depth=7, min_samples_split=4 |
| Random Forest | 0.8001 | 0.7661 | max_depth=5, n_estimators=100, min_samples_split=4 |
| XGBoost | 0.8251 | 0.7971 | max_depth=7, n_estimators=100 |
| **CatBoost** | **0.8296** | **0.8033** | defaults |
| Neural network | 0.7935 | 0.7623 | Dense 64-64, ReLU |

## Data

| | |
|---|---|
| Source | KRX market data provided to competition participants: daily prices, 1-minute and 10-minute intraday prices, and trading by investor type |
| Period | January 2019 – July 2022 |
| Size | About 68 million raw rows, aggregated to about 656,000 stock-days |
| Target | 1 if the day's close is above the open, otherwise 0 (44% / 56%) |

The data was licensed to participants and is **not included** in this repository.

## Approach (competition)

```
Load daily, intraday (1-min, 10-min) and investor-type data
 → Keep 09:00–11:00, separate foreign investor trades
 → Engineer 24 features (supply and demand, information asymmetry, trend)
 → Remove preferred shares and SPACs, missing and infinite values
 → EDA: distributions, winsorizing (0.5% each tail), Q-Q plots, Anderson-Darling tests
 → Stepwise feature selection (AIC, p < 0.05) → 22 features; correlation and VIF checks
 → Models: logistic regression, decision tree, random forest, XGBoost, CatBoost, neural network
 → Robustness check with a different split
```

| Group | Features |
|---|---|
| Supply and demand | foreign net buying (value, volume), foreign buy/sell strength, signed turnover, trading value / market cap |
| Information asymmetry | quoted and relative spreads, adverse-selection cost, market depth, order cancellation rates, trade and order imbalances, depth-weighted quote / price |
| Trend and position | intraday intensity, daily volume index, TRIN, pivot-point position, money flow index, 1-min and 10-min returns |

## Repository structure

```
├── notebooks/
│   ├── 01_validation_and_leakage.ipynb   # Re-validation: time split, baseline, leak audit (English)
│   └── original/
│       └── Micro-ANT_final_ko.ipynb      # Competition submission as a record (Korean)
├── docs/                                  # Competition proposal and final presentation (Korean)
├── requirements.txt
├── README.md
└── README.ko.md
```

## Reproducing

```bash
pip install -r requirements.txt
```

`01_validation_and_leakage.ipynb` expects three files in `data/`: the team's feature tables (`최종train(2019_2021).csv`, `최종test(2022).csv`) and the daily price file (`df_일별데이터.csv`). It was run with Python 3.13, scikit-learn 1.9, XGBoost 3.4 and CatBoost 1.2.

The original notebook ran on Python 3.8 with scikit-learn below 1.2 and needs the raw KRX files.

## Notes on the original notebook

The saved outputs are from the original run and match the final presentation. For publication, I fixed only data-loading bugs that do not affect those outputs:

- File lists and paths for 2022 intraday, investor and 10-minute data that pointed to 2021
- Missing DataFrame initializations for 2019, a missing path separator and a missing closing bracket
- A 2021 foreign-investor filter that used the wrong DataFrame as its mask
- A reload cell whose file names and encodings did not match the files saved just above it
- An unused import, mislabeled year comments, a missing image (replaced with the same table) and per-epoch training logs

## Next steps

- Rebuild the trading value / market cap feature with the previous day's market cap from the raw minute data. The leak experiment above uses an approximation: average morning price ≈ (open + 11:00 price) / 2.
- Reframe the task around the harder question that matters in practice: which stocks **reverse** their morning direction by the close.

## Team

Five members: project management and development (2), business development and branding (2), UI and PR (1).

### My role

I led the project from start to finish as team lead (July–September 2022).

- Defined the research question and built the analysis framework (data, features, models and validation), drawing on my background in economics, finance and machine learning.
- Wrote the first draft of the competition proposal and took part in every stage of the pipeline, from data processing to model evaluation.
- Planned the three-month schedule from proposal to finals, assigned roles across the five-person team, and resolved disagreements within the team.
- Reviewed and revised the final slides and presentation script; a teammate delivered the presentation.
- After the competition, re-validated the model with an out-of-time split and a baseline comparison, and traced the inflated score to a look-ahead leak.
