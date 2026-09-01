# Capstone Report — Refresh Opportunity Scoring

- **Author:** Jasper Owen
- **Lane:** Refresh Opportunity Scoring
- **Repo:** https://github.com/JasperOwen/flyrank-ml-internship.git
- **Date:** 30/08/2026

## 0. Abstract

Search traffic decay can cause a major loss of discoverability and revenue for website owners. Therefore, as part of the Flyrank Refresh Opportunity Scoring Lane, I investigated whether observable page performance data can be used to identify signs of future content decay and prioritize web pages for review.

Using 13,531 page records from March 2026, I created a leak-free 12-feature dataset, and evaluated multiple models on it using a 5-fold client-holdout split. My Random Forest model achieved a mean Precision@50 score of 0.876, which was the highest mean Precision@50 out of all my models.

Finally, I converted these predictions into a priority score between 0 and 100, and created a series of recommended actions to accompany these priority scores. This resulted in 18.6% of the pages in the dataset (2,520/13,531) being given the review_for_refresh suggested action, which human editors can use as a guide when deciding if a page needs refreshing or not.


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

I split my data into 5 GroupKFolds by client_hash_id. This meant that I used 5 folds where, in each fold, all the pages belonging to one client were either in the testing or training sections of the data. This meant that I could test my models on pages that belonged to clients they had never seen before, which prevented my models from accidentally trying to identify which client owned a page, instead of if that page was in decline or not. When I tried to use RandomKFolds, where a client's pages could be in either training or testing data, my Random Forest achieved an unusually high Precision@50 score of 0.9, which suggested that the random split was affected by data leakage, so I split my data using GroupKFolds for my final model.

I evaluated the models using the same five client folds so that their Precision@20, Precision@50 and Precision@100 scores would reflect the models' performance across the same data, making the comparison fair. While I chose Precision@50 as my main metric, I also recorded each model's Precision@20 and Precision@100 scores to see how well each model performed across larger queues, and if there was a certain queue size that a particular model would be more suited for. 

My Random Forest model achieved the highest Precision scores across each metric, with a Precision@20 score of 0.9100, Precision@50 score of 0.8760 and Precision@100 score of 0.8660 compared to my baseline's scores of 0.8800, 0.8160, 0.7500. 

I also examined the errors that my Random Forest model made. An example of a false positive error was when my model gave a page a high probability of declining after my model observed that the page experienced a ranking drop in early March, but it recovered in the second half of March. This shows that my model can struggle to differentiate SERP turbulence from actual page decline. Also, an example of a false negative error was when a page did not decline in search rankings between weeks 1 and 2, so it was not flagged by my model. However, the page's search rankings did suddenly drop in late March. This shows that my model cannot identify when a page will suddenly decline.


## 6. Interpretation

When I examined the importance of the features used by my Random Forest model, the top 3 most important features were:
* historical_tier_change
* week1_avg_position
* w1_tier

Week1_avg_position and w1_tier are different versions of the same data: a page's search rankings in week 1. Combined with the fact that historical_tier_change is the most important feature to the model, this suggests that my model is using the momentum of a page’s search rankings to predict if it will decline.

However, this does not necessarily mean that search rankings alone indicate how well a page is performing online. This just means that the main features used by my model to predict whether a page will decline in late march are related to a page's search rankings and the direction it is moving in those rankings.


## 7. Recommendation

I have used my model to create a ranked queue that should be used to help FlyRank editors decide which pages to review first.

The pages are ranked using a priority score that combines the Random Forest's probability of future decline with my baseline score. Each page is also given a reason code that explains why it has been ranked where it is and a suggested action for the editor to take, which will be one of the following:

* review_for_refresh: The page's content should be reviewed for refresh
* review_title_and_ctr: The page's title should be reviewed, to improve Click Through Rate (CTR)
* review_for_expansion: The page's content does not need refreshing, but it could be expanded to improve the number of views
* monitor: The page is not predicted to decline, it does not require a refresh at this time

An editor could start with the highest-ranked pages, use the reason codes and performance data to understand why they were prioritised, and then look at the page and decide whether to refresh it, expand it, change the title, or do nothing at all. 

I am confident that my ranked queue will be helpful to a Flyrank editor, as both my Random Forest and my baseline have high Precision@50 values across the 5 folds I evaluated them on: 0.876 for the Random Forest and 0.816 for my baseline rule. This suggests that both the model and the baseline rule are helpful for identifying pages that are likely to decline, allowing editors to prioritise these pages for review. However, the ranked queue is only meant to support the decisions of the editors. It does not guarantee that a page is in decline, or that carrying out the recommended action will improve the page's performance. The final decision of whether a page is declining, whether it should be refreshed or not, and what that refresh should be, is always up to the editor.

## 8. Reproducibility

To reproduce my work, clone this repository and open work/notebooks/capstone.ipynb in Google Colab (Google account required). The default Colab environment will be sufficient to run the notebook. No additional packages or specialist environments will be required.

However, the notebook does require access to the FlyRank ML internship dataset on Hugging Face. To do this, you will need permission to access the dataset in Hugging Face, and you will need to add a Hugging Face token to Google Colab Secrets under the name HF_TOKEN. Once you have done this and given the notebook access to the dataset, you can click Runtime → Run all to run the notebook, which will create my features and target, evaluate my models using the 5-fold client-held-out validation, create the ranked queue, and export the queue and supporting figures, which will be found in Google Colab files in work/outputs and work/figures.


## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset **https://flyrank.ai**.

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
