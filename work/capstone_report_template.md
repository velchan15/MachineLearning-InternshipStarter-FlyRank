# Capstone Report — Content Decline Risk Ranking

- **Author:** Evelyn Anastasia
- **Lane:** Content Opportunity Scoring (Refresh/Content lane)
- **Repo:** https://github.com/velchan15/MachineLearning-InternshipStarter-FlyRank
- **Date:** September 08, 2026

> Copy this file to `work/capstone_report.md` and fill it in as you build. The eight
> sections mirror the Pass / Needs-Work rubric axes, so nothing here is optional.

## 1. Problem framing

This work supports a content/SEO team's decision of **which pages to review first** in a limited-time refresh cycle. The unit of analysis is one published content page. The output is a **risk score and rank** (probability a page is already declining), grouped into five archetypes with a recommended action per archetype. The action a human takes from it: pull the top-ranked pages into the next content review queue and decide, page by page, whether to refresh, consolidate, or leave alone. The cost of a wrong call is asymmetric — missing a genuinely declining high-traffic page costs more (lost visibility compounds) than over-flagging a stable page for a review that turns out unnecessary (wasted reviewer time only). ML helps here because manually reviewing hundreds of pages every cycle isn't feasible; a ranking model turns "read everything" into "read the top 50 first."

## 2. Data safety

**Data used:** `content_refresh_anonymized.csv`, a single trailing-90-day snapshot, 30,000 content pages across 32 client accounts. All IDs (`content_id`, `client_id`) are pseudonyms, used only for grouping/splitting, never as model features.

**Columns deliberately excluded:**
- `trend_direction` and `trend_pct` — these directly define the label (`is_declining`), so including them would be circular.
- `impressions_last_30d`, `clicks_last_30d`, `sessions_last_30d` — verified that `trend_pct ≈ (impressions_last_30d − impressions_prev_30d) / impressions_prev_30d` (correlation ≈ 0.99999). Keeping the "last 30 days" window would let the model reconstruct the label directly rather than predict it from prior information. Only the `*_prev_30d` window (fully prior to the label's own window) is used.

**Leakage risks considered:** beyond excluding the columns above, the exclusion was tested directly — deliberately re-adding `impressions_last_30d` spiked precision@50 to a suspicious 1.000, confirming both that the leak was real and that the evaluation harness is sensitive to it. Remaining features were checked for raw correlation with the label; the strongest (`days_with_impressions`) sits at 0.19, far from the near-1.0 signature of a leak.

**Client safety confirmation:** no client names, real URLs, or raw search queries appear anywhere in `work/` or in the deployed paper — only pseudonymous `content_id`/`client_id` values, which carry no identifying information on their own.

## 3. Baseline

The baseline is a transparent rule, not a fitted model: flag a page if it is **stale** (no update in 91+ days) **and** still **visible** (impression tier "moderate" or better), ranked by `impressions_prev_30d`. This is a fair comparison because it uses only information available before the label's own window, and represents what a reasonable person could build with a spreadsheet filter — the bar the model needs to clear to be worth using.

**Baseline numbers**, recomputed on the same held-out test split used for the model (n=6,163, base rate 51.1%): Precision@10 = 0.300, Precision@50 = 0.320.

## 4. Model / analysis

**Method:** Logistic Regression, selected after a direct comparison against Random Forest on the same split and metric — Logistic Regression scored higher, so the simpler model is the one used, per the principle that added complexity should earn its place rather than be assumed better.

**Feature list:** `search_volume`, `competition`, `cpc`, `word_count`, `char_count`, `impressions_90d`, `clicks_90d`, `pageviews_90d`, `sessions_90d`, `users_90d`, `engaged_sessions_90d`, `ai_sessions_90d`, `scroll_events_90d`, `days_with_impressions`, `days_with_sessions`, `impressions_prev_30d`, `clicks_prev_30d`, `sessions_prev_30d`, `content_age_days`, `days_since_last_update`, `ctr`, `avg_position`, `engagement_rate`, `scroll_rate`, `ai_traffic_pct`, plus categorical fields (`content_type`, `main_intent`, `provider_used`, `model_used`, `competition_level`, `age_tier`, `freshness_tier`, `word_count_tier`, `char_count_tier`, `impression_tier`, `position_tier`) and explicit `has_*` missingness flags for fields where missingness is non-random (missingness follows `content_type`).

**Left out on purpose:** the three `*_last_30d` columns and the label-derived `trend_direction`/`trend_pct` (see Section 2).

**Target definition, in one sentence:** `is_declining` = 1 if the page's most recent 30-day impressions are lower than its prior 30-day impressions by enough to register `trend_direction == "down"`, else 0.

## 5. Evaluation

**Split:** grouped by `client_id` (32 clients, 80/20), not a random row split — pages from the same client share templates and traffic patterns, so a random split risks letting the model memorize client-specific quirks instead of learning a pattern that generalizes to an unseen client. Tested directly: the same model under a naive random split measured Precision@50 = 0.940, a full 10 points higher than the grouped split's 0.840 — evidence of how much score a careless split can quietly borrow from memorization.

**Metrics, model vs. baseline, same split** (n=6,163, base rate 51.1%):

| Method | Precision@10 | Precision@50 |
|---|---|---|
| Baseline rule | 0.300 | 0.320 |
| Random Forest | 0.500 | 0.560 |
| **Logistic Regression** | **0.800** | **0.840** |

**Error analysis:** reviewing the top-50 ranked pages, false positives cluster in the "excellent" impression-tier bucket with much higher average `impressions_prev_30d` than the true positives in the same top-50 — the model appears to over-weight sheer traffic volume, occasionally flagging large, stable pages as declining risks. This is a small-sample observation (a handful of rows), noted as directional rather than conclusive.

## 6. Interpretation

**Top features (permutation importance, Logistic Regression):** `impressions_90d`, `days_with_impressions`, `users_90d`, `impressions_prev_30d`, `days_with_sessions`, `avg_position` — all traffic-volume and visibility-consistency signals, with no single feature dominating suspiciously. This makes sense descriptively: a page visible on fewer days recently, or with lower overall traffic history, is plausibly the one already declining.

**Surprise / negative result worth flagging:** the decline rate by freshness tier is not monotonic. Pages last updated 91–180 days ago show the *highest* observed decline rate (61.1%), but the 181+ day tier shows a *lower* rate (47.1%) than even fresh content (51.1%) — backwards from what "older content decays more" would predict. The most likely explanation is survivorship bias: pages still tracked at 181+ days without having been retired or de-indexed are probably the ones that already stabilized, while pages that kept declining past 180 days may have already been pulled from the portfolio before this snapshot. This is reported as an open question, not smoothed over.

## 7. Recommendation

The validated model's scores are turned into a ranked queue with reason codes and five archetypes:

| Archetype | n (test) | Recommended action |
|---|---|---|
| High-Traffic Decliner | 786 | Priority refresh this quarter |
| Stale & Fading | 799 | Refresh or consolidate into a stronger page |
| Early Warning | 2,066 | Add to next review cycle, not urgent |
| Stable Performer | 560 | No action, light monitoring only |
| Low-Priority / New | 1,952 | No action, revisit next quarter |

**How a FlyRank editor uses this tomorrow:** open the ranked queue, start from the top (highest risk score), skip anything already reviewed this cycle, and use the reason code to decide the specific fix (e.g. "stale-content + high-visibility" → check for outdated facts/links before rewriting).

**Confidence and limits, stated explicitly:** Precision@50 = 0.84 is a batch statistic for this test sample, not a per-page guarantee — some of the top 50 will be wrong, and the model can't say in advance which ones. This is a cross-sectional snapshot; the model has not been validated against actual refresh outcomes, so it supports *what to review first*, not proof that reviewing it will help. It should never be used to auto-publish, auto-delete, or evaluate content creators — those decisions stay human.

## 8. Reproducibility

**Repo:** https://github.com/velchan15/MachineLearning-InternshipStarter-FlyRank

**Notebooks (run in this order to reproduce every number in this report):**
- `work/notebooks/w05_model.ipynb` — model build, baseline comparison
- `work/notebooks/w06_validation_audit.ipynb` — split validation, leakage audit
- `work/notebooks/w07_action_playbook.ipynb` — ranked queue, archetype mapping
- `work/notebooks/capstone.ipynb` — consolidated end-to-end run behind this report and the deployed paper

**Random seed:** `random_state=42` used throughout (train/test split and both models), for exact reproducibility.

**Environment:** Python 3, `pandas`, `numpy`, `scikit-learn`, `matplotlib`. No GPU or special hardware required; runs end-to-end in Google Colab's default environment.

**To rerun from a fresh clone:** open any notebook above in Colab, Runtime → Run all — each notebook downloads its own copy of `data/raw/content_refresh_anonymized.csv` directly from this repo, so no manual data setup is needed.

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
> **Metrics vs. base rate:** base rate reported (51.1%) next to every precision@K number
> throughout this report, so a reader can judge how much the score adds over chance.
