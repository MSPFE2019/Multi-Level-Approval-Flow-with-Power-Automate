Below is the complete guide as a single Markdown document. Copy everything between the horizontal rules into a `README.md` (or any `.md` file) in your repository.

***

# Multi-Level Approval Flow with Power Automate

A step-by-step guide to building a **dynamic, configuration-driven, multi-level approval workflow** using Microsoft Power Automate and SharePoint — written so beginners can follow it end-to-end and architects can see the reasoning behind every decision.

> **Why this matters**
> Hard-coding approvers inside a flow is fast on day one and painful by month six. This guide shows you how to externalize *who approves what* into a configuration list so the same flow can serve many business processes without redeployment.

***

## Table of Contents

1.  \#1-overview
2.  \#2-sequential-vs-multi-level-approvals
3.  \#3-when-to-use-this-pattern
4.  \#4-architecture-at-a-glance
5.  \#5-prerequisites
6.  \#step-1--design-the-approval-matrix
7.  \#step-2--create-the-request-list
8.  \#step-3--build-the-flow-skeleton
9.  \#step-4--resolve-approvers-dynamically
10. \#step-5--loop-through-approval-levels
11. \#step-6--handle-approvals-and-rejections
12. \#step-7--update-the-request-and-notify
13. \#step-8--add-timeouts-and-escalation
14. \#step-9--test-the-flow
15. \#15-best-practices-and-common-pitfalls
16. \#16-generalizing-to-other-platforms
17. \#17-license

***

## 1. Overview

A **multi-level approval flow** routes a request through several approvers in a defined order — for example: *Team Lead → Department Head → Finance → CFO*. The pattern in this guide adds three properties that simple sequential flows usually lack:

*   **Dynamic approver resolution** — approvers are looked up at runtime from a configuration source (an "approval matrix"), not hard-coded in the flow.
*   **Variable depth** — different request types can have different numbers of levels.
*   **Central governance** — non-developers can update approvers by editing a list, not a flow.

> **Architect Note — Separation of Concerns**
> The single most important design decision is to keep *routing data* (who approves) separate from *orchestration logic* (how the flow runs). Treat the matrix as a small product: schema, version, ownership, and change control all matter.

***

## 2. Sequential vs. Multi-Level Approvals

A **simple sequential approval** uses a fixed chain — typically the requestor's manager and skip-manager — derived directly from Microsoft Entra ID. A **dynamic multi-level approval** resolves the chain from a configuration store keyed by request attributes (department, amount, region, item type).

| Dimension                         | Simple Sequential                  | Dynamic Multi-Level                                        |
| --------------------------------- | ---------------------------------- | ---------------------------------------------------------- |
| Build complexity                  | Low                                | Medium–High                                                |
| Flexibility                       | Low — flow edit required to change | High — data edit is enough                                 |
| Governance                        | Decentralized across many flows    | Centralized in one matrix                                  |
| Scalability across business units | Poor                               | Strong                                                     |
| Day-1 effort                      | Small                              | Larger                                                     |
| Long-term maintenance             | Grows over time                    | Lower; non-devs maintain matrix                            |
| Performance                       | Fast, predictable                  | Slightly slower (lookup + loop)                            |
| Debuggability                     | Easy (flat structure)              | Harder (data-driven)                                       |
| Blast radius of a bug             | One flow                           | All routed requests                                        |
| Typical failure modes             | Hard-coded approver leaves company | Empty/duplicate matrix rows, wrong filters, null approvers |
| Suits parallel/hybrid stages      | Awkward                            | Natural                                                    |
| 30-day run cap risk               | Low                                | Higher                                                     |
| Best for                          | 2–3 step, identity-derived chains  | Variable depth, attribute-driven routing                   |

> **Rule of thumb**
> Reach for **sequential** first. Graduate to **dynamic multi-level** when you would otherwise clone the same flow more than two or three times.

***

## 3. When to Use This Pattern

**Use dynamic multi-level approvals when at least one is true:**

*   Approver identity depends on attributes other than the requestor (department, region, threshold tier, item category).
*   The number of stages varies by request type.
*   Approvers change frequently and must be editable by business users.
*   The same workflow must serve many business units with different rules.
*   You need an auditable, single source of truth for "who approves what."

**Stay with simple sequential when all are true:**

*   The chain is fully derivable from the requestor's manager hierarchy.
*   The chain is short (2–3 stages) and stable.
*   One organization-wide rule applies.

***

## 4. Architecture at a Glance

    +-----------------------+        +-----------------------+
    |  Request List         |        |  Approval Matrix      |
    |  (e.g., Purchase      |        |  (Department/Amount   |
    |   Requests)           |        |   -> ordered Approvers)|
    +----------+------------+        +-----------+-----------+
               |                                 |
               | trigger: item created           | lookup by routing keys
               v                                 |
    +-----------------------+                    |
    |  Power Automate Flow  | <------------------+
    |  1. Resolve approvers                       
    |  2. Loop through levels                     
    |  3. Send approval at each level             
    |  4. Short-circuit on reject                 
    |  5. Update request + notify                 
    +-----------------------+

The flow is a **generic engine**. Adding a new business process means adding rows to the matrix, not editing the flow.

***

## 5. Prerequisites

*   A Microsoft 365 tenant with **Power Automate** and **SharePoint Online**.
*   Permission to create SharePoint lists in a target site.
*   Permission to create automated cloud flows.
*   Familiarity with Power Automate basics: triggers, actions, expressions, `Apply to each`, conditions, variables.
*   A test group of users you can route approvals to (do not test on production approvers).

> **Architect Note — Connector Governance**
> The flow runs under a **service identity** (the maker's account or a service principal). Apply least-privilege permissions on the matrix and request lists, and ensure your tenant's **Data Loss Prevention (DLP)** policies allow the connectors you intend to use. Avoid the "shared user account that everyone knows the password to" anti-pattern — it breaks audit and ownership.

***

## Step 1 — Design the Approval Matrix

The matrix is the heart of the design. Two schema styles are common; prefer the **normalized** style.

### Option A — Normalized (recommended)

One row per (RequestType, Level, Approver). Scales to any number of levels without schema changes.

| Column        | Type            | Notes                                                     |
| ------------- | --------------- | --------------------------------------------------------- |
| `RequestType` | Choice / Text   | e.g., `Laptop`, `PurchaseOrder`, `Contract`               |
| `Department`  | Choice / Lookup | Or any other routing attribute                            |
| `Level`       | Number          | 1, 2, 3, … evaluated in ascending order                   |
| `Approver`    | Person          | Single user, group, or shared mailbox                     |
| `MinAmount`   | Number          | Optional threshold (level only applies above this amount) |
| `MaxAmount`   | Number          | Optional threshold (level only applies up to this amount) |
| `Quorum`      | Choice          | `All` / `Any` — semantics within the level                |
| `Active`      | Yes/No          | Soft-disable a row without deleting it                    |

### Option B — Denormalized (quick start, hard to scale)

One row per routing key with fixed columns `Approver1`, `Approver2`, …, `ApproverN`. Easy to read, but capped at N levels and requires null-handling in the flow.

> **Architect Note — Schema Choice**
> Denormalized matrices feel intuitive but they cap your design. Every time you outgrow `ApproverN` you change the schema *and* the flow. Normalized matrices cost slightly more on day one and pay back forever.

> **Tip**
> Prefer **groups** over individuals where possible (e.g., "Finance Approvers" Microsoft 365 group). Group membership changes don't require matrix edits.

***

## Step 2 — Create the Request List

Create a SharePoint list (e.g., `Purchase Requests`) representing the business object that needs approval.

Suggested columns:

| Column          | Type              | Purpose                                                     |
| --------------- | ----------------- | ----------------------------------------------------------- |
| `Title`         | Text              | Short summary                                               |
| `RequestType`   | Choice            | Drives matrix lookup                                        |
| `Department`    | Choice / Lookup   | Drives matrix lookup                                        |
| `Amount`        | Number / Currency | Used for threshold rules                                    |
| `Justification` | Multiline text    | Approver context                                            |
| `Status`        | Choice            | `Pending`, `In Review`, `Approved`, `Rejected`, `Escalated` |
| `CurrentLevel`  | Number            | Persisted level cursor (useful for state-machine designs)   |
| `LastApprover`  | Person            | Whoever last decided                                        |
| `ApprovalLog`   | Multiline text    | Append-only audit trail                                     |
| `Requestor`     | Person            | Captured automatically                                      |

> **Why this matters**
> `Status`, `CurrentLevel`, and `ApprovalLog` are not just for the UI. They make the request **resumable**: if the flow run dies, you can recover state from the list rather than from a black-box run history.

***

## Step 3 — Build the Flow Skeleton

Create an **Automated cloud flow** with these high-level actions, in this order:

1.  **Trigger** — *When an item is created* (SharePoint, on the Request List).
2.  **Initialize variables** — at minimum:
    *   `varApprovers` — Array — the resolved approver list.
    *   `varOverallStatus` — String — defaults to `Approved`; flips to `Rejected` on first reject.
    *   `varApprovalLog` — String — appended on every decision.
3.  **Get items** — query the Approval Matrix.
4.  **Filter / Compose** — produce an ordered, deduplicated approver array.
5.  **Apply to each** — iterate the approver array.
6.  **Update item** — write final status back to the Request List.
7.  **Send notifications** — email or Teams message to requestor.

> **Architect Note — Keep the Engine Thin**
> Resist the urge to encode business rules in the flow. The flow should *interpret* the matrix, not duplicate it. If you find yourself adding `If Department = HR Then …` inside the flow, push that condition into the matrix instead.

***

## Step 4 — Resolve Approvers Dynamically

The resolver turns a request into an ordered list of approvers.

### 4.1 Query the Matrix

Use **Get items** on the Approval Matrix list with an OData filter on the routing keys captured in the trigger. Conceptually:

    Filter Query (pseudo):
      RequestType eq '<trigger.RequestType>'
      and Department eq '<trigger.Department>'
      and Active eq 1
      and (MinAmount le <trigger.Amount> or MinAmount eq null)
      and (MaxAmount ge <trigger.Amount> or MaxAmount eq null)
    Order By:
      Level asc

> **Tip — Performance**
> Always push filters into the **OData filter query** rather than filtering in the flow. Returning a few rows is dramatically faster than returning the whole list and then filtering. Set a reasonable **Top Count** to bound payload size.

### 4.2 Build a Safe Approver Array

After the query, use a `Select` action to project each row into an object you'll use later:

    Select - From: outputs of Get items
    Map:
      level     -> item()?['Level']
      approver  -> item()?['Approver']?['Email']  // or claim
      quorum    -> item()?['Quorum']?['Value']

Then **filter out nulls and duplicates** with a `Filter array` action and an expression such as:

    @and(
      not(empty(item()?['approver'])),
      greater(item()?['level'], 0)
    )

> **Why this matters**
> Empty rows are the single most common source of skipped or stuck levels. Defending against them once in the resolver is cheaper than debugging a hundred runs later.

### 4.3 Group by Level (optional but recommended)

If multiple approvers can sit at the same level (parallel-within-level), group the array by `level` so each iteration of the outer loop processes one *level*, not one *person*. A common technique is:

1.  Compose a `union` of distinct levels: `union(body('Filter_array'), body('Filter_array'))` over the `level` field.
2.  For each level, filter the approver array down to that level's members.

***

## Step 5 — Loop Through Approval Levels

Use **Apply to each** over the level list. Inside the loop:

1.  **Compose** the approvers for the current level.
2.  **Start and wait for an approval** action. Choose the type that matches the level's quorum:

| Quorum at the level                              | Approval type to use                     |
| ------------------------------------------------ | ---------------------------------------- |
| One approver only                                | *Approve/Reject – First to respond*      |
| Multiple approvers, all must approve             | *Approve/Reject – Everyone must approve* |
| Multiple approvers, any one suffices             | *Approve/Reject – First to respond*      |
| Custom outcomes (e.g., Approve/Reject/Need Info) | *Custom Responses – Wait for all / one*  |

3.  **Append to `varApprovalLog`** the level number, approver(s), outcome, comment, and timestamp.
4.  **Condition** — if the level's outcome is `Reject`:
    *   Set `varOverallStatus` to `Rejected`.
    *   **Terminate** the loop early (use a `Terminate` action with status `Succeeded`, or a boolean flag checked at the top of each iteration — Power Automate doesn't support `break`).

> **Architect Note — Short-Circuit Semantics**
> Almost every enterprise process short-circuits on first reject. Make this a deliberate design decision and document it. If your process needs *all* levels to vote even after a reject (e.g., for analytics), capture votes without halting and aggregate at the end — but expect higher latency and cost.

> **Architect Note — The 30-Day Run Cap**
> Power Automate caps a single flow run at **30 days**. Multi-level approvals with human delay easily exceed this. Two mitigations:
>
> *   **State-machine design**: persist `CurrentLevel` and outcome to the Request List after each level, then trigger the next level via a separate flow run (e.g., on item update). Each run is short and resumable.
> *   **Aggressive timeouts + escalation** so no single level holds the run open for weeks.

***

## Step 6 — Handle Approvals and Rejections

Inside each level's iteration, branch on the outcome.

### 6.1 On Approve

*   Append to `varApprovalLog`:
        Level <n> APPROVED by <approver display name> at <utcNow()> — "<comment>"
*   Continue to the next iteration.

### 6.2 On Reject

*   Append to `varApprovalLog` with `REJECTED`.
*   Set `varOverallStatus` to `Rejected`.
*   Set a boolean `varStop` to `true` (checked at the top of the next iteration to skip remaining work) **or** use a `Terminate` action.

### 6.3 On Timeout (configured in Step 8)

*   Append `TIMED OUT` to the log.
*   Either escalate (preferred) or treat as reject, per business policy.

> **Tip**
> Always store the **email/UPN** of the approver in the log, not just the display name. Names are not unique; UPNs are.

***

## Step 7 — Update the Request and Notify

After the loop:

1.  **Update item** on the Request List:
    *   `Status` ← `varOverallStatus`
    *   `LastApprover` ← last approver of record
    *   `ApprovalLog` ← `varApprovalLog`
    *   `CurrentLevel` ← final level reached
2.  **Send a notification** (email or Teams adaptive card) to the requestor with the outcome and the consolidated log.
3.  Optionally **fan out** to downstream systems (e.g., create a PO in ERP if approved).

> **Why this matters**
> The list is now your **system of record** for this request. Any reporting, KPI dashboard, or audit query can run against the list without touching flow run history (which is volatile and time-bounded).

***

## Step 8 — Add Timeouts and Escalation

Approvers go on vacation. Build for it from the start.

For each approval action:

1.  Set the action's **Timeout** (under Settings) to an ISO 8601 duration such as `P3D` (three days).
2.  Add a parallel branch configured to run on `has timed out`:
    *   Notify the approver's manager.
    *   Either **reassign** the approval to the manager, or **add** the manager as a parallel approver, or **mark as escalated** and continue.
3.  Append `ESCALATED` to `varApprovalLog`.

> **Architect Note — Escalation Policy is a Business Decision**
> Escalation rules belong in the matrix, not in the flow. Add columns like `TimeoutHours`, `EscalationPath` (e.g., `Manager`, `Group:Backup`, `User:specific`), and `OnTimeoutAction` (`Escalate`, `Reject`, `Approve`). The flow reads these per level.

> **Tip**
> If you use **delegation** (Outlook out-of-office or directory delegation), check delegation state during resolution rather than after the fact. It's cheaper to assign correctly than to escalate later.

***

## Step 9 — Test the Flow

Test with **representative data**, not just a happy path.

### 9.1 Functional Tests

*   A request with **one level** and one approver — approve, then reject.
*   A request with **multiple levels** — approve all, reject at level 1, reject at last level.
*   A request with **a level that has multiple approvers** in `All` quorum — approve all, reject one.
*   A request with **threshold rules** — amounts that should and should not trigger the high-tier level.

### 9.2 Edge Cases

*   Matrix row with a **null approver** (resolver should drop it).
*   Matrix row marked **Inactive** (resolver should ignore it).
*   An approver who is a **disabled user** in the directory.
*   An approver who **never responds** (verify timeout + escalation fire).
*   A **duplicate approver** at two different levels (verify no double-prompt within a level; cross-level duplication is policy-dependent).

### 9.3 Non-Functional Tests

*   **Concurrency** — submit several requests in parallel; verify no race conditions on `Update item`.
*   **Long-running** — start a request and let it idle 24+ hours; verify it doesn't lose state.
*   **Connector failure** — temporarily revoke a permission and verify the flow surfaces a useful error.

> **Tip — Test in Isolation**
> Use a separate `Approval Matrix (Test)` and `Request List (Test)` and a dedicated test group. Do **not** point your test flow at production data.

***

## 15. Best Practices and Common Pitfalls

### Best Practices

*   **Externalize everything that changes**: approvers, timeouts, thresholds, escalation paths.
*   **Prefer groups over individuals** in the matrix.
*   **Version the matrix**: enable list versioning and require approval to publish changes.
*   **Instrument the engine**: log every decision to an audit list; build a Power BI dashboard on cycle time, escalation rate, rejection rate by level.
*   **Use environment variables / solution-aware flows** so dev, test, and prod point at the right lists without manual edits.
*   **Document the schema** in this README so future maintainers don't have to reverse-engineer it.

### Common Pitfalls

| Pitfall                                    | Symptom                                   | Mitigation                                 |
| ------------------------------------------ | ----------------------------------------- | ------------------------------------------ |
| Hard-coded approvers in the flow           | Flow edits required to change people      | Externalize to matrix                      |
| Filtering inside the flow instead of OData | Slow runs, throttling                     | Push filters into `Get items`              |
| No timeout on approval actions             | Stalled requests held indefinitely        | Set explicit timeout + escalation          |
| 30-day run cap reached                     | Run terminates with pending approval lost | Re-architect as state machine              |
| Empty / duplicate matrix rows              | Skipped or doubled levels                 | Validate in resolver; pre-flight tests     |
| Using display names as identity            | Wrong approver assigned                   | Use UPN/email/object ID                    |
| Over-permissioned service identity         | Larger blast radius for bugs              | Apply least privilege; DLP policies        |
| No audit log on the request                | Compliance gaps                           | Append-only log column on the request      |
| Single shared flow account                 | No traceability of changes                | Use solutions and proper ownership         |
| Tests only on the happy path               | Production surprises                      | Test rejects, timeouts, nulls, concurrency |

***

## 16. Generalizing to Other Platforms

The same pattern transfers cleanly to other workflow engines. The vocabulary changes, the design does not.

| Concept              | Power Automate                          | Azure Logic Apps                   | ServiceNow                | Camunda / BPMN            |
| -------------------- | --------------------------------------- | ---------------------------------- | ------------------------- | ------------------------- |
| Trigger              | SharePoint *When item is created*       | Recurrence / event trigger         | Business rule on table    | Start event               |
| Configuration store  | SharePoint list / Dataverse             | Storage / SQL                      | sys\_user\_group + tables | DMN decision table        |
| Resolver             | `Get items` + `Select` + `Filter array` | HTTP / SQL action + composition    | Script include            | DMN evaluation            |
| Loop                 | `Apply to each`                         | `For each`                         | Flow Designer loop        | Multi-instance subprocess |
| Approval primitive   | Approvals connector                     | Approvals API / email-with-options | sysapproval\_approver     | User Task                 |
| Timeout & escalation | Action timeout + parallel branch        | Until + timeout                    | SLA + escalation rule     | Boundary timer event      |

> **Architect Note — Treat the Engine as a Product**
> Whatever the platform, success depends on three habits: (1) version the matrix, (2) automate tests for representative routing scenarios, (3) make SLA breaches visible. The platform is the cheapest part of the system; the data model and the operating discipline are the expensive parts.

***

## 17. License

You may use this guide as a starting point for your own internal documentation. Replace this section with the license that applies to your repository (for example, `MIT`, `Apache-2.0`, or `CC-BY-4.0`).

***

*End of guide.*
