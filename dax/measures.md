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

Calculated column on `Transactions`:

```dax
Age Group =
SWITCH(
    TRUE(),
    Transactions[CustomerAge] < 25, "18-24",
    Transactions[CustomerAge] < 40, "25-39",
    Transactions[CustomerAge] < 60, "40-59",
    "60+"
)
```
Buckets each transaction into an age band. Conditions are checked top to bottom, so the first match wins.

---

## 5. Risk indicators (rule-based)

The dataset has no fraud label. These measures flag unusual activity using simple rules.

```dax
High Login Txns =
CALCULATE([Total Transactions], Transactions[LoginAttempts] >= 3)
```
Transactions with 3 or more login attempts.

```dax
High Login % = DIVIDE([High Login Txns], [Total Transactions])
```
Share of transactions with 3 or more login attempts. Put Channel or Location on the axis to compare segments.

```dax
Large Txns =
VAR Threshold = 1000
RETURN
    CALCULATE([Total Transactions], Transactions[TransactionAmount] > Threshold)
```
Count of transactions above a fixed amount. Choose the threshold after checking the amount distribution.

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
Combined amount of the five highest-value locations. Returns a single number, so use it on a card. For a chart of the five locations, use a Top N visual filter.

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
| `VAR / RETURN` for readability and a single evaluation | Sections 5 and 6 |
| Table functions (`TOPN`, `PREVIOUSMONTH`) sit inside `CALCULATE` | Sections 6 and 7 |
| `RANKX` over `ALL`, never `VALUES` | Section 7 |
