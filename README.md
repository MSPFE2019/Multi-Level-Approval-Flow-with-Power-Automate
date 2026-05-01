```markdown
# Multi‑Level Approval Flow with Power Automate (Approval‑Only Guide)

A step‑by‑step guide to building a **dynamic, configuration‑driven, multi‑level approval engine** using Microsoft Power Automate.  
This version **focuses only on the approval flow itself** (no request intake design), making it reusable across scenarios where an approval is triggered by *any* upstream system.

> **Audience**
> - **Beginners / citizen developers**: clear steps you can follow end‑to‑end  
> - **Architects**: callouts explaining design trade‑offs, governance, and scale considerations

---

## Table of Contents
1. Overview  
2. Sequential vs Multi‑Level Approvals  
3. When to Use This Pattern  
4. Architecture at a Glance  
5. Prerequisites  
6. Step 1 – Design the Approval Matrix  
7. Step 2 – Create the Approval Flow  
8. Step 3 – Resolve Approvers Dynamically  
9. Step 4 – Loop Through Approval Levels  
10. Step 5 – Handle Approvals and Rejections  
11. Step 6 – Timeouts and Escalation  
12. Step 7 – Completion and Outputs  
13. Testing the Approval Engine  
14. Best Practices and Common Pitfalls  
15. Generalizing Beyond Power Automate  
16. License

---

## 1. Overview

A **multi‑level approval flow** routes an approval through multiple ordered stages (levels), where each level can contain one or more approvers.

This guide describes an **approval engine**, not a business request:

- The flow can be triggered by **any source** (SharePoint, Dataverse, Forms, API, another flow).
- All approval routing rules live in a **configuration list (approval matrix)**.
- The same flow supports different approval chains without modification.

> **Why this matters**  
> Hard‑coding approvers inside a flow does not scale. Externalizing them into a matrix allows governance, reuse, and change without redeployment.

---

## 2. Sequential vs Multi‑Level Approvals

| Aspect | Sequential Approval | Dynamic Multi‑Level Approval |
|---|---|---|
| Approvers defined | Inside the flow | In a configuration list |
| Number of levels | Fixed | Variable |
| Change management | Flow edit required | Data change only |
| Reusability | Low | High |
| Governance | Decentralized | Centralized |
| Best fit | Simple manager chains | Enterprise approval scenarios |

> **Rule of thumb**  
> If approval logic varies by department, amount, region, or policy → use **multi‑level**.

---

## 3. When to Use This Pattern

Use a dynamic multi‑level approval engine when:

- Approval stages vary by **business rule**.
- Approvers change frequently.
- Non‑developers must manage approvers.
- You need **auditability and consistency** across many processes.

---

## 4. Architecture at a Glance

```

Trigger (any source)
|
v
+-----------------------+

| Approval Flow Engine      |
| ------------------------- |
| 1. Load approval rules    |
| 2. Build approver list    |
| 3. Loop by level          |
| 4. Send approvals         |
| 5. Short‑circuit on       |
| rejection                 |
| 6. Return outcome         |
| +-----------------------+ |

        |
        v

Outcome + Audit Log

```

The flow is **generic**. The approval matrix drives behavior.

---

## 5. Prerequisites

- Power Automate access
- SharePoint Online (or Dataverse) for configuration storage
- Permission to create flows and lists
- Basic Power Automate knowledge (variables, conditions, loops)

> **Architect Note – Security**  
> The flow identity must have **read access** to the approval matrix and **send approval permissions** only. Apply least privilege.

---

## 6. Step 1 – Design the Approval Matrix

The approval matrix defines *who approves at each level*.

### Recommended (Normalized) Schema

| Column | Type | Description |
|---|---|---|
| `ApprovalType` | Choice / Text | Logical approval scenario |
| `Level` | Number | 1, 2, 3… evaluated in order |
| `Approver` | Person or Group | User or group |
| `Quorum` | Choice | `All` or `Any` |
| `TimeoutHours` | Number | Optional |
| `EscalationTarget` | Person / Group | Optional |
| `Active` | Yes/No | Enables/disables rule |

> **Architect Note – Why Normalized**
> Avoid `Approver1`, `Approver2`, … columns. Normalized rows allow unlimited levels without schema or flow changes.

---

## 7. Step 2 – Create the Approval Flow

Create an **Automated cloud flow**.

### Core Flow Structure

1. **Trigger** – Any trigger that indicates approval should start  
2. **Initialize variables**
   - `varOverallStatus` (String, default: `Approved`)
   - `varApprovalLog` (String)
   - `varStopProcessing` (Boolean, default: `false`)
3. **Get items** – Approval Matrix
4. **Resolve approvers** – Build ordered list
5. **Apply to each (Level loop)**
6. **Return outcome**

> **Why this matters**  
> The flow does not care *what* is being approved—only *how* approvals are processed.

---

## 8. Step 3 – Resolve Approvers Dynamically

Query the approval matrix using the approval context (for example, `ApprovalType`).

**Conceptual filter:**
```

ApprovalType eq '\<trigger.ApprovalType>'
and Active eq true

```

Sort by:
```

Level asc

````

Transform results into an array like:
```json
{
  "level": 1,
  "approver": "user@contoso.com",
  "quorum": "All",
  "timeout": 48,
  "escalation": "manager@contoso.com"
}
````

Remove nulls and duplicates.

> **Architect Note – Data First**
> Most approval bugs are data issues. Validate and sanitize here once instead of compensating later.

***

## 9. Step 4 – Loop Through Approval Levels

Use **Apply to each** over the distinct levels.

At the top of the loop:

*   If `varStopProcessing` is `true` → skip remaining levels.

Inside each level:

1.  Identify approvers for that level
2.  Select approval type based on quorum
3.  Send approval request

| Quorum | Approval Action                        |
| ------ | -------------------------------------- |
| All    | Approve/Reject – Everyone must approve |
| Any    | Approve/Reject – First to respond      |

***

## 10. Step 5 – Handle Approvals and Rejections

### On Approval

*   Append to log:
        Level X approved by <approver> at <timestamp>
*   Continue to next level

### On Rejection

*   Append to log
*   Set `varOverallStatus = Rejected`
*   Set `varStopProcessing = true`

> **Architect Note – Short‑Circuiting**
> Most enterprise processes stop on first rejection. Make this explicit and documented.

***

## 11. Step 6 – Timeouts and Escalation

Configure each approval action:

*   **Timeout** (ISO 8601, e.g. `PT48H`)
*   Parallel branch on timeout:
    *   Notify escalation target
    *   Reassign or resend approval
    *   Log escalation

> **Architect Note – 30‑Day Limit**
> Power Automate runs expire after 30 days. For long approvals:
>
> *   Persist state externally
> *   Resume approvals in new runs (state‑machine pattern)

***

## 12. Step 7 – Completion and Outputs

After all levels (or rejection):

*   Output final status (`Approved` / `Rejected`)
*   Output approval log
*   Optionally return results to caller (HTTP response, child flow output)

> **Why this matters**  
> Treat the approval flow as a **service** that returns a decision and audit trail.

***

## 13. Testing the Approval Engine

Test scenarios:

*   Single‑level approval
*   Multi‑level approval
*   Rejection at level 1 and final level
*   Timeout + escalation
*   Disabled matrix rows
*   Duplicate approvers

***

## 14. Best Practices and Common Pitfalls

### Best Practices

*   Externalize all routing logic
*   Use groups where possible
*   Enable versioning on the matrix
*   Log every decision
*   Separate dev/test/prod matrices

### Common Pitfalls

| Pitfall                  | Result              |
| ------------------------ | ------------------- |
| Hard‑coded approvers     | Flow redeployments  |
| No timeouts              | Stuck approvals     |
| Flat schema              | Limited scalability |
| No audit log             | Compliance gaps     |
| Long‑running single flow | 30‑day failures     |

***

## 15. Generalizing Beyond Power Automate

This pattern applies to:

*   Azure Logic Apps
*   ServiceNow Flow Designer
*   Camunda / BPMN engines
*   Custom workflow services

Only the **orchestration syntax** changes—the **design does not**.

***

## 16. License

Replace this section with your repository license (MIT, Apache‑2.0, etc.).

***

*End of approval‑only guide.*

```
```
