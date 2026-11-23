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

## Power Query refresh and source-row validation

The Power Query partitions for Product, Sales, Customer, Reseller, Sales Territory, Sales Order, and Date were refreshed successfully in Power BI Desktop; all seven report a Ready state.

The imported output was checked against rows in the source workbook:

- Product keys 210–217 matched the source for SKU, product name, subcategory, and category.
- Sales lines 43659001–43659005 matched the source for ProductKey, quantity, unit price, Extended Amount, Unit Price Discount Pct, and Sales Amount. Floating-point values were equivalent within numeric precision.
- Model totals were 121,253 sales rows and distinct line keys, 31,455 orders, 274,776 units, 397 products and distinct product keys, 18,485 customers, and 1,461 dates.
- Imported net sales totaled 109,809,274.203; gross sales totaled 110,336,782.0914. These are source-field sums, not a claim that the source's discount field explains their difference.

## Session 2: Star schema and DAX validation

### Model review

The model is a sales-line star schema:

- **Fact:** `Sales` contains one row per `SalesOrderLineKey`.
- **Dimensions:** `Date`, `Product`, `Customer`, `Reseller`, and `SalesTerritory`.
- **Order dimension:** `SalesOrder` describes each sales order line and connects to `Sales` by `SalesOrderLineKey`.
- **Active relationships:** Order date (`Sales[OrderDateKey]` to `Date[DateKey]`) and the five fact-to-dimension relationships filter the fact table.
- **Inactive relationships:** Due date and ship date remain available for explicit `USERELATIONSHIP` measures.
- `Sales`-to-`SalesOrder` uses a many-to-one, one-direction relationship. This avoids bidirectional filtering and preserves the fact-to-dimension grain.

### Implemented measures

| Measure | DAX purpose |
|---|---|
| Gross Sales | Sum of `Extended Amount`. |
| Net Sales | Sum of `Sales Amount`. |
| Total Cost | Sum of `Total Product Cost`. |
| Total Profit | Net Sales minus Total Cost. |
| Gross Margin % | Profit divided by Net Sales. |
| Total Units | Sum of `Order Quantity`. |
| Total Orders | Distinct count of `SalesOrder[Sales Order]`. |
| Total Customers | Distinct count of `Sales[CustomerKey]`. |
| Average Sales per Order | Net Sales divided by Total Orders. |
| Average Sales per Customer | Net Sales divided by Total Customers. |
| Prior Month Net Sales | Net Sales shifted back one month with `DATEADD`. |
| Month-over-Month Growth % | `(current - prior month) / prior month`, using `DIVIDE` for safe zero handling. |
| Prior Year Net Sales | Net Sales shifted back one year with `DATEADD`. |
| Year-over-Year Growth % | `(current - prior year) / prior year`, using `DIVIDE` for safe zero handling. |

### Manual validation

Whole-model source/measure checks:

- Net Sales: 109,809,274.203, matching the source-column sum.
- Gross Sales: 110,336,782.0914, matching the source-column sum.
- Total Cost: 97,257,907.9547.
- Total Profit: 109,809,274.203 - 97,257,907.9547 = 12,551,366.2483.
- Gross Margin: 12,551,366.2483 / 109,809,274.203 = 11.4302%.
- Total Units: 274,776.
- Total Orders: 31,455, not 121,253 sales lines.
- Total Customers: 18,485.
- Average per Order: 109,809,274.203 / 31,455 = 3,490.9958.
- Average per Customer: 109,809,274.203 / 18,485 = 5,940.4530.

March 2020 manual time-intelligence check:

- March 2020 Net Sales: 4,429,832.5275.
- February 2020 Net Sales: 4,070,045.8933.
- MoM: (4,429,832.5275 - 4,070,045.8933) / 4,070,045.8933 = 8.8399%.
- March 2019 Net Sales: 2,092,434.3516.
- YoY: (4,429,832.5275 - 2,092,434.3516) / 2,092,434.3516 = 111.7071%.

Edge cases:

- July 2017 has no preceding month or prior-year data in this model, so both growth measures correctly return blank.
- The growth measures also return blank for a zero or missing prior-period denominator through `DIVIDE`.
- The 121,253 fact rows and 121,253 distinct line keys confirm that order count must use `SalesOrder[Sales Order]`, not `COUNTROWS(Sales)`.

## Session 3: Report quality and documentation

### Recommended visual narrative

Use one 1920×1080 executive sales page with a top-to-bottom reading and filtering order:

1. **Page title and context.** “Sales and Product Performance” with the reporting-period context shown separately.
2. **Headline KPI cards:** Net Sales, Gross Sales, Total Profit, Gross Margin %, Total Orders, and Total Units.
3. **Monthly trend:** A line chart of Net Sales and a separate MoM growth measure. Display the same date hierarchy and avoid dual unrelated axes.
4. **Product category breakdown:** A clustered bar chart with Net Sales and Total Profit for four categories. Category is the primary comparison dimension; units can be a tooltip.
5. **Top products:** A ranked horizontal bar chart for Net Sales or Units, limited to the top 10 and sorted descending.
6. **Territory view:** A matrix or map only for broad Region/Country values. Do not expose City or Postal Code.
7. **Filters:** Reporting date, fiscal year/quarter, product category/subcategory, territory region/country, and sales channel.

Avoid using the fact table or surrogate keys as fields. Use measures and dimension attributes, and keep line-detail fields unavailable in the report.

### Accessibility check

- Use the Fluent theme with high-contrast text and avoid color alone to communicate positive or negative growth.
- Add explicit chart titles and meaningful axis labels. For example, use “Net Sales by Month” rather than “Sum of Sales Amount”.
- Provide subtitles for KPI units and time context.
- Use tooltips for supporting measures, but not as the only location for critical values.
- Make growth trends distinguishable by position and direct labels as well as color.
- Preserve keyboard navigation and provide an on-page reset or clear filters control.
- Verify visual alternatives and alt text manually in Power BI Desktop after the report canvas is populated. This PBIP currently has no populated report visuals, so alt text cannot be checked automatically from PBIP JSON.
- Run Power BI Desktop’s Accessibility checker before publishing.

### Performance check

Trace-based DAX samples completed successfully:

- KPI row query: 7 ms.
- Customer/reseller distinct-count query: 4 ms; 16 ms CPU.
- Product category summary: 7 ms; 16 ms CPU.
- 37-row monthly trend: 23 ms; 16 ms CPU.
- Month-over-month query returned 37 periods and included `null` for July 2017, as expected.

Performance actions:

- Import mode is retained for the 121,253-line fact and small dimensions.
- Calculations use direct sums and distinct counts rather than table materialization.
- The Date table and active order-date relationship support time-intelligence.
- Date and product/territory dimensions should remain the main slicer fields.
- Keep the monthly trend and category summary separate; avoid one high-cardinality visual that mixes many customer, product, order, and date values.
- Profile in Power BI Desktop Performance Analyzer after visuals are added and remove high-cost visual-level summaries before publishing.

### Data dictionary

| Table | Key fields | Grain / purpose |
|---|---|---|
| Sales | SalesOrderLineKey, foreign keys, quantities and amounts | One sales-order-line fact row; central fact table |
| SalesOrder | SalesOrderLineKey, Sales Order, Sales Order Line, Channel | Order and channel attributes for each line |
| Date | DateKey, Date, Fiscal Year, Fiscal Quarter, Month | Reporting date dimension; active OrderDateKey relationship |
| Product | ProductKey, SKU, Product, Category, Subcategory, Color, Model, prices/cost | Product hierarchy and attributes |
| Customer | CustomerKey, hidden Customer ID/name/city/postal code, broad region attributes | Customer dimension; detailed fields hidden |
| Reseller | ResellerKey, hidden Reseller ID/name/city/postal code, Business Type, broad region | Reseller dimension; detailed fields hidden |
| SalesTerritory | SalesTerritoryKey, Region, Country, Group | Broad geographic reporting |

Primary measures:

| Measure | Formula / purpose |
|---|---|
| Gross Sales | `SUM(Sales[Extended Amount])`; gross source amount |
| Net Sales | `SUM(Sales[Sales Amount])`; net source amount |
| Total Cost | `SUM(Sales[Total Product Cost])` |
| Total Profit | Net Sales minus Total Cost |
| Gross Margin % | `DIVIDE([Total Profit], [Net Sales])` |
| Total Units | `SUM(Sales[Order Quantity])` |
| Total Orders | `DISTINCTCOUNT(Sales[Sales Order])` |
| Total Customers | `DISTINCTCOUNT(Sales[CustomerKey])` |
| Average Sales per Order | Net Sales divided by Total Orders |
| Average Sales per Customer | Net Sales divided by Total Customers |
| Prior Month Net Sales | `DATEADD('Date'[Date], -1, MONTH)` |
| Month-over-Month Growth % | `(Current - Prior) / Prior` |
| Prior Year Net Sales | `DATEADD('Date'[Date], -1, YEAR)` |
| Year-over-Year Growth % | `(Current - Prior) / Prior` |

### Sensitive-data review

The source contains or can derive potentially sensitive fields:

- 18,485 customer names and customer IDs.
- 702 reseller IDs and 700 reseller names.
- City and postal-code values for customers and resellers.
- 324 distinct customer postal codes and 502 distinct reseller postal codes.

The model now hides Customer ID, Customer name, Customer City, Customer Postal Code, Reseller ID, Reseller name, Reseller City, Reseller Postal Code, and fact/dimension surrogate keys. State-Province and Country-Region remain available for broad reporting, subject to row-level privacy and publication policy.

Before publishing:

1. Confirm that hiding a field in the model does not by itself enforce row-level security or prevent export if the field remains reachable through the data source.
2. Restrict report access and exports to approved audiences.
3. Avoid visual combinations that enable re-identification of small customer or postal-code cohorts.
4. Do not publish Customer or Reseller IDs, names, cities, or postal codes.
5. Configure and test row-level security if users must receive different regional or account views.
6. Document the workbook file path and replace it with a governed shared source before cloud refresh.

## Session 4: Capstone build

### Business question

**How should the sales leadership team allocate growth and profitability attention across fiscal periods, channels, territories, and product categories?**

The scenario is to separate the following decisions:

- Which fiscal periods are growing or contracting.
- Whether Internet or Reseller channel growth is more profitable.
- Which territories contribute the most revenue and should receive further investment.
- Which product categories and products drive sales and profit.
- Whether order and customer counts support the revenue narrative at the correct grain.

### Source-to-data-to-model trace

| Requirement | Source / transformation | Model object | Capstone output |
|---|---|---|---|
| Net sales and gross sales | `Sales[Sales Amount]` and `Sales[Extended Amount]` | Sales fact plus Net/Gross Sales measures | Executive revenue cards |
| Profit and margin | Net Sales and `Total Product Cost` | Total Profit and Gross Margin % measures | Category/channel/territory profitability |
| Order count | `SalesOrder[Sales Order]` through `SalesOrderLineKey` | `Sales[Sales Order]` fact calculated column and `Total Orders` measure | Correct order counts in all filter contexts |
| Channel | `SalesOrder[Channel]` through `SalesOrderLineKey` | Hidden `Sales[Sales Channel]` fact calculated column | Internet versus Reseller analysis |
| Time growth | Active Order Date relationship | Date dimension plus MoM/YoY measures | Fiscal trend and growth |
| Product performance | Product category/subcategory hierarchy | Product dimension plus category summaries | Revenue and profit ranking |
| Territory performance | Sales territory dimensions | SalesTerritory region/country/group attributes | Regional comparison |
| Customer reach | CustomerKey in the fact | Total Customers measure | Customer coverage |

### Capstone validation

- 121,253 fact rows reconcile to 121,253 distinct SalesOrderLineKeys.
- 31,455 distinct order IDs reconcile at the whole-model level.
- 18,485 distinct customers and 350 sold products are represented.
- Net Sales reconciles to 109,809,274.203 and Profit to 12,551,366.2483.
- Internet Net Sales: 29,358,677.2207 across 27,659 orders.
- Reseller Net Sales: 80,450,596.9823 across 3,796 orders.
- Internet and Reseller Net Sales sum to full Net Sales.
- The Internet margin is 41.1493%; the Reseller margin is 0.5848%.
- Bikes generate 94,620,526.2077 Net Sales and 10,515,096.6084 Profit.
- Fiscal-year Net Sales are 23,860,891.1663 for FY2018, 34,070,108.5001 for FY2019, and 51,878,274.5366 for FY2020.

The fact-level order count must remain the primary order measure. It avoids the prior incorrect behavior in territory or channel slices where the SalesOrder lookup could repeat the full global order count.

### Realistic team handoff

**Sales leadership:** Review Net Sales, Profit, margin, MoM growth, and YoY growth.

**Channel managers:** Compare Internet and Reseller by order count, average sales per order, and margin.

**Product managers:** Rank categories and products by Net Sales, Units, and Profit. Do not rank unfiltered products from a total that includes unrelated contexts.

**Territory managers:** Use Region and Country rather than hidden City/Postcode fields. Validate order counts with the fact-level Total Orders measure.

**Finance:** Reconcile to Gross Sales, Net Sales, Total Cost, and Total Profit. Investigate source discount fields and gross/net differences before publishing cost explanations.

**Data/BI team:** Refresh all partitions, preserve 2,113 blank ship-date lines, and validate foreign-key matches and source totals after each workbook refresh.

## Validation checklist for future refreshes

1. Confirm `SalesOrderLineKey` remains unique and that fact foreign keys match their dimension keys.
2. Check that order, due, and ship date keys have the expected matches; investigate new unmatched dates and keep valid blank ship dates distinct from invalid keys.
3. Confirm quantity and monetary columns remain numeric, the discount field remains decimal, and gross/net differences are reviewed rather than silently overwritten.
4. Recheck product key uniqueness and that category/subcategory values are populated or intentionally blank.
5. Reconcile totals to the source for a selected period before publishing KPI cards.

## Session 5: Capstone finish

### Final measure set

The semantic model now contains 20 measures:

- Revenue/profit: Gross Sales, Net Sales, Total Cost, Total Profit, Gross Margin %, Total Units.
- Order/customer: Total Orders, Total Customers, Average Sales per Order, Average Sales per Customer, Net Sales Per Order.
- Time intelligence: Prior Month Net Sales, Month-over-Month Growth %, Prior Year Net Sales, Year-over-Year Growth %.
- Variance: Sales Variance, Margin Variance.
- Contribution: Product Contribution %, Territory Contribution %, Channel Contribution %.

### Report build specification

The report should use a single 1920×1080 executive page named **Sales & Product Performance**:

1. Page title and a clear reporting-period subtitle.
2. Six KPI cards: Net Sales, Gross Sales, Total Profit, Gross Margin %, Total Orders, and Total Units.
3. One monthly line chart: Date[Month] on the axis and Net Sales plus Month-over-Month Growth % in tooltips or an accompanying growth card.
4. One category bar chart: Product[Category] and Net Sales; use Total Profit and Total Units as tooltips.
5. One territory bar chart: SalesTerritory[Region] and Net Sales; use Territory Contribution % and Total Orders as tooltips.
6. One channel matrix: Sales[Sales Channel], Net Sales, Total Orders, Gross Margin %, and Channel Contribution %.
7. Slicers for Date[Year/Fiscal Year], Product[Category], and SalesTerritory[Region].

Recommended visual ordering follows the narrative:

- First understand the amount and profitability.
- Second identify month movement.
- Third identify product and territory drivers.
- Fourth compare channel economics.

Do not place customer name, customer ID, reseller name, reseller ID, city, or postal code in report visuals. Keep hidden fields out of export and drill-through experiences.

### DAX validation and edge cases

Whole-model manual reconciliation:

- Net Sales = 109,809,274.203.
- Gross Sales = 110,336,782.0914.
- Total Cost = 97,257,907.9547.
- Total Profit = 12,551,366.2483.
- Gross Margin % = 11.4302%.
- Total Orders = 31,455.
- Net Sales Per Order = 3,490.9958.
- Average Sales per Customer = 5,940.4530.

March 2020:

- Net Sales = 4,429,832.5275.
- Prior-month Net Sales = 4,070,045.8933.
- MoM = 8.8399%.
- Sales Variance = 359,786.6342.

July 2017 edge case:

- Prior Month Net Sales, MoM Growth %, Sales Variance, and Margin Variance return blank.

Contribution reconciliation:

- Product category contributions sum to 100%.
- Reseller plus Internet channel contributions sum to 100%.
- Territory contribution values sum to 100% across the 10 reporting regions.
- Internet and Reseller Net Sales sum to 109,809,274.203.

### Capstone rubric self-check

| Rubric item | Status |
|---|---|
| Business question stated | Complete |
| KPI definitions written | Complete |
| Source and grain profiled | Complete |
| Power Query transformations generated and explained | Complete |
| Query output checked against source rows | Complete |
| Star schema reviewed and relationships documented | Complete |
| DAX measures created and explained | Complete |
| Measures matched to manual calculations | Complete |
| Edge cases tested | Complete |
| Data dictionary written | Complete |
| Sensitive-data fields identified and hidden | Complete |
| Visual and narrative specification written | Complete |
| Export and refresh reconciliations performed | Complete |
| Report visuals added in Power BI Desktop | Manual handoff required |
| Accessibility checker run | Manual handoff required |
| Performance Analyzer run | Manual handoff required |

### Work and handoff

Repository and PBIP:

- Updated: [Sales_KPI_Definitions.md](./Sales_KPI_Definitions.md)
- Updated: `Sales.SemanticModel/definition/tables/Sales.tmdl`
- Updated: `Sales.SemanticModel/definition/relationships.tmdl`
- Existing report canvas: `Sales.Report/definition/pages/51ae08578c4181e78602/page.json`

AI/Copilot features used:

- Power Query cleanup and numeric type guidance.
- DAX measure generation and filter-context debugging.
- Semantic-model relationship review.
- Source-row, measure, and edge-case validation queries.
- Report narrative, accessibility, data-dictionary, and governance documentation.

Hand changes still required:

1. Add the specified visuals to the Page 1 report canvas.
2. Set visual titles, subtitles, axis labels, tooltips, sort order, and slicer defaults.
3. Set each KPI, chart, and slicer alternative text in Power BI Desktop.
4. Run Power BI Desktop Accessibility checker.
5. Run Performance Analyzer and verify no visual is unusually slow.
6. Save and refresh the PBIP, then publish or share through the approved workspace.
