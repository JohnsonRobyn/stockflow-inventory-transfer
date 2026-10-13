# Problems solved: StockFlow Inventory Transfer

*Every real problem met while building StockFlow, in the order the build met them: what I saw, what caused it, what fixed it, and what I took from it. Harbor & Pine Foods data is fictional; every problem below is real.*

The pattern across all of them: **read what the system actually received or returned (the run history, the raw inputs, the exported results) before changing anything.** Almost every fix was one small change once the cause was clear.

---

## Data (Dataverse)

### 1. Duplicates and generated sample rows
- **Symptom:** after the first import, the tables held duplicate rows and sample data the planning tool had generated ("Organic Apples", "Pine Store", "Harbor Warehouse"), including two different olive-oil items and look-alike location names.
- **Cause:** the plan generator seeds sample rows, and the import added mine on top.
- **Fix:** cleared all four tables and re-imported clean, in dependency order: Location → Item → Inventory → Transfer Request. Also remapped the seed file's Priority values to the High / Medium / Low choices the system had created, rather than forcing the system to match the file.
- **Lesson:** the screening rules are only as good as the reference data. Two "olive oil" rows would have quietly broken every check that looks an item up by name.

### 2. "Organic Flour is not a valid primary id Guid value"
- **Symptom:** importing Inventory from Excel failed on every row that named its item and location.
- **Cause:** the new Import from Excel (preview) resolves lookup columns by the hidden row ID, not by name.
- **Tried:** alternate keys on Item Name and Location Name, which is Microsoft's documented way to identify rows by business name. After the keys turned Active the import still failed the same way, so this importer does not use alternate keys for lookups.
- **Fix:** a two-step load: import the rows without lookups, then set Item and Location in the grid (24 lookups, checked against a written checklist). The keys stayed, because they now reject duplicate names.
- **Lesson:** test the documented route, record what actually happens, and keep the parts of a failed fix that still add value.

---

## Rules (Power Automate screening flow)

### 3. Every risk case skipped
- **Symptom:** the flow ran green, but no Risk Level was ever written.
- **Cause:** three small things, all found in the run history. A step was named "Scr" where the expressions said "Src". The Update steps had no Risk Level chosen. And the Switch received **"/Review"**, with a stray slash in its input, so it matched none of its cases.
- **Fix:** renamed the step, set the Update fields, and re-picked the Switch input as the output of the Risk step.
- **Lesson:** the risk calculation itself was right every time; the plumbing around it wasn't. Read each step's inputs, not just its green tick.

### 4. A crash on a real scenario: no stock record at the destination
- **Symptom:** sending salmon to Lakeside Store, which has never stocked salmon, crashed the flow.
- **Cause:** the destination stock lookup came back empty, and the days-of-supply maths had nothing to work with.
- **Fix:** two new checks test whether each stock lookup is empty; blanks count as zero in the maths, destination rules are skipped when there's no record, and a missing record counts as a hit, so the request goes to **Review** with the note *"No inventory record for this item at the destination: confirm the location carries it before approving."* The earlier scenario was re-run afterwards with an unchanged result.
- **Also caught:** Copilot's "Fix my expression" suggested `item()` in place of `outputs('Src')`. That only works inside a loop, so the suggestion was declined.
- **Lesson:** sending an item to a location that has never carried it is a real operational case. The system should flag it, not fail, and AI suggestions get reviewed like anyone else's.

### 5. The approval went nowhere
- **Symptom:** the first approval failed with "no valid users".
- **Cause:** the approver was an address outside the organisation, and Approvals only accepts approvers in the tenant.
- **Fix:** an approver inside the tenant.
- **Lesson:** that refusal is a governance control working, not a bug.

---

## Agents (Copilot Studio)

### 6. The agent landed in the wrong solution
- **Symptom:** the new agent wasn't in the StockFlow transfers solution.
- **Cause:** the create dialog defaults to the Default Solution, behind an expander labelled optional.
- **Fix:** added the agent to the solution by hand, and created every later agent with the solution chosen up front.
- **Lesson:** for anything that has to move between environments, the solution picker is not optional.

### 7. ActionSchemaInvalid on publish
- **Symptom:** the Submit transfer request tool wouldn't publish.
- **Cause:** an agent flow's "Respond to the agent" steps must declare an identical output schema, down to the description text. The success and failure branches had different descriptions.
- **Fix:** gave both branches the same output name, type and description.
- **Lesson:** a tool has one contract back to the agent, however many branches produce it.

### 8. Four "successful" runs that found nothing
- **Symptom:** four test runs of the Submit tool all succeeded, and all said the location was not found.
- **Cause:** one step's raw inputs showed the filter as `cr425_locationname eq ' Riverside Warehouse'`, with a single leading space inside the quotes.
- **Fix:** removed the space.
- **Lesson:** when a lookup finds nothing, read what was sent, not what you meant to type.

### 9. Every message returned SystemError
- **Symptom:** the test pane failed on every message for about an hour.
- **Cause:** a service-side fault. A known-good agent in another environment failed the same way, and Microsoft's service health advisory **PC1486478** later confirmed intermittent failures on the GPT-5.5 model both agents use.
- **Fix:** none needed in the build; waited for the service to recover.
- **Lesson:** isolate the platform before touching the configuration, and keep the incident ID. Forty minutes went into the agent before the known-good test settled it.

### 10. The requester's name in the destination slot
- **Symptom:** the agent filled the Submit tool with the right six values in the wrong slots: the requester's name as the destination, and the destination as the requester.
- **Cause:** every input carried the same placeholder description, so the model had names but no meaning.
- **Fix:** a one-line description on each input ("The exact name of the destination location…").
- **Lesson:** a tool's parameters are a prompt; describe them like one.

### 11. Need By Date one day early
- **Symptom:** agent-created rows showed 10/19 for a need-by date of 2026-10-20.
- **Cause:** the Date-only column used user-local behaviour, so "2026-10-20" was stored as UTC midnight and displayed a day earlier in Pacific time.
- **Fix:** set the column to **Time zone independent**.
- **Lesson:** dates are a governance detail, not a display detail.

### 12. The handoff crash: a blank ID
- **Symptom:** the risk screener passed every test on its own. Connected to the staff assistant, the first question ("200 cases of olive oil") ended in a gateway error.
- **Cause:** the everyday name "olive oil" passed straight through, the item lookup found nothing, and the stock query went out with a blank ID, which Dataverse rejected (*"Expression expected at position 87"*).
- **Fix:** two layers. The flow now substitutes a null ID when a lookup misses, so a bad name returns NOT FOUND instead of crashing. And both agents map everyday names to the policy's exact names before calling any tool.
- **Lesson:** test the chain, not just the links; names are an interface.

### 13. Get Transfer took four runs
- **Symptom:** the review assistant's read-only tool failed or returned the wrong thing four times.
- **Cause and fix, one per run:** an exact-match filter where a partial match was needed (switched to `contains()`); a missing parenthesis; a function the platform doesn't support in agent flows (`select`, replaced with explicit indexing); and a date returned as a raw timestamp (formatted).
- **Lesson:** each one was read off the run history and fixed in one change.

### 14. "Will be for approval" on a rejected request
- **Symptom:** the review assistant described an already-rejected request as if it were still waiting for approval.
- **Cause:** the data was right; the sentence order wasn't.
- **Fix:** one instruction line, "State the current Status first."
- **Lesson:** test the sentence the user reads, not just the tool's output.

---

## Evaluation

### 15. The default grader failed the right answers
- **Symptom:** Run 1 scored 83%, and both failures were correct refusals (declining to approve the user's own request, and declining a weather question).
- **Cause:** the default Answer quality grader scores whether a question was answered; for a governed agent, refusing is often the right answer.
- **Fix:** a custom **Safe behavior** grader with Pass and Fail criteria written for this agent.
- **Lesson:** measure the job, not a generic notion of helpfulness.

### 16. Then the custom grader was wrong too
- **Symptom:** Run 2 scored 92%; the one failure was a correct answer about the 21-day limit, marked "off-topic".
- **Cause:** the grader's criteria failed answers outside inventory transfers and never said that questions about the policy itself are inside.
- **Fix:** one opening sentence making policy questions explicitly on topic. Run 3 scored 100%.
- **Lesson:** a grader is a prompt too, and it needs the same precision as the agent.

---

## Power BI

### 17. The custom tables weren't in the connector
- **Symptom:** Power BI's Dataverse connector listed hundreds of system tables but not the four custom ones.
- **Cause:** the connector's navigator didn't surface them, although the Web API returned them fine.
- **Fix:** connected through the **OData feed** instead (`https://<your-org>.crm.dynamics.com/api/data/v9.2/`), which lists every table, and mapped the choice codes to words in Power Query.
- **Lesson:** when the obvious door is shut, check whether the data is actually there before stopping.

### 18. A dashboard that looked finished but said something false
- **Symptom 1:** Decided On showed a time for every row, including requests nobody had decided.
- **Fix:** it now stays blank unless the status is Approved or Rejected.
- **Symptom 2:** the first colour rule painted 21 or more days of supply green.
- **Cause:** in the Transfer Policy, 21 days is an **overstock** limit, not a target.
- **Fix:** red under 7 days, green 7 to 21, slate blue above 21, identical on the matrix and the chart.
- **Lesson:** colour is a claim; it has to match the policy it illustrates.

### 19. The first build was lost
- **Symptom:** the first build was never saved, and the work was gone.
- **Fix:** recovered from AutoRecovery, then rebuilt and saved early.
- **Lesson:** save before the first relationship drag.

---

## Shipping (solutions and ALM)

### 20. Two agents would have arrived without their tools
- **Symptom:** the pre-export check showed the solution held all three agents but only two of the four flows.
- **Cause:** Check Stock and Get Transfer had been built inside Copilot Studio and never joined the solution.
- **Fix:** added both, published, then exported.
- **Lesson:** the solution is the unit of delivery; check what's in it before you ship it.

### 21. The import blocked on Outlook
- **Symptom:** importing into the test environment stopped at the Office 365 Outlook connection.
- **Cause:** the account had no mailbox licence; the mailbox had come with a trial that lapsed.
- **Fix:** licensed the account. No workaround in the flow.
- **Lesson:** a package can be perfect and still fail on the target environment's licences and connections. Check the environment, not just the solution.

*All Harbor & Pine Foods data is fictional, for demonstration only.*
