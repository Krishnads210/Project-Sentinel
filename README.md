# Project Sentinel – POD Discussion Report

This repository contains the pod discussion report for the **SENTINEL LM** project, a fraud-detection model.

> **Note for reviewers:** This task is a **discussion report**, not a coding assignment. The deliverable is the PDF report below; there is no source code for this submission.

## Contents

| File | Description |
|------|-------------|
| `Sentinel_POD_Report.pdf` | Full pod discussion report |

## About the report

DELIVERABLE: ONE CONSOLIDATED POD REPORT
1. Dataset overview: purpose, structure, number of records and
attributes
Purpose: Data for building the Sentinel LM.
Structure and number of records: 14 months of labelled transactions (about 1.2 crore rows,
Jan 2025 – Feb 2026)
Attributes
Field Type
transaction_id string
customer_id string (hashed)
transaction_amount decimal ( ) ₹
merchant_category categorical
transaction_time timestamp (IST)
device_id string (hashed)
customer_age integer (years)
customer_location lat/long + city
merchant_location lat/long + city
previous_transactions integer
average_transaction_value decimal ( ) ₹
failed_transactions integer
account_age integer (days)
fraud_label binary (0/1)
2. Data dictionary: every attribute with its data type, business
meaning and example values
Field Type Description Example
transaction_id string Unique transaction reference TX98231
customer_id string (hashed) Pseudonymised payer identifier C_7f31a9
transaction_amount decimal ( ) ₹ Value of the transaction 50000.00
merchant_category categorical MCC group (travel, gaming,
electronics, etc.)
electronics
transaction_time timestamp (IST) Date and time of the authorisation
request
2025-11-14 03:12:44
device_id string (hashed) Fingerprint of the device used D_44be20
customer_age integer (years) Payer's age from KYC 27
customer_location lat/long + city Location of the payer at transaction
time
Pune
merchant_location lat/long + city Registered location of the merchant Delhi
1
Field Type Description Example
previous_transactions integer Count of the payer's transactions in
the last 30 days
14
average_transaction_valu
e
decimal ( ) ₹ Payer's mean transaction value over
90 days
2,350.00
failed_transactions integer Failed attempts by the payer in the
last 24 hours
4
account_age integer (days) Days since the payer's account was
created
6
fraud_label binary (0/1) 1 = confirmed fraud, 0 = genuine 1
3. Business mapping: how the dataset relates to the SENTINEL
client requirements
The main goal of the Sentinel LM project is to prevent or reduce fraud. In the current dataset we
have attributes like transaction amount, customer_location, merchant location, account age, etc.
The example rule that was provided to us in the Project document was:
“A 50,000 transaction is not suspicious by itself. The same 50,000 from a new device, in ₹ ₹
a new city, at 3 AM, right after several failed attempts, very likely is. Fraud is a combination
of context, behaviour and timing, which rules capture poorly and which a learned model
can capture well.”
So the data basically has all these data points and will be the main source for building the model.
4. Data quality observations: missing values, duplicates,
inconsistencies and other findings
The file is clean structurally: it parses fully, every ID is well formed, and there are no duplicates.
The problems are in a few missing fields, some extreme amounts, column names that don’t
match what the data holds, and how the train/validation/holdout splits are built.
4.1 Missing values
Only four columns have gaps. All of them are empty strings rather than NA/null text.
Column Missing Pattern
customer_location 3,257 (3.0%) Spread across all channels and splits; only 62 fall on a
customer’s first transaction
average_transaction_valu
e
2,829 (2.6%) Not random. It’s empty exactly when the customer had no
transaction in the previous 90 days (2,480 are first transactions,
the rest follow gaps of more than 90 days)
device_id 1,138 (1.0%) All UPI. 858 of them are in the train split
everything else —
4.2 Duplicates
• No full-row duplicates, no repeated transaction_id, and no duplicates even when the ID is
ignored.
2
• There are 318 back-to-back transactions from the same customer less than 1 second apart
(smallest gap: 2 ms), and 3,169 less than 10 seconds apart. These could be bursts of
fraud, or retries or double submissions that need deduplicating.
4.3 Inconsistencies and misleading semantics
• Redundant columns: amount_paise always equals transaction_amount × 100, with 0
mismatches.
• Constant column: is_synthetic is true on every row, so it carries no information.
• Timestamp precision varies: milliseconds have trailing zeros stripped (.602, .73, .9, or
none at all). Pandas parses them fine, but strict parsers may not. Can be removed.
• Customer location jumps around. 2,080 of 2,480 customers appear in 2 to 9 different
cities, with 9,002 city changes in total. 20 of those changes happen within an hour (4 within
10 minutes), which is physically impossible.
• Shared devices: 120 devices are used by more than one customer, one of them by 17.
2,141 customers use more than one device (up to 8).
• Age vs account age: on 4,942 rows the account is older than the customer’s adult years
(account_age/365 > age − 18). That includes 1,986 rows where an 18–19-year-old has an
account more than 2 years old.
• account_age = 0 on 25 rows. 24 customers have a constant account_age; most of them
have only 1–3 transactions.
• What is consistent: each customer has one age, each merchant has one category and one
location, each city name always has the same coordinates, account_age grows exactly
with elapsed time, and failed_transactions never exceeds previous_transactions.
4.4 Outliers
• Very skewed amounts: the median is 1,250, the 99th percentile 25k, and the maximum ₹ ₹
79.96 lakh
₹ . 147 transactions are over 1 lakh and 17 are over 50 lakh, typically about ₹ ₹
37× the customer’s own average.
• Implausible for the channel: 97 UPI transactions are over 1 lakh and 28 are over 5 ₹ ₹
lakh (UPI’s per-transaction caps are around 1–5 lakh). Some are in categories like ₹
grocery and quick-commerce. They look like injected anomalies.
• These outliers inflate the rolling average: 413 rows have average_transaction_value over
₹
1 lakh.
• 4 transactions are under 10 (the minimum is 8.20). ₹ ₹
4.5 Splits and coverage
• The splits are cleanly time-based on IST month boundaries: warmup is Oct–Dec 2024,
train Jan–Oct 2025, validation Nov–Dec 2025, holdout Jan–Feb 2026.
• Customer overlap: 2,429 of 2,480 customers appear in more than one split. That’s normal
for a time-based split, but customer-level features or target encoding will leak across splits.
50 customers appear only in holdout (cold start). Important while building the model.
• Volume roughly doubles from about 3.3k transactions a month in Oct–Dec 2024 to about
7.2k a month from Jan 2025.
• Timestamps are in UTC. The “18:xx” boundaries are midnight in IST, and the dip from
18:00 to 23:00 UTC is night-time in India. Use IST for any hour-of-day features.
3
4.6 Structural gaps
• There’s no label column (no is_fraud or similar), even though the features look built for
fraud detection. If the labels live in another table, it needs a join key, which is presumably
transaction_id.
• 16 customers have only one transaction and 49 have fewer than five, so their velocity
features are thin.
5. AI/ML opportunities: potential use cases and relevant attributes
As we are in the early stage of the discussion below is what we can follow from the project
document
1. 2. 3. Base model selection. Shortlist at least three open-weight models with a commercial-use
licence in the 1B–8B parameter range (for example Llama 3.x, Qwen 2.5, Mistral,
Gemma). Pick one on validation PR-AUC after a small pilot fine-tune, licence terms and
inference cost.
Continued pre-training (domain adaptation). Train on the ~135 M-token fraud text
corpus (Section 7.3) so the model learns payments vocabulary and investigator reasoning.
Supervised fine-tuning (SFT). Train on serialised transactions with two targets: the fraud
label (through a classification head on the final hidden state) and, for the 8,000 gold cases,
a short JSON explanation. Use parameter-efficient methods (LoRA / QLoRA) unless full
fine-tuning is shown to be clearly better.
6. Discussion notes: key observations, questions, decisions and
learnings
Key Observations
1. device_id missing with UPI type transaction can be directly classified as a fraud, as it could
be done through a theft device. This needs be investigated.
2. If the device location permission is off then whether the app can handle the transaction
without location permission? And is it needed and customers privacy also matter
sometimes
3. Customer and merchant cities barely match: only 6% are the same city, and that holds
even for grocery and quick-commerce (6%), where you’d expect local merchants.
Questions
A. We want understand more about is_syenthic field in the different tables.
B. A payer far from the merchant, or in a new city, raises risk - Why is this a risk. We need
more clarifications (refer 3rd Key observation)
C. Night transactions are not always risky. This needs to be clarified further and we need
other points as well.
D. Why is average only calculated for 90 days
4
7. Individual contributions: what each pod member explored and
explained
Both of us (Naga Teja and Krishna) worked together throughout the doc and discussed each
area in detail. Hence we explored the complete data together and explained each point in detail.
We worked as a team.
Naga Teja:
Krishna
5
## Pod members

- Krishna
- Naga Teja

## How to view

Open [`Sentinel_POD_Report.pdf`](./Sentinel_POD_Report.pdf) directly on GitHub, or download it.
