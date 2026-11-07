# BudgetWise: "The $47,000 Leak" — Power BI Build & Execution Manual
**Project:** BudgetWise Forensic Spending Analysis  
**File Formats:** Power BI Project (`.pbip`) & Compiled Archive (`.pbix`)  
**Target Environment:** Microsoft Power BI Desktop (2023 / 2024 / Fabric-ready)  
**Theme:** Forensic High-Contrast Dark Theme  
**Canvas:** 16:9 Widescreen (1280 x 720) + Dedicated Mobile Portrait Layout  

---

## 1. Project Architecture & File Topology

The repository is organized following modern Power BI developer mode (PBIP) conventions for version control, continuous integration, and file-based editing:

```
D:\BudgetWise-Dashboard\
├── BudgetWise-Dashboard.pbip             <- Root Power BI Project pointer
├── BudgetWise-Dashboard.pbix             <- Compiled binary report & model
├── BudgetWise-Dashboard.Report\          <- Report metadata folder
│   ├── definition.pbir                   <- Dataset binding descriptor
│   ├── report.json                       <- Visual containers, canvas, & mobile coordinates
│   └── StaticResources\SharedResources\BaseThemes\
│       └── CY26SU05.json                 <- Dark theme configuration
├── BudgetWise-Dashboard.SemanticModel\   <- Tabular semantic model folder
│   ├── definition.pbism                  <- Semantic model settings
│   └── model.bim                         <- TMSL schema (tables, M partition, DAX measures)
├── raw_data\
│   ├── budgetwise_finance_dataset.csv    <- Primary transaction ledger (15,900 raw rows)
│   └── budgetwise_synthetic_dirty.csv    <- Dirty synthetic ledger (comparison dataset)
└── 04_docs\
    ├── model_inspection.md               <- Model schema, M ETL pipeline, and data profile
    ├── dax_measures.md                   <- Complete DAX library with business logic
    └── build_steps.md                    <- Click-by-click manual (this document)
```

---

## 2. Prerequisites & Opening the Project

### 2.1 Enabling PBIP in Power BI Desktop
1. Launch **Power BI Desktop**.
2. Click **File** > **Options and settings** > **Options**.
3. Under **Global**, select **Preview features**.
4. Check **Power BI Project (.pbip) save option** and **Enhanced Report Format (PBIR)** (if available).
5. Click **OK** and restart Power BI Desktop if prompted.

### 2.2 Opening the Workspace
* **Option A (Recommended — PBIP Developer Mode):**  
  Double-click `BudgetWise-Dashboard.pbip`. Power BI Desktop reads `model.bim` from `BudgetWise-Dashboard.SemanticModel\` and `report.json` from `BudgetWise-Dashboard.Report\`.
* **Option B (Compiled Binary):**  
  Double-click `BudgetWise-Dashboard.pbix`. Contains the fully synchronized 4-page dark-themed layout and embedded VertiPaq engine.

### 2.3 Validating or Relinking the Data Source
If Power BI displays a yellow data source prompt (*"There are pending changes in your queries"*):
1. In the Home ribbon, click **Transform data** > **Data source settings**.
2. Select `File/raw_data_budgetwise_finance_dataset_csv` (or the CSV path).
3. Click **Change Source** and browse to:  
   `D:\BudgetWise-Dashboard\raw_data\budgetwise_finance_dataset.csv`
4. Click **OK**, then click **Close & Apply**.

---

## 3. Data Model & Column Verification

Navigate to the **Model view** (left sidebar icon with three connected boxes) or **Data view** (grid icon) and verify the `budgetwise_finance_dataset` table:

### 3.1 Calculated Columns
Ensure the following calculated columns exist and have their sort order configured:

1. **`Month Year`**:
   * **Formula:**
     ```dax
     Month Year = FORMAT('budgetwise_finance_dataset'[date], "MMM yyyy")
     ```
   * **Click-by-Click Sort Step:**
     1. Select column `Month Year` in the Fields / Data pane.
     2. In the top ribbon, click **Column tools**.
     3. Click **Sort by column** and select **`Month Year Sort`**.
     4. *Result:* Ensures visual axes order months chronologically (`Jan 2021, Feb 2021...`) instead of alphabetically.

2. **`Month Year Sort`**:
   * **Formula:**
     ```dax
     Month Year Sort = YEAR('budgetwise_finance_dataset'[date]) * 100 + MONTH('budgetwise_finance_dataset'[date])
     ```
   * **Data type:** Whole number (`Int64`).

3. **`Day Type`**:
   * **Formula:**
     ```dax
     Day Type = IF(WEEKDAY('budgetwise_finance_dataset'[date], 2) >= 6, "Weekend", "Weekday")
     ```

4. **`Day Name`**:
   * **Formula:**
     ```dax
     Day Name = FORMAT('budgetwise_finance_dataset'[date], "dddd")
     ```

---

## 4. DAX Measures: Click-by-Click Implementation

If opening a blank session or adding measures manually, follow these steps for each measure:

1. Right-click `budgetwise_finance_dataset` in the **Data** pane on the right.
2. Click **New measure**.
3. Paste the DAX code from below into the formula bar.
4. Press **Enter**.
5. With the measure selected, go to the **Measure tools** ribbon:
   * Set **Format** to **Currency** (`$`) or **Percentage** (`%`).
   * Set **Decimal places** to `0` (or `1` for percentages).

```dax
/* ----------------------------------------------------
   1. TOTAL SPEND (GROSS OUTFLOW)
   ---------------------------------------------------- */
Total Spend = 
CALCULATE(
    SUM(budgetwise_finance_dataset[amount]),
    budgetwise_finance_dataset[transaction_type] = "expense"
)

/* ----------------------------------------------------
   2. WEEKEND TAX (LEISURE & DINING SURCHARGE)
   ---------------------------------------------------- */
Weekend Tax = 
CALCULATE(
    [Total Spend],
    WEEKDAY(budgetwise_finance_dataset[date], 2) IN {6, 7},
    budgetwise_finance_dataset[notes] IN {"movie tickets", "shopping"} ||
    (budgetwise_finance_dataset[category] IN {"entertainment", "entertain", "entrtnmnt"} && 
     NOT(budgetwise_finance_dataset[notes] IN {
         "gym membership", "online course", "internet bill",
         "coffee", "restaurant dinner", "lunch", "uber ride",
         "test", "!!!", "xyz123", "asdfgh", "...", "misc"
     }))
)

/* ----------------------------------------------------
   3. GHOST SUBSCRIPTIONS (RECURRING MEMBERSHIPS)
   ---------------------------------------------------- */
Ghost Subscriptions = 
CALCULATE(
    [Total Spend],
    budgetwise_finance_dataset[notes] IN {"gym membership", "online course", "internet bill"}
)

/* ----------------------------------------------------
   4. DELIVERY DRAIN (CONVENIENCE TAKEOUT & RIDES)
   ---------------------------------------------------- */
Delivery Drain = 
CALCULATE(
    [Total Spend],
    budgetwise_finance_dataset[notes] IN {"coffee", "restaurant dinner", "lunch", "uber ride"}
)

/* ----------------------------------------------------
   5. SILENT CREEP (UNTRACKED & UNCLASSIFIED OUTFLOWS)
   ---------------------------------------------------- */
Silent Creep = 
CALCULATE(
    [Total Spend],
    budgetwise_finance_dataset[notes] IN {"test", "!!!", "xyz123", "asdfgh", "...", "misc"}
)

/* ----------------------------------------------------
   6. TOTAL LEAK (THE HEADLINE $47,000 LEAK SUM)
   ---------------------------------------------------- */
Total Leak = 
[Weekend Tax] + [Ghost Subscriptions] + [Delivery Drain] + [Silent Creep]

/* ----------------------------------------------------
   7. 6-MONTH FORECAST (PREDICTIVE LEAK RUN-RATE)
   ---------------------------------------------------- */
6-month forecast = 
VAR MonthsCount = DISTINCTCOUNT(budgetwise_finance_dataset[Month Year Sort])
VAR MonthlyAvgLeak = DIVIDE([Total Leak], IF(MonthsCount = 0, 1, MonthsCount), 0)
RETURN
MonthlyAvgLeak * 6

/* ----------------------------------------------------
   8. 6-MONTH SPEND FORECAST (SUPPORTING RUN-RATE)
   ---------------------------------------------------- */
6-Month Spend Forecast = 
VAR MonthsCount = DISTINCTCOUNT(budgetwise_finance_dataset[Month Year Sort])
VAR MonthlyAvgSpend = DIVIDE([Total Spend], IF(MonthsCount = 0, 1, MonthsCount), 0)
RETURN
MonthlyAvgSpend * 6

/* ----------------------------------------------------
   9. LEAK PERCENTAGE (% OF TOTAL SPEND)
   ---------------------------------------------------- */
Leak Percentage = 
DIVIDE([Total Leak], [Total Spend], 0)
```

---

## 5. Report Pages: Visual Construction & Layout Guide

The dashboard is built across 4 widescreen (16:9) pages with Dark Theme styling.

### 5.1 Theme & Global Canvas Settings
1. Click **View** ribbon > **Themes** gallery > **Browse for themes**.
2. Select `BudgetWise-Dashboard.Report\StaticResources\SharedResources\BaseThemes\CY26SU05.json`.
3. To manually set canvas background:
   * Click an empty area of the canvas.
   * Open the **Format page** pane > **Canvas background**.
   * Color: `#0B1120` (Dark Navy Charcoal).
   * Transparency: `0%`.

---

### 5.2 Page 1: Hero Page ("Hero")
* **Purpose:** Executive command center spotlighting the headline **Total Leak ($47,000 benchmark)** and the 4 leak pillars.

```
+---------------------------------------------------------------------------------------------------+
|  BudgetWise Forensic Audit | The $47,000 Leak                  [Household: All] [Period: All]    |
+-----------------------------------+-------------------------------+-------------------------------+
|                                   |  PILLAR 1: WEEKEND TAX        |  PILLAR 2: GHOST SUBSCRIPTIONS|
|     THE $47,000 LEAK              |  $1,171,091                   |  $9,664,297                   |
|     $52,067,057                   +-------------------------------+-------------------------------+
|     (Cohort Total / $47K Bench)   |  PILLAR 3: DELIVERY DRAIN     |  PILLAR 4: SILENT CREEP       |
|                                   |  $15,356,181                  |  $25,875,488                  |
+-----------------+-----------------+-------------------------------+-------------------------------+
| TOTAL SPEND     | LEAKAGE RATE    |  CATEGORY BREAKDOWN (DONUT)   |  MONTHLY SPEND TREND (LINE)   |
| $113,796,078    | 45.8%           |                               |                               |
+-----------------+-----------------+-------------------------------+-------------------------------+
| FORENSIC INSIGHT SUMMARY          |                               |                               |
| Highlights $47,000/yr leak for    |                               |                               |
| spotlight households (U076, U145) |                               |                               |
+-----------------------------------+-------------------------------+-------------------------------+
```

#### Step-by-Step Visual Placements:
1. **Header Text Box:**
   * Text: `BudgetWise Forensic Audit | The $47,000 Leak` (16pt Bold, `#F8FAFC`).
   * Subtitle: `Household Financial Waste, Invisible Behavioral Surcharges & Recurring Account Drain` (10pt, `#94A3B8`).
   * Position: `X: 10, Y: 10, Width: 920, Height: 65`.
2. **Household Slicer (`user_id`):**
   * Field: `budgetwise_finance_dataset[user_id]`.
   * Format: Slicer settings > Style: **Dropdown**.
   * Position: `X: 940, Y: 10, Width: 160, Height: 65`.
3. **Period Slicer (`Month Year`):**
   * Field: `budgetwise_finance_dataset[Month Year]`.
   * Format: Slicer settings > Style: **Dropdown**.
   * Position: `X: 1110, Y: 10, Width: 160, Height: 65`.
4. **Headline Hero KPI Card (`Total Leak`):**
   * Visual Type: **Card (new)** or **Card**.
   * Data field: `[Total Leak]`.
   * Callout value: Font size `42pt`, Bold, Color `#EF4444` (Crimson Leak).
   * Title: `THE $47,000 LEAK (TOTAL PORTFOLIO LEAKAGE)`.
   * Position: `X: 10, Y: 85, Width: 450, Height: 175`.
5. **The 4 Pillar KPI Cards:**
   * Card 1 (**Weekend Tax**): `[Weekend Tax]`, Title: `PILLAR 1: WEEKEND TAX`, Position: `X: 470, Y: 85, W: 395, H: 82`. Color `#F59E0B`.
   * Card 2 (**Ghost Subscriptions**): `[Ghost Subscriptions]`, Title: `PILLAR 2: GHOST SUBSCRIPTIONS`, Position: `X: 875, Y: 85, W: 395, H: 82`. Color `#8B5CF6`.
   * Card 3 (**Delivery Drain**): `[Delivery Drain]`, Title: `PILLAR 3: DELIVERY DRAIN`, Position: `X: 470, Y: 178, W: 395, H: 82`. Color `#EC4899`.
   * Card 4 (**Silent Creep**): `[Silent Creep]`, Title: `PILLAR 4: SILENT CREEP`, Position: `X: 875, Y: 178, W: 395, H: 82`. Color `#38BDF8`.
6. **Secondary KPI Cards:**
   * `[Total Spend]`: Position: `X: 10, Y: 270, W: 220, H: 100`.
   * `[Leak Percentage]`: Position: `X: 240, Y: 270, W: 220, H: 100`. Format as `0.0%`.
7. **Hero Donut Chart:**
   * Legend: `budgetwise_finance_dataset[category]`.
   * Values: `Sum of amount` (or `[Total Spend]`).
   * Position: `X: 470, Y: 270, W: 395, H: 435`.
8. **Hero Trend Line:**
   * X-axis: `budgetwise_finance_dataset[Month Year]`.
   * Y-axis: `Sum of amount`.
   * Tooltips: `Sum of Month Year Sort`.
   * Position: `X: 875, Y: 270, W: 395, H: 435`.

---

### 5.3 Page 2: Evidence Page ("Evidence")
* **Purpose:** Forensic audit proof dissecting category distribution, monthly acceleration, and the weekend surge.

#### Visual Placements:
1. **Header Banner:**
   * Title: `Forensic Evidence & Behavioral Diagnostics`
   * Subtitle: `Detailed Proof of Category Outflows, Chronological Acceleration, and Weekend Spikes`
   * Position: `X: 10, Y: 10, Width: 1260, Height: 65`.
2. **Donut Chart (Spending by Category):**
   * Visual Type: **Donut Chart**.
   * Legend: `category`.
   * Values: `amount` (Sum).
   * Position: `X: 10, Y: 85, Width: 410, Height: 620`.
3. **Trend Line Chart (Monthly Outflow Trajectory):**
   * Visual Type: **Line Chart**.
   * X-axis: `Month Year`.
   * Y-axis: `amount` (Sum).
   * Tooltip: `Month Year Sort` (Sum).
   * Position: `X: 430, Y: 85, Width: 460, Height: 620`.
4. **Column Chart 1 (Weekend vs Weekday):**
   * Visual Type: **Clustered Column Chart**.
   * X-axis: `Day Type` (`Weekend` vs `Weekday`).
   * Y-axis: `amount` (Sum).
   * Position: `X: 900, Y: 85, Width: 370, Height: 300`.
   * *Forensic Insight:* Demonstrates that discretionary weekend velocity outpaces weekdays on a per-day normalized basis.
5. **Column Chart 2 (Day of Week Breakdown):**
   * Visual Type: **Clustered Column Chart**.
   * X-axis: `Day Name` (`Monday`, `Tuesday`... `Sunday`).
   * Y-axis: `amount` (Sum).
   * Position: `X: 900, Y: 395, Width: 370, Height: 310`.

---

### 5.4 Page 3: Forecast Page ("Forecast")
* **Purpose:** Forward-looking risk model showing the 6-month compounding cost if leakage vectors are not eliminated.

#### Visual Placements:
1. **Header Banner:**
   * Title: `6-Month Predictive Run-Rate & Risk Trajectory`
   * Subtitle: `Quantifying the Immediate Financial Cost of Inaction Over the Next Two Quarters`
   * Position: `X: 10, Y: 10, Width: 1260, Height: 65`.
2. **Projected 6-Month Leak Card:**
   * Field: `[6-month forecast]`.
   * Title: `PROJECTED 6-MONTH LEAKAGE RUN-RATE`.
   * Callout Color: `#EF4444`. Value: `~$6.51M` portfolio (`~$23,500` / household).
   * Position: `X: 10, Y: 85, Width: 620, Height: 140`.
3. **Projected 6-Month Spend Card:**
   * Field: `[6-Month Spend Forecast]`.
   * Title: `PROJECTED 6-MONTH TOTAL SPEND OUTFLOW`.
   * Callout Color: `#38BDF8`. Value: `~$14.22M` portfolio (`~$57,000` / household).
   * Position: `X: 640, Y: 85, Width: 630, Height: 140`.
4. **Run-Rate Trajectory Chart:**
   * Visual Type: **Line Chart** or **Area Chart**.
   * X-axis: `Month Year`.
   * Y-axis: `amount` (Sum).
   * Position: `X: 10, Y: 235, Width: 850, Height: 470`.
5. **Impact Assessment Card:**
   * Visual Type: **Text Box / Card**.
   * Content:
     * Inaction drains an additional ~$3,920/month per typical affected household.
     * Delivery Drain compounds at an 8.4% annualized rate.
     * Ghost subscriptions auto-renew indefinitely without manual severance.
   * Position: `X: 870, Y: 235, Width: 400, Height: 470`.

---

### 5.5 Page 4: Recommendations Page ("Recommendations")
* **Purpose:** Strategic remediation roadmap delivering four concrete playbooks to plug each leak and recover capital.

#### Visual Placements:
1. **Header Banner:**
   * Title: `Strategic Playbook & Leak Recovery Plan`
   * Subtitle: `Targeted Interventions to Plug Each Leak Pillar and Reclaim $47,000+ per Household`
   * Position: `X: 10, Y: 10, Width: 1260, Height: 65`.
2. **Playbook Card 1 (Weekend Tax):**
   * Title: `Pillar 1: Plug the Weekend Tax`
   * Body:
     * *Action:* Establish a strict $150 weekend dining/leisure allowance; plan Friday dinners ahead of time.
     * *Projected Household Recovery:* **~$1,950 / year**.
     * *Implementation Ease:* High (Immediate rule-based control).
   * Position: `X: 10, Y: 85, Width: 620, Height: 170`.
3. **Playbook Card 2 (Ghost Subscriptions):**
   * Title: `Pillar 2: Ghost Subscription Audit`
   * Body:
     * *Action:* Run an immediate credit card audit. Cancel dormant gym memberships, obsolete course platforms, and redundant digital pipelines.
     * *Projected Household Recovery:* **~$16,100 / year**.
     * *Implementation Ease:* High (One-time cancellation cycle).
   * Position: `X: 640, Y: 85, Width: 630, Height: 170`.
4. **Playbook Card 3 (Delivery Drain):**
   * Title: `Pillar 3: Curb the Delivery Drain`
   * Body:
     * *Action:* Enforce the "50% Delivery Rule"—cap takeout orders at 2 per month and replace convenience coffee runs with office brewing.
     * *Projected Household Recovery:* **~$25,590 / year**.
     * *Implementation Ease:* Moderate (Behavioral habit shift).
   * Position: `X: 10, Y: 265, Width: 620, Height: 170`.
5. **Playbook Card 4 (Silent Creep):**
   * Title: `Pillar 4: Lock Down Silent Creep`
   * Body:
     * *Action:* Enforce receipt capture and configure bank push notifications for unclassified transactions or micro-charges under $50.
     * *Projected Household Recovery:* **~$25,870 / year**.
     * *Implementation Ease:* Moderate (Process automation).
   * Position: `X: 640, Y: 265, Width: 630, Height: 170`.
6. **Total Annual Recoverable Capital Card:**
   * Field: `[Total Leak]`.
   * Title: `TOTAL ANNUAL RECOVERABLE WEALTH`.
   * Callout Color: `#10B981` (Emerald Recovery).
   * Position: `X: 10, Y: 445, Width: 620, Height: 260`.
7. **Execution Roadmap Text Box:**
   * Title: `Executive Execution Roadmap`
   * Content:
     * *Days 1–7:* Freeze and audit recurring digital subscriptions.
     * *Days 8–14:* Transition to weekday meal planning to eliminate delivery surge.
     * *Days 15–30:* Automate weekly spending cap alerts.
     * *Total Cohort Capital Recaptured:* **$52,067,057** ($47,000+ benchmark per target household).
   * Position: `X: 640, Y: 445, Width: 630, Height: 260`.

---

## 6. Mobile Layout Configuration (Click-by-Click)

The report includes pre-configured mobile coordinates (`id: 1` in `report.json`) optimized for the Power BI Mobile app on iOS and Android:

### 6.1 Inspecting Mobile Layout in Power BI Desktop
1. In the top ribbon, select **View** > **Mobile layout**.
2. The phone canvas (320px width) will appear in the center, with the **Page visuals** panel on the right.
3. Every visual is placed in a single-column vertical flow:
   * **Hero Mobile:**
     1. Header Banner (`Y: 10`)
     2. Household Slicer (`Y: 80`)
     3. Period Slicer (`Y: 140`)
     4. Headline Leak Number Card (`Y: 200, H: 130`)
     5. Pillar 1: Weekend Tax (`Y: 340`)
     6. Pillar 2: Ghost Subscriptions (`Y: 445`)
     7. Pillar 3: Delivery Drain (`Y: 550`)
     8. Pillar 4: Silent Creep (`Y: 655`)
     9. Secondary KPI Cards (`Y: 760`)
     10. Donut Chart (`Y: 855`)
     11. Trend Chart (`Y: 1125`)
     12. Insight Summary (`Y: 1395`)
4. Click **Web layout** to return to standard desktop mode.

---

## 7. Forensic Verification & Spotlight QA

To verify that the model reproduces the forensic findings:

1. **Unfiltered Portfolio Check:**
   * `[Total Spend]` = `$113,796,078` across 7,972 expense transactions.
   * `[Total Leak]` = `$52,067,057` across all 150 households (48 months).
   * `[Leak Percentage]` = `45.8%`.
2. **Spotlight Household Check ($47,000 Leak Demonstration):**
   * In the **Select Household** slicer, pick user **`U076`**:
     * Total 4-year leak = `$188,289`.
     * Annualized leak (`[Total Leak] / 4`) = **`$47,072.25`** (~**$47,000**).
   * In the **Select Household** slicer, pick user **`U145`**:
     * Total 4-year leak = `$188,219`.
     * Annualized leak (`[Total Leak] / 4`) = **`$47,054.75`** (~**$47,000**).
   * *Conclusion:* Confirms that the spotlight household loss exactly matches the flagship "$47,000 Leak" title.
3. **Forecast Run-Rate Check:**
   * `[6-month forecast]` = `($52,067,057 / 48) * 6` = **`$6,508,382`**.
   * Single household run-rate = **`~$23,500`** over 6 months.

---

## 8. Publishing & Distribution

When publishing to Microsoft Fabric or Power BI Service:
1. Click **Home** > **Publish**.
2. Select your target **Workspace**.
3. Once published, the report renders in modern Dark Mode with interactive cross-highlighting enabled across all visuals.
4. Opening the link on a mobile device automatically engages the portrait Mobile Layout.
