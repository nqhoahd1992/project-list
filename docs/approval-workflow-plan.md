# Approval Workflow & Notification Flow — Design Spec

How the 2-level approval (Manager → Executive) works, and the spec for the `Project_Notify` Power Automate flow. Data model in `docs/sharepoint-schema.md`.

## 1. Principles

- **No form ever writes `Project_List`.** Every Create / Update / Delete intent is stored as a row in `Project_ChangeRequests` (CR). The master list changes only inside the Executive-approve handler on `ApprovalsScreen`.
- **Apply happens via Power Apps `Patch`**, not a flow — synchronous error feedback, consistent with the procurement app. Flows are used only for email.
- **Apply BEFORE flipping CR status.** On Executive approve the master Patch runs first; only if it succeeds does the CR become `Approved`. A failed apply leaves the CR at `Pending Executive` so the Executive can retry. (This ordering differs from procurement on purpose — a CR marked Approved but never applied would be unrecoverable without admin surgery.)
- **Reject requires a Remark** at either step — enforced in the app before the log Patch.

## 2. State machine

```
Requester submits (Create / Update / Delete form)
        │  Patch Project_ChangeRequests → ApprovalStatus = "Pending Manager"
        │  Project_Notify.Run("SubmittedToManager", <active Manager emails>)
        ▼
Pending Manager ── ApprovalsScreen, role Manager, StepNumber 1 ──
        │  Approve → log(1, Approved) → CR = "Pending Executive"
        │            → Notify("ManagerApprovedToExecutive", <active Executive emails>)
        │  Reject  → Remark ⚠ → log(1, Rejected, Remark) → CR = "Rejected"
        │            → Notify("FinalRejected", requester)
        ▼
Pending Executive ── ApprovalsScreen, role Executive, StepNumber 2 ──
        │  Reject  → Remark ⚠ → log(2, Rejected, Remark) → CR = "Rejected"
        │            → Notify("FinalRejected", requester)
        │  Approve → APPLY to Project_List (Switch on RequestType, guarded Patch)
        │            success → log(2, Approved) → CR = "Approved"
        │                      → Notify("FinalApproved", requester)
        │            failure → Notify error, CR STAYS "Pending Executive" (retryable)
        ▼
Approved / Rejected (terminal)
```

Anyone with a tenant email can submit (`Requester` above is not a `Project_User` role — Managers and Executives can submit requests too). There is no substitute-approver role: each step can only be acted on by its own role.

### Apply logic per RequestType (Executive approve)

| RequestType | Apply |
|---|---|
| `Create` | Patch new `Project_List` row from CR values (`ApprovalStatus: Approved`, `ApprovedBy` = executive) → 2nd Patch sets `ProjectID = "PROJ-" & <DeptCode> & "-" & <Year> & "-" & Text(newRow.ID, "000")` |
| `Update` | `LookUp` target by `TargetItemID` (guard: still exists) → Patch only `ProjectDescription` + `EndDate` |
| `Delete` | Patch target `ProjectStatus = {Value: "Deleted"}` (soft delete — row is hidden from All Projects by default) |

### Guards enforced in the app

- Only one pending CR per target project: Update/Delete screens disable submit when a CR with `TargetItemID = project.ID` is still pending.
- A root project (level 0) with non-deleted children cannot be deleted.
- Update form only exposes `Project Description` and `End Date`; everything else read-only.
- `ActualCost` is never editable and never included in any CR.

## 3. `Project_Notify` flow — build spec (manual, Power Automate portal)

- **Trigger**: Power Apps (V2). Add 9 inputs in this exact order, declaring types explicitly (do NOT let a Blank in the first test run infer the type):
  `text` (Text), `text_1` (Text), `text_2` (Text), `number` (Number), `text_3` (Text), `text_4` (Text), `text_5` (Text), `text_6` (Text), `text_7` (Text) — meanings in `sharepoint-schema.md` §Flows.
- **Actions**:
  1. `Initialize variable` — `varSubject` (String), `varBody` (String).
  2. `Switch` on `text` (notificationType) with 4 cases setting `varSubject` / `varBody` (templates below). **Default branch → Terminate (Failed, "Unknown notificationType")** so a typo in the app surfaces as a flow failure instead of silent no-mail.
  3. `Send an email (V2)` — To: `text_1` (semicolon-separated works natively), Subject: `varSubject`, Body: `varBody` (HTML on).
- **Connection**: `app.admin@maxbiocare.com` shared mailbox connection; pin it under *Run only users* so every app user sends through the same connection.
- App deep link used in bodies (fill after the app is published):
  `https://apps.powerapps.com/play/e/<ENV_ID>/a/<APP_ID>?tenantId=<TENANT_ID>`

### Email templates (English)

**Case `SubmittedToManager`** — To: all active Managers

- Subject: `[Project List] Approval needed: @{triggerBody()['text_2']}`
- Body:
  ```
  A new project change request is waiting for your review.

  Request:      @{triggerBody()['text_2']} (#@{triggerBody()['number']})
  Type:         @{triggerBody()['text_3']}
  Project:      @{triggerBody()['text_4']}
  Requested by: @{triggerBody()['text_5']}

  Please open the Project List app → Approvals to review it.
  <app deep link>
  ```

**Case `ManagerApprovedToExecutive`** — To: all active Executives

- Subject: `[Project List] Executive approval needed: @{triggerBody()['text_2']}`
- Body: same layout, plus `Manager approved by: @{triggerBody()['text_6']}`.

**Case `FinalApproved`** — To: requester

- Subject: `[Project List] Approved: @{triggerBody()['text_2']}`
- Body:
  ```
  Your request has been fully approved and applied.

  Request:     @{triggerBody()['text_2']} (#@{triggerBody()['number']})
  Type:        @{triggerBody()['text_3']}
  Project:     @{triggerBody()['text_4']}
  Approved by: @{triggerBody()['text_6']}
  ```

**Case `FinalRejected`** — To: requester

- Subject: `[Project List] Rejected: @{triggerBody()['text_2']}`
- Body:
  ```
  Your request has been rejected.

  Request:     @{triggerBody()['text_2']} (#@{triggerBody()['number']})
  Type:        @{triggerBody()['text_3']}
  Project:     @{triggerBody()['text_4']}
  Rejected by: @{triggerBody()['text_6']}
  Remark:      @{triggerBody()['text_7']}

  You may submit a new request after addressing the remark.
  ```

### Recipient resolution (app side, Power Fx)

```
// Managers (submit): all active Manager role emails, ";"-joined
Concat(
    Filter(Project_User, Role.Value = "Manager" && IsActive),
    LookUp('Employee List', ID = EmployeeID.Id).email, ";"
)
// Executives (manager approved): same with Role.Value = "Executive"
// Requester (final result): gSelectedCR.RequesterEmail
```

## 4. FUTURE — `Project_SyncActualCost` (placeholder, do not build yet)

Goal: `Project_List.ActualCost` accumulates actual spend from the procurement SharePoint data automatically; users never type it.

- **Blocked on join key**: procurement lists (`Procurement_Requests`, `Procurement_InvoiceData`) have no project reference today. Preferred design: add a `ProjectID` (Text) column to `Procurement_Requests`, exposed as an optional picker in the procurement app.
- Sketch: recurrence or SharePoint-trigger flow → for each distinct `ProjectID`, sum `TotalAmount` from `Procurement_InvoiceData` → Patch `Project_List.ActualCost` where `ProjectID` matches → on change, call `Project_Notify` with a new case `ActualCostUpdated` (To: Project Manager) — add the 10th arg / new case then.
- Until then `ActualCost` stays blank/manual-admin-only and read-only in the app.
