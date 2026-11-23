# Sales and Product KPI Definitions

## Scope and model grain

The `Sales` table is a sales-order-line fact table: one row per `SalesOrderLineKey`. Use `SalesOrder[Sales Order]` for order counts; counting fact rows or sales-order-line keys counts lines, not orders.

The definitions below use:

- **Net sales**: sum of `Sales[Sales Amount]`.
- **Gross sales**: sum of `Sales[Extended Amount]`.
- **Cost**: sum of `Sales[Total Product Cost]`.
- **Profit**: net sales less cost.
- **Reporting date**: `Date[Date]`, related actively to `Sales[OrderDateKey]`.

All monetary KPIs should use the report's chosen currency and display rounding; preserve full precision in calculations. Unless stated otherwise, KPI results respect report filters such as date, product, customer, and territory.

## Revenue metrics

| KPI | Definition |
|---|---|
| Total Sales Revenue | Net sales in the current filter context. This is intentionally the same calculation as **Net Sales**; use one measure and give it whichever display name suits the report. |
| Gross Sales | Sum of `Sales[Extended Amount]`, the source's pre-adjustment line amount. |
| Net Sales | Sum of `Sales[Sales Amount]`, the source's recorded sales amount. Do not recalculate it from quantity and unit price. |
| Average Sales per Order | Net sales divided by the distinct count of `SalesOrder[Sales Order]` for orders with sales in the current context. |
| Average Sales per Customer | Net sales divided by distinct `Sales[CustomerKey]` values with sales in the current context. |
| Sales Growth % | `(current-period net sales - prior comparable-period net sales) / prior comparable-period net sales`. Return blank when the prior-period denominator is zero or blank. |
| Month-over-Month Sales Growth | Sales Growth % comparing a calendar month with the immediately preceding calendar month. |
| Year-over-Year Sales Growth | Sales Growth % comparing a period with the equivalent period one year earlier (for example, the same month last year). |

## Customer metrics

| KPI | Definition |
|---|---|
| Total Customers | Distinct customers with at least one sales line in the selected period. This is an active-customer count, not the full customer master count. |
| New Customers | Customers whose first recorded order date is within the selected period. “First” is limited to the sales history available in this dataset. |
| Returning Customers | Customers with sales in the selected period and at least one recorded order before the period began. This is distinct from retention, which uses a prior-period cohort. |
| Customer Retention Rate | Customers who purchased in both the immediately preceding comparable period and the current period, divided by customers who purchased in the preceding period. Return blank when the prior-period customer count is zero. |
| Revenue per Customer | Net sales divided by distinct customers with sales in the current context. With this customer population, it is the same calculation as Average Sales per Customer. |
| Top Customers by Revenue | Rank customers by net sales in the selected context, descending. Show ties at the cutoff or state an explicit tie-break rule. |

## Product performance

| KPI | Definition |
|---|---|
| Total Products Sold | Sum of `Sales[Order Quantity]` (units), not the count of sales lines or distinct products. |
| Top Selling Products | Rank products by units sold, descending; show net sales as a companion value. If ranking by revenue instead, label the visual accordingly. |
| Lowest Performing Products | Rank products with sales by net sales, ascending. Keep unsold products separate from low-selling products; use profit when the intent is to identify loss-making products. |
| Revenue by Product Category | Net sales grouped by `Product[Category]`. |
| Revenue by Product Subcategory | Net sales grouped by `Product[Subcategory]`. |
| Product Contribution % | Product net sales divided by total net sales after removing all Product-table filters while retaining other report filters. |

## Cost and profitability

| KPI | Definition |
|---|---|
| Total Cost | Sum of `Sales[Total Product Cost]`. |
| Total Profit | Net sales minus total cost. |
| Gross Margin % | Total profit divided by net sales. Return blank when net sales are zero or blank. |
| Profit by Product Category | Net sales less total cost, grouped by product category. |
| Profit by Region | Net sales less total cost, grouped by `SalesTerritory[Region]`. |
| Cost Distribution by Product Category | Category cost divided by total cost for the current context, removing the category filter from the denominator. |
| Cost Distribution by Sales Territory | Territory cost divided by total cost for the current context, removing the territory filter from the denominator. |

## Data preparation applied to the PBIP

The semantic model was prepared for these KPIs without changing the source workbook:

1. **Keep the sales-line grain.** `SalesOrderLineKey` is unique in the loaded fact data. Use the order ID in `SalesOrder` for order counts and customer/product keys for distinct counts.
2. **Normalize descriptive text.** Power Query trims leading/trailing whitespace, removes non-printing characters, and converts empty strings to nulls in the Product, Customer, Reseller, Sales Territory, Sales Order, and Date descriptive columns. Keys and postal codes remain text/integer as appropriate.
3. **Preserve numeric precision.** The unit-price discount percentage is typed as a decimal number (not an integer) and formatted as a percentage. The source currently contains only zero values in this field.
4. **Use a proper date dimension.** The Date table is marked as the date table. `Date[DateKey]` relates to `Sales[OrderDateKey]` actively; Due Date and Ship Date relationships are inactive so measures can opt into those roles without creating ambiguous date filtering.
5. **Retain missing ship dates.** Blank ship dates are not treated as zero or as a real date; they remain blank and are excluded from ship-date analyses.
6. **Keep the source mapping intact.** The model still reads the existing AdventureWorks Sales workbook from its configured local path. For sharing or scheduled refresh on another machine, replace this machine-specific file path with a shared source or a Power Query parameter.

## Data-quality findings and decisions

The loaded source was profiled before preparing the model:

- 121,253 sales lines and 121,253 distinct sales-line keys; 31,455 distinct order IDs; 397 product rows and keys.
- The Date table contains 1,461 unique dates from 2017-07-01 through 2021-06-30. Sales order dates run from 2017-07-01 through 2020-06-15.
- All sales order-date and due-date keys matched the Date table. There are 2,113 sales lines with a blank Ship Date; preserve them rather than dropping valid sales.
- `Extended Amount` reconciles to `Order Quantity × Unit Price` within floating-point precision. However, 3,282 lines have different Extended Amount and Sales Amount values, totalling a gross-to-net difference of about 527,507.89, while the source's `Unit Price Discount Pct` is zero on every line.
- Therefore, use the provided Extended Amount and Sales Amount fields for gross and net sales respectively, but do **not** claim that the discount-percentage field explains their difference. Confirm the source's gross/net business meaning before presenting a discount or discount-rate KPI.
- Sales history begins in July 2017. New-customer and retention KPIs are therefore “first observed in this dataset” and “retained across observed periods,” not necessarily lifetime measures.

## Validation checklist for future refreshes

1. Confirm `SalesOrderLineKey` remains unique and that fact foreign keys match their dimension keys.
2. Check that order, due, and ship date keys have the expected matches; investigate new unmatched dates and keep valid blank ship dates distinct from invalid keys.
3. Confirm quantity and monetary columns remain numeric, the discount field remains decimal, and gross/net differences are reviewed rather than silently overwritten.
4. Recheck product key uniqueness and that category/subcategory values are populated or intentionally blank.
5. Reconcile totals to the source for a selected period before publishing KPI cards.
