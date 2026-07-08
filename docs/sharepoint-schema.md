# SharePoint Lists & Flows — Schema Reference

Authoritative data model for the Max Biocare Project List app. Site: `maxbiocare.sharepoint.com/sites/Powerapps`.

> Only **custom / app-relevant columns** are listed. Every SharePoint list also carries the standard system columns (`ID`, `Created`, `Modified`, `Author`, `Editor`, `Attachments`, content-type, moderation, etc.) — omitted here for clarity. Update this file whenever a SharePoint column is added/renamed.

## Conventions

- **System columns use English internal names** (site locale = en) — use `Title` directly in Power Fx, no quoting needed.
- **Choice** columns: write `{Value: "..."}`, read `Col.Value`.
- **Multi-choice** columns: write a table of `{Value: "..."}` records (`ForAll(items, {Value: Value})`), read as a table.
- **Lookup / Person** columns: write `{Id: ..., Value: ...}`, read `Col.Value` / `Col.Id`. (These are SharePoint *lookup* columns pointing at another list — never real Person columns, which require Claims records on Patch.)
- **Multi-value Lookup** columns (e.g. `RelatedSKU`): write a table of `{Id: ..., Value: ...}` records (`ForAll(items, {Id: ID, Value: Title})`), read as a table (`Concat(Col, Value, ", ")`).
- **Currency / Number** columns: Patch plain numbers (`Value(txt.Value)`), no `{Value:}` wrapper.
- **Exception:** `Department` on `Project_List` / `Project_ChangeRequests` is plain **Text** (not Choice) — Patch the raw string, no `{Value:}` wrapper. Its picker options come from `Choices('Employee List'.department)` — the Choice column's defined value set on `Employee List` (delegable; a `Distinct(Filter(...))` over the data rows hit a SharePoint delegation wall on both `<>` and `Not()`).
- **Required** columns are flagged ⚠ — a `Patch` that omits them fails.
- `ChangeRequestIDText` (plain-text copy of the change-request ID) is the delegable join key for log lookups: `LookUp(Project_ApprovalLog, ChangeRequestIDText = Text(cr.ID) && StepNumber = N)`.
- **The app never writes `Project_List` directly from forms.** The ONLY writer is the Executive-approve handler on `ApprovalsScreen`. All user intent goes through `Project_ChangeRequests`.

---

## SharePoint list permissions (governance — configure on the site, not in code)

Canvas Apps' SharePoint connector always runs under the **signed-in user's own permissions** — there is no service-account indirection. So "nobody should be able to open a list directly (browser/Excel/REST) and add/edit/delete rows outside the app's guarded `Patch` flow" has to be enforced with SharePoint list permissions, not anything in `Src/*.pa.yaml`.

Assign via 3 SharePoint groups (Requester = default site Members/Visitors — everyone with a tenant account, no explicit group needed; Manager / Executive = dedicated groups mirroring `Project_User.Role`):

| List | Requester (everyone) | Manager | Executive |
|---|---|---|---|
| `Employee List` | Read | Read | Read |
| `Project_User` | Read (required — every user's `App.OnStart` reads this to resolve `gUserRole`) | Read | Read |
| `Product_Database_SKU_Master` | Read | Read | Read |
| `Project_ChangeRequests` | **Add Items + Read only** (custom permission level, see below) | Edit (needed to Patch `ApprovalStatus` + write the Step 1 log) | Edit (needed to Patch `ApprovalStatus` at Step 2) |
| `Project_ApprovalLog` | Read only | **Add Items + Read only** — the app only ever inserts new log rows, never edits one | **Add Items + Read only** |
| `Project_List` | Read only | Read only | Edit (needed to Patch the apply step) |

**"Add Items + Read only" is not a SharePoint built-in level** — create a custom Permission Level (Site Settings → Permission Levels → copy `Read`, additionally check `Add Items`, leave `Edit Items`/`Delete Items` unchecked). This lets Requesters submit a CR and read the list (needed for the `Mine` tab) but blocks them from tampering with any CR — their own or anyone else's — once submitted.

**Known residual risk — cannot be fully closed without an architecture change:** Manager needs raw `Edit Items` on `Project_ChangeRequests`, and Executive needs raw `Edit Items` on both `Project_ChangeRequests` and `Project_List`, because `Patch()` always executes under the acting user's own identity. A Manager/Executive with SharePoint UI access could in principle bypass the app's guarded formulas (e.g. hand-flip a CR to `Approved`, or edit `Project_List` directly). Closing this fully would mean moving the apply-step `Patch` into a Power Automate flow with a locked-down "Run only users" connection (the flow's own connection does the write, not the human's) — a deliberate, larger architecture change, not something list permissions alone can fix. Not pursued unless explicitly requested.

---

## `Project_List` — master project record (approved data only)

Required ⚠: `ProjectStatus`, `ApprovalStatus`, `ProjectLevel`.

| Column (internal) | Type | Notes |
|---|---|---|
| `Title` | Text | **Project Name** (reuses system Title) |
| `ProjectID` | Text | Unique business key, e.g. `PROJ-QA-2026-017`; generated by the 2nd Patch at apply time (see CLAUDE.md — ID generation) |
| `ProjectDescription` | Text (multiline) | Detailed scope |
| `ProjectType` | Choice | `New Product Development`, `Product Improvement`, `Regulatory & Compliance`, `Marketing Campaign`, `Market Expansion`, `Operational Improvement`, `IT & Digital`, `Research & Clinical`, `Other` |
| `Department` | Text | Single department name; picker options come from `Choices('Employee List'.department)`, not a Choice column itself — see `.department` on Employee List |
| `ProjectOwner` | Lookup→Employee List | Business owner, `{Id, Value}` (→Title). Picker on `CreateProjectScreen` is scoped to active `Project_User` accounts with `Role.Value = "Executive"` (`ForAll(Filter(Project_User, IsActive && Role.Value = "Executive"), {ID: EmployeeID.Id, Title: EmployeeID.Value})` — reshapes the already-available `EmployeeID` fields, no secondary `LookUp` needed) — only Executives are selectable |
| `ProjectManager` | Lookup→Employee List | Execution lead, `{Id, Value}` (→Title). Same pattern as `ProjectOwner` but scoped to `Role.Value = "Manager"` — only Managers are selectable |
| `Priority` | Choice | `Low`, `Medium`, `High`, `Critical` |
| `StartDate` | Date | Planned start |
| `EndDate` | Date | Planned completion |
| `ProjectStatus` ⚠ | Choice | `Not Started` (default), `In Planning`, `In Progress`, `On Hold`, `Completed`, `Cancelled`, `Deleted` — **`Deleted` is the soft-delete value**, hidden from All Projects by default |
| `BudgetAmount` | Currency | Total budget (no separate Budget Approved flag — dropped by decision, Project Status covers it) |
| `ActualCost` | Currency | Spend to date — **never written by the app**; reserved for the future `Project_SyncActualCost` flow (see approval-workflow-plan.md). Always rendered read-only |
| `CapexOpex` | Choice | `CapEx`, `OpEx`, `Mixed` |
| `RelatedSKU` | Lookup→`Product_Database_SKU_Master` (multi) | Table of `{Id, Value}` (→Title) — allow multiple values; verify the target list's Title/display column before wiring the picker |
| `Market` | Choice | `AU`, `MY`, `SG`, `VN` |
| `Channel` | Choice | `Retail Pharmacy`, `Hospital`, `Distributor`, `E-commerce`, `Direct-to-Consumer`, `Cross-border`, `Multiple` |
| `ApprovalStatus` ⚠ | Choice | `Approved` — always `Approved` on master rows (rows only exist after full approval); kept per spec for reporting |
| `ApprovedBy` | Lookup→Employee List | Executive who applied the create |
| `RequestedBy` ⚠ | Lookup→Employee List | Original requester who submitted the Create request (`{Id, Value}`, copied from `gSelectedCR.RequesterID` at apply time) — **gates who can submit Update/Delete for this project** (`AllProjectsScreen`'s Update/Delete row buttons only show when `RequestedBy.Id = gCurrentEmployee.ID`). Rows created before this column existed have it blank, which locks everyone out of Update/Delete for that row until fixed manually |
| `ProjectDocuments` | Text (URL) | Document link — plain text column, NOT the Hyperlink column type (awkward to Patch) |
| `StrategicObjective` | Choice | `Revenue Growth`, `Market Expansion`, `Brand Building`, `Product Innovation`, `Operational Excellence`, `Regulatory Compliance`, `Cost Optimization` |
| `CostCenter` | Choice | `Office Melbourne (Head Quarter)`, `Port Melbourne Warehouse`, `Max Biocare Research Park - Natural Inspirations@Yinnar`, `Max Biocare Research Park - Mar-Nuka Bay`, `Malay Warehouse`, `Singapore Warehouse` |
| `ProjectLevel` ⚠ | Number | `0` = root project, `1` = child project |
| `RootProjectID` | Text | Parent's `ProjectID` when level 1; blank for level 0 |

### Department → code map (ProjectID generation)

`Quality Assurance`→QA · `Regulatory Affairs`→RA · `Marketing`→MK · `Sales`→SL · `Supply Chain`→SC · `Finance`→FN · `IT`→IT · `R&D`→RD · `HR`→HR · `Operations`→OP · fallback→GN. `Department` is a single Text value, matched directly via `Switch(Department, ...)`.

---

## `Project_ChangeRequests` — staging list (the approval state machine lives here)

Every Create / Update / Delete intent becomes one row here. Only after Executive approval does the app apply the change to `Project_List`.

Required ⚠: `RequestType`, `ApprovalStatus`, `RequesterEmail`.

### Request metadata

| Column | Type | Notes |
|---|---|---|
| `Title` | Text | Auto-built: `<RequestType> - <ProjectName> - <requester> - dd/mm/yyyy` |
| `RequestType` ⚠ | Choice | `Create`, `Update`, `Delete` |
| `ApprovalStatus` ⚠ | Choice | `Pending Manager` → `Pending Executive` → `Approved` \| `Rejected` (exact strings — used as literals across all screens) |
| `TargetItemID` | Number | SharePoint `ID` of the master row (Update/Delete only) — join key for live-diff lookups |
| `TargetProjectID` | Text | Business `ProjectID` snapshot for display (Update/Delete only) |
| `RequesterEmail` ⚠ | Text | `User().Email`; "my requests" filter key |
| `RequesterID` | Lookup→Employee List | `{Id, Value}` |
| `RequestReason` | Text (multiline) | **Required for Delete and Update** (enforced in app); Create no longer has a Request Notes field, so it's always blank on Create CRs |

### Proposed values (Create: all set · Update: only `ProjectDescription` + `EndDate` + `ProjectName` for display · Delete: all blank)

Same column names and **identical Choice value sets** as `Project_List` (keep them in sync — a `{Value:"X"}` not present in the target choice column fails the apply Patch):

| Column | Type |
|---|---|
| `ProjectName` | Text (master uses Title; CR needs its own column) |
| `ProjectDescription` | Text (multiline) |
| `ProjectType` | Choice — `New Product Development`, `Product Improvement`, `Regulatory & Compliance`, `Marketing Campaign`, `Market Expansion`, `Operational Improvement`, `IT & Digital`, `Research & Clinical`, `Other` |
| `Priority` | Choice — `Low`, `Medium`, `High`, `Critical` |
| `ProjectStatus` | Choice — `Not Started`, `In Planning`, `In Progress`, `On Hold`, `Completed`, `Cancelled`, `Deleted` |
| `CapexOpex` | Choice — `CapEx`, `OpEx`, `Mixed` |
| `Market` | Choice — `AU`, `MY`, `SG`, `VN` |
| `Channel` | Choice — `Retail Pharmacy`, `Hospital`, `Distributor`, `E-commerce`, `Direct-to-Consumer`, `Cross-border`, `Multiple` |
| `StrategicObjective` | Choice — `Revenue Growth`, `Market Expansion`, `Brand Building`, `Product Innovation`, `Operational Excellence`, `Regulatory Compliance`, `Cost Optimization` |
| `CostCenter` | Choice — `Office Melbourne (Head Quarter)`, `Port Melbourne Warehouse`, `Max Biocare Research Park - Natural Inspirations@Yinnar`, `Max Biocare Research Park - Mar-Nuka Bay`, `Malay Warehouse`, `Singapore Warehouse` |
| `Department` | Text (same convention as master — single value, options sourced from Employee List) |
| `ProjectOwner`, `ProjectManager` | Lookup→Employee List |
| `StartDate`, `EndDate` | Date |
| `BudgetAmount` | Currency |
| `RelatedSKU` | Lookup→Product_Database_SKU_Master (multi) |
| `ProjectDocuments` | Text (URL) |
| `ProjectLevel` | Number (0/1) |
| `RootProjectID` | Text |

No `ActualCost` column (never user-proposable). No `ManagerApprovedBy/At` columns — who/when acted comes from `Project_ApprovalLog` via `ChangeRequestIDText` + `StepNumber`.

---

## `Project_User` — role assignment (mirror of `Procurement_User`)

Required ⚠: `Role`, `IsActive`, `EmployeeID`.

| Column | Type | Notes |
|---|---|---|
| `Title` | Text | **Not populated by the app** — the record is already tied to an employee via `EmployeeID`. For display, resolve the name through `EmployeeID.Value` / `LookUp('Employee List', ID = EmployeeID.Id).Title`, never `Project_User.Title` |
| `Role` ⚠ | Choice | `Manager`, `Executive` — no `Requester`/`Admin` value; any tenant email can create change requests regardless of role |
| `IsActive` ⚠ | Yes/No | default true |
| `EmployeeID` ⚠ | Lookup→Employee List | `{Id, Value}` (→Title) only — confirmed `EmployeeID.Email` does not resolve even with SharePoint's "additional columns" Lookup setting configured for `Email` and the Power Apps data source refreshed; use `LookUp('Employee List', ID = EmployeeID.Id).Email` for email instead |
| `Note` | Text | |

Resolved in `App.OnStart` → `gCurrentUser` / `gUserRole`. Employees absent here (or not `IsActive`) default to `gUserRole = "Requester"` — an internal-only value, never written to `Project_User.Role`, meaning "not an approver."

---

## `Project_ApprovalLog` — manager & executive decisions

Required ⚠: `ChangeRequestID`, `StepNumber`.

| Column | Type | Notes |
|---|---|---|
| `Title` | Text | `Step <n> - <action> - <CR title>` |
| `ChangeRequestID` ⚠ | Lookup→Project_ChangeRequests (→Title) | `{Id, Value}` |
| `ChangeRequestIDText` | Text | delegable join key = `Text(cr.ID)` |
| `StepNumber` ⚠ | Number | **`1` = Manager, `2` = Executive** (document in the column description) |
| `ApproverID` | Lookup→Employee List | `{Id, Value}` |
| `Action` | Choice | `Approved`, `Rejected` |
| `Remark` | Text (multiline) | **Required when Action = Rejected** (enforced in app) |
| `ActionAt` | DateTime | `Now()` |

---

## `Employee List` — staff directory (existing, shared with procurement)

Read-only dependency. Matched in `App.OnStart` via `LookUp('Employee List', Email = User().Email)`.

| Column | Type | Notes |
|---|---|---|
| `Title` | Text | employee display name |
| `email` | Text | internal name is **lowercase** (display name capitalized) |
| `department` | Choice | internal name is **lowercase**; source of the `Department` picker options on `Project_List`/`Project_ChangeRequests` via `Choices('Employee List'.department)` — shows every department defined on this Choice column, not just ones currently assigned to an employee |
| `JobTitle`, `City`, `Country`, `EmployeeCode` | Text | |

---

## `Product_Database_SKU_Master` — product catalog (existing, read-only)

Target of the `RelatedSKU` lookup (multi-select — a project can relate to more than one SKU). App uses `ID` + `Title` only.
Picker: `ComboBox@0.0.51`, `SelectMultiple: =true`, `Items: =Sort(Product_Database_SKU_Master, Title)`; write `ForAll(cmbSKU.SelectedItems, {Id: ID, Value: Title})`; `DefaultSelectedItems` via `Filter(...)`.

> ⚠ Verify the actual display/Title column name in SharePoint before wiring — this list is not referenced by the sibling apps, so its schema is unconfirmed.

---

## Power Automate flows

### `Project_Notify.Run(...)` — generic email notification (PowerAppsV2 trigger)

Called from Power Fx after every successful state transition. **Always pass all 9 args** — use `""` for unused remark. Declare each trigger input's type explicitly when building the flow (Blank-type inference bug — see invoice-batch-app `docs/finalize-invoice-flow-plan.md`).

**Recipients are the CR's/project's specific `ProjectManager`/`ProjectOwner`, not a role broadcast** (see `CLAUDE.md` §Role-based visibility for the full authorization model) — always wrapped in `Coalesce(LookUp('Employee List', ID = <ProjectManager.Id or ProjectOwner.Id>).Email, "app.admin@maxbiocare.com")` so a blank/unassigned person never leaves the flow's "Send an email (V2)" action with an empty `To`.

| # | Trigger param (rename from Power Automate default) | Type | Meaning |
|---|---|---|---|
| 1 | `notificationType` (default `text`) | Text | `SubmittedToManager` \| `ManagerApprovedToExecutive` \| `FinalApproved` \| `FinalRejected` |
| 2 | `recipientEmails` (default `text_1`) | Text | recipient emails, `;`-separated |
| 3 | `requestTitle` (default `text_2`) | Text | change request title |
| 4 | `requestId` (default `number`) | Number | change request ID |
| 5 | `requestType` (default `text_3`) | Text | request type (`Create`/`Update`/`Delete`) |
| 6 | `projectName` (default `text_4`) | Text | project name |
| 7 | `requesterName` (default `text_5`) | Text | requester display name |
| 8 | `actionByName` (default `text_6`) | Text | action-by display name |
| 9 | `remark` (default `text_7`) | Text | remark (`""` when N/A — never omit) |

Rename each "Ask in PowerApps" input to the name above when building the flow (Power Automate defaults to `text`/`text_1`/`number`/... if left unrenamed). Power Apps still calls `Project_Notify.Run(...)` **positionally** in this exact order — renaming only changes the identifier used inside the flow body (`triggerBody()['notificationType']`, etc.) and the intellisense hint shown in the Power Apps formula bar.

Flow body: **Switch on `notificationType`** (not nested If) → each case sets `Subject`/`Body` variables → single "Send an email (V2)" with To = `recipientEmails`. Switch **Default → Terminate (Failed, "Unknown notificationType")**. Full email templates in `docs/approval-workflow-plan.md`.

### `Project_SyncActualCost` — FUTURE (spec placeholder, not built)

Accumulates `ActualCost` on `Project_List` from procurement spend. Blocked on adding a `ProjectID` (Text) join column to the procurement lists (`Procurement_Requests` / `Procurement_InvoiceData`) — join key TBD. On completion it should notify the Project Manager (`ActualCostUpdated` case to be added to `Project_Notify`). See `docs/approval-workflow-plan.md` §Future.
