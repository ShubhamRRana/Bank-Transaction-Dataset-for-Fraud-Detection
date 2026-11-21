# DAX Measures

All measures used in the Bank Transaction Analysis report, with a plain-language explanation of each.

## Model reference

| Table | Role | Key columns |
|---|---|---|
| `Transactions` | Fact | TransactionID, AccountID, TransactionAmount, TransactionDate, TransactionType, CustomerAge, LoginAttempts, AccountBalance, ChannelID, LocationID |
| `Dim_Channel` | Dimension | ChannelID, Channel |
| `Dim_Location` | Dimension | LocationID, Location |
| `Calendar` | Date table | Date, Month, Year |

Relationships: each dimension is one-to-many to `Transactions`, filtering one direction. `Calendar[Date]` is related to `Transactions[TransactionDate]` and marked as the date table.

Adjust table and column names if they differ in your model.

---

## 1. Calendar table

```dax
Calendar =
CALENDAR(
    MIN(Transactions[TransactionDate]),
    MAX(Transactions[TransactionDate])
)
```

Creates one row per date between the first and last transaction. Mark it as a date table so time intelligence works.

Helper columns on `Calendar` (Table tools > New column):

```dax
Year = YEAR(Calendar[Date])
Month Label = FORMAT(Calendar[Date], "MMM yyyy")
Month Sort = YEAR(Calendar[Date]) * 100 + MONTH(Calendar[Date])
```

Set `Month Label` > Column tools > Sort by column > `Month Sort`, otherwise months sort alphabetically on chart axes.

The data runs from 2 Jan 2023 to 1 Jan 2024. The Trends page uses a page-level filter `Calendar[Year] = 2023` to exclude the one-day 2024 stub, which would otherwise create a near-empty 13th month and distort MoM % and YTD.

---

## 2. Core KPIs

```dax
Total Transactions = COUNTROWS(Transactions)
```
Number of transactions in the current filter context.

```dax
Total Amount = SUM(Transactions[TransactionAmount])
```
Total value of transactions.

```dax
Avg Amount = AVERAGE(Transactions[TransactionAmount])
```
Average value per transaction.

```dax
Unique Accounts = DISTINCTCOUNT(Transactions[AccountID])
```
Number of distinct accounts that transacted.

```dax
Avg Balance = AVERAGE(Transactions[AccountBalance])
```
Average account balance across transactions.

---

## 3. Share and percent of total

```dax
Debit Amount =
CALCULATE([Total Amount], Transactions[TransactionType] = "Debit")
```
Total amount of debit transactions only. The filter replaces any existing TransactionType filter.

```dax
Credit Amount =
CALCULATE([Total Amount], Transactions[TransactionType] = "Credit")
```
Total amount of credit transactions only.

```dax
Debit % = DIVIDE([Debit Amount], [Total Amount])
```
Share of the total amount that is debit. `DIVIDE` returns blank instead of an error when the denominator is zero.

```dax
Channel % =
DIVIDE(
    [Total Transactions],
    CALCULATE([Total Transactions], ALL(Dim_Channel[Channel]))
)
```
Each channel's share of all transactions. `ALL` removes the channel filter in the denominator so every row is divided by the grand total.

---

## 4. Customer segments

Both are **calculated columns** on `Transactions` (Table tools > New column), not measures. They read one row's age at a time, which a measure cannot do.

```dax
Age Group =
SWITCH(
    TRUE(),
    ISBLANK(Transactions[CustomerAge]), "Unknown",
    Transactions[CustomerAge] < 25, "18-24",
    Transactions[CustomerAge] < 40, "25-39",
    Transactions[CustomerAge] < 60, "40-59",
    "60+"
)
```
Buckets each transaction into an age band. Conditions are checked top to bottom, so the first match wins.

```dax
Age Group Sort =
SWITCH(
    TRUE(),
    ISBLANK(Transactions[CustomerAge]), 5,
    Transactions[CustomerAge] < 25, 1,
    Transactions[CustomerAge] < 40, 2,
    Transactions[CustomerAge] < 60, 3,
    4
)
```
Sort helper. Set `Age Group` > Column tools > Sort by column > `Age Group Sort` so slicers and charts show age bands in order instead of alphabetically.

---

## 5. Risk indicators (rule-based)

The dataset has no fraud label. These measures flag unusual activity using simple rules: 3 or more login attempts, and the top 5% of transaction amounts.

```dax
High Login Txns =
CALCULATE([Total Transactions], Transactions[LoginAttempts] >= 3)
```
Transactions with 3 or more login attempts.

```dax
High Login % = DIVIDE([High Login Txns], [Total Transactions])
```
Share of transactions with 3 or more login attempts. Put Channel or Age Group on the axis to compare segments.

```dax
Large Txns =
VAR Threshold =
    CALCULATE(PERCENTILE.INC(Transactions[TransactionAmount], 0.95), ALL(Transactions))
RETURN
    CALCULATE([Total Transactions], Transactions[TransactionAmount] > Threshold)
```
Count of transactions above the 95th percentile amount. `ALL(Transactions)` fixes the threshold across the whole table, so slicers do not move it. By construction this flags about 5% of transactions, so it is a rule, not a finding.

```dax
Large Txn % = DIVIDE([Large Txns], [Total Transactions])
```
Share of transactions flagged as large.

```dax
High Risk Txns =
VAR Threshold =
    CALCULATE(PERCENTILE.INC(Transactions[TransactionAmount], 0.95), ALL(Transactions))
RETURN
    CALCULATE(
        [Total Transactions],
        Transactions[LoginAttempts] >= 3,
        Transactions[TransactionAmount] > Threshold
    )
```
Transactions that meet both rules at once (3+ login attempts and top 5% amount). The strictest flag, and the one to review first.

```dax
Max Login Attempts = MAX(Transactions[LoginAttempts])
```
Highest login attempt count in context. Used in the flagged accounts table.

---

## 6. Time intelligence

Requires the `Calendar` table to be marked as a date table.

```dax
YTD Amount = TOTALYTD([Total Amount], Calendar[Date])
```
Running total from 1 January to the latest date in the current context.

```dax
Prev Month Amount =
CALCULATE([Total Amount], PREVIOUSMONTH(Calendar[Date]))
```
Total amount for the month before the current one.

```dax
MoM % =
VAR Cur  = [Total Amount]
VAR Prev = [Prev Month Amount]
RETURN
    DIVIDE(Cur - Prev, Prev)
```
Month-over-month growth. The first month is blank because it has no previous month.

```dax
Avg Monthly Amount =
AVERAGEX(VALUES(Calendar[Month Label]), [Total Amount])
```
Average of the monthly totals. `VALUES` makes the iterator visit each month in context, and `[Total Amount]` is evaluated per month through context transition.

```dax
Rolling 3M Amount =
CALCULATE(
    [Total Amount],
    DATESINPERIOD(Calendar[Date], MAX(Calendar[Date]), -3, MONTH)
)
```
Total amount over the 3 months ending at the last date in each cell. It is a 3-month total, not an average. The first two months use shorter windows because earlier data does not exist.

```dax
Peak Month =
MAXX(
    TOPN(1, VALUES(Calendar[Month Label]), [Total Amount]),
    Calendar[Month Label]
)
```
Name of the month with the highest total amount. `TOPN` returns a one-row table and `MAXX` turns it into a single text value.

---

## 7. Ranking

```dax
Location Rank =
IF(
    HASONEVALUE(Dim_Location[Location]),
    RANKX(ALL(Dim_Location[Location]), [Total Amount])
)
```
Rank of each location by total amount (1 = highest). `ALL` ranks against every location, and `HASONEVALUE` blanks the total row.

```dax
Top 5 Location Amount =
CALCULATE(
    [Total Amount],
    TOPN(5, VALUES(Dim_Location[Location]), [Total Amount])
)
```
Combined amount of the five highest-value locations. Returns a single number, so use it on a card. The Top 5 bar chart on the Overview page uses a Top N visual filter instead.

```dax
Top 5 Share % =
DIVIDE(
    [Top 5 Location Amount],
    CALCULATE([Total Amount], ALL(Dim_Location))
)
```
Share of total amount contributed by the top 5 locations.

---

## DAX patterns used

| Pattern | Where |
|---|---|
| Aggregate in a measure, row-level logic in a calculated column | Sections 2 and 4 |
| `CALCULATE` replaces filters only on the columns it names | Sections 3 and 5 |
| `ALL` in the denominator for percent of total | Sections 3 and 7 |
| `ALL` inside a variable to fix a threshold across slicers | Section 5 |
| Iterate over `VALUES` of a grouping column, then aggregate | Section 6 (`Avg Monthly Amount`) |
| `DATESINPERIOD` anchored on `MAX(date)` for rolling windows | Section 6 |
| `VAR / RETURN` for readability and a single evaluation | Sections 5 and 6 |
| Table functions (`TOPN`, `PREVIOUSMONTH`) sit inside `CALCULATE` | Sections 6 and 7 |
| `RANKX` over `ALL`, never `VALUES` | Section 7 |
