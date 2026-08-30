# Capstone Report — Refresh Opportunity Scoring

- **Author:** Jasper Owen
- **Lane:** Refresh Opportunity Scoring
- **Repo:** https://github.com/JasperOwen/flyrank-ml-internship.git
- **Date:** 30/08/2026

## 0. Abstract

Five sentences, written last, placed first: question → data → method → headline result →
what the output is for. This is the top of your deployed paper.

## 1. Problem framing

**Research question:** Which pages should be reviewed first for expansion, reduction or refreshing based on changes in visibility and engagement compared to their previous rates?

**Decision:** My work improves the decision of which pages to review first for potential updates. This is because instead of manually reviewing each page they find, or using static heuristics, an editing team can use my work to see which pages my model recommends that they should review for refresh. They can then use that data to help them decide which pages to review.

**Unit of analysis:** One page that belongs to one client, with its performance summarised across the March 2026 observation period.

**Output:** A ranked queue, where each page in the dataset is ranked by its priority score, which suggests the order that the pages should be reviewed for refresh. Each ranking also comes with a reason code, to let the user know why the page is ranked where it is, and an action they could take to improve the page.

**Action:** The user can prioritize their editorial workflow by focusing on the pages at the top of the queue, as these will be the pages most likely to need their attention. They can then review those pages and determine if the action recommended by the queue will improve the page's performance, and choose to carry out that action, a different action, or no action at all.

**Cost of being wrong:**

* False positive: The editor potentially wastes time and resources reviewing a page that does not need to be improved. This could reduce their trust in the model.

* False negative: A page with declining performance may go unnoticed, which means an opportunity to improve it may be missed.

**Why Machine Learning helps:** An editing team may have too many pages to review individually, and manual heuristic rules may not be accurate, so combining multiple performance signals into a ranked queue can help them use their limited review time more efficiently by prioritising pages that need reviewing for refresh the most.


## 2. Data safety

**Dataset:** I used the March 2026 production warehouse release from FlyRank/internship-warehouse on Hugging Face. The table I used was fact_content_daily_performance filtered by month=2026-03 (March 2026). In the fact_content_daily_performance table, one row is the performance of one page that belongs to one client in one day.

However, in the dataset I used on my model, the data is aggregated so each row represents the performance of one page that belongs to one client, summarised across March 2026.

I divided the data into three time periods:

* Week 1: March 1–7, 2026
* Week 2: March 8–15, 2026
* Late March: March 16–31, 2026

Week 1 and Week 2 were used to construct features available at the March 15 decision point, while Late March was used to measure the future outcome. After filtering for pages with gsc and ga4 information available across March 2026, the modelling dataset contained 13,531 page records.

**Excluded data:** The fact_content_daily_performance table contained many columns and records that I did not use in my final model. For example, client_hash_id and content_hash_id were retained as context to identify pages, but they were not used as features in my model.

Also, columns related to the date such as report_date and month were only used to identify rows that were in March 2026. Similarly, client_has_gsc, client_has_ga4, gsc_data_available and ga4_data_available were used to identify which records had available features, without being used as features themselves.

In addition, any data from outside March 2026 was not included, as it was outside of the time period I was studying. Also, any data that did not have both gsc and ga4 data available was excluded, as that would mean it did not have the data I used as my features. Finally, any data from after my decision point: 15th March 2026, was excluded from being used as a feature from my model.

**Leakage risks:** None of my features come from after my decision point (15th March 2026), meaning no Late March data was used as a feature. Also, I measured correlations of each of my features against my future_decline label to check that no feature had an unusually strong correlation with future_decline, which could indicate data leakage. I did not see any unusually strong correlations.

**Privacy:** The names of each client and their pages did not appear in the dataset. Instead, each client was given a hashed ID to use instead, in the column client_hash_id. The same process was used for each page, resulting in the creation of the content_hash_id column. This means that I never saw the names of any clients or pages, and that no identifying information appeared in any of my work.

## 3. Baseline

I created a simple baseline rule to evaluate my models against  

**baseline_score = historical_tier_change × visibility_score**

where historical_tier_change is the difference between search ranking tiers of a page between Week 1 and Week 2 of March 2026 and Visibility score is determined by the number of impressions a page received in week 2:  
Tiers: (1 = top 3 search results, 2 = positions 4-10, 3 = positions 11-20, 4 = positions 21–50, 5 = > position 50)  
Visibility Score: (low ≤8 = 1, medium 9–459 = 2, high > 459 = 4)

I used this rule to calculate the baseline score of each page across the same Group KFold that my models used. The mean Precision@20, Precision@50 and Precision@100 scores of my baseline rule across the 5 folds were 0.88, 0.816 and 0.750.


The baseline provided a simple benchmark to compare my models against. It uses the exact same data as the rest of my models and maintains the idea that a highly visible page that is showing signs of declining performance should be prioritised for review. This baseline allowed me to test if Machine Learning models were more suitable to solve the problem of which pages to prioritise for review than simple heuristics.


## 4. Model / analysis

I created 3 models and compared them against each other and the baseline: a Logistic Regression model, a Decision Tree model and a Random Forest model

The mean results I got across each client fold were:  
| Model | Mean Base Rate | Precision@20 Mean | Precision@50 Mean | Precision@100 Mean |
| :--- | :---: | :---: | :---: | :---: |
| **Baseline score** | 0.2041 | 0.8800 | 0.8160 | 0.7500 |
| **Logistic Regression** | 0.2041 | 0.8400 | 0.8200 | 0.8260 |
| **Decision Tree** | 0.2041 | 0.8200 | 0.8440 | 0.8220 |
| **Random Forest** | 0.2041 | **0.9100** | **0.8760** | **0.8660** |

As my Random Forest had the highest Precision@50 mean out of all my models and the baseline, that was the model I used to create my Ranked Queue. I did this by combining the probability that a page would decline produced by my model with the baseline score to create a final priority score for each page. I then ranked the pages by the priority score, and added reason codes and suggested actions to make the queue useful to editors.

The features I used in my Random Forest model were, in order of importance to my model:  
historical_tier_change
week1_avg_position
w1_tier
week2_avg_position
week1_impressions
week2_impressions
w2_tier
log_w2_impressions
impression_change_w1_w2
w2_ctr
log_w2_clicks
click_change_w1_w2

These features represented 3 types of performance signal:
* Ranking: Week 1/Week 2 average position, Week 1/Week 2 tier, historical tier change
* Visibility: Week 1/Week 2 impressions, log Week 2 impressions, impression change
* Engagement: Week 2 CTR, log Week 2 clicks, click change

I did not use client or page identifiers as model features, as these were retained only to identify and group pages into folds. I also did not use date or data-availability columns as predictive features. Information outside of the March 1-15 decision time was not used as a model feature either.

I defined the target label, future_decline, as 1 when a page's Late March (16-31) position tier was worse than its Week 1 position tier, and 0 otherwise. This was so the model could identify an outcome to predict. No Late March data was used as a feature, only to construct the target label.

## 5. Evaluation

Your split (grouped by client? time-aware?) and why. Metrics, model vs baseline **on the same
split**. What the errors look like — a short error analysis beats a big metric table.

## 6. Interpretation

What the model/clusters actually found. Feature importances or cluster profiles in plain
words. Surprises and negative results — a well-understood "no effect" is a valid result.

## 7. Recommendation

The ranked actions or decisions your output supports, and how a FlyRank editor would use them
tomorrow. State your confidence and the limits explicitly.

## 8. Reproducibility

The exact commands to re-run everything from a fresh clone, your random seeds, and your
environment (`pip freeze` highlights or `requirements.txt` deltas). If you claim a sealed or
holdout evaluation, two things must be committed: the cell/script that builds the sealed
frame, and the metrics file it produced — "evaluated once, blind" should be checkable from
your repo, not taken on faith.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset" **https://flyrank.ai**.

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
