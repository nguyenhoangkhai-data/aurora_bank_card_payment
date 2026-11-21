# Card Payment Performance: Aurora Bank (Jan 2022 – Oct 2024)

A Power BI report that answers two questions for the Card Division: how reliable card payments are today, and which group of declined transactions should be addressed first.

## Table of Contents

1. [Background and Problem](#1-background-and-problem)
2. [Approach](#2-approach)
3. [Data and Model](#3-data-and-model)
4. [Key Metrics](#4-key-metrics)
5. [Power BI Report](#5-power-bi-report)
6. [Insights and Recommendations](#6-insights-and-recommendations)

---

## 1. Background and Problem

Between January 2022 and October 2024, Aurora Bank recorded 157,224 card transactions, of which 1.74% (2,738 transactions) were declined. Each team tracks errors in its own report with its own definitions, so the Card Division has no shared view to answer the following:

- Is the decline rate currently rising?
- Are declines caused by customers, by the bank's systems, or by security checks?
- Which group of transactions should be addressed first, and which team owns it?
- Which transactions look unusual and need to be reviewed right away?

The report is built for the Head of Card Division, who assigns follow-up actions after each monthly meeting, and for the team leads involved:

| User | Main need |
| --- | --- |
| Head of Card Division | Understand the overall picture and assign work to the right team |
| Card Product | Know where insufficient-balance declines happen and which customers they affect |
| Digital Payments | Understand why the online channel has more declines than the other channels |
| Risk Operations | Have a list of unusual transactions, ranked by review priority |

---

## 2. Approach

The project followed four steps: defining user needs with Design Thinking, preparing the data in Power Query, building the data model, and writing the DAX measures. After the first version of the report was complete, I reviewed it end to end and restructured the storyline so that each page leads into the next.

<details>
<summary><b>Design Thinking</b></summary>

Before building the report, I used Design Thinking to clarify who the audience is, what decisions they need to make, and which metrics should measure success.

**Step 1: Empathize.** The primary audience is the Head of Card Division. The need is framed as a single verb: *prioritize* which errors to fix first and assign them to the right team. From there, I defined seven key questions for the report, moving from the success rate, to where errors are concentrated, to the list of transactions that need review.

<details>
<summary><b><i>5W1H</i></b></summary>

![Design Thinking: 5W1H](<docs/image/Design Thinking/5W1H.png>)

</details>

<details>
<summary><b><i>Empathy Map</i></b></summary>

![Design Thinking: Empathy Map](<docs/image/Design Thinking/Empathy Map.png>)

</details>

<details>
<summary><b><i>Explore Dataset</i></b></summary>

![Design Thinking: Explore Dataset](<docs/image/Design Thinking/Explore Dataset.png>)

</details>

**Step 2: Define Point of View.** Two North Star metrics were selected: *Success Rate* (the share of successful transactions, used as the primary metric) and *Successful Amount* (the value of successful transactions net of refunds, used for context). Four points of view were then chosen: decline reason, payment channel, merchant category group, and transaction risk level.

<details>
<summary><b><i>North Star Metric</i></b></summary>

![Design Thinking: North Star Metric](<docs/image/Design Thinking/Northstar Metric.png>)

</details>

<details>
<summary><b><i>Define Point of View</i></b></summary>

![Design Thinking: Define Point of View](<docs/image/Design Thinking/Define Point of View.png>)

</details>

<details>
<summary><b><i>Growth Formula</i></b></summary>

![Design Thinking: Growth Formula](<docs/image/Design Thinking/Growth Formula.png>)

</details>

**Step 3: Ideate.** I listed the metrics and charts for each point of view, then organized them into four pages across three levels of information: critical, important, and detail.

<details>
<summary><b><i>Brainstorming</i></b></summary>

![Design Thinking: Brainstorming](<docs/image/Design Thinking/Brainstorming.png>)

</details>

<details>
<summary><b><i>Structure Idea</i></b></summary>

![Design Thinking: Structure Idea](<docs/image/Design Thinking/Structure Idea.png>)

</details>

Full Design Thinking workbook: [Design Thinking – Aurora Bank Project 4.xlsx](<power bi/Design Thinking – Aurora Bank Project 4.xlsx>)

</details>

<details>
<summary><b>Power Query</b></summary>

The source data consists of four Excel files in a local `Dataset` folder (not included in this repository). The folder path is stored in the `DataFolder` parameter, so moving the project to another machine only requires changing one value.

| Query | Source | Main steps |
| --- | --- | --- |
| fact_transactions_data | `transactions_data.xlsx` | Keep the 8 required columns, set data types, rename `mcc` to `mcc_id` |
| dim_users_data | `users_data.xlsx` | Keep only `id` |
| dim_cards_data | `cards_data.xlsx` | Keep only `id` and `card_type` |
| dim_mcc_codes | `mcc_codes.xlsx` | Group 109 MCC codes into 14 categories (`MCC Group` column) |

**Merchant category grouping.** The 109 MCC codes are grouped into 14 categories such as Food, Retail, Financial Services, and Transportation.

**Transactions with multiple errors.** Transactions with two or more errors are assigned to the *Multiple Errors* group, keeping one row per transaction.

</details>

<details>
<summary><b>Data Modeling</b></summary>

The model follows a star schema, with the fact table `fact_transactions_data`, three dimension tables taken from the source data, and two helper tables built in the model (see the diagram in [section 3](#3-data-and-model)):

- `dim_users_data[id]` relates to `client_id` in the transactions table.
- `dim_cards_data[id]` relates to `card_id`.
- `dim_mcc_codes[mcc_id]` relates to `mcc_id`.
- `dim_date` is a date table (`Date`, `Month Start`, `Year`) linked to the transaction date, used for the Year slicer and the monthly charts.
- `dim_error` lists each `Error Type` with its `Decline Reason Group` and sort order, linked on `Error Type`, used for the Decline Reason slicer and the reason charts.
- All relationships are one-to-many, filtering in a single direction from dimension to fact.

</details>

<details>
<summary><b>Transaction Flagging Rules</b></summary>

The data has no column confirming which transactions are fraudulent, so I used four simple rules to select transactions that should be reviewed:

| Rule | Condition | Transactions |
| --- | --- | --- |
| T1 High Value | Amount above $1,000 | 122 |
| T2 Peer Outlier | Amount in the top 1% of transactions with the same category group and channel (groups with at least 30 transactions) | 1,589 |
| T3 Authentication Error | Incorrect PIN or CVV | 445 |
| T4 Multiple Errors | Two or more errors on a single transaction | 18 |

Review priority is based on the number and combination of rules triggered:

| Priority | Condition | Transactions |
| --- | --- | --- |
| Urgent | Authentication error combined with an unusual amount (T3 + T1 or T3 + T2), or three or more rules | 9 |
| High | Exactly two rules | 30 |
| Monitor | One rule | 2,096 |

Rule T2 compares each transaction with similar transactions instead of applying a single amount threshold. As a result, the same amount can be normal at a supermarket but unusually high at a fast-food restaurant.

</details>

---

## 3. Data and Model

The dataset was provided in the Unigap Power BI module and consists of four Excel files describing Aurora Bank card transactions from January 2022 to October 2024.

| Table | Content | Grain | Rows |
| --- | --- | --- | --- |
| fact_transactions_data | Date, amount, payment channel, merchant category, and errors for each transaction | One transaction | 157,224 |
| dim_users_data | Customer list | One customer | 2,000 |
| dim_cards_data | Card list and card type (Credit, Debit, Debit (Prepaid)) | One card | 6,146 |
| dim_mcc_codes | Merchant category code, description, and category group | One MCC code | 109 |
| dim_date | Date table with date, month start, and year (built in the model) | One day | 1,035 |
| dim_error | Error type, decline reason group, and sort order (built in the model) | One error type | 9 |

Of the 2,000 customers and 6,146 cards, only 303 customers and 896 cards have transactions. The findings in the report therefore apply to active card users.

<details>
<summary><b>Main columns in the transactions table</b></summary>

| Column | Data type | Description |
| --- | --- | --- |
| id | Whole number | Transaction ID |
| date | Date/time | Transaction timestamp |
| client_id | Whole number | Customer ID |
| card_id | Whole number | Card ID |
| amount | Decimal number | Transaction amount; negative values are treated as refunds |
| use_chip | Text | Payment channel: Chip Transaction, Swipe Transaction, Online Transaction |
| mcc_id | Whole number | Merchant category code |
| errors | Text | Decline reason; blank if the transaction succeeded |

Main calculated columns: `Channel` (Chip, Swipe, Online), `Error Type`, `Error Attempt Flag`, `Peer Amount Percentile`, the four rule flags T1–T4, `Triggered Rules`, `Rule Combination`, and `Review Priority`.

</details>

![Data Model](<docs/image/Data Model/Data Model.png>)

---

## 4. Key Metrics

| Metric | Formula | Purpose |
| --- | --- | --- |
| Success Rate | (Total transactions − Declined transactions) ÷ Total transactions | North Star: share of customers who complete a payment |
| Decline Rate | Declined transactions ÷ Total transactions | Share of declined transactions, can be split by reason |
| Successful Amount | Total value of successful transactions, net of refunds | Payment volume through cards, used for context |
| Declined Amount | Total value of declined transactions | Value of payments that were not completed |
| Balance Decline Rate | Insufficient-balance declines ÷ Total transactions | Tracking KPI for the balance alert recommendation |
| Online Card Details Rate | Online transactions with a wrong card number, expiration date, or CVV ÷ Online transactions | Tracking KPI for the card detail validation recommendation |
| Technical Glitch Rate | Technical glitch transactions ÷ Total transactions | Share of errors caused by the bank's own systems |
| Review Cases | Transactions triggering two or more rules | Cases to review first (Urgent + High) |

---

## 5. Power BI Report

The report has four pages, read in order from overview to action. Each of the three analysis pages opens with a key takeaway box, and every page has a button that leads to the question answered on the next page. Click a page name to view its screenshot.

Full files: [PDF](<power bi/Aurora_Bank_Card_Payments.pdf>) · [Power BI (.pbix)](<power bi/Aurora_Bank_Card_Payments.pbix>)

<details>
<summary><b>1. Overview</b>: How stable card payments are and the main reasons transactions are declined</summary>

![1. Overview](<docs/image/Dashboard/1. Overview.png>)

</details>

<details>
<summary><b>2. Where and Why</b>: The decline reasons, payment channels, customer groups, and merchant categories most affected</summary>

![2. Where and Why](<docs/image/Dashboard/2. Where and Why.png>)

</details>

<details>
<summary><b>3. Review Queue</b>: Unusual transactions, ranked by review priority</summary>

![3. Review Queue](<docs/image/Dashboard/3. Review Queue.png>)

</details>

<details>
<summary><b>4. Recommendations</b>: Four recommendations, the supporting evidence, and the KPIs to track</summary>

![4. Recommendations](<docs/image/Dashboard/4. Recommendations.png>)

</details>

---

## 6. Insights and Recommendations

### Key Findings

- Card payments are stable: **98.26%** of transactions succeed, and the monthly decline rate stays between **1.35% and 2.13%** across 34 months.
- **Insufficient balance** is the largest cause, accounting for **64%** of declined transactions and affecting **290 of the 303** active customers.
- The online channel has the highest decline rate (**2.50%**), but most of the gap comes from **incorrect card details**, not from the online channel itself.

### Insights

**1. The decline rate is stable, with no upward trend.**
The yearly decline rate was 1.80% (2022), 1.65% (2023), and 1.78% (2024). The lowest month was September 2022 at 1.35%, and the highest was April 2024 at 2.13%. Successful Amount stayed around $197K per month, ranging from $176K to $214K. The Card Division is dealing with a steady, persistent level of declines rather than a sudden incident.

**2. Three reasons account for 90% of declined transactions.**
Insufficient balance accounts for 1,760 transactions (64.3%), bad PIN for 388 (14.2%), and technical glitches for 324 (11.8%). The remaining six reasons together make up only about 10%.

**3. Insufficient balance affects almost every customer, and usually on larger payments.**
290 of the 303 active customers were declined for insufficient balance at least once. The median amount of these transactions is $71.86, more than twice the $30.65 median of successful transactions. Customers are typically declined when making a payment larger than usual.

**4. Financial Services has the highest decline rate of the 14 category groups.**
This group has a decline rate of 3.00%, compared with 1.74% overall. Money transfers make up 92% of the group's transactions, and 77% of its declines are due to insufficient balance. Customers tend to transfer large amounts without checking their balance first.

**5. Online declines come mainly from entering card details.**
Wrong card number, wrong expiration date, and wrong CVV errors (231 transactions, grouped as Card Details) appear only in the online channel and represent 1.30% of online transactions. In contrast, bad PIN errors appear only in Chip (291) and Swipe (97), since online purchases do not require a PIN. Excluding Card Details, the online decline rate drops to 1.22%, lower than Chip (1.62%).

**6. Technical glitches are small and evenly spread across channels.**
The 324 technical glitch transactions represent only 0.21% of all transactions, or about 10 cases per month (ranging from 4 to 15). The rate by channel ranges from 0.18% to 0.26%, so no single channel has its own system issue.

**7. Of 2,135 flagged transactions, only 39 need to be reviewed first.**
Most flagged transactions trigger only one rule and fall into the Monitor level. Thirty-nine transactions trigger two or more rules: 9 Urgent cases and 30 High cases.

**8. All 9 Urgent cases share the same pattern: an incorrect PIN or CVV during an unusual payment.**
Every one of them is a Peer Outlier + Authentication Error combination. The amounts range from only about $80 to $300, yet they fall in the top 1% of comparable transactions. A $1,000 threshold alone would miss all of them. At the High level, 17 cases are successful transactions with unusually large amounts (High Value + Peer Outlier), and 13 cases are authentication errors combined with multiple errors (Authentication Error + Multiple Errors).

### Recommendations

| Priority | Action | Applies to | Proposed owner | KPI to track |
| --- | --- | --- | --- | --- |
| 1. Act now | Send balance and available-limit alerts before large payments and money transfers | Money transfers and payments larger than usual | Card Product | Balance Decline Rate (currently 1.12%) |
| 2. Plan next | Validate the card number, expiration date, and CVV format on the online checkout page before the transaction is submitted | Online channel | Digital Payments | Online Card Details Rate (currently 1.30%) |
| 2. Plan next | Review the 9 Urgent cases first, then the 30 High cases; keep single-rule transactions on the watch list | 39 transactions in the Review Queue | Risk Operations | Urgent and High cases (currently 39) |
| 3. Monitor | Track technical glitches monthly and investigate when the count moves outside the usual range (4 to 15 cases) | All transactions | IT Operations | Technical Glitch Rate (currently 0.21%) |

Recommendation 1 has the highest priority because insufficient balance accounts for most declined transactions and affects nearly every active card user. The owners listed in the table are proposed assignments, not conclusions drawn from the data.
