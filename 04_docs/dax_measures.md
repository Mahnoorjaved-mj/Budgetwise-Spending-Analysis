# BudgetWise: DAX Measures Specification & Dictionary
**Project:** BudgetWise "The $47,000 Leak"  
**Scope:** Forensic Spending DAX Calculations & Time Series Projections  
**Status:** Approved  

---

## 1. Overview of Forensic DAX Architecture
The forensic DAX architecture translates raw transactional entries into structured behavioral leakage pillars. All leakage measures are built on top of `[Total Spend]`, guaranteeing strict filter context propagation across dates, categories, payment modes, and household identifiers.

---

## 2. Core Requested DAX Measures

### 2.1 `Total Spend`
* **Purpose:** Calculates gross outflow across all expense transactions.
* **DAX Formula:**
  ```dax
  Total Spend = 
  CALCULATE(
      SUM(budgetwise_finance_dataset[amount]),
      budgetwise_finance_dataset[transaction_type] = "expense"
  )
  ```
* **Format:** Currency (`$#,##0`)
* **Behavior:** Automatically slices by user, category, date, and payment mode.

---

### 2.2 `Weekend Tax`
* **Purpose:** Quantifies discretionary leisure and premium expenditure incurred on Saturday and Sunday.
* **DAX Formula:**
  ```dax
  Weekend Tax = 
  CALCULATE(
      [Total Spend],
      WEEKDAY(budgetwise_finance_dataset[date], 2) IN {6, 7},
      budgetwise_finance_dataset[notes] IN {"movie tickets", "shopping"} ||
      (budgetwise_finance_dataset[category] IN {"entertainment", "entertain", "entrtnmnt"} && 
       NOT(budgetwise_finance_dataset[notes] IN {"gym membership", "online course", "internet bill", "coffee", "restaurant dinner", "lunch", "uber ride", "test", "!!!", "xyz123", "asdfgh", "...", "misc"}))
  )
  ```
* **Format:** Currency (`$#,##0`)
* **Business Rationale:** Isolates weekend-specific discretionary indulgence without double-counting delivery or subscriptions.

---

### 2.3 `Ghost Subscriptions`
* **Purpose:** Detects passive, recurring memberships and digital subscriptions that drain accounts automatically.
* **DAX Formula:**
  ```dax
  Ghost Subscriptions = 
  CALCULATE(
      [Total Spend],
      budgetwise_finance_dataset[notes] IN {"gym membership", "online course", "internet bill"}
  )
  ```
* **Format:** Currency (`$#,##0`)
* **Business Rationale:** Targets unused gyms, obsolete educational courses, and premium digital pipelines.

---

### 2.4 `Delivery Drain`
* **Purpose:** Measures the convenience tax of on-demand takeout, quick-service coffees, and ridesharing apps.
* **DAX Formula:**
  ```dax
  Delivery Drain = 
  CALCULATE(
      [Total Spend],
      budgetwise_finance_dataset[notes] IN {"coffee", "restaurant dinner", "lunch", "uber ride"}
  )
  ```
* **Format:** Currency (`$#,##0`)
* **Business Rationale:** Identifies the cumulative financial drag of habitual convenience consumption.

---

### 2.5 `Silent Creep`
* **Purpose:** Identifies untracked, irregular, or suspicious outflows that slip past traditional budgeting.
* **DAX Formula:**
  ```dax
  Silent Creep = 
  CALCULATE(
      [Total Spend],
      budgetwise_finance_dataset[notes] IN {"test", "!!!", "xyz123", "asdfgh", "...", "misc"}
  )
  ```
* **Format:** Currency (`$#,##0`)
* **Business Rationale:** Captures phantom charges, unallocated discretionary spending, and administrative leakage.

---

### 2.6 `Total Leak`
* **Purpose:** The headline forensic aggregate summing the 4 distinct leakage pillars.
* **DAX Formula:**
  ```dax
  Total Leak = 
  [Weekend Tax] + [Ghost Subscriptions] + [Delivery Drain] + [Silent Creep]
  ```
* **Format:** Currency (`$#,##0`)
* **Business Rationale:** Serves as the central KPI for the dashboard ($47,000 headline benchmark per household).

---

### 2.7 `6-month forecast`
* **Purpose:** Projects the forward-looking 6-month financial loss if leaks are not addressed.
* **DAX Formula:**
  ```dax
  6-month forecast = 
  VAR MonthsCount = DISTINCTCOUNT(budgetwise_finance_dataset[Month Year Sort])
  VAR MonthlyAvgLeak = DIVIDE([Total Leak], IF(MonthsCount = 0, 1, MonthsCount), 0)
  RETURN
  MonthlyAvgLeak * 6
  ```
* **Format:** Currency (`$#,##0`)
* **Business Rationale:** Quantifies the immediate cost of inaction over the next 2 quarters.

---

## 3. Supporting & Diagnostic Measures

### 3.1 `6-Month Spend Forecast`
* **DAX Formula:**
  ```dax
  6-Month Spend Forecast = 
  VAR MonthsCount = DISTINCTCOUNT(budgetwise_finance_dataset[Month Year Sort])
  VAR MonthlyAvgSpend = DIVIDE([Total Spend], IF(MonthsCount = 0, 1, MonthsCount), 0)
  RETURN
  MonthlyAvgSpend * 6
  ```
* **Format:** Currency (`$#,##0`)

### 3.2 `Leak Percentage`
* **DAX Formula:**
  ```dax
  Leak Percentage = 
  DIVIDE([Total Leak], [Total Spend], 0)
  ```
* **Format:** Percentage (`0.0%`)

### 3.3 `Annualized Leak`
* **DAX Formula:**
  ```dax
  Annualized Leak = 
  DIVIDE([Total Leak], 4, 0)
  ```
* **Format:** Currency (`$#,##0`)

### 3.4 `Weekend vs Weekday Spend Ratio`
* **DAX Formula:**
  ```dax
  Weekend vs Weekday Ratio = 
  VAR WeekendSpend = CALCULATE([Total Spend], WEEKDAY(budgetwise_finance_dataset[date], 2) IN {6, 7})
  VAR WeekdaySpend = CALCULATE([Total Spend], WEEKDAY(budgetwise_finance_dataset[date], 2) IN {1, 2, 3, 4, 5})
  RETURN
  DIVIDE(WeekendSpend, WeekdaySpend, 0)
  ```
* **Format:** Percentage (`0.0%`)
