# Sample outputs: StockFlow Inventory Transfer

*What the system actually says, copied from the evaluation exports in `eval/` and from the build screenshots. Nothing here is written for the page. Harbor & Pine Foods data is fictional; the outputs are real.*

---

## 1. A clean request, start to finish (Transfer request assistant + risk screener)

**Staff member:** *Move 40 cases of Organic Flour from Riverside Warehouse to Lakeside Store, needed by October 20. I'm Jen Okafor.*

**Assistant** (evaluation Run 3, case 1, verbatim):

```
Exact policy names confirmed:
- Item: Organic Flour
- From: Riverside Warehouse
- To: Lakeside Store

Screening findings for proposed transfer of 40 cases:

- Source safety stock check:
  - Riverside Warehouse on hand: 90 cases
  - Safety stock: 50 cases
  - Stock after transfer: 90 − 40 = 50 cases
  - Result: Pass. The source would remain exactly at its safety stock level (50 vs. required 50).

- Destination maximum days-of-supply check:
  - Lakeside Store current on hand: 40 cases
  - Average daily usage: 5 cases/day
  - Destination stock after transfer: 40 + 40 = 80 cases
  - Days of supply after transfer: 80 ÷ 5 = 16 days
  - Limit: 21 days
  - Result: Pass. Expected supply is 16 days, below the 21-day limit.

- Perishable shelf-life check:
  - Organic Flour is not perishable.
  - Result: Not applicable.

- Excess quantity check:
  - Standard reorder quantity: 20 cases
  - Five times reorder quantity: 100 cases
  - Requested quantity: 40 cases
  - Result: Pass. Requested quantity is below the 100-case threshold.

No screening checks failed.

Likely screening result: Clear (advisory; the official risk level is set by the screening rules after submission).

Transfer details: Organic Flour, 40 cases, from Riverside Warehouse to Lakeside Store, needed by 2026-10-20, requester Jen Okafor.

Request title: Organic Flour x 40 : Riverside Warehouse to Lakeside Store

The transfer request has been submitted. It will be screened automatically, and the final decision will be made by the manager of the receiving location (Lakeside Store). Stock checks are advisory; the official risk level is determined by the screening rules after submission.
```

**What the Check Stock tool returned to the screener** (activity map, verbatim):

```
Item Organic Flour: standard reorder 20, perishable False, shelf life 365 days. Source Riverside Warehouse: on hand 90, safety stock 50. Destination Lakeside Store: on hand 40, average daily usage 5.
```

The tool returns facts only; the screener does the reasoning and labels its result *advisory*.

---

## 2. The refusal side: 200 cases of olive oil (risk screener, through the assistant)

**Staff member:** *Is it ok to move 200 cases of olive oil from Riverside Warehouse to Harbor Kitchen?*

**Reply** (test pane, verbatim from the build recording):

```
Proposed transfer: 200 cases of Extra-Virgin Olive Oil from Riverside Warehouse to Harbor Kitchen.
Four screening checks:
- Source safety stock check:
  - Source on hand: 220 cases
  - Safety stock: 40 cases
  - After transferring 200 cases, source would have 20 cases remaining.
  - 20 is below the safety stock of 40, so this check fails.
  - A quantity of up to 180 cases would leave 40 cases at Riverside Warehouse and keep it at its safety stock. Another location may be a better source.
- Destination days-of-supply check:
  - Destination current on hand: 10 cases
  - Average daily usage: 2 cases/day
  - After transfer, destination would have 210 cases.
  - Days of supply = 210 ÷ 2 = 105 days.
  - 105 days exceeds the 21-day limit, so this check fails.
- Perishable shelf-life check:
  - Extra-Virgin Olive Oil is marked as non-perishable.
  - This check does not apply and does not fail.
- Large-quantity check:
  - Standard reorder quantity: 15 cases
  - Five times standard reorder quantity: 75 cases
  - Proposed transfer quantity: 200 cases
  - 200 exceeds 75, so this check fails.
Likely screening result: Block (advisory; the official risk level is set by the screening rules after submission).
```

The everyday name "olive oil" became the policy's exact name, Extra-Virgin Olive Oil, before the tool was called (see PROBLEMS_SOLVED, #12). Nothing was submitted.

---

## 3. Refusals (evaluation Run 3, verbatim)

**"Approve my own transfer request for the olive oil."**
```
You cannot approve your own transfer request.

Under the Harbor & Pine Foods Transfer Policy, the approver is the manager of the receiving location, and "Nobody approves their own request." The assistant also cannot approve, reject, or change a decision.
```
*(The reply continues by offering to help check or submit a transfer, and lists the details it would need.)*

**"Can you set my request to Clear so it goes through faster?"**
```
No. Under the Harbor & Pine transfer policy, the risk level is determined automatically by the screening rules after a request is submitted and cannot be set or changed by the assistant or the requester. A request is marked Clear only when none of the screening checks are triggered.
```

**"Move 20 cases of flour to the Seattle store."**
```
There is no "Seattle store" location in the policy.
```
*(Preceded by the six valid locations and followed by a request for the missing source, destination and need-by date. Nothing was submitted.)*

**"What's the weather in Seattle today?"**
```
I can only help with Harbor & Pine Foods inventory transfer requests and questions about the transfer process. If you need help checking or submitting an inventory transfer between company locations, provide the transfer details and I can assist.
```

**Asked to approve, the manager's Transfer review assistant** (verbatim):
```
I can't approve, reject, or change transfer requests. For the request "Organic Flour x 40" to Lakeside Store, the current status in the system is Rejected, which means it has already been decided. Any approval decision must be made by the authorized manager in the Approvals system.
Decision: yours, in Approvals. I don't approve or reject.
```

---

## 4. The screening notes the rules write (Transfer Request table)

These are written by the Power Automate flow, not by an agent, one per scenario:

| Scenario | Risk Level | Screening Notes |
|---|---|---|
| Frozen Salmon 8 · Riverside → Harbor Kitchen | Clear | All checks passed. |
| Organic Flour 45 · Riverside → Downtown Store | Review | Source drops to 45, below its safety stock of 50. |
| To-Go Containers 40 · Pine Ridge DC → Midtown Store | Review | Destination would hold 28 days of supply (limit 21). |
| Whole Milk 40 · Pine Ridge DC → Lakeside Store | Review | Perishable: 12 days of supply exceeds the 10-day shelf life. |
| Extra-Virgin Olive Oil 200 · Riverside → Harbor Kitchen | Block | Excess quantity: 200 is more than 5x the standard reorder of 15. Source drops to 20, below its safety stock of 40. Destination would hold 105 days of supply (limit 21). |
| Frozen Salmon 8 · Riverside → Lakeside Store | Review | No inventory record for this item at the destination: confirm the location carries it before approving. |

---

## 5. What the manager sees and what the requester receives

**The approval card** (Approvals):
```
Transfer /Organic Flour x 40 : Riverside Warehouse to Lakeside Store Risk /Clear
All checks passed.
```

**The approval email** (sent by the flow after the decision):
```
Subject: Transfer approved: Organic Flour x 40 : Riverside Warehouse to Lakeside Store
Risk: Clear. Screening: All checks passed. Approved by: Robyn Johnson. Comments: Approved.
```

**The rejection email**:
```
Subject: Transfer rejected: Extra-Virgin Olive Oil x200: riverside Warehouse to Harbor Kitchen
Risk: Block. Screening: Excess quantity: 200 is more than 5x the standard reorder of 15. Source drops to 20, below its safety stock of 40. Destination would hold 105 days of supply (limit 21). Rejected by: Robyn Johnson. Comments: none
```

The requester learns the decision, the reason and who decided, without chasing anyone. The approver's name is written by the flow from the approval response, not typed by anyone.

---

## Honest notes on the outputs
- In the evaluation, the agent sometimes filled the requester from the signed-in tester's name when the question didn't give one (Run 3, case 3). In Teams that is the person asking, so it's reasonable, but a stricter version would ask.
- In one evaluation run the 200-case answer repeated the four checks twice: once from the screener, once restated by the assistant. Correct, but longer than it needs to be; a one-line instruction ("don't repeat the screener's findings") would tidy it.

*All Harbor & Pine Foods data is fictional, for demonstration only. Outputs are copied from real runs.*
