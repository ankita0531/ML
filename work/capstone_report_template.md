# Capstone Report — <Refresh / Content Opportunity Scoring>

- **Author: Ankita Rout**
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/ankita0531/ML
- **Date:** 04 Sep 2026

## 0. Abstract

This project asks whether pre-outcome search and engagement signals can identify content likely to experience a significant decline in Google Search impressions and convert those signals into a ranked refresh-priority queue. 
The analysis uses February 2026 signals from Google Search Console and GA4 at the client-content level, with March 2026 impressions used only to define the outcome. 
A Logistic Regression model was evaluated using a client-grouped holdout, with a transparent rule-based baseline evaluated on the same held-out clients. 
The log-transformed Logistic Regression achieved a ROC-AUC of 0.5668 and Precision@5% of 0.2908, compared with 0.2799 for the baseline at the same top-5% cutoff, while its lift reached 1.3484 at the top 20% of the queue. The final output is a ranked content-priority queue with reason codes and recommended actions intended to support editorial review, not to prove that refreshing content will cause improved search performance.

## 1. Problem framing

The decision supported by this analysis is: **which content should a FlyRank editor prioritize for refresh or further investigation?**
The unit of analysis is `client_hash_id × content_hash_id`, representing a client-content pair.
The output is a ranked content-priority queue containing a priority score, reason code, and recommended action.

A human editor can use the queue to review the highest-priority content first. Recommended actions include validating GA4 coverage, reviewing title/meta and search-intent alignment, and reviewing content relevance and depth.
A false positive may cause editorial time to be spent reviewing content that did not experience the defined decline. A false negative may cause content that actually declined to be missed.
Data and ML help by combining multiple February search and engagement signals into a single ranking score instead of relying only on a manually defined rule.

This is a decision-support analysis. It does not claim that refreshing a page will cause its search performance to improve, and it does not attempt to predict or prove Google's ranking algorithm.

## 2. Data safety

The analysis uses pseudonymous client and content identifiers only for grouping and evaluation. `client_hash_id` and `content_hash_id` are not model features.
The feature window is February 2026, covering February 1 through February 28. The outcome window is March 2026, covering March 1 through March 31.
The analysis uses February search and engagement features:

- `feb_impressions`
- `feb_clicks`
- `feb_avg_position`
- `feb_sessions`
- `feb_engaged_sessions`
- `feb_ctr`
- `feb_engagement_rate`

The target is defined as March 2026 GSC impressions declining by more than 20% relative to February 2026.
There were 153,559 February content rows before filtering. After requiring February impressions greater than zero, 134,238 content items remained.
The target distribution was:

| Outcome | Count | Proportion |
|---|---:|---:|
| Non-declining | 107,413 | 80.02% |
| Declining | 26,825 | 19.98% |

The February source data contained 7,355,108 rows from February 1–28, 2026. The March source data contained 9,841,378 rows from March 1–31, 2026.
The following were deliberately excluded:

- client-identifying fields
- raw URLs and domains
- private search queries
- credentials
- `trend_direction`
- `trend_pct`
- other outcome-derived fields

The exclusion of outcome-derived variables reduces the risk of target leakage. Pseudonymous IDs are used for grouping only and are not included as model features.
February GA4 session and engagement fields contained 67,969 missing values. The final recommendation logic therefore includes a specific GA4 coverage-validation action rather than automatically interpreting missing GA4 values as zero engagement.
No client names, domains, raw URLs, private queries, or credentials are included in `work/`.

## 3. Baseline

A transparent rule-based baseline was created for comparison with the Logistic Regression model.
The baseline assigns:

- Priority score 2 when February GSC impressions are at least 100 and average position is greater than 10.
- Priority score 1 when February GSC impressions are at least 10 and average position is greater than 10.
- Priority score 0 otherwise.

A positive baseline prediction is defined as a baseline score greater than zero.
The baseline uses only February information, matching the model's feature window, and is evaluated on the same held-out clients. This makes it a transparent reference for assessing whether the model provides additional ranking value.
On the held-out test set, the baseline produced:

| Metric | Baseline |
|---|---:|
| Precision | 0.2293 |
| Recall | 0.2479 |
| F1 | 0.2382 |

Baseline Precision@K was:

| Top fraction | K | Baseline Precision@K |
|---|---:|---:|
| 5% | 2,294 | 0.2799 |
| 10% | 4,589 | 0.2325 |
| 20% | 9,178 | 0.2023 |
| 30% | 13,768 | 0.2192 |

## 4. Model / analysis

The primary model is Logistic Regression. It was selected because it provides a transparent classification approach while combining multiple search and engagement signals.
The model features are:

- `feb_impressions`
- `feb_clicks`
- `feb_avg_position`
- `feb_sessions`
- `feb_engaged_sessions`
- `feb_ctr`
- `feb_engagement_rate`

The target is binary: 1 when March 2026 GSC impressions declined by more than 20% relative to February 2026, and 0 otherwise.
March outcome information was not used as a model feature.
A second Logistic Regression model used log-transformed numerical features to reduce the influence of highly skewed search and engagement distributions.
The log-transformed model achieved a higher ROC-AUC than the original model:

| Metric | Original Logistic Regression | Log-transformed Logistic Regression |
|---|---:|---:|
| Precision | 0.2595 | 0.2819 |
| Recall | 0.7779 | 0.5988 |
| F1 | 0.3892 | 0.3833 |
| ROC-AUC | 0.5479 | 0.5668 |

The log-transformed Logistic Regression was therefore used for the final ranking.

## 5. Evaluation

The validation strategy uses a client-grouped holdout so that content from the same client cannot appear in both training and test sets.
The split contained:

| Split | Rows | Clients |
|---|---:|---:|
| Training | 88,344 | 33 |
| Test | 45,894 | 9 |

There were zero overlapping clients between the training and test sets.
The log-transformed model achieved:

- Precision: 0.2819
- Recall: 0.5988
- F1: 0.3833
- ROC-AUC: 0.5668

For the ranked queue:

| Top fraction | K | Log-model Precision@K | Baseline Precision@K | Lift |
|---|---:|---:|---:|---:|
| 5% | 2,294 | 0.2908 | 0.2799 | 1.0389 |
| 10% | 4,589 | 0.2874 | 0.2325 | 1.2362 |
| 20% | 9,178 | 0.2728 | 0.2023 | 1.3484 |
| 30% | 13,768 | 0.2703 | 0.2192 | 1.2329 |

The strongest observed lift was 1.3484 at the top 20% of the queue.
The task's declining-content base rate was 19.98%. The majority class was 80.02% non-declining. This base rate is important when interpreting precision values.
The model shows modest discrimination rather than highly accurate prediction. Its main value is therefore ranking content for review rather than making definitive decisions about individual pages.

## 6. Interpretation

The original Logistic Regression coefficients provide a directional view of how the model combined the available features.

| Feature | Coefficient |
|---|---:|
| `feb_engaged_sessions` | 0.1754 |
| `feb_sessions` | -0.1278 |
| `feb_ctr` | -0.0981 |
| `feb_avg_position` | -0.0723 |
| `feb_clicks` | -0.0704 |
| `feb_engagement_rate` | -0.0482 |
| `feb_impressions` | -0.0376 |

These coefficients represent model associations and should not be interpreted as causal effects.
The log transformation produced an improvement in ROC-AUC from 0.5479 to 0.5668 and improved Precision@5% from 0.2428 to 0.2908.
The final ranked queue contained 45,894 held-out content items.
An important operational finding appeared in the Top-100 queue. Ninety of the top 100 items, or 90%, were assigned the reason code **"GA4 coverage missing; Low CTR"**.

The Top-100 reason-code distribution was:

| Reason code | Count | Share |
|---|---:|---:|
| GA4 coverage missing; Low CTR | 90 | 90% |
| Low CTR; Established visibility | 7 | 7% |
| Position > 10; Low CTR | 3 | 3% |

This means the highest-priority queue is strongly affected by missing GA4 coverage. The appropriate interpretation is therefore to validate measurement coverage before taking editorial action.
This is a useful negative or cautionary finding: a high model score does not automatically mean that content should be refreshed.

## 7. Recommendation

The final output should be used as a ranked review queue rather than an automatic refresh list.
A FlyRank editor can use the queue in the following order:
1. **Validate GA4 coverage first.**  
   For content with missing GA4 sessions or engagement data, verify measurement coverage before interpreting the signal or taking editorial action.
2. **Review low-CTR content with established visibility.**  
   Review title and metadata and assess alignment between the page and search intent.
3. **Review lower-position content with low CTR.**  
   Review content relevance, depth, and title/meta alignment.
4. **Work from the highest priority score downward.**  
   The score provides a way to allocate limited editorial review capacity.
5. **Use reason codes as diagnostic prompts.**  
   The reason code explains which signal combination triggered the recommended action.
   
Confidence in the ranking should be considered limited to moderate. The log-transformed model achieved ROC-AUC of 0.5668, and its improvement over the baseline was strongest at the 10%, 20%, and 30% ranking cutoffs.
The results support directional prioritization and decision support. They do not establish that an individual page will decline, and they do not establish that refreshing a page will reverse or improve its search performance.

## 8. Reproducibility

The analysis is implemented in:

`work/notebooks/capstone.ipynb`

The repository is:

https://github.com/ankita0531/ML

The notebook uses DuckDB for warehouse querying, pandas for data preparation, and scikit-learn for modeling and evaluation.
The main analysis configuration is:

- Feature window: February 2026
- Outcome window: March 2026
- Unit of analysis: `client_hash_id × content_hash_id`
- Model: Logistic Regression
- Final model: log-transformed Logistic Regression
- Validation: client-grouped holdout
- Training rows: 88,344
- Test rows: 45,894
- Training clients: 33
- Test clients: 9
- Client overlap: 0

The notebook should be run from top to bottom in a clean Colab runtime to reproduce the analysis.
The Hugging Face authentication token is stored in Colab Secrets and is not committed to the repository.

The final paper URL must be stored in:

`submission/paper_url.txt`

The file should contain exactly one line containing the direct URL of the deployed research paper.

## 9. Acknowledgments & data credit

Built on the **FlyRank ML Internship dataset**.

Data source: [FlyRank](https://flyrank.ai)

This project uses the FlyRank ML Internship dataset for educational and research purposes. No client-identifying information, raw URLs, private queries, or credentials are included in this public report.
