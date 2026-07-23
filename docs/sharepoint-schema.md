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
- **The app never writes `Project_List` directly from forms.** The ONLY writer is the Sponsor final-approve handler (Step 2) on `ApprovalsScreen`. All user intent goes through `Project_ChangeRequests`.

---

## SharePoint list permissions (governance — configure on the site, not in code)

Canvas Apps' SharePoint connector always runs under the **signed-in user's own permissions** — there is no service-account indirection. So "nobody should be able to open a list directly (browser/Excel/REST) and add/edit/delete rows outside the app's guarded `Patch` flow" has to be enforced with SharePoint list permissions, not anything in `Src/*.pa.yaml`.

Assign via 1 dedicated SharePoint group plus "Everyone except external users" (Requester = everyone with a tenant account, no explicit group needed; Sponsor = dedicated group mirroring `Project_User.Role = "Sponsor"`; there is no dedicated Manager group — see the 2026-07-23 decision below).

**Use SharePoint's built-in Permission Levels by their exact dropdown name** (`Site Permissions` → `Grant Permissions` → `Select a permission level`: `Full Control` / `Design` / `Edit` / `Contribute` / `Read` / `Restricted View`) — none of the tables below ever need `Full Control`, `Design`, or `Edit`:
- **`Read`** — view items/pages, no changes. Used for every read-only grant below.
- **`Contribute`** — can view, add, update, and delete list items (item-level CRUD), but **cannot** alter the list's own structure (columns/views) or delete the list itself. This is the correct level everywhere the table below says "item CRUD" — **not** `Edit`.
- **`Edit`** — everything `Contribute` does **plus** the ability to add/edit/delete the list's own columns and views (and the list itself). Deliberately **never used** in this app's permission model, even for Sponsor/Everyone grants that need full item CRUD — granting `Edit` broadly (especially to "Everyone except external users" on `Project_ChangeRequests`/`Project_ApprovalLog`) would let any user reshape or delete the list schema, which `Contribute` avoids while still allowing every `Patch()` the app performs.

There is no "Manager" row/group in this matrix — `ProjectManager` is a per-project `Employee List` assignment, not a SharePoint principal (see `Project_User.Role` below), so a Manager is just whichever employee is assigned and is covered entirely by the "Everyone" grants:

| List | Everyone except external users | Sponsor (dedicated group) |
|---|---|---|
| `Employee List` | `Read` | *(covered)* |
| `Project_User` | `Read` (required — every user's `App.OnStart` reads this to resolve `gUserRole`) | *(covered)* |
| `Product_Database_SKU_Master` | `Read` | *(covered)* |
| `Project_ChangeRequests` | **`Contribute`** (see 2026-07-23 decision below; needed so any employee assigned as a project's `ProjectManager` can Patch `ApprovalStatus`) | *(covered)* |
| `Project_ApprovalLog` | **"Add Items + Read only"** ⚠ *(custom level, broadened 2026-07-23, see below)* — the app only ever inserts log rows, never edits one | *(covered)* |
| `Project_List` | `Read` | **`Contribute`** (needed to Patch the apply step) — the only grant that isn't blanket "Everyone" |

**"Add Items + Read only" is not one of the 6 built-in levels above** — create a custom Permission Level (`Site Settings` → `Site permissions` → `Permission Levels` → `Add a Permission Level`, copy `Read`, additionally check only the `Add Items` permission, leave `Edit Items`/`Delete Items` unchecked) and grant it on `Project_ApprovalLog` to "Everyone except external users". The quick `Select a permission level` dropdown (shown when using "Grant Permissions"/"Shared with") only lists the 6 built-ins — the custom level still shows up there once created, but it must be created first via the Permission Levels page, not picked from that dropdown directly.

> **Model change (2026-07-11):** `ProjectSponsor` is now a `Project_User` role (renamed from "Executive"). `Project_List` `Contribute` goes to the Sponsor group (they run the apply step), not "Executive".

> **Decision (2026-07-23):** `ProjectManager` assignment (`cmbAssignManager`) now sources the full `Employee List`, not just accounts with `Project_User.Role = "Manager"` (see `Project_List.ProjectManager` below and `CLAUDE.md` §Known caveats), so a per-role SharePoint "Manager" group can no longer reliably cover whoever gets assigned — and the same problem applies to `Project_ApprovalLog`, since approving Step 1 first writes a log row there before the `Project_ChangeRequests` status Patch even runs. **Resolved by broadening both lists to "Everyone except external users"**: `Project_ChangeRequests` gets blanket `Contribute` (mirroring the historical `ProjectSponsor`-could-be-anyone precedent), and `Project_ApprovalLog` gets the custom "Add Items + Read only" level instead of restricting it to a Manager/Sponsor group. This removes the standalone "Requester: Add Items + Read only" tier on `Project_ChangeRequests` (superseded by the blanket `Contribute` grant) and the dedicated Manager group entirely. **Residual risk, accepted:** every signed-in user can now edit/delete *any* `Project_ChangeRequests` row and insert *any* `Project_ApprovalLog` row directly via SharePoint UI/Excel/REST, not just their own — the app's own buttons/formulas still gate this in the UI (only the original requester, no pending CR, `ApproverID` set to the acting user, etc.), but that gating is bypassable outside the app. Same posture as the pre-existing Sponsor `Project_List` `Contribute` risk below; not closable without moving the guarded Patches into a locked-down Power Automate flow.

**Known residual risk — cannot be fully closed without an architecture change:** Sponsor needs raw `Contribute` on `Project_List` (everyone else stays `Read`-only there), because `Patch()` always executes under the acting user's own identity. Anyone with SharePoint UI access to `Project_ChangeRequests`/`Project_ApprovalLog` (now everyone) or `Project_List` (Sponsor) could in principle bypass the app's guarded formulas and edit those lists directly. Closing this fully would mean moving the apply-step `Patch` into a Power Automate flow with a locked-down "Run only users" connection (the flow's own connection does the write, not the human's) — a deliberate, larger architecture change, not something list permissions alone can fix. Not pursued unless explicitly requested.

---

## `Project_List` — master project record (approved data only)

Required ⚠: `ProjectStatus`, `ProjectLevel`.

| Column (internal) | Type | Notes |
|---|---|---|
| `Title` | Text | **Project Name** (reuses system Title) |
| `ProjectID` | Text | Unique business key, e.g. `PROJ-AU-QA-2026-017` (`PROJ-<MarketCode>-<DeptCode>-<Year>-<seq>`); generated by the 2nd Patch at apply time (see CLAUDE.md — ID generation) |
| `ProjectDescription` | Text (multiline) | Detailed scope |
| `Deliverables` | Text (multiline) | Concrete outputs/results the project is expected to produce — required at Create; editable via Update alongside `ProjectDescription`/`EndDate` |
| `ProjectType` | Choice | `New Product Development`, `Product Improvement`, `Regulatory & Compliance`, `Marketing Campaign`, `Market Expansion`, `Operational Improvement`, `IT & Digital`, `Research & Clinical`, `Other` |
| `Department` | Text | Single department name; picker options come from `Choices('Employee List'.department)`, not a Choice column itself — see `.department` on Employee List |
| `ProjectOwner` | Lookup→Employee List | **DEPRECATED — no longer used by the app (2026-07-11).** The former "Owner/Executive" second-approver was merged into `ProjectSponsor`. The SharePoint column may remain for historical rows but is never read or written by any screen. Do not reintroduce it |
| `ProjectSponsor` | Lookup→Employee List | Owns the project's outcome **and** is the final (Step 2) approver — the merged Owner+Sponsor role, `{Id, Value}` (→Title). **Gated by `Project_User` role** (`Role.Value = "Sponsor"`, renamed from "Executive"): the picker on `CreateProjectScreen` is scoped to active Sponsors (`ForAll(Filter(Project_User, IsActive && Role.Value = "Sponsor"), {ID: EmployeeID.Id, Title: EmployeeID.Value})` — reshapes the already-available `EmployeeID` fields, no secondary `LookUp` needed), so only Sponsors are selectable. Set at Create time; does **two** things: (1) Step 0 — picks the `ProjectManager` on the CR right after submit (`ApprovalsScreen`'s "Assign Manager" tab), (2) Step 2 — final approve + applies to `Project_List` |
| `ProjectManager` | Lookup→Employee List | Execution lead / supervisor, `{Id, Value}` (→Title). **No longer scoped to `Role.Value = "Manager"` (2026-07-23)** — `cmbAssignManager` on `ApprovalsScreen` now sources the full `Employee List`, so any employee can be assigned; `Project_ChangeRequests`'s SharePoint `Contribute` grant was broadened to "Everyone except external users" to match (see §SharePoint list permissions). **Not chosen on `CreateProjectScreen`** — blank on a newly-created row's originating CR until the `ProjectSponsor` assigns it (see `ApprovalStatus` below); always populated by the time a row reaches `Project_List` (apply only happens after Step 2, by which point a Manager was assigned and Step 1 completed). Does Step 1 approval |
| `Priority` | Choice | `Low`, `Medium`, `High`, `Critical` |
| `StartDate` | Date | Planned start |
| `EndDate` | Date | Planned completion |
| `ProjectStatus` ⚠ | Choice | `Not Started` (default), `In Planning`, `In Progress`, `On Hold`, `Completed`, `Cancelled`, `Deleted` — **`Deleted` is the soft-delete value**, hidden from All Projects by default |
| `BudgetAmount` | **Number** (not Currency) | Total budget, in whatever unit `Currency` (below) says — no separate Budget Approved flag — dropped by decision, Project Status covers it |
| `ActualCost` | **Number** (not Currency) | Spend to date, in whatever unit `Currency` (below) says — **never written by the app**; reserved for the future `Project_SyncActualCost` flow (see approval-workflow-plan.md). Always rendered read-only |
| `Currency` | Text | 3-letter code (`AUD`/`MYR`/`SGD`/`VND`) applying to both `BudgetAmount` and `ActualCost` on this row — plain Text like `Department`, not a Choice column. Set once at Create time (copied from the CR's own `Currency`, itself derived from the form's Market selection — see `CreateProjectScreen.pa.yaml`'s `currency` control) and never changed afterward; `ActualCost` (whenever the future sync flow starts writing it) reuses this same field rather than carrying its own currency |
| `CapexOpex` | Choice | `CapEx`, `OpEx`, `Mixed` |
| `RelatedSKU` | Lookup→`Product_Database_SKU_Master` (multi) | Table of `{Id, Value}` (→Title) — allow multiple values; verify the target list's Title/display column before wiring the picker |
| `Market` | Choice | `AU`, `MY`, `SG`, `VN` — no longer the source `Currency` is derived from at display time (that coupling was removed); `Currency` is now its own stored field, only *initially populated* from Market's mapping at Create |
| `Channel` | Choice | `Retail Pharmacy`, `Hospital`, `Distributor`, `E-commerce`, `Direct-to-Consumer`, `Cross-border`, `Multiple` |
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

Every Create / Update / Delete intent becomes one row here. Only after the Sponsor's Step 2 approval does the app apply the change to `Project_List`.

Required ⚠: `RequestType`, `ApprovalStatus`, `RequesterEmail`.

### Request metadata

| Column | Type | Notes |
|---|---|---|
| `Title` | Text | Auto-built: `<RequestType> - <ProjectName> - <requester> - dd/mm/yyyy` |
| `RequestType` ⚠ | Choice | `Create`, `Update`, `Delete` |
| `ApprovalStatus` ⚠ | Choice | `Pending Sponsor` → `Pending Manager Approval` → `Pending Sponsor Approval` → `Approved` \| `Rejected` (exact strings — used as literals across all screens). The Sponsor acts twice: `Pending Sponsor` = Step 0 (assign the `ProjectManager`, via `ApprovalsScreen`'s "Assign Manager" tab), `Pending Sponsor Approval` = Step 2 (final approve + apply). **`Pending Sponsor` exists only on Create CRs** — the state between submit and the Sponsor assigning a Manager; Update/Delete CRs skip it and start directly at `Pending Manager Approval`, since their target project already has a `ProjectManager` from its own original Create flow |
| `TargetItemID` | Number | SharePoint `ID` of the master row (Update/Delete only) — join key for live-diff lookups |
| `TargetProjectID` | Text | Business `ProjectID` snapshot for display (Update/Delete only) |
| `RequesterEmail` ⚠ | Text | `User().Email`; "my requests" filter key |
| `RequesterID` | Lookup→Employee List | `{Id, Value}` |
| `RequestReason` | Text (multiline) | **Required for Delete and Update** (enforced in app); Create no longer has a Request Notes field, so it's always blank on Create CRs |

### Proposed values (Create: all set · Update: only `ProjectDescription` + `Deliverables` + `EndDate` + `ProjectName` for display · Delete: all blank)

Same column names and **identical Choice value sets** as `Project_List` (keep them in sync — a `{Value:"X"}` not present in the target choice column fails the apply Patch):

| Column | Type |
|---|---|
| `ProjectName` | Text (master uses Title; CR needs its own column) |
| `ProjectDescription` | Text (multiline) |
| `Deliverables` | Text (multiline) — required at Create, editable on Update alongside `ProjectDescription`/`EndDate` |
| `ProjectType` | Choice — `New Product Development`, `Product Improvement`, `Regulatory & Compliance`, `Marketing Campaign`, `Market Expansion`, `Operational Improvement`, `IT & Digital`, `Research & Clinical`, `Other` |
| `Priority` | Choice — `Low`, `Medium`, `High`, `Critical` |
| `ProjectStatus` | Choice — `Not Started`, `In Planning`, `In Progress`, `On Hold`, `Completed`, `Cancelled`, `Deleted` |
| `CapexOpex` | Choice — `CapEx`, `OpEx`, `Mixed` |
| `Market` | Choice — `AU`, `MY`, `SG`, `VN` |
| `Channel` | Choice — `Retail Pharmacy`, `Hospital`, `Distributor`, `E-commerce`, `Direct-to-Consumer`, `Cross-border`, `Multiple` |
| `StrategicObjective` | Choice — `Revenue Growth`, `Market Expansion`, `Brand Building`, `Product Innovation`, `Operational Excellence`, `Regulatory Compliance`, `Cost Optimization` |
| `CostCenter` | Choice — `Office Melbourne (Head Quarter)`, `Port Melbourne Warehouse`, `Max Biocare Research Park - Natural Inspirations@Yinnar`, `Max Biocare Research Park - Mar-Nuka Bay`, `Malay Warehouse`, `Singapore Warehouse` |
| `Department` | Text (same convention as master — single value, options sourced from Employee List) |
| `ProjectSponsor` | Lookup→Employee List — set on the Create form (scoped to `Role.Value = "Sponsor"`). `ProjectOwner` is no longer written — see `Project_List.ProjectOwner` (deprecated) |
| `ProjectManager` | Lookup→Employee List — **blank on submit**; set by the `ProjectSponsor` right after, via `ApprovalsScreen`'s "Assign Manager" tab (`ApprovalStatus` flips `Pending Sponsor` → `Pending Manager Approval` in that same Patch). Picker sources the full `Employee List` (not scoped to `Project_User` `Role.Value = "Manager"`, unlike `ProjectSponsor` below — see 2026-07-23 model change) |
| `StartDate`, `EndDate` | Date |
| `BudgetAmount` | **Number** (not Currency) |
| `Currency` | Text — same convention as `Project_List.Currency` above; set at Create time from the form's `currency` control, carried through unchanged on Update/Delete CRs (not user-editable on those screens) |
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
| `Role` ⚠ | Choice | **`Sponsor` only (2026-07-23) — `Manager` removed from the value set.** There is no "Manager" role anywhere in this app anymore: `ProjectManager` is whoever the Sponsor picks from the full `Employee List` on `ApprovalsScreen`'s "Assign Manager" tab (`cmbAssignManager`) — any employee, not a `Project_User` account. No `Requester`/`Admin` value either; any tenant email can create change requests regardless of role. `ProjectSponsor` **is** the `Sponsor` role (picked on the Create form from active Sponsors) — see `Project_List.ProjectSponsor` above. Existing rows with the old `Manager` Choice value (from before 2026-07-23) are harmless leftovers — nothing in `Src/*.pa.yaml` or SharePoint permissions reads `Role.Value = "Manager"` — but remove `Manager` from the Choice column's defined value set going forward so it can't be picked again |
| `IsActive` ⚠ | Yes/No | default true |
| `EmployeeID` ⚠ | Lookup→Employee List | `{Id, Value}` (→Title) only — confirmed `EmployeeID.Email` does not resolve even with SharePoint's "additional columns" Lookup setting configured for `Email` and the Power Apps data source refreshed; use `LookUp('Employee List', ID = EmployeeID.Id).Email` for email instead |
| `Note` | Text | |

Resolved in `App.OnStart` → `gCurrentUser` / `gUserRole`. Employees absent here (or not `IsActive`) default to `gUserRole = "Requester"` — an internal-only value, never written to `Project_User.Role`, meaning "not an approver."

---

## `Project_ApprovalLog` — manager & sponsor decisions

Required ⚠: `ChangeRequestID`, `StepNumber`.

| Column | Type | Notes |
|---|---|---|
| `Title` | Text | `Step <n> - <action> - <CR title>` |
| `ChangeRequestID` ⚠ | Lookup→Project_ChangeRequests (→Title) | `{Id, Value}` |
| `ChangeRequestIDText` | Text | delegable join key = `Text(cr.ID)` |
| `StepNumber` ⚠ | Number | **`1` = Manager, `2` = Sponsor** (final approve; document in the column description) |
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

**Recipients are the CR's/project's specific `ProjectSponsor`/`ProjectManager`, not a role broadcast** (see `CLAUDE.md` §Role-based visibility for the full authorization model) — always wrapped in `Coalesce(LookUp('Employee List', ID = <...Id>).Email, "app.admin@maxbiocare.com")` so a blank/unassigned person never leaves the flow's "Send an email (V2)" action with an empty `To`.

| # | Trigger param (rename from Power Automate default) | Type | Meaning |
|---|---|---|---|
| 1 | `notificationType` (default `text`) | Text | `SubmittedToSponsor` \| `SubmittedToManager` \| `ManagerApprovedToSponsor` \| `FinalApproved` \| `FinalRejected` \| `FinalApprovedManagerCopy` \| `FinalRejectedManagerCopy` |
| 2 | `recipientEmails` (default `text_1`) | Text | recipient email — always exactly one recipient per call (each `notificationType` = one recipient + one template, see `approval-workflow-plan.md` §2a); the underlying "Send an email (V2)" action natively accepts a `;`-separated list, but the app never relies on that |
| 3 | `requestTitle` (default `text_2`) | Text | change request title |
| 4 | `requestId` (default `number`) | Number | change request ID |
| 5 | `requestType` (default `text_3`) | Text | request type (`Create`/`Update`/`Delete`) |
| 6 | `projectName` (default `text_4`) | Text | project name |
| 7 | `requesterName` (default `text_5`) | Text | requester display name |
| 8 | `actionByName` (default `text_6`) | Text | action-by display name |
| 9 | `remark` (default `text_7`) | Text | remark (`""` when N/A — never omit) |

Rename each "Ask in PowerApps" input to the name above when building the flow (Power Automate defaults to `text`/`text_1`/`number`/... if left unrenamed). Power Apps still calls `Project_Notify.Run(...)` **positionally** in this exact order.

**Renaming does NOT change the key used inside the flow body.** It only relabels the input in the designer and in the Power Apps formula-bar intellisense hint. Every expression inside the flow itself (the `Switch`'s `On` field, both `Set variable` actions, the email templates) must still reference the field by its **original default key**, with the safe-navigation `?` operator — e.g. `@{triggerBody()?['text_2']}` for `requestTitle` (the 3rd input added), never `@{triggerBody()?['requestTitle']}`. See `docs/approval-workflow-plan.md` §3 for the full default-key mapping and `flows/Project_Notify/templates/` for templates already written against the correct keys.

Flow body: **Switch on `notificationType`** (not nested If) → each case sets `Subject`/`Body` variables → single "Send an email (V2)" with To = `recipientEmails`. Switch **Default → Terminate (Failed, "Unknown notificationType")**. Full email templates in `docs/approval-workflow-plan.md`.

### `Project_SyncActualCost` — FUTURE (spec placeholder, not built)

Accumulates `ActualCost` on `Project_List` from procurement spend. Originally blocked on adding a `ProjectID` (Text) join column to the procurement lists (`Procurement_Requests` / `Procurement_InvoiceData`) — join key TBD. On completion it should notify the Project Manager (`ActualCostUpdated` case to be added to `Project_Notify`). See `docs/approval-workflow-plan.md` §Future.

> **Update (2026-07-23):** `ProjectID` (Text) has since been added directly to `Procurement_InvoiceData` on SharePoint (ahead of this doc — confirmed live in the SharePoint list UI, column present though unpopulated on older rows). This flow itself is still **not built** — but `ViewProjectScreen`'s "Actual Cost — Procurement Invoices" section now does a **live read** of `Procurement_InvoiceData` filtered by `ProjectID = gSelectedProject.ProjectID` directly in the app (no batch/flow involved), showing a per-invoice table (Title/TotalAmount/Currency) plus a total-by-currency summary (`GroupBy`/`AddColumns`/`Sum` over the already-filtered result). `Project_List.ActualCost` itself is still never written by the app (see `Project_List.ActualCost` above) — this live-read table is a separate, additive display, not a replacement for the eventual sync flow. Requires `Procurement_InvoiceData` added as its own connected data source in this app (cross-app list, not one of this app's native 6) — see `CLAUDE.md`'s data-source note. Older `Procurement_InvoiceData` rows created before this column existed have it blank and won't match any project.
