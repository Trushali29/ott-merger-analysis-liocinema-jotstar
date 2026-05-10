# OTT Merger ML Project — Complete Explanation


## 1. WHAT WAS THE BUSINESS SITUATION?

Two OTT (streaming) platforms — **Jotstar** and **LioCinema** — were merging
into one single platform.

Jotstar had **44,620 subscribers** and LioCinema had **183,446 subscribers**
giving a combined dataset of **228,066 users**.

After any merger, subscriber behaviour changes. Some users leave. Some
downgrade their plan. Some upgrade. The business needed to know IN ADVANCE
which users were at risk — so they could take action before losing them.

---

## 2. WHAT WAS THE PROBLEM STATEMENT?

> Given a subscriber's profile, subscription plan history, and content
> consumption behaviour — predict what will happen to them after the merger.

This is a **classification problem** — for each user, predict one of 4 outcomes:

| Class | Meaning | Count | % |
|---|---|---|---|
| Retained   | Still active, never changed plan | 107,005 | 46.9% |
| Churned    | Stopped using the platform       |  88,957 | 39.0% |
| Downgraded | Moved to a lower plan            |  26,054 | 11.4% |
| Upgraded   | Moved to a higher plan           |   6,050 |  2.7% |

---

## 3. WHY DO WE EVEN NEED ML FOR THIS?

You might ask — why not just look at the data manually?

Because we have 228,066 users. No human can look at each user and decide
"this person will churn". But a ML model can learn patterns from thousands
of users and apply them instantly to everyone.

Also, the decision is not simple. Churn depends on multiple things at once:
- How much someone watches
- How long they have been a subscriber
- What plan they are on
- Whether they changed plans before

A human cannot weigh all these factors together for 228,066 users.
A ML model can — in seconds.

**When to use ML in general:**
- When you have a lot of data (we had 228,066 rows)
- When the pattern is too complex for simple rules
- When you want to predict something about the future
- When the cost of being wrong is high (losing a customer is expensive)

---

## 4. WHAT DATA DID WE HAVE?

We had 6 files across 2 platforms:

**Subscriber files** — who the user is:
- age_group, city_tier, subscription_plan
- subscription_date, last_active_date, plan_change_date

**Consumption files** — what the user did:
- total_watch_time per device (Mobile, TV, Laptop)

**Content files** — what content exists on the platform

We used subscribers and consumption files for this project.

---

## 5. HOW DID WE DEFINE CHURN FROM RAW DATA?

The raw data did not have a "churned" column. We had to create it.

The platform recorded a `last_active_date` when a user went inactive.

```
last_active_date is EMPTY → user is still active → NOT churned (0)
last_active_date has DATE → user stopped         → CHURNED (1)
```

Similarly for plan changes:
```
plan_change_date is EMPTY → never changed plan
new plan rank > old rank  → UPGRADED
new plan rank < old rank  → DOWNGRADED
```

This became our target variable — `subscription_plan_status`:
```
0 = Retained
1 = Downgraded
2 = Upgraded
3 = Churned
```

---

## 6. DATA CLEANING — WHAT PROBLEMS DID WE FIX?

**Problem 1 — Date formats were different**
- Jotstar used YYYY-MM-DD
- LioCinema used DD-MM-YYYY
- Fix: pd.to_datetime() with correct format for each

**Problem 2 — Text columns**
- ML models only understand numbers, not "Tier 1" or "18-24"
- Fix: mapped all text to numbers manually

```
age_group:         18-24=0, 25-34=1, 35-44=2, 45+=3
city_tier:         Tier 1=1, Tier 2=2, Tier 3=3
subscription_plan: Free=0, Basic=1, Premium=2, VIP=3
platform:          Jotstar=0, LioCinema=1
new_subscription_plan: same as above, -1 if never changed
```

**Problem 3 — Skewed numeric columns**
- total_watch_time had skewness of 2.6 (most users watch little,
  few users watch a lot — extreme outliers)
- Fix: applied log1p transformation to compress outliers

**Problem 4 — Raw dates cannot go into a model**
- You cannot feed "2024-06-10" to a ML model
- Fix: extracted meaningful numbers from dates

---

## 7. FEATURE ENGINEERING — WHAT NEW COLUMNS DID WE CREATE?

This was the most important step. We created 4 new features from dates:

**days_since_last_active**
```
If user still active  → 2024-12-31 minus subscription_date
If user churned       → last_active_date minus subscription_date
Meaning: total days the user spent on the platform
```

**days_before_new_plan**
```
If never changed plan → 0
If changed plan       → plan_change_date minus subscription_date
Meaning: how many days before they switched their plan
A user who switches after 10 days is very different
from one who switches after 300 days
```

**days_after_new_plan**
```
If never changed plan      → 0
If changed, still active   → 2024-12-31 minus plan_change_date
If changed, then churned   → last_active_date minus plan_change_date
Meaning: how long did the new plan satisfy them?
```

**days_in_new_plan**
```
If never changed plan → 0
If changed plan       → 2024-12-31 minus plan_change_date
Meaning: total duration on the new plan till end of dataset
```

**total_watch_time**
```
Summed watch time across all devices per user
Mobile + TV + Laptop = total engagement
```

**Final feature list going into the model:**
1. age_group
2. city_tier
3. subscription_plan
4. new_subscription_plan
5. platform
6. total_watch_time
7. days_since_last_active
8. days_before_new_plan
9. days_in_new_plan

---

## 8. HOW DID WE TRAIN THE MODEL?

**Train Test Split:**
```
80% of data = Training set (182,452 users) → model learns from this
20% of data = Test set     (45,614 users)  → model is tested on this

The model NEVER sees the test set during training.
This tells us how it performs on brand new users.
```

**Models we trained:**

Model 1 — Decision Tree
- Simple model that asks yes/no questions like a flowchart
- Easy to visualise and explain
- Accuracy: 81.98%

Model 2 — Random Forest
- Builds 100 decision trees and takes a majority vote
- More accurate than a single tree
- Accuracy: 87.56%

---

## 9. WHAT WERE THE RESULTS?

**Random Forest — 87.56% accuracy**

| Class | Precision | Recall | F1-score |
|---|---|---|---|
| Retained  | 0.86 | 0.88 | 0.87 |
| Downgrade | 1.00 | 1.00 | 1.00 |
| Upgrade   | 1.00 | 1.00 | 1.00 |
| Churned   | 0.85 | 0.82 | 0.84 |

**What these numbers mean:**

Downgrade and Upgrade → Perfect 1.00
The model predicted every single downgrade and upgrade correctly.
This is because days_before_new_plan and days_in_new_plan are
extremely strong signals for these classes.

Retained → F1 of 0.87
Out of 21,401 retained users, the model correctly identified
18,905. It confused 2,496 retained users as churned. This is
understandable — a user who watches little but stays looks
similar to one who is about to churn.

Churned → F1 of 0.84
Out of 17,792 churned users, the model caught 14,612 correctly.
It missed 3,180 — predicted them as retained. These are likely
users who were still somewhat active before leaving.

---

## 10. WHAT DID FEATURE IMPORTANCE TELL US?

| Feature | Importance | Business meaning |
|---|---|---|
| total_watch_time     | 0.378 | Biggest churn signal |
| days_since_last_active | 0.236 | How long they stayed |
| new_subscription_plan | 0.111 | Plan change direction |
| days_before_new_plan | 0.082 | Speed of plan switch |
| days_in_new_plan     | 0.072 | Satisfaction with new plan |
| subscription_plan    | 0.070 | Original plan tier |
| age_group            | 0.020 | Minor signal |
| platform             | 0.016 | Minor signal |
| city_tier            | 0.014 | Weakest signal |

**The single most important insight:**

BEHAVIOUR beats DEMOGRAPHICS.

Watch time and activity duration together explain 61% of all
predictions. Age, city, and platform barely matter at all.

This tells the business: churn is not about who the user is.
It is about whether they are engaged with the content or not.

---

## 11. WHAT SHOULD THE BUSINESS DO WITH THIS MODEL?

**For users predicted as CHURNED:**
→ Send them personalised content recommendations
→ Offer a discount or free trial of premium content
→ Send re-engagement notifications

**For users predicted as DOWNGRADE:**
→ They are losing perceived value — show them premium content
→ Offer a loyalty discount before they downgrade
→ Highlight features they are missing on their current plan

**For users predicted as UPGRADE:**
→ These are your happiest users — understand what they watched
→ Replicate those content conditions for other users
→ These users are good candidates for referral programs

**For users predicted as RETAINED:**
→ Keep doing what is working
→ Monitor their watch time — if it drops, they may be at risk

---

## 12. COMPLETE PROJECT SUMMARY

```
Step 1 — Data Collection
         6 files, 2 platforms, 228,066 subscribers

Step 2 — Data Merging
         Combined Jotstar + LioCinema into one dataframe
         Added platform flag column

Step 3 — Target Variable Creation
         Created subscription_plan_status from
         last_active_date and plan_change_date

Step 4 — Feature Engineering
         Created 4 date-based features
         Summed watch time per user across devices

Step 5 — Data Cleaning
         Parsed dates, encoded text to numbers,
         fixed skewed columns with log1p,
         handled nulls in new_subscription_plan

Step 6 — Model Training
         80/20 train test split
         Trained Decision Tree and Random Forest

Step 7 — Evaluation
         Random Forest → 87.56% accuracy
         Perfect scores on Downgrade and Upgrade
         Strong scores on Retained and Churned

Step 8 — Feature Importance
         total_watch_time is the #1 churn predictor
         Demographics (age, city) barely matter
         Behaviour is everything
```

---

## FINAL TAKEAWAY

The model answers one question for every user:

"Given everything we know about this user —
 how much they watch, how long they have been here,
 what plan they are on, and whether they changed it —
 will they stay, leave, upgrade, or downgrade?"

With 87.56% accuracy, the business can now act on
predictions BEFORE users actually churn — saving
revenue and improving retention at scale.