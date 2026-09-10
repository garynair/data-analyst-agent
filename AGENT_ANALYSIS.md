# Agent Analysis: Top Selling Products & Top Customers

## Purpose
Reproducible record of the analysis behind `Top_Products_Customers_Analysis.xlsx`. Re-run the queries below against the same Supabase project to refresh the workbook, or hand this file to an agent to regenerate the report end to end.

## Source
- **Supabase project:** (redacted — point at your own Supabase project with the same schema)
- **Tables used:** `public.transactions` (3,000 rows), `public.customers` (1,000 rows)
- **Analysis date:** 2026-08-09
- **Scope:** Excludes returned transactions (`Returned = true`)

## Schema reference

**transactions**: `Transaction_ID` (bigint), `Date_of_Purchase` (text), `Customer_ID` (bigint), `Product_Category` (text), `Product_Name` (text), `Units` (bigint), `Price` (float8), `Discounts` (float8), `Returned` (bool), `Mode_of_Payment` (text), `Purchase_Channel` (text)

**customers**: `Customer_ID` (bigint), `Customer_Name` (text), `Customer_Email` (text), `Customer_Number` (text), `Age` (bigint), `Gender` (text), `Location` (text)

## Methodology
1. Ranked all products by total units sold (non-returned transactions only).
2. Took the top 5 products by units.
3. For each of those 5 products, joined transactions to customers and ranked buyers by **net spend** (`Units * Price - Discounts`), keeping the top 5 per product.
4. Net revenue = `SUM(Units * Price) - SUM(Discounts)`. Gross revenue = `SUM(Units * Price)`.

## Step 1 — Top selling products (by units)

```sql
SELECT "Product_Name", "Product_Category",
  SUM("Units") AS total_units,
  SUM("Units"*"Price") AS gross_revenue,
  SUM("Units"*"Price" - COALESCE("Discounts",0)) AS net_revenue,
  COUNT(*) AS transaction_count
FROM transactions
WHERE "Returned" = false OR "Returned" IS NULL
GROUP BY "Product_Name", "Product_Category"
ORDER BY total_units DESC
LIMIT 10;
```

## Step 2 — Top 5 customers for each of the top 5 products

```sql
WITH top_products AS (
  SELECT "Product_Name"
  FROM transactions
  WHERE "Returned" = false OR "Returned" IS NULL
  GROUP BY "Product_Name"
  ORDER BY SUM("Units") DESC
  LIMIT 5
),
cust_agg AS (
  SELECT t."Product_Name", t."Customer_ID", c."Customer_Name", c."Customer_Email", c."Location",
    SUM(t."Units") AS units_purchased,
    SUM(t."Units"*t."Price" - COALESCE(t."Discounts",0)) AS net_spend,
    COUNT(*) AS transaction_count
  FROM transactions t
  JOIN customers c ON c."Customer_ID" = t."Customer_ID"
  WHERE t."Product_Name" IN (SELECT "Product_Name" FROM top_products)
    AND (t."Returned" = false OR t."Returned" IS NULL)
  GROUP BY t."Product_Name", t."Customer_ID", c."Customer_Name", c."Customer_Email", c."Location"
),
ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY "Product_Name" ORDER BY net_spend DESC) AS rnk
  FROM cust_agg
)
SELECT * FROM ranked WHERE rnk <= 5 ORDER BY "Product_Name", rnk;
```

> Note the identifiers are quoted (`"Product_Name"`) because the columns were created with mixed-case names in Postgres — unquoted references fail with `column "product_name" does not exist`.

## Results snapshot (2026-08-09)

### Top 10 products by units sold
| Rank | Product | Category | Units | Gross Rev | Net Rev | Txns |
|---|---|---|---|---|---|---|
| 1 | Motor Oil | Automotive | 220 | $53,152.33 | $48,471.71 | 66 |
| 2 | Laptop | Electronics | 213 | $52,380.38 | $48,393.47 | 69 |
| 3 | Board Game | Toys | 201 | $49,355.47 | $45,566.90 | 58 |
| 4 | Car Charger | Automotive | 187 | $49,826.82 | $46,525.38 | 61 |
| 5 | Jeans | Fashion | 181 | $49,608.38 | $45,242.25 | 60 |
| 6 | Headphones | Electronics | 172 | $44,740.09 | $40,913.23 | 54 |
| 7 | Doll | Toys | 171 | $50,826.06 | $46,683.58 | 53 |
| 8 | Watch | Fashion | 162 | $37,648.45 | $34,506.14 | 51 |
| 9 | Bed Sheets | Home | 162 | $41,692.74 | $38,543.19 | 53 |
| 10 | Dress | Fashion | 160 | $43,020.45 | $39,972.74 | 53 |

### Top 5 customers per top-5 product (by net spend)
Full detail is in the workbook's "Top Customers by Product" tab. Highlights: William Wilkinson (Laptop, 2 txns, $3,083.31), Jill Howell (Car Charger, 2 txns, $1,910.43), and Scott Williams (Jeans, 2 txns, $1,869.43) are repeat buyers within their product — candidates for retention/cross-sell follow-up. Crystal Johnson appears in both the Laptop and Board Game top-5 lists.

## How to regenerate the Excel file
1. Run Step 1 query → gives the "Top Selling Products" tab data (take top 10, or adjust `LIMIT`).
2. Run Step 2 query → gives the "Top Customers by Product" tab data (adjust `LIMIT 5` in `top_products` CTE and `rnk <= 5` filter to change how many products/customers are included).
3. Rebuild `report.xlsx` using `build_report.py` (openpyxl), substituting refreshed values into the `products` list and `data2` dict.
4. Run `recalc.py report.xlsx` to recalculate the SUM formulas and confirm zero formula errors before delivering.

## Assumptions / caveats
- "Top selling" = ranked by total units sold, not revenue. Revenue ranking would reorder slightly (e.g., Doll would outrank Car Charger on net revenue despite fewer units).
- Customer ranking within each product is by net spend, not units purchased — a customer with fewer units but a higher-priced order can rank above one with more units.
- Analysis excludes returned transactions entirely; it does not net returns against a customer's other same-product purchases.

---

# Report 2: Sales by Location & Channel

Reproducible record of `Sales_by_Location_Channel_Analysis.xlsx`. Same source, scope, and exclusion rule as above.

## Step 1 — Revenue by location

```sql
SELECT c."Location",
  SUM(t."Units") AS total_units,
  SUM(t."Units"*t."Price") AS gross_revenue,
  SUM(t."Units"*t."Price" - COALESCE(t."Discounts",0)) AS net_revenue,
  COUNT(*) AS transaction_count,
  COUNT(DISTINCT t."Customer_ID") AS unique_customers
FROM transactions t
JOIN customers c ON c."Customer_ID" = t."Customer_ID"
WHERE t."Returned" = false OR t."Returned" IS NULL
GROUP BY c."Location"
ORDER BY net_revenue DESC;
```

## Step 2 — Revenue by purchase channel

```sql
SELECT "Purchase_Channel",
  SUM("Units") AS total_units,
  SUM("Units"*"Price") AS gross_revenue,
  SUM("Units"*"Price" - COALESCE("Discounts",0)) AS net_revenue,
  COUNT(*) AS transaction_count
FROM transactions
WHERE "Returned" = false OR "Returned" IS NULL
GROUP BY "Purchase_Channel"
ORDER BY net_revenue DESC;
```

## Step 3 — Top product per location (by units)

```sql
WITH loc_prod AS (
  SELECT c."Location", t."Product_Name",
    SUM(t."Units") AS units,
    SUM(t."Units"*t."Price" - COALESCE(t."Discounts",0)) AS net_revenue,
    ROW_NUMBER() OVER (PARTITION BY c."Location" ORDER BY SUM(t."Units") DESC) AS rnk
  FROM transactions t
  JOIN customers c ON c."Customer_ID" = t."Customer_ID"
  WHERE t."Returned" = false OR t."Returned" IS NULL
  GROUP BY c."Location", t."Product_Name"
)
SELECT "Location", "Product_Name", units, net_revenue
FROM loc_prod WHERE rnk = 1
ORDER BY units DESC;
```

## Step 4 — Revenue by location x channel

```sql
SELECT c."Location", t."Purchase_Channel",
  SUM(t."Units"*t."Price" - COALESCE(t."Discounts",0)) AS net_revenue,
  SUM(t."Units") AS units
FROM transactions t
JOIN customers c ON c."Customer_ID" = t."Customer_ID"
WHERE t."Returned" = false OR t."Returned" IS NULL
GROUP BY c."Location", t."Purchase_Channel"
ORDER BY c."Location", t."Purchase_Channel";
```

## Results snapshot (2026-08-09)

Top location by net revenue: **Pune** ($124,909.91, 515 units, 93 unique customers), followed by Mumbai ($119,240.72) and Hyderabad ($117,131.91). Channel split overall: In-store $554,560.89 net vs Online $514,761.14 net. Mumbai and Hyderabad skew heavily In-store; Bangalore and Ahmedabad skew Online. Top product varies by city — see the "Top Product by Location" tab (Laptop leads Pune, Motor Oil leads Hyderabad/Mumbai/Delhi, Board Game leads Jaipur, Dress leads Surat, Puzzle leads Bangalore, Watch leads Kolkata, Smartwatch leads Chennai, Car Seat Cover leads Ahmedabad).

## How to regenerate this file
1. Run Steps 1–4 above.
2. Rebuild using `build_location_channel_report.py` (openpyxl), substituting refreshed values into `loc_data`, `chan_data`, `lc_data`, and `tp_data`.
3. Run `recalc.py location_channel_report.xlsx` and confirm zero formula errors before delivering.
