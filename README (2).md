# Superstore Sales Dashboard (Power BI)

An interactive sales and profit dashboard built on the Superstore dataset (US retail, orders from 4 Jan 2011 to 31 Dec 2014).

## Deliverables

| File | Purpose |
|---|---|
| `Superstore.csv` / `Superstore.xlsx` | Source data (9,994 order lines, 21 columns) |
| `superstore_dashboard.html` | Interactive preview of the dashboard (open in any browser) |
| `README.md` | This guide: data notes, model, DAX measures, page layout |
| Superstore report | Written findings and recommendations |

## Dataset summary

- 9,994 rows, 5,009 orders, 793 customers, 49 states, one country (United States)
- No missing values and no duplicate rows
- Dimensions: Region (4), Segment (3), Category (3), Sub-Category (17), Ship Mode (4)
- Measures: Sales, Quantity, Discount, Profit

## Data quality notes

1. **Encoding:** `Superstore.csv` is Latin-1 (not UTF-8). In Power BI choose **File Origin: 1252 Western European (Windows)** when importing, or use `Superstore.xlsx` to avoid the issue.
2. **Date format:** `Order Date` and `Ship Date` are text in `dd-mm-yyyy`. Convert them with the locale **English (United Kingdom)** (Transform > Data Type > Using Locale), otherwise days and months swap.
3. **Postal Code** is numeric and drops leading zeros for New England states. Set it to Text and pad to 5 digits if you map by ZIP.

## Step-by-step build

### 1. Load and clean (Power Query)

1. Home > Get Data > Text/CSV > `Superstore.csv` (File Origin 1252) > Transform Data.
2. Right-click `Order Date` and `Ship Date` > Change Type > Using Locale > Date, English (United Kingdom).
3. Change `Postal Code` to Text. Change `Row ID` to Text (it is an identifier, not a number).
4. Add custom column **Ship Days**: `Duration.Days([Ship Date] - [Order Date])`.
5. Add custom column **Discount Band**:

```
if [Discount] = 0 then "No discount"
else if [Discount] <= 0.2 then "Up to 20%"
else if [Discount] <= 0.4 then "20-40%"
else "Over 40%"
```

6. Close & Apply.

### 2. Data model

Create a calendar table and mark it as the date table:

```DAX
Calendar =
ADDCOLUMNS (
    CALENDAR ( DATE ( 2011, 1, 1 ), DATE ( 2014, 12, 31 ) ),
    "Year", YEAR ( [Date] ),
    "Quarter", "Q" & QUARTER ( [Date] ),
    "Month No", MONTH ( [Date] ),
    "Month", FORMAT ( [Date], "MMM" ),
    "Year-Month", FORMAT ( [Date], "YYYY-MM" )
)
```

Relationship: `Calendar[Date]` (one) to `Superstore[Order Date]` (many), single direction. Sort `Month` by `Month No`.

### 3. DAX measures

```DAX
Total Sales    = SUM ( Superstore[Sales] )
Total Profit   = SUM ( Superstore[Profit] )
Profit Margin  = DIVIDE ( [Total Profit], [Total Sales] )
Total Quantity = SUM ( Superstore[Quantity] )
Total Orders   = DISTINCTCOUNT ( Superstore[Order ID] )
Customers      = DISTINCTCOUNT ( Superstore[Customer ID] )
Avg Discount   = AVERAGE ( Superstore[Discount] )
Avg Order Value = DIVIDE ( [Total Sales], [Total Orders] )
Avg Ship Days  = AVERAGE ( Superstore[Ship Days] )

Sales PY       = CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Calendar'[Date] ) )
Sales YoY %    = DIVIDE ( [Total Sales] - [Sales PY], [Sales PY] )
Profit PY      = CALCULATE ( [Total Profit], SAMEPERIODLASTYEAR ( 'Calendar'[Date] ) )
Profit YoY %   = DIVIDE ( [Total Profit] - [Profit PY], [Profit PY] )

Loss-Making Lines =
CALCULATE ( COUNTROWS ( Superstore ), Superstore[Profit] < 0 )

Profit Colour =
IF ( [Total Profit] < 0, "#C2410C", "#0E7C6B" )   -- use in Conditional formatting > Field value
```

### 4. Report pages

**Page 1: Executive overview**

- Slicers (top bar): Year, Region, Segment, Category
- KPI cards: Total Sales, Total Profit, Profit Margin, Total Orders, Avg Discount (with YoY % as the card's tooltip or subtitle)
- Line chart: `Calendar[Year-Month]` vs Total Sales and Total Profit
- Clustered bar: Region vs Total Profit
- Donut or bar: Category vs Total Sales

**Page 2: Products and profitability**

- Bar chart: Sub-Category vs Total Profit, conditional colour using `Profit Colour`
- Scatter: Sales (X) vs Profit (Y), detail = Sub-Category, size = Total Quantity
- Column chart: Discount Band vs Profit Margin
- Matrix: Category > Sub-Category with Sales, Profit, Margin (data bars on Profit)

**Page 3: Geography and customers**

- Filled map: State (location), colour saturation by Total Profit (diverging, red to green)
- Bar: Top 10 / Bottom 10 states by profit (Top N filter)
- Table: Top 10 customers by Total Sales
- Bar: Ship Mode vs Avg Ship Days

### 5. Interactivity checklist

- Sync the Year and Region slicers across all pages (View > Sync slicers)
- Enable cross-filtering between visuals (Format > Edit interactions)
- Add a **Clear filters** bookmark button on each page
- Add drill-through from Sub-Category to a detail table of order lines
- Set a tooltip page showing Sales, Profit and Margin for any hovered category

### 6. Design guidance

- Palette: profit teal `#0E7C6B`, loss orange `#C2410C`, neutral blue `#1F4E79`, background `#F3F5F4`
- Font: Segoe UI (Power BI default) or Public Sans
- Keep one colour meaning: teal is always profit or gain, orange is always loss
- Aim for 5 to 7 visuals per page; use consistent card sizes and a 16:9 canvas

## Key findings (summary)

- Sales $2.30M, profit $286K, margin 12.5%
- Technology and Office Supplies earn about 17% margin; Furniture earns 2.5%
- Tables, Bookcases and Supplies lose money overall
- Discounts above 20% push margin negative (-15% for 20-40%, -77% above 40%)
- Central region has the weakest margin (7.9%); West is the strongest (14.9%)
- Texas, Ohio, Pennsylvania and Illinois are the largest profit drains

See the report for details and recommendations.

## Refreshing the data

Replace the source file, keep the same column names, then Home > Refresh. Extend the `CALENDAR` range if new years are added.
