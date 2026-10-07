# JCars Logistics Power BI Business Intelligence Assessment

![Dashboard](/images/dashboard.png)
  
**JCars Logistics** is a commercial automotive dealership and logistics company operating across **8 branches** (*Thika, Kakamega, Nakuru, Nairobi, Kisumu, Mombasa, Eldoret, Athi River*) in **7 regions of Kenya**.

This repository contains the complete enterprise Business Intelligence solution developed in Power BI. The project transforms a raw, uncleaned, flat operational dataset into a fully interactive, audit-ready decision support system.

### 📊 Key Enterprise Metrics
* **Gross Revenue:** `Ksh 1.85 Bn`
* **Net Revenue:** `Ksh 1.77 Bn` (accounting for `Ksh 15.07M` in refunds)
* **Net Profit:** `Ksh 447.52M`
* **Gross Profit Margin:** `25.34%`
* **Sales Volume:** `408 Units Sold` across `245 Orders`
* **Avg. Order-to-Delivery (OTD):** `28 Days`
* **Historical Cancellation Rate:** `7.35%`

---

## 🏗️ Repository Structure

```text
| jcars_analysis
|
├── /images     # Image files for the project
|
├── jcars_analysis.pbix    # Master Power BI Report File
├── jcars_analysis.pdf
|
├── Jcars_data.csv  # Flat table
|
├── readme.md

```
---

## 🧹 Data Quality Audit & Power Query ETL
A comprehensive data quality audit uncovered **10 major data quality issues** in the raw dataset:

1. **Multi-Currency Inconsistencies:** Mixed monetary entries standardized to **Kenya Shillings (KES)** using Central Bank of Kenya(CBK) average exchange rates for 2026.

| Currency | Rate |
|---|---|
| EUR | 150.7 |
| $ | 129.3 |
| R | 7.9 |
| KES| 1 |

*Table: ExchangeRate table - CBK average exchange rates for 2026*

Also, formatting of some unit values were indicated i.e. as 1.4M implying they were in millions, we used a helper column to get a multiplier i.e for any Value ending in M the Multiplier would be 1000000, K = 1000, else 1.

2. **Invalid Timestamps (Year 1899):** Mixed date strings (DD/MM/YYYY vs MM/DD/YYYY). Parsed explicitly in Power Query using Locale setting (English - Kenya). Other errors were replaced with null after setting the correct type for both the Order Date and Delivery Date. The following custom M code handled the date issue i.e for the Order Date by creating a new column. Follow the comments in the code for the steps:

```
// 1. Clean the text, but keep slashes/hyphens for now

rawTxt = Text.Clean(Text.Trim(Text.From([Order Date]))),

// 2. Try standard parsing first (handles Excel serial numbers and system-matching dates)

parseStandard = try Date.From(Number.FromText(rawTxt)) otherwise try Date.FromText(rawTxt) otherwise null,

// 3. If standard parsing fails, strip out hyphens, slashes, and dots to inspect the raw digits

cleanDigits = Text.Select(rawTxt, {"0".."9"}),
len = Text.Length(cleanDigits),

// 4. Conditional logic to extract components from the raw digits

parseCustom = if parseStandard <> null then parseStandard
else if len = 8 and (Text.StartsWith(cleanDigits, "20") or
Text.StartsWith(cleanDigits, "19")) then

// Handles YYYYMMDD (Assumes Year first)

try #date(Int16.From(Text.Start(cleanDigits, 4)),
Int16.From(Text.Middle(cleanDigits, 4, 2)),
Int16.From(Text.End(cleanDigits, 2)))

// Alternate backup: If Month > 12, handles YYYYDDMM

otherwise try #date(Int16.From(Text.Start(cleanDigits,
4)), Int16.From(Text.End(cleanDigits, 2)),
Int16.From(Text.Middle(cleanDigits, 4, 2)))

otherwise null
else if len = 8 then

// Handles MMDDYYYY (Assumes Year is at the end)

try #date(Int16.From(Text.End(cleanDigits, 4)),
Int16.From(Text.Start(cleanDigits, 2)),
Int16.From(Text.Middle(cleanDigits, 2, 2)))

// Alternate backup: If Month > 12, handles DDMMYYYY

otherwise try #date(Int16.From(Text.End(cleanDigits,
4)), Int16.From(Text.Middle(cleanDigits, 2, 2)),
Int16.From(Text.Start(cleanDigits, 2)))
otherwise null
else null
in
parseCustom

```
3. **Branch & Geographic Variations:** Standardized text string casing and trimmed whitespace across branches and counties and corrected the spelling mistakes using the correct spellings.
4. **Missing Payment & Channel Attributes:** Replaced null values with `'Unspecified / Direct'` to preserve aggregate volume.
5. **Zero / Negative Selling Prices:** Flagged and filtered test transactions.
6. *Mixed and Incorrectly Formatted Order ID:* The Order ID was incorrectly formatted as a text field where non digits and digit values were combined for some IDs while some had either the digit part or the non digit part only and some blank. After observations, the field had a sequential digit part starting at 1000. To solve this, we decided to use only the digit part to fill down the Order ID column by creating a new Custom Index Column.
7. **Delivery Status Anomalies:** Validated zero-fee delivered orders.
8. **Discount Exceeds Margin:** Created exception flags for transactions where discounts eroded gross margin and set them to `null`.
9. **Missing Ratings:** Assigned neutral default weights for sentiment analysis.

![Power Query](/images/power_query.png) 

---

## 📐 Star Schema Data Model
The raw flat dataset was restructured into an analytical **Star Schema** to optimize Power BI VertiPaq engine compression and DAX performance:

![Star Schema Modelling](/images/star_schema.png)

### Model Relationships:
* **`FactOrders`**: Contains foreign keys (`Order ID`, `Branch ID`, `Region ID`, `Sales Rep ID`, `Customer ID`, `Order Date`, `City ID`, `County ID`) and numerical measures(`Unit Selling Price(KES)`, `Units Sold`, `Unit Cost(KES)`, `Logistic Cost(KES)`, `Delivery Fees(KES)`), and Categorical measures like (`Vehicle Type`, `Color`, `Payment Method`, `Payment Status`, `Delivery Status`, `Car Make`, e.t.c.)
* **Dimensions**: Filter direction is strictly **Single-Direction** from Dimension tables to Fact table exhibing a 1-to-many(1:*) cardinality.

![Relationships](/images/rlts.png)

---

## 🧮 Key DAX Measures

To enforce consistent business rules across all visual elements, DAX measures were constructed. Core DAX formulas used include:

1. Gross Revenue(Create a new custom column)

`Gross Revenue = FactOrders[Units Sold] * FactOrders[Unit Selling Price(KES)]`

2. Recorded Revenue(Create a new custom column)

`Revenue Recorded = (FactOrders[Unit Selling Price(KES)]*FactOrders[Units Sold]*(1-FactOrders[Discount])) + FactOrders[Delivery Fees(KES)]`

3. Discount Amount(Create a new custom column)

`Discount Amount = (FactOrders[Unit Selling Price(KES)]/(1 - FactOrders[Discount])) - FactOrders[Unit Selling Price(KES)]`

4. Net Profit Margin(New custom column)

`Net Profit Margin = FactOrders[Revenue Recorded] - ((FactOrders[Units Sold]*FactOrders[Unit Cost(KES)])+FactOrders[Logistic Cost(KES)]+FactOrders[Delivery Fees(KES)])`

5. Volume Quarter 1 2025

`VolumeQtr125 = CALCULATE(SUM(FactOrders[Units Sold]), AND(QUARTER(FactOrders[Order Date]) = 1, YEAR(FactOrders[Order Date]) = 2025), NOT(ISBLANK(FactOrders[Order Date])))`

6. Percentage Volume Quarter 1(Compare between Q1 2025 and Q1 2026 Performance. Create a new measure)

`VolumePercentageQ1 = (([VolumeQtr126]-[VolumeQtr125])/[VolumeQtr125])`

7. Total Revenue Recorded

`TotalRevenueRecorded = SUM(FactOrders[Revenue Recorded])`

## 💡 Top Strategic Insights & Findings

1. **High-Margin Premium Trims:** **Volkswagen** yields an astounding **77.19% profit margin** (`Ksh 126.50M` net profit on 17 units), while **Toyota** serves as the volume driver (**116 units**, `29.21%` margin, `Ksh 201.98M` profit).
2. **Branch Profit Margin Disparity:** **Nairobi branch** achieves a **70.46% profit margin** (`Ksh 151.37M` profit), whereas **Kisumu branch** operates at a low **10.64% margin** (`Ksh 21.34M` profit) due to heavy customer refunds (`Ksh 11.94M`).
3. **Messaging Channel Bottlenecks:** **WhatsApp** orders exhibit severe fulfillment delays with an average **Order-to-Delivery (OTD) time of 68 days**, compared to the enterprise baseline of 28 days.
4. **Tender Contract Cancellations:** Tender sales record an abnormally high **18.75% cancellation rate**, driven by 42-day transit friction and strict contract specifications.
5. **Sales Rep Concentration Risk:** **Faith Achieng** (`Ksh 131.61M`) and **Grace Njeri** (`Ksh 141.26M`) generate **60.98% of total enterprise net profit**.

---

## 🎯 Executive Recommendations

1. **Reallocate Capital to High-Margin Trims:** Expand inventory of high-margin Crossover/SUV trims (Volkswagen Tiguan/Golf, Toyota Prado, Mercedes C200/E250) to boost profit by `~Ksh 65M` annually.
2. **Optimize Digital Channel Fulfillment:** Streamline WhatsApp order workflows to compress OTD days from 68 down to 28 days, reducing refund exposure.
3. **Enforce Branch Discount Guardrails:** Limit branch manager discount authority to a maximum of 5% on low-margin inventory, recovering `~Ksh 18M–25M` in margin leakage.

