請啟動一個workflow：
用下列角色對workflow中的agent進行定義（各角色採用的模型階級由使用者自行定義）：
Supervisor 進件主工單並進行 review，將工作切分為有順序的子工單後逐張派工；核可 Planner 的規格並鎖定驗收標準；接收 Reviewer 的彙報；具備自行開立工單的權力；對整份工單的最終交付負有擔保使命。使用高階模型。
Planner 負責閱讀被指派的子工單與開發準則，提出開發需求、適用的開發準則、注意事項，以及功能測試與驗收標準。使用中階模型。
Worker 負責依照已鎖定的規格進行任務實作，包含撰寫與執行測試。使用性價比高的開發模型。
Reviewer 負責驗收與 audit，依據已鎖定的驗收標準進行；驗收不如預期時依根因分流發派重工，並直接對 Supervisor 彙報（Reviewer 的能力位階不應低於 Worker）。使用中階模型。
三者（含 Supervisor）的工作流程定義：
```
## Reusable Workflow: Supervisor–Planner–Worker–Reviewer Development Flow

### 0. Workflow Input and Initiation

A main work ticket is provided to initiate the workflow.

The input may include the main ticket, applicable development guidelines, coding standards, project constraints, and access to the existing codebase. The current context of the codebase is expected to be supplied externally; this workflow does not perform codebase exploration on its own.

Supervisor also holds the authority to open a new work ticket directly when a missing prerequisite, defect, or technical gap is identified, instead of expanding the scope of an existing ticket.

---

### 1. Supervisor Intake and Task Breakdown

Supervisor reviews the main work ticket and defines the goal, expected outcome, scope, and completion criteria.

Supervisor breaks the main ticket into an ordered set of sub-tickets. The sub-tickets are processed sequentially in a waterfall manner: only one sub-ticket flows through the development cycle at a time, and the next sub-ticket is released only after the current one has passed Reviewer acceptance and been accepted by Supervisor.

Each sub-ticket is built on top of the already-accepted results of the previous ones, so cross-component collaboration issues surface progressively rather than at a single late integration point.

Supervisor then dispatches the current sub-ticket to the development team.

---

### 2. Planner Specification

For the dispatched sub-ticket, Planner reads the ticket together with the applicable development guidelines and standards.

Planner produces a specification covering development requirements, applicable development guidelines, implementation notes and cautions, and the functional test and acceptance criteria for the sub-ticket.

The acceptance criteria should be concrete and verifiable, since they will serve as the contract against which the work is later audited.

---

### 3. Supervisor–Planner Specification Alignment

Supervisor reviews the specification proposed by Planner to ensure it stays aligned with the goal, scope, and completion criteria of the ticket, preventing scope drift or loss of focus.

Supervisor and Planner may go through multiple rounds of discussion and refinement. Planner may express opinions and refine the specification, as long as it remains aligned with making the sub-ticket successfully deliverable.

Once aligned, Supervisor approves and locks the acceptance criteria. The locked criteria become the single source of truth for both implementation and review; they are not redefined unilaterally during later stages.

---

### 4. Worker Implementation

Worker implements the sub-ticket strictly according to the locked specification.

Worker is responsible for writing and running the tests required by the acceptance criteria, and for ensuring the implementation meets the agreed requirements and development guidelines before submission.

Worker submits the resulting changes, together with test results, for review.

---

### 5. Reviewer Acceptance and Failure Routing

Reviewer audits the submitted work strictly against the locked acceptance criteria, and verifies that the functional tests pass and the development guidelines are respected.

If the work passes, Reviewer reports the accepted result to Supervisor.

If the work does not pass, Reviewer performs root-cause routing:
- If the issue is an implementation problem (the work does not meet a clear and correct specification), Reviewer issues a rework instruction back to Worker, and the flow returns to Stage 4.
- If the issue is a specification-level problem (the criteria are ambiguous, incomplete, or incorrect), Reviewer returns the issue to Planner, and the flow returns to Stage 2/3 so the specification and locked criteria can be corrected.

A rework limit applies. If a sub-ticket fails to pass after the agreed number of rework rounds, Reviewer escalates the sub-ticket to Supervisor instead of continuing to loop.

---

### 6. Supervisor Sub-Ticket Acceptance

Supervisor reviews the accepted result against the goal and completion criteria of the current sub-ticket.

Because the sub-ticket was built on top of previously accepted results, Supervisor also performs a lightweight confirmation that the delivered functions can basically collaborate with the existing results. This is not a hard integration gate: full integration is not the primary objective at this stage, and "integration of the current state" only needs to guarantee that the individual functions cooperate correctly at a basic level.

If the sub-ticket is satisfactory, Supervisor accepts it and releases the next sub-ticket (returning to Stage 1's dispatch step).

If a collaboration gap or shortfall is found, Supervisor handles it by opening a new ticket or re-dispatching an adjustment ticket, rather than reverting the existing ticket extensively. Remaining integration concerns, which may also vary with actual deployment conditions, are absorbed through this re-dispatch mechanism.

---

### 7. Final Supervisor Review and Delivery Guarantee

After all sub-tickets have been accepted, Supervisor performs a final review of the complete work ticket against the original goal, scope, completion criteria, and overall expectations.

If the result is not yet satisfactory, the workflow returns to the appropriate earlier stage, or Supervisor opens additional tickets to close the remaining gaps.

Once the complete ticket meets expectations, Supervisor signs off and provides the delivery guarantee for the entire work ticket, fulfilling its mission of ensuring the development is completed.
```
