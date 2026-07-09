# Approval Workflow & Notification Flow — Design Spec

How the 2-level approval (Manager → Executive) works, and the spec for the `Project_Notify` Power Automate flow. Data model in `docs/sharepoint-schema.md`.

## 1. Principles

- **No form ever writes `Project_List`.** Every Create / Update / Delete intent is stored as a row in `Project_ChangeRequests` (CR). The master list changes only inside the Executive-approve handler on `ApprovalsScreen`.
- **Apply happens via Power Apps `Patch`**, not a flow — synchronous error feedback, consistent with the procurement app. Flows are used only for email.
- **Apply BEFORE flipping CR status.** On Executive approve the master Patch runs first; only if it succeeds does the CR become `Approved`. A failed apply leaves the CR at `Pending Executive` so the Executive can retry. (This ordering differs from procurement on purpose — a CR marked Approved but never applied would be unrecoverable without admin surgery.)
- **Reject requires a Remark** at either step — enforced in the app before the log Patch.
- **`ProjectManager` is never chosen by the requester.** On a Create request the requester instead picks a `ProjectSponsor` (owns the project's outcome, not a supervisory role); right after submit, that Sponsor is the one who assigns the actual `ProjectManager` (who then runs Step 1). Update/Delete requests skip this — they target an existing project whose `ProjectManager` was already assigned at its own Create time.

## 2. State machine

```
Requester submits (Create / Update / Delete form)
        │  Patch Project_ChangeRequests
        │
        ├─ Create ──▶ ApprovalStatus = "Pending Sponsor"
        │             Project_Notify.Run("SubmittedToSponsor", <picked ProjectSponsor's email>)
        │             ▼
        │             Pending Sponsor ── ApprovalsScreen "Assign Manager" tab, actor = that CR's ProjectSponsor only ──
        │             │  Sponsor picks a ProjectManager, submits
        │             │      → ApprovalStatus = "Pending Manager"
        │             │      → Notify("SubmittedToManager", newly-assigned Manager's email)
        │             ▼
        └─ Update / Delete ──▶ ApprovalStatus = "Pending Manager" directly (target project already has a ProjectManager)
                      Project_Notify.Run("SubmittedToManager", <that project's existing ProjectManager email>)
        ▼
Pending Manager ── ApprovalsScreen, StepNumber 1, actor = the CR's/project's ProjectManager only ──
        │  Approve → log(1, Approved) → CR = "Pending Executive"
        │            → Notify("ManagerApprovedToExecutive", <that project's ProjectOwner email>)
        │  Reject  → Remark ⚠ → log(1, Rejected, Remark) → CR = "Rejected"
        │            → Notify("FinalRejected", requester)
        ▼
Pending Executive ── ApprovalsScreen, StepNumber 2, actor = the CR's/project's ProjectOwner only ──
        │  Reject  → Remark ⚠ → log(2, Rejected, Remark) → CR = "Rejected"
        │            → Notify("FinalRejected", requester); Notify("FinalRejectedManagerCopy", that project's ProjectManager)
        │  Approve → APPLY to Project_List (Switch on RequestType, guarded Patch)
        │            success → log(2, Approved) → CR = "Approved"
        │                      → Notify("FinalApproved", requester); Notify("FinalApprovedManagerCopy", that project's ProjectManager)
        │            failure → Notify error, CR STAYS "Pending Executive" (retryable)
        ▼
Approved / Rejected (terminal)
```

Anyone with a tenant email can submit (`Requester` above is not a `Project_User` role — Managers and Executives can submit requests too).

### 2a. Notification matrix

Who gets emailed for each decision — **the Manager is only ever cc'd at the Executive step**, not at their own step (they already know they just acted):

| Step | Actor | Decision | Notified |
|---|---|---|---|
| Submit (Create) | Requester | — | Sponsor (`SubmittedToSponsor`) |
| Submit (Update/Delete) | Requester | — | Manager (`SubmittedToManager`) |
| 0 (Sponsor, Create only) | ProjectSponsor | Assign Manager | Manager (`SubmittedToManager`) |
| 1 (Manager) | ProjectManager | Approve | Executive (`ManagerApprovedToExecutive`) |
| 1 (Manager) | ProjectManager | Reject ⚠ Remark | Requester (`FinalRejected`) |
| 2 (Executive) | ProjectOwner | Approve | Requester (`FinalApproved`) **+ Manager (`FinalApprovedManagerCopy`)** |
| 2 (Executive) | ProjectOwner | Reject ⚠ Remark | Requester (`FinalRejected`) **+ Manager (`FinalRejectedManagerCopy`)** |

`SubmittedToManager` is reused for two distinct triggers (Update/Delete submit vs. Sponsor's assignment on a Create CR) — same template, same single-recipient shape, just a different code path resolving the recipient.

The bolded **+ Manager** legs are the delta from the original spec. Each recipient gets its own `notificationType`/template/`Project_Notify.Run` call rather than being folded into one `;`-joined `recipientEmails` — the Requester and Manager copies read differently ("your request" vs. "a project you manage"), and this keeps the existing "one `notificationType` = one recipient + one template" contract from `SubmittedToManager`/`ManagerApprovedToExecutive` intact (9-arg trigger schema unchanged; no new args, just 2 new `Switch` cases). At Step 2 the app therefore chains two `Project_Notify.Run(...)` calls with `;` (same statement-chaining pattern already used elsewhere in `ApprovalsScreen.pa.yaml`, e.g. `Project_Notify.Run(...); Notify(...); Navigate(...)`). At Step 1, `FinalRejected` keeps its original single-call, requester-only form since there is no "Manager" leg to add at that step.

**Authorization is per-project, not per-role.** For Step 1/Step 2, having `Role = "Manager"`/`"Executive"` in `Project_User` is necessary but not sufficient to act on a given CR — the user must also be that specific CR's/project's `ProjectManager` (Step 1) or `ProjectOwner` (Step 2). Step 0 (Sponsor, Create only) works differently: `ProjectSponsor` carries **no** `Project_User`-role prerequisite at all — being that CR's `ProjectSponsor` is both necessary and sufficient, since anyone in `Employee List` can be picked as one (see `CLAUDE.md` §Role-based visibility). For a Create CR, "that project" means the values proposed on the CR itself (`ProjectSponsor`/`ProjectManager`/`ProjectOwner`, since the project doesn't exist yet); for Update/Delete, it means the live target row in `Project_List` (via `TargetItemID`) — Update/Delete CRs never carry a `ProjectSponsor` value, since Step 0 only ever happens once, at the project's original Create. There is no substitute-approver role — a Manager who isn't the assigned `ProjectManager` for a project never sees that project's CR in their queue at all. See `CLAUDE.md` §Role-based visibility for the exact formula and the places it's duplicated.

### Apply logic per RequestType (Executive approve)

| RequestType | Apply |
|---|---|
| `Create` | Patch new `Project_List` row from CR values (`ApprovalStatus: Approved`, `ApprovedBy` = executive) → 2nd Patch sets `ProjectID = "PROJ-" & <MarketCode> & "-" & <DeptCode> & "-" & <Year> & "-" & Text(newRow.ID, "000")` (e.g. `PROJ-AU-GN-2026-002`; MarketCode = `Market.Value`, already `AU`/`MY`/`SG`/`VN`) |
| `Update` | `LookUp` target by `TargetItemID` (guard: still exists) → Patch only `ProjectDescription` + `Deliverables` + `EndDate` |
| `Delete` | Patch target `ProjectStatus = {Value: "Deleted"}` (soft delete — row is hidden from All Projects by default) |

### Guards enforced in the app

- Only one pending CR per target project: Update/Delete screens disable submit when a CR with `TargetItemID = project.ID` is still pending.
- A root project (level 0) with non-deleted children cannot be deleted.
- Update form only exposes `Project Description`, `Deliverables`, and `End Date`; everything else read-only.
- `ActualCost` is never editable and never included in any CR.

## 3. `Project_Notify` flow — build spec (manual, Power Automate portal)

- **Trigger**: Power Apps (V2). Add 9 inputs in this exact order, declaring types explicitly (do NOT let a Blank in the first test run infer the type), and rename each from its Power Automate default:
  `notificationType` (Text, default `text`), `recipientEmails` (Text, default `text_1`), `requestTitle` (Text, default `text_2`), `requestId` (Number, default `number`), `requestType` (Text, default `text_3`), `projectName` (Text, default `text_4`), `requesterName` (Text, default `text_5`), `actionByName` (Text, default `text_6`), `remark` (Text, default `text_7`) — meanings in `sharepoint-schema.md` §Flows.
  **Renaming only relabels the input in the designer/Power Apps intellisense — it does NOT change the underlying trigger-body key.** Every expression inside this flow (Switch's `On`, both `Set variable` actions, every email template) must still address the field by its **original default key** (`text`, `text_1`, `text_2`, `number`, `text_3`, `text_4`, `text_5`, `text_6`, `text_7`, in that add-order) with the safe-navigation `?` operator, e.g. `@{triggerBody()?['text_2']}` for `requestTitle` — never `@{triggerBody()?['requestTitle']}`.
- **Actions** (each `Initialize variable` below is its own separate action — the action only ever initializes one variable, it cannot be shared across three):
  1. `Initialize variable` → `varSubject` (String, blank initial value).
  2. `Initialize variable` → `varBody` (String, blank initial value — holds **HTML**, not plain text).
  3. `Initialize variable` → `varAppLink` (String) = the app deep link, set **once** here (fill in after the app is published): `https://apps.powerapps.com/play/e/<ENV_ID>/a/<APP_ID>?tenantId=<TENANT_ID>`.
  4. `Switch` on `notificationType` with 7 cases, each using `Set variable` to set `varSubject` / `varBody` (templates below). **Default branch → Terminate (Failed, "Unknown notificationType")** so a typo in the app surfaces as a flow failure instead of silent no-mail.
  5. `Send an email (V2)` — To: `recipientEmails` (always exactly one recipient per call, see §2a), Subject: `varSubject`, Body: `varBody`, **Is HTML = Yes**.
- **Connection**: `app.admin@maxbiocare.com` shared mailbox connection; pin it under *Run only users* so every app user sends through the same connection.
- Every email body links back to the app via `@{variables('varAppLink')}` — set the URL once in action 3 above and every template (all 7 cases) picks it up; nothing to edit per-template. See `flows/Project_Notify/README.md`.

### Email templates (English)

**Body is HTML, not plain text.** The ready-to-paste HTML source for `varBody` in each `Switch` case lives in [`flows/Project_Notify/templates/`](../flows/Project_Notify/) (one file per case — see [`flows/Project_Notify/README.md`](../flows/Project_Notify/README.md) for how to paste it into Power Automate's Body field via Code View). What follows below is the **content spec** (subject line + body copy in plain-text form) — the source of truth for wording; the HTML files are that same copy laid out in a branded table template.

**Case `SubmittedToSponsor`** — To: the CR's `ProjectSponsor` (Create requests only)

- Subject: `[Project List] Action needed: assign a Project Manager for @{triggerBody()?['text_2']}`
- Body:
  ```
  A new project has been requested and needs you, as Project Sponsor, to assign a Project Manager before it can move to approval.

  Request:      @{triggerBody()?['text_2']} (#@{triggerBody()?['number']})
  Type:         @{triggerBody()?['text_3']}
  Project:      @{triggerBody()?['text_4']}
  Requested by: @{triggerBody()?['text_5']}

  Please open the Project List app → Approvals → Assign Manager to pick a Project Manager and submit.
  <app deep link>
  ```

**Case `SubmittedToManager`** — To: the CR's/project's specific `ProjectManager` — fired either when an Update/Delete CR is submitted directly (target project already has a Manager), or when the `ProjectSponsor` finishes assigning one on a Create CR

- Subject: `[Project List] Approval needed: @{triggerBody()?['text_2']}`
- Body:
  ```
  A new project change request is waiting for your review.

  Request:      @{triggerBody()?['text_2']} (#@{triggerBody()?['number']})
  Type:         @{triggerBody()?['text_3']}
  Project:      @{triggerBody()?['text_4']}
  Requested by: @{triggerBody()?['text_5']}

  Please open the Project List app → Approvals to review it.
  <app deep link>
  ```

**Case `ManagerApprovedToExecutive`** — To: all active Executives

- Subject: `[Project List] Executive approval needed: @{triggerBody()?['text_2']}`
- Body: same layout, plus `Manager approved by: @{triggerBody()?['text_6']}`.

**Case `FinalApproved`** — To: requester

- Subject: `[Project List] Approved: @{triggerBody()?['text_2']}`
- Body:
  ```
  Your request has been fully approved and applied.

  Request:     @{triggerBody()?['text_2']} (#@{triggerBody()?['number']})
  Type:        @{triggerBody()?['text_3']}
  Project:     @{triggerBody()?['text_4']}
  Approved by: @{triggerBody()?['text_6']}
  ```

**Case `FinalRejected`** — To: requester

- Subject: `[Project List] Rejected: @{triggerBody()?['text_2']}`
- Body:
  ```
  Your request has been rejected.

  Request:     @{triggerBody()?['text_2']} (#@{triggerBody()?['number']})
  Type:        @{triggerBody()?['text_3']}
  Project:     @{triggerBody()?['text_4']}
  Rejected by: @{triggerBody()?['text_6']}
  Remark:      @{triggerBody()?['text_7']}

  You may submit a new request after addressing the remark.
  ```

**Case `FinalApprovedManagerCopy`** — To: that project's ProjectManager (Step 2/Executive approve only — Manager-facing copy of `FinalApproved`, distinct wording since the Manager isn't the requester)

- Subject: `[Project List] Project approved: @{triggerBody()?['text_2']}`
- Body:
  ```
  A project you manage has been fully approved by the Executive and applied.

  Request:      @{triggerBody()?['text_2']} (#@{triggerBody()?['number']})
  Type:         @{triggerBody()?['text_3']}
  Project:      @{triggerBody()?['text_4']}
  Requested by: @{triggerBody()?['text_5']}
  Approved by:  @{triggerBody()?['text_6']}
  ```

**Case `FinalRejectedManagerCopy`** — To: that project's ProjectManager (Step 2/Executive reject only — Manager-facing copy of `FinalRejected`, distinct wording since the Manager isn't the requester)

- Subject: `[Project List] Project rejected: @{triggerBody()?['text_2']}`
- Body:
  ```
  A project you manage was rejected by the Executive.

  Request:      @{triggerBody()?['text_2']} (#@{triggerBody()?['number']})
  Type:         @{triggerBody()?['text_3']}
  Project:      @{triggerBody()?['text_4']}
  Requested by: @{triggerBody()?['text_5']}
  Rejected by:  @{triggerBody()?['text_6']}
  Remark:       @{triggerBody()?['text_7']}
  ```

### Recipient resolution (app side, Power Fx)

Per-project, not a role broadcast — see `CLAUDE.md` §Role-based visibility for the full authorization model. Each `notificationType` is always exactly one recipient — never a `;`-joined list.

```
// SubmittedToSponsor: the CR's own ProjectSponsor (Create only — the project doesn't exist yet, nothing to fall back to but the CR itself)
Coalesce(
    LookUp('Employee List', ID = <ProjectSponsor.Id>).Email,
    "app.admin@maxbiocare.com"
)
// SubmittedToManager: the CR's/project's specific ProjectManager
Coalesce(
    LookUp('Employee List', ID = <ProjectManager.Id>).Email,
    "app.admin@maxbiocare.com"
)
// ManagerApprovedToExecutive: the CR's/project's specific ProjectOwner, same Coalesce fallback
// FinalRejected / FinalApproved (requester leg, both steps): gSelectedCR.RequesterEmail
// FinalApprovedManagerCopy / FinalRejectedManagerCopy (Step 2 only, Manager leg): the CR's/project's specific ProjectManager, same Coalesce fallback as ProjectOwner above
```

`<ProjectSponsor.Id>` / `<ProjectManager.Id>` / `<ProjectOwner.Id>` resolve differently depending on where the call happens:
- Submitting a Create request (`CreateProjectScreen`): the form's own `cmbSponsor.Selected.ID` for `ProjectSponsor` — there is no `ProjectManager` to resolve yet, it's left blank on the CR until the Sponsor assigns one.
- Submitting an Update/Delete request (`Update`/`DeleteProjectScreen`): the target project's `gSelectedProject.ProjectManager.Id` (no `ProjectSponsor` involved — Step 0 doesn't apply to these).
- Sponsor assigning a Manager (`ApprovalsScreen`, "Assign Manager" tab, Step 0 → notifies the newly-assigned Manager): the tab's own `cmbAssignManager.Selected.ID`.
- Manager approving (`ApprovalsScreen`, Step 1 → notifies the Executive): `If(gSelectedCR.RequestType.Value = "Create", gSelectedCR.ProjectOwner.Id, gLiveTarget.ProjectOwner.Id)`.
- Executive approving/rejecting (`ApprovalsScreen`, Step 2 → notifies requester, then separately the Manager): same `Create` vs. live-target branch as above, but resolving `ProjectManager.Id` instead of `ProjectOwner.Id` for the second `Project_Notify.Run` call.

## 4. FUTURE — `Project_SyncActualCost` (placeholder, do not build yet)

Goal: `Project_List.ActualCost` accumulates actual spend from the procurement SharePoint data automatically; users never type it.

- **Blocked on join key**: procurement lists (`Procurement_Requests`, `Procurement_InvoiceData`) have no project reference today. Preferred design: add a `ProjectID` (Text) column to `Procurement_Requests`, exposed as an optional picker in the procurement app.
- Sketch: recurrence or SharePoint-trigger flow → for each distinct `ProjectID`, sum `TotalAmount` from `Procurement_InvoiceData` → Patch `Project_List.ActualCost` where `ProjectID` matches → on change, call `Project_Notify` with a new case `ActualCostUpdated` (To: Project Manager) — add the 10th arg / new case then.
- Until then `ActualCost` stays blank/manual-admin-only and read-only in the app.
