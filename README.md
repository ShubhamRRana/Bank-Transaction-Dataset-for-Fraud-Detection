# Bank Transaction Analysis: Power BI Dashboard

An interactive Power BI report that analyzes bank transactions to understand spending behavior, channel usage, customer segments and suspicious activity patterns.

Built as a hands-on portfolio project to practice DAX, data modeling (star schema) and dashboard design for fintech analytics roles.

## Business problem

A retail bank wants to answer three questions:

1. How are customers transacting (volume, value, channel, location)?
2. Which customer segments drive the most spend?
3. Where does risky or unusual activity show up, and how is it trending over time?

## Dataset

- **Source:** Kaggle, [Bank Transaction Dataset for Fraud Detection](https://www.kaggle.com/datasets/valakhorasani/bank-transaction-dataset-for-fraud-detection)
- **Grain:** one row per transaction
- **Key fields:** TransactionID, AccountID, TransactionAmount, TransactionDate, TransactionType, Channel, Location, CustomerAge, CustomerOccupation, LoginAttempts, AccountBalance
- **Note:** the dataset has no fraud label. "Suspicious" activity in this project is a rule-based proxy (for example, 3 or more login attempts), not a confirmed fraud flag.

## Data model

Star schema built in Power Query and the Power BI model view.

```
Calendar ──┐
Dim_Channel ──┼──> Transactions (fact)
Dim_Location ─┘
```

| Table | Type | Key |
|---|---|---|
| Transactions | Fact | ChannelID, LocationID, TransactionDate |
| Dim_Channel | Dimension | ChannelID |
| Dim_Location | Dimension | LocationID |
| Calendar | Dimension (date table) | Date |

All relationships are one-to-many with single-direction filtering from the dimension to the fact table.

## Questions answered

### Overview
| Question | Visual |
|---|---|
| How many transactions, how much value, how many unique accounts, what is the average balance? | KPI cards |
| Which channel carries the most amount? | Bar chart |
| Which 5 locations have the highest amount? | Top 5 bar chart |
| Which occupations spend the most? | Bar chart |
| What is the debit vs credit split? | Donut / bar |

### Risk (in progress)
| Question | Visual |
|---|---|
| How many transactions had 3 or more login attempts? | Card |
| Which channel has the highest share of high-login-attempt transactions? | Bar chart |
| How many large transactions are there? | Card |

### Trends (in progress)
| Question | Visual |
|---|---|
| What is the monthly amount trend? | Line chart |
| What is the month-over-month change? | Line / table |
| What is the year-to-date amount? | Card / line |

## Key findings (Overview page)

- Total transaction value is about **$747.6K** across **495 unique accounts**, with an average transaction of about **$297.6** and an average account balance of about **$5.1K**.
- Spend is spread almost evenly across channels: ATM about $0.26M, Branch about $0.25M, Online about $0.24M.
- Students contribute the most by occupation (about $0.21M), with Doctor, Engineer and Retired each at about $0.18M.
- The top 5 locations by amount are Austin, Oklahoma City, Memphis, Fort Worth and Detroit, each between about $21K and $23K.

## DAX highlights

Full list with explanations in [`dax/measures.md`](dax/measures.md). Concepts used:

- Aggregations: `SUM`, `AVERAGE`, `COUNTROWS`, `DISTINCTCOUNT`, `DIVIDE`
- Filter context: `CALCULATE`, `ALL`, `REMOVEFILTERS`, `FILTER`, `VALUES`
- Logic: `VAR / RETURN`, `SWITCH(TRUE())`, `IF`
- Time intelligence: `TOTALYTD`, `PREVIOUSMONTH`, `DATEADD`
- Ranking: `RANKX`, `TOPN`

## Repository structure

```
bank-transaction-analysis/
├── README.md
├── data/                  raw CSV (or a link if the file is too large)
├── powerbi/               .pbix report
├── dax/measures.md        all measures with explanations
└── screenshots/           one image per dashboard page
```

## Screenshots

| Overview | Risk | Trends |
|---|---|---|
| ![Overview](images/overview.png) | ![Risk](images/risk.png) | ![Trends](images/trends.png) |

## How to open

1. Download the dataset from the Kaggle link above and place the CSV in `data/`.
2. Open `powerbi/bank-transaction-analysis.pbix` in Power BI Desktop.
3. If prompted, update the data source path: Home > Transform data > Data source settings.
4. Refresh.

## Tools

Power BI Desktop, Power Query (M), DAX

## Limitations and next steps

- The data is a single snapshot with no fraud label, so risk indicators are rule-based and should be validated against confirmed fraud cases.
- Next steps: add a customer segmentation page, add a Region field to `Dim_Location` for a map view, and publish the report to Power BI Service.

## Author

Shubham Rana
