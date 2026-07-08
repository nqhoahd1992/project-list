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
        │  Project_Notify.Run("SubmittedToManager", <that project's ProjectManager email>)
        ▼
Pending Manager ── ApprovalsScreen, StepNumber 1, actor = the CR's/project's ProjectManager only ──
        │  Approve → log(1, Approved) → CR = "Pending Executive"
        │            → Notify("ManagerApprovedToExecutive", <that project's ProjectOwner email>)
        │  Reject  → Remark ⚠ → log(1, Rejected, Remark) → CR = "Rejected"
        │            → Notify("FinalRejected", requester)
        ▼
Pending Executive ── ApprovalsScreen, StepNumber 2, actor = the CR's/project's ProjectOwner only ──
        │  Reject  → Remark ⚠ → log(2, Rejected, Remark) → CR = "Rejected"
        │            → Notify("FinalRejected", requester)
        │  Approve → APPLY to Project_List (Switch on RequestType, guarded Patch)
        │            success → log(2, Approved) → CR = "Approved"
        │                      → Notify("FinalApproved", requester)
        │            failure → Notify error, CR STAYS "Pending Executive" (retryable)
        ▼
Approved / Rejected (terminal)
```

Anyone with a tenant email can submit (`Requester` above is not a `Project_User` role — Managers and Executives can submit requests too).

**Authorization is per-project, not per-role.** Having `Role = "Manager"` in `Project_User` is necessary but not sufficient to act on a given CR — the user must also be that specific CR's/project's `ProjectManager` (Step 1) or `ProjectOwner` (Step 2). For a Create CR, "that project" means the values proposed on the CR itself (`ProjectManager`/`ProjectOwner`, since the project doesn't exist yet); for Update/Delete, it means the live target row in `Project_List` (via `TargetItemID`). There is no substitute-approver role — a Manager who isn't the assigned `ProjectManager` for a project never sees that project's CR in their queue at all. See `CLAUDE.md` §Role-based visibility for the exact formula and the 3 places it's duplicated.

### Apply logic per RequestType (Executive approve)

| RequestType | Apply |
|---|---|
| `Create` | Patch new `Project_List` row from CR values (`ApprovalStatus: Approved`, `ApprovedBy` = executive) → 2nd Patch sets `ProjectID = "PROJ-" & <MarketCode> & "-" & <DeptCode> & "-" & <Year> & "-" & Text(newRow.ID, "000")` (e.g. `PROJ-AU-GN-2026-002`; MarketCode = `Market.Value`, already `AU`/`MY`/`SG`/`VN`) |
| `Update` | `LookUp` target by `TargetItemID` (guard: still exists) → Patch only `ProjectDescription` + `EndDate` |
| `Delete` | Patch target `ProjectStatus = {Value: "Deleted"}` (soft delete — row is hidden from All Projects by default) |

### Guards enforced in the app

- Only one pending CR per target project: Update/Delete screens disable submit when a CR with `TargetItemID = project.ID` is still pending.
- A root project (level 0) with non-deleted children cannot be deleted.
- Update form only exposes `Project Description` and `End Date`; everything else read-only.
- `ActualCost` is never editable and never included in any CR.

## 3. `Project_Notify` flow — build spec (manual, Power Automate portal)

- **Trigger**: Power Apps (V2). Add 9 inputs in this exact order, declaring types explicitly (do NOT let a Blank in the first test run infer the type), and rename each from its Power Automate default:
  `notificationType` (Text, default `text`), `recipientEmails` (Text, default `text_1`), `requestTitle` (Text, default `text_2`), `requestId` (Number, default `number`), `requestType` (Text, default `text_3`), `projectName` (Text, default `text_4`), `requesterName` (Text, default `text_5`), `actionByName` (Text, default `text_6`), `remark` (Text, default `text_7`) — meanings in `sharepoint-schema.md` §Flows.
- **Actions**:
  1. `Initialize variable` — `varSubject` (String), `varBody` (String).
  2. `Switch` on `notificationType` with 4 cases setting `varSubject` / `varBody` (templates below). **Default branch → Terminate (Failed, "Unknown notificationType")** so a typo in the app surfaces as a flow failure instead of silent no-mail.
  3. `Send an email (V2)` — To: `recipientEmails` (semicolon-separated works natively), Subject: `varSubject`, Body: `varBody` (HTML on).
- **Connection**: `app.admin@maxbiocare.com` shared mailbox connection; pin it under *Run only users* so every app user sends through the same connection.
- App deep link used in bodies (fill after the app is published):
  `https://apps.powerapps.com/play/e/<ENV_ID>/a/<APP_ID>?tenantId=<TENANT_ID>`

### Email templates (English)

**Case `SubmittedToManager`** — To: all active Managers

- Subject: `[Project List] Approval needed: @{triggerBody()['requestTitle']}`
- Body:
  ```
  A new project change request is waiting for your review.

  Request:      @{triggerBody()['requestTitle']} (#@{triggerBody()['requestId']})
  Type:         @{triggerBody()['requestType']}
  Project:      @{triggerBody()['projectName']}
  Requested by: @{triggerBody()['requesterName']}

  Please open the Project List app → Approvals to review it.
  <app deep link>
  ```

**Case `ManagerApprovedToExecutive`** — To: all active Executives

- Subject: `[Project List] Executive approval needed: @{triggerBody()['requestTitle']}`
- Body: same layout, plus `Manager approved by: @{triggerBody()['actionByName']}`.

**Case `FinalApproved`** — To: requester

- Subject: `[Project List] Approved: @{triggerBody()['requestTitle']}`
- Body:
  ```
  Your request has been fully approved and applied.

  Request:     @{triggerBody()['requestTitle']} (#@{triggerBody()['requestId']})
  Type:        @{triggerBody()['requestType']}
  Project:     @{triggerBody()['projectName']}
  Approved by: @{triggerBody()['actionByName']}
  ```

**Case `FinalRejected`** — To: requester

- Subject: `[Project List] Rejected: @{triggerBody()['requestTitle']}`
- Body:
  ```
  Your request has been rejected.

  Request:     @{triggerBody()['requestTitle']} (#@{triggerBody()['requestId']})
  Type:        @{triggerBody()['requestType']}
  Project:     @{triggerBody()['projectName']}
  Rejected by: @{triggerBody()['actionByName']}
  Remark:      @{triggerBody()['remark']}

  You may submit a new request after addressing the remark.
  ```

### Recipient resolution (app side, Power Fx)

Per-project, not a role broadcast — see `CLAUDE.md` §Role-based visibility for the full authorization model.

```
// SubmittedToManager: the CR's/project's specific ProjectManager
Coalesce(
    LookUp('Employee List', ID = <ProjectManager.Id>).Email,
    "app.admin@maxbiocare.com"
)
// ManagerApprovedToExecutive: the CR's/project's specific ProjectOwner, same Coalesce fallback
// FinalApproved / FinalRejected (requester): gSelectedCR.RequesterEmail
```

`<ProjectManager.Id>` / `<ProjectOwner.Id>` resolve differently depending on where the call happens:
- Submitting a Create request (`CreateProjectScreen`): the form's own `cmbManager.Selected.ID` (the CR doesn't exist yet).
- Submitting an Update/Delete request (`Update`/`DeleteProjectScreen`): the target project's `gSelectedProject.ProjectManager.Id`.
- Manager approving (`ApprovalsScreen`, Step 1 → notifies the Executive): `If(gSelectedCR.RequestType.Value = "Create", gSelectedCR.ProjectOwner.Id, gLiveTarget.ProjectOwner.Id)`.

## 4. FUTURE — `Project_SyncActualCost` (placeholder, do not build yet)

Goal: `Project_List.ActualCost` accumulates actual spend from the procurement SharePoint data automatically; users never type it.

- **Blocked on join key**: procurement lists (`Procurement_Requests`, `Procurement_InvoiceData`) have no project reference today. Preferred design: add a `ProjectID` (Text) column to `Procurement_Requests`, exposed as an optional picker in the procurement app.
- Sketch: recurrence or SharePoint-trigger flow → for each distinct `ProjectID`, sum `TotalAmount` from `Procurement_InvoiceData` → Patch `Project_List.ActualCost` where `ProjectID` matches → on change, call `Project_Notify` with a new case `ActualCostUpdated` (To: Project Manager) — add the 10th arg / new case then.
- Until then `ActualCost` stays blank/manual-admin-only and read-only in the app.
