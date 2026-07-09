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
| `Project_ChangeRequests` | **Edit** — broadened from the original "Add Items + Read only" design specifically so that any employee picked as a `ProjectSponsor` (not a `Project_User`-gated role, see `Project_List.ProjectSponsor` below) can `Patch` `ProjectManager` + `ApprovalStatus` at Step 0 ("Assign Manager") without needing a separate grant | Edit (needed to Patch `ApprovalStatus` + write the Step 1 log) | Edit (needed to Patch `ApprovalStatus` at Step 2) |
| `Project_ApprovalLog` | Read only | **Add Items + Read only** — the app only ever inserts new log rows, never edits one | **Add Items + Read only** |
| `Project_List` | Read only | Read only | Edit (needed to Patch the apply step) |

**"Add Items + Read only" is not a SharePoint built-in level** — create a custom Permission Level (Site Settings → Permission Levels → copy `Read`, additionally check `Add Items`, leave `Edit Items`/`Delete Items` unchecked). Still used for `Project_ApprovalLog` (Manager/Executive can insert log rows but never edit one); no longer used for `Project_ChangeRequests`/Requester since that was broadened to plain `Edit` — a built-in level, nothing custom to create there.

**Accepted trade-off — every user, not just Manager/Executive, now has raw `Edit` on `Project_ChangeRequests`.** This was a deliberate choice (not a default) to solve the Sponsor permission gap: since `ProjectSponsor` can be any employee rather than someone in a dedicated `Project_User`/SharePoint group, there was no way to grant *only* the right people `Edit` for the "Assign Manager" step without granting it to everyone. The cost: the tamper-protection that "Add Items + Read only" gave Requesters — blocking them from editing any CR, their own or anyone else's, once submitted — is gone for **all** users, not only Sponsors. Anyone with SharePoint UI access can now hand-edit any CR's fields directly (e.g. flip `ApprovalStatus`, rewrite proposed values) bypassing the app's guarded `Patch` formulas entirely. Revisit if this turns out to matter in practice — the narrower fix (Power Automate flow with a locked-down connection for just the Step 0 Patch, mirrored on the residual-risk note below) is still available later.

**Known residual risk — cannot be fully closed without an architecture change:** Executive needs raw `Edit Items` on `Project_List` (everyone else stays Read-only there), because `Patch()` always executes under the acting user's own identity. An Executive with SharePoint UI access could in principle bypass the app's guarded formulas and edit `Project_List` directly (the `Project_ChangeRequests` version of this risk is no longer Manager/Executive-specific — see the accepted trade-off above, which extends it to everyone). Closing this fully would mean moving the apply-step `Patch` into a Power Automate flow with a locked-down "Run only users" connection (the flow's own connection does the write, not the human's) — a deliberate, larger architecture change, not something list permissions alone can fix. Not pursued unless explicitly requested.

---

## `Project_List` — master project record (approved data only)

Required ⚠: `ProjectStatus`, `ApprovalStatus`, `ProjectLevel`.

| Column (internal) | Type | Notes |
|---|---|---|
| `Title` | Text | **Project Name** (reuses system Title) |
| `ProjectID` | Text | Unique business key, e.g. `PROJ-AU-QA-2026-017` (`PROJ-<MarketCode>-<DeptCode>-<Year>-<seq>`); generated by the 2nd Patch at apply time (see CLAUDE.md — ID generation) |
| `ProjectDescription` | Text (multiline) | Detailed scope |
| `Deliverables` | Text (multiline) | Concrete outputs/results the project is expected to produce — required at Create; editable via Update alongside `ProjectDescription`/`EndDate` |
| `ProjectType` | Choice | `New Product Development`, `Product Improvement`, `Regulatory & Compliance`, `Marketing Campaign`, `Market Expansion`, `Operational Improvement`, `IT & Digital`, `Research & Clinical`, `Other` |
| `Department` | Text | Single department name; picker options come from `Choices('Employee List'.department)`, not a Choice column itself — see `.department` on Employee List |
| `ProjectOwner` | Lookup→Employee List | Business owner, `{Id, Value}` (→Title). Picker on `CreateProjectScreen` is scoped to active `Project_User` accounts with `Role.Value = "Executive"` (`ForAll(Filter(Project_User, IsActive && Role.Value = "Executive"), {ID: EmployeeID.Id, Title: EmployeeID.Value})` — reshapes the already-available `EmployeeID` fields, no secondary `LookUp` needed) — only Executives are selectable. Does Step 2 approval |
| `ProjectSponsor` | Lookup→Employee List | Owns the project's outcome — **not** a supervisory role — `{Id, Value}` (→Title). **Not gated by a `Project_User` role, unlike `ProjectOwner`/`ProjectManager`** — the picker on `CreateProjectScreen` sources directly from `Employee List` (`Sort('Employee List', Title, SortOrder.Ascending)`), so any employee can be picked, not just accounts with a `Project_User` row. Set at Create time; sole job is picking the `ProjectManager` on the CR right after submit (`ApprovalsScreen`'s "Assign Manager" tab) — takes no further part in the approval chain itself. `Project_ChangeRequests` permissions were broadened to plain `Edit` for everyone specifically so any Sponsor can complete that Patch — see §SharePoint list permissions |
| `ProjectManager` | Lookup→Employee List | Execution lead / supervisor, `{Id, Value}` (→Title). Scoped to `Role.Value = "Manager"`. **Not chosen on `CreateProjectScreen`** — blank on a newly-created row's originating CR until the `ProjectSponsor` assigns it (see `ApprovalStatus` below); always populated by the time a row reaches `Project_List` (apply only happens after Step 2, by which point a Manager was assigned and Step 1 completed). Does Step 1 approval |
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
| `ApprovalStatus` ⚠ | Choice | `Pending Sponsor` → `Pending Manager` → `Pending Executive` → `Approved` \| `Rejected` (exact strings — used as literals across all screens). **`Pending Sponsor` exists only on Create CRs** — the state between submit and the `ProjectSponsor` assigning a `ProjectManager` (`ApprovalsScreen`'s "Assign Manager" tab); Update/Delete CRs skip it and start directly at `Pending Manager`, since their target project already has a `ProjectManager` from its own original Create flow |
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
| `ProjectOwner`, `ProjectSponsor` | Lookup→Employee List — both set on the Create form |
| `ProjectManager` | Lookup→Employee List — **blank on submit**; set by the `ProjectSponsor` right after, via `ApprovalsScreen`'s "Assign Manager" tab (`ApprovalStatus` flips `Pending Sponsor` → `Pending Manager` in that same Patch) |
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
| `Role` ⚠ | Choice | `Manager`, `Executive` — no `Requester`/`Admin`/`Sponsor` value; any tenant email can create change requests regardless of role. `ProjectSponsor` is deliberately **not** a `Project_User` role — see `Project_List.ProjectSponsor` above |
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

**Recipients are the CR's/project's specific `ProjectSponsor`/`ProjectManager`/`ProjectOwner`, not a role broadcast** (see `CLAUDE.md` §Role-based visibility for the full authorization model) — always wrapped in `Coalesce(LookUp('Employee List', ID = <...Id>).Email, "app.admin@maxbiocare.com")` so a blank/unassigned person never leaves the flow's "Send an email (V2)" action with an empty `To`.

| # | Trigger param (rename from Power Automate default) | Type | Meaning |
|---|---|---|---|
| 1 | `notificationType` (default `text`) | Text | `SubmittedToSponsor` \| `SubmittedToManager` \| `ManagerApprovedToExecutive` \| `FinalApproved` \| `FinalRejected` \| `FinalApprovedManagerCopy` \| `FinalRejectedManagerCopy` |
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

Accumulates `ActualCost` on `Project_List` from procurement spend. Blocked on adding a `ProjectID` (Text) join column to the procurement lists (`Procurement_Requests` / `Procurement_InvoiceData`) — join key TBD. On completion it should notify the Project Manager (`ActualCostUpdated` case to be added to `Project_Notify`). See `docs/approval-workflow-plan.md` §Future.
