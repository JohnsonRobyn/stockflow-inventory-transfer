# The planner's dashboard: StockFlow Inventory Transfer

*A three-page Power BI report reading live from Dataverse. It turns the rows the agents and rules produce into the pictures a planner would open on a Monday: what the rules caught, where stock is thin, and how fast managers decide. Harbor & Pine Foods data is fictional; every number below was checked against the source tables.*

![Days of Supply page](screenshots/fig-dashboard-days-of-supply.png)

## Connection
- **Source:** the Dataverse Web API through Power BI's **OData feed** connector: `https://<your-org>.crm.dynamics.com/api/data/v9.2/`, signed in with an organisational account.
- **Why not the Dataverse connector:** its navigator listed hundreds of system tables but not the four custom ones, while the Web API returned them fine (PROBLEMS_SOLVED, #17).
- **Tables used:** `cr425_transferrequests`, `cr425_inventories`, `cr425_items`, `cr425_locations`.

## Shaping (Power Query)
- Kept only the columns the report needs and renamed them in plain English (Request Title, Quantity, Need By, On Hand, Avg Daily, Safety Stock, …).
- Lookups arrive as `_..._value` ID columns, used for the relationships and then hidden.
- **Choice codes turned into words** with conditional columns, using codes read from the table metadata:

| Column | Code → label |
|---|---|
| Status | 472290000 Submitted · 472290001 Approved · 472290002 Rejected · 472290003 In Transit · 472290004 Received |
| Risk Level | 472290000 Clear · 472290001 Review · 472290002 Block · anything else → **Not screened** |

"Not screened" keeps the early test rows (created before the rules existed) honest instead of hiding them.

## Model
Four tables, five relationships, all many-to-one and single direction:

| From | To | Active |
|---|---|---|
| Transfer Request[Item ID] | Item[Item ID] | ✓ |
| Transfer Request[To Location ID] | Location[Location ID] | ✓, the receiving location is the location of record, because its manager approves |
| Transfer Request[From Location ID] | Location[Location ID] | inactive (two paths to one table can't both be active) |
| Inventory[Item ID] | Item[Item ID] | ✓ |
| Inventory[Location ID] | Location[Location ID] | ✓ |

Transfer Request is never joined directly to Inventory: both already reach Item and Location, and a direct line would create ambiguous paths.

## The nine measures

**Inventory: is the stock healthy?**
```DAX
Days of Supply = DIVIDE( SUM(Inventory[On Hand]), SUM(Inventory[Avg Daily]) )
Below Safety Stock = IF( SUM(Inventory[On Hand]) < SUM(Inventory[Safety Stock]), 1, 0 )
Pairs Below Safety = SUMX( SUMMARIZE(Inventory, Location[Location], Item[Item]), [Below Safety Stock] )
```

**Transfer Request: what's flowing through the system?**
```DAX
Requests = COUNTROWS('Transfer Request')
Blocked Requests = CALCULATE( [Requests], 'Transfer Request'[Risk Level] = "Block" )
Block Rate = DIVIDE( [Blocked Requests], [Requests] )
Decided Requests = CALCULATE( [Requests], 'Transfer Request'[Status] IN { "Approved", "Rejected" } )
Cycle Time (hrs) = AVERAGEX( FILTER('Transfer Request', 'Transfer Request'[Status] IN { "Approved", "Rejected" }), DATEDIFF('Transfer Request'[Created], 'Transfer Request'[Modified], MINUTE) / 60 )
Overridden Blocks = CALCULATE( [Requests], 'Transfer Request'[Risk Level] = "Block", 'Transfer Request'[Status] = "Approved" ) + 0
```
`+ 0` makes the card show 0 instead of blank when nothing matches. **Pairs Below Safety** checks every location × item pair separately, so one healthy item can't hide a short one.

**Two display columns:**
```DAX
Created On = FORMAT( 'Transfer Request'[Created], "yyyy-mm-dd hh:mm" )
Decided On = IF( 'Transfer Request'[Status] IN { "Approved", "Rejected" }, FORMAT( 'Transfer Request'[Modified], "yyyy-mm-dd hh:mm" ), BLANK() )
```
Text columns replace Power BI's automatic date hierarchies, which split a timestamp into year / quarter / month / day. Decided On stays blank for undecided requests (PROBLEMS_SOLVED, #18).

## The three pages

| Page | Question it answers | Visuals |
|---|---|---|
| **Risk mix** | What did the rules catch? | Cards: Requests, Blocked Requests, Block Rate, Overridden Blocks · location slicer · requests by Risk Level, split by Status (approved green, rejected red, submitted amber, received grey) · request table with the screening notes |
| **Days of Supply** | Where is stock thin, and where is it piling up? | Location × item matrix · bar chart by location with a dashed line at the 21-day policy limit · Pairs Below Safety card · item slicer |
| **Approval cycle** | How fast do managers decide? | Cards: Cycle Time (hrs), Decided Requests · cycle time by receiving location · decided-request table |

A page navigator, one theme, a header and a one-line question on every page.

### The colour rule (Days of Supply, matrix and chart identical)

| Days of supply | Colour | Meaning |
|---|---|---|
| under 7 | red | thin: could run out within a week |
| 7 to 21 | green | healthy, inside the policy |
| over 21 | slate blue | held above the 21-day limit (expected for a warehouse that supplies others; a flag for a store) |

## The numbers, checked
Every Days of Supply value was recomputed from the inventory rows (On Hand ÷ Avg Daily, summed across the location's items):

| Location | On hand ÷ daily usage | Days | Colour |
|---|---|---|---|
| Riverside Warehouse | 370 ÷ 13 | 28.5 | blue |
| Pine Ridge DC | 900 ÷ 35 | 25.7 | blue |
| Midtown Store | 300 ÷ 12 | 25.0 | blue |
| Lakeside Store | 60 ÷ 10 | 6.0 | red |
| Downtown Store | 40 ÷ 12 | 3.3 | red |
| Harbor Kitchen | 16 ÷ 5 | 3.2 | red |
| **All locations** | 1,686 ÷ 87 | 19.4 | |

Risk mix: 25 requests, 3 blocked, a 12% block rate, 0 overridden; the bars sum to 25. Approval cycle: 6.7 hours on average across 13 decided requests.

## What it says, read with context
- **A location's number blends its items.** Riverside's 28.5 looks overstocked, but drill in and flour and salmon sit at a healthy **15.0** days; extra-virgin olive oil alone, at **73.3**, pulls the total up. The location view is the summary; the item view is the truth.
- **One store sits above the limit.** Midtown Store holds 25 days of to-go containers. Blue is expected for the warehouse and the distribution centre, but for a store it's worth a question.
- **A risk the dashboard can't flag yet.** Pine Ridge DC holds **20 days of whole milk against a 10-day shelf life**. The spoilage rule only runs when a transfer is requested, so stock already sitting past its shelf life goes unnoticed. A next measure: days of supply versus shelf life for every perishable pair.
- **No location is below safety stock today** (Pairs Below Safety = 0); the three red locations are thin, not short.
- **Lakeside looks like the slow approver** at about 9.6 hours, partly because some evaluation-run requests were decided the next morning. A dashboard on honest test data still needs a person to read it.
- **Stock levels are a snapshot.** Approving a transfer doesn't move stock in this build (there is no "received" step), so Days of Supply reflects the inventory table, not stock after approved transfers.

## Kept honest on purpose
The real build history stays in the data: duplicates, retyped titles, and the early rows marked "Not screened". Cleaning them for the demo would have made a tidier chart and a less truthful one.

*All Harbor & Pine Foods data is fictional, for demonstration only.*
