# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Max Biocare Project List — Canvas App

Power Apps canvas app for managing the company project portfolio on SharePoint, with **2-level approval (Manager → Executive) for every data change**. No Create/Update/Delete touches the master list until fully approved.

## What this repo is

Unpacked canvas source (`Src/*.pa.yaml`) + design docs. There is **no `.msapp` here and the Src tree is not repackable** — screens are authored/edited as YAML and pasted into Power Apps Studio (or reviewed against Studio's YAML view). Same workflow as the sibling `procurement-procedure` repo.

## Working in this repo

- **No build, lint, or test commands.** There is no package manager or toolchain. All UI text and source code must be English.
- Verification happens in Power Apps Studio: paste a `Src/*.pa.yaml` screen into Studio's YAML view and confirm zero `PA1001`/`PA2108` errors. Before pasting, the app must already have the 6 data sources (`Project_List`, `Project_ChangeRequests`, `Project_User`, `Project_ApprovalLog`, `Employee List`, `Product_Database_SKU_Master`) and the `Project_Notify` flow added, or every reference errors.
- A quick local syntax check: parse the YAML with any YAML library (e.g. Python + PyYAML). Known false positive: `Label: =` (a CheckBox's empty formula) trips strict parsers on the `=` value tag — it is valid Power Apps YAML.
- Editing `.pa.yaml` files: **`.claude/pa-yaml-rules.md` is binding** — read it first. The two most common mistakes: a formula string containing `: ` (colon + space) must use the `|-` block literal, and AutoLayout children with an explicit `Height` need `FillPortions: =0`.
- End-to-end test checklist (create → 2-step approve → apply, reject with remark, soft delete, duplicate-pending guard, child-project rules) lives in the approved plan and mirrors `docs/approval-workflow-plan.md` §2 guards.

```
project-list/
├── CLAUDE.md                       ← this file
├── .claude/pa-yaml-rules.md        ← Power Apps YAML gotchas (control versions, YAML pitfalls) — READ BEFORE EDITING ANY .pa.yaml
├── docs/
│   ├── sharepoint-schema.md        ← authoritative data model (4 lists + flow trigger args)
│   └── approval-workflow-plan.md   ← state machine, Project_Notify flow spec, email templates
└── Src/
    ├── _EditorState.pa.yaml        ← screen order
    ├── App.pa.yaml                 ← App.OnStart global state
    ├── AllProjectsScreen.pa.yaml   ← home: approved projects gallery
    ├── CreateProjectScreen.pa.yaml ← full form → Create change request
    ├── UpdateProjectScreen.pa.yaml ← Description + End Date only → Update change request
    ├── DeleteProjectScreen.pa.yaml ← confirmation + reason → Delete change request
    └── ApprovalsScreen.pa.yaml     ← shared approval queue (role-filtered) + apply logic
```

## Backend (SharePoint site `maxbiocare.sharepoint.com/sites/Powerapps`)

| List | Role |
|---|---|
| `Project_List` | Master, approved data only — written ONLY by the Executive-approve handler |
| `Project_ChangeRequests` | Staging: every Create/Update/Delete intent + the approval state machine |
| `Project_User` | Roles: Manager / Executive only (absent or inactive ⇒ internal `Requester` — not an approver, but can still create change requests) |
| `Project_ApprovalLog` | Decision log — StepNumber **1 = Manager, 2 = Executive** |
| `Employee List` | Shared staff directory (existing; internal names `email`/`department` lowercase) |
| `Product_Database_SKU_Master` | Product catalog (existing, read-only) — `RelatedSKU` lookup target (multi-select) |

Flow: `Project_Notify` (PowerAppsV2, 9 positional args) — email on submit / manager-approve / final result. Spec + templates in `docs/approval-workflow-plan.md`.

## Workflow state machine (`Project_ChangeRequests.ApprovalStatus`)

```
Pending Manager ──approve──▶ Pending Executive ──approve──▶ APPLY to Project_List ─▶ Approved
      │ reject (Remark ⚠)            │ reject (Remark ⚠)      │ apply fails
      ▼                              ▼                        ▼
   Rejected                       Rejected            stays Pending Executive (retry)
```

- **Apply-before-flip**: on Executive approve, the master Patch runs FIRST; the CR becomes `Approved` only if it succeeds. Never mark a CR Approved before its change is applied.
- Apply per type — Create: new master row (+ 2nd Patch for ProjectID) · Update: only `ProjectDescription` + `EndDate` · Delete: soft delete `ProjectStatus = "Deleted"`.
- Guards: one pending CR per target project; a level-0 project with active children cannot be deleted; Reject requires Remark.

## ProjectID generation (race-free)

Generated at apply time from the new row's SharePoint ID (two-step Patch):
`"PROJ-" & <DeptCode> & "-" & Text(Year(Today())) & "-" & Text(wNew.ID, "000")`
DeptCode = `Switch(Department, "Quality Assurance","QA", "Regulatory Affairs","RA", "Marketing","MK", "Sales","SL", "Supply Chain","SC", "Finance","FN", "IT","IT", "R&D","RD", "HR","HR", "Operations","OP", "GN")`. `Department` is a single Text value (see `docs/sharepoint-schema.md`), not a Choice column.
The numeric part is a global sequence (not per-department/year) — accepted trade-off; count-based schemes race and hit delegation limits. If the app dies between the two Patches the row has a blank ProjectID — the gallery shows `PROJ-PENDING-<ID>` via `Coalesce`; fix by patching ProjectID manually.

## Global state (App.OnStart)

`gCurrentEmployee` (Employee List row by `User().Email`) · `gCurrentUser`/`gUserRole` (Project_User, fallback Requester) · `gIsApprover` · `gApprovalsPendingCount` (see below) · `gSelectedProject` (gallery → Update/Delete screens) · `gSelectedCR` + `gLiveTarget` (Approvals master-detail; `gLiveTarget` = live master row for diff) · `gShowRejectPanel` · `gShowDeleted` · `gStatusFilter` · `gLevelFilter` · `gApprovalsTab` · `gHasPendingCR` · `gAppReady`.

- **`CountRows` against SharePoint is never delegable, full stop** — Studio flags the delegation warning on the formula itself regardless of which property hosts it (`Text`, `OnStart`, `OnVisible`, `OnSelect` all show it identically; moving the call between them does not suppress it). `gApprovalsPendingCount` is computed with `CountRows(Filter(Project_ChangeRequests, ...))` in `App.OnStart` and refreshed in `AllProjectsScreen.OnVisible` purely to avoid re-querying on every render (perf/freshness), **not** to dodge the warning — the warning is expected and accepted, same as the other delegation caveats below. Accurate only within the app's non-delegable query row limit (default 500, raisable to 2000 in Advanced Settings).

## Role-based visibility

Anyone with an email in the tenant can create change requests, including Managers and Executives — there is no separate "Requester" role in `Project_User`; it is only the internal default `gUserRole` for anyone with no active `Project_User` row.

| Role | Sees |
|---|---|
| Requester (default, no `Project_User` row) | All Projects; own change requests (Mine tab) |
| Manager | + Approvals queue filtered `Pending Manager` (acts as Step 1) |
| Executive | + Approvals queue filtered `Pending Executive` (acts as Step 2 + applies); History tab; Show-deleted toggle |

## Conventions

- **UI text and all source: English only.**
- Guarded writes: `With({wX: Patch(...)}, If(IsBlank(wX.ID), Notify(error), <next step>))` — chain nested `With` for multi-step (log → status → notify).
- Choice: write `{Value: "..."}`; multi-choice: table of `{Value}` records; Lookup/Person: `{Id, Value}`; multi-lookup (`RelatedSKU`): table of `{Id, Value}` records; Currency/Number: plain numbers. Exception: `Department` is plain Text (options sourced from `Choices('Employee List'.department)` — the Choice column's defined value set, not the distinct values actually in use), not a Choice column itself.
- Status strings (`Pending Manager`, `Pending Executive`, `Approved`, `Rejected`, `Deleted`, …) are **literals in every screen** — no shared constant; renaming one means touching every filter, color map, and Patch.
- Join key for logs: `ChangeRequestIDText = Text(cr.ID)` (delegable), never lookup-column comparison.
- `Project_User.Title` is never populated — always resolve a user's display name/email through its `EmployeeID` lookup into `Employee List`, never read `Project_User.Title` directly.
- `Project_Notify.Run(...)` recipient args are always `Coalesce(Concat(Filter(Project_User, Role.Value = "..." && IsActive), ..., ";"), "app.admin@maxbiocare.com")` — an empty active-approver Filter makes `Concat` return `""`, which fails the flow's "Send an email" action on an empty `To`.
- Inline `RGBA(...)` colors; brand purple `RGBA(83, 74, 183, 1)`; status pill palette mirrors procurement (green `RGBA(15,110,86,1)` / red `RGBA(163,45,45,1)` / amber `RGBA(133,79,11,1)`).
- Control versions and YAML pitfalls: **`.claude/pa-yaml-rules.md` is binding** (`Label@2.5.1`, `Button@0.0.45`, `TextInput@0.0.54`, `Radio@0.0.25` no Default, `ComboBox@0.0.51` DefaultSelectedItems=Filter, `CheckBox@0.0.30`, `Gallery@2.15.0`, `GroupContainer@1.5.0`, `Rectangle@2.3.0`; `|-` block literal for any formula containing `: `; `FillPortions: =0` for AutoLayout children with explicit Height).
- `ActualCost` is display-only everywhere — reserved for the future procurement cost-sync flow (`docs/approval-workflow-plan.md` §Future).

## Project status (last updated: 2026-07-06)

### Done — repo authoring phase complete
- All docs written: `docs/sharepoint-schema.md` (4 lists + `Project_Notify` 9-arg table), `docs/approval-workflow-plan.md` (state machine, flow build spec, 4 English email templates, future cost-sync placeholder), this CLAUDE.md, `.claude/pa-yaml-rules.md` (copied from procurement-procedure).
- All 7 `Src/*.pa.yaml` files written and YAML-validated: `_EditorState`, `App` (OnStart role resolution), `AllProjectsScreen` (filters, search, pending-change badge, approvals counter), `CreateProjectScreen` (full form, Actual Cost read-only, Root/Child hierarchy picker), `UpdateProjectScreen` (Description + End Date only, pending-CR guard), `DeleteProjectScreen` (reason + confirm required, active-children guard), `ApprovalsScreen` (master-detail, update diff view, manager/executive approve chains with apply-before-flip, reject with required remark).
- Cross-checked: column names in formulas match the schema doc; status strings consistent; 8 `Project_Notify.Run` call sites all pass 9 args.
- Decisions locked with the user: staging list `Project_ChangeRequests`; single shared Approvals screen; Update limited to Description + End Date; soft delete; **Budget Approved column dropped** (Budget Amount only); SKU lookup target = `Product_Database_SKU_Master`; emails on submit / manager-approve / final result.

### Next — manual setup outside this repo (in order)
1. Create the 4 SharePoint lists per `docs/sharepoint-schema.md` — Choice value sets on `Project_ChangeRequests` must exactly match `Project_List`. Apply the permission model in `docs/sharepoint-schema.md` §SharePoint list permissions (custom "Add Items only" level on `Project_ChangeRequests`/`Project_ApprovalLog` for non-approvers; Read-only on `Project_List`).
2. Seed `Project_User` with at least one account per role (Manager/Executive).
3. Build the `Project_Notify` flow per `docs/approval-workflow-plan.md` §3 (declare all 9 trigger input types explicitly; Switch with Default → Terminate; pin the `app.admin@maxbiocare.com` connection in Run-only users).
4. Create the canvas app in Power Apps Studio, add the 6 data sources + the flow, then paste the `Src/*.pa.yaml` screens (order: App OnStart, then screens). Verify the exact modern-control version Studio assigns to `DatePicker`/`Toggle` and add it to `.claude/pa-yaml-rules.md`.
5. Verify two unconfirmed schema points before wiring: the display/Title column of `Product_Database_SKU_Master`, and Employee List internal name `email` (lowercase).
6. Run the E2E test checklist: create → 2-step approve → row created with generated ProjectID + 2 log rows; update diff + apply; delete (reason required, soft delete, restore via Show Deleted); reject at each step with remark; duplicate-pending guard; child-project root picker.
7. Future phase (spec only, do not build yet): `Project_SyncActualCost` flow — blocked on adding a `ProjectID` join column to the procurement lists; then add an `ActualCostUpdated` case to `Project_Notify` (notify Project Manager).

## Known caveats

- This SharePoint connector delegates neither `<>` nor `Not(...)` — never write `Col.Value <> "X"` or `Not(Col.Value = "X")` against a data source. Instead OR together `=` comparisons against every value you DO want (e.g. exclude `Deleted` by writing `Col.Value = "Not Started" || Col.Value = "In Planning" || ...` for every non-Deleted status). Verbose, but `=`/`&&`/`||` are the only combination confirmed delegable here.
- Per-row `LookUp` badges (pending-change on All Projects, manager trail on Approvals) are N+1 queries — fine at portfolio scale; revisit past ~500 rows.
- Recipient email resolution (`Project_Notify` calls) uses `LookUp('Employee List', ID = EmployeeID.Id).Email` per active Manager/Executive — another accepted N+1 pattern, and this is final: `Project_User.EmployeeID` only ever exposes `.Id`/`.Value` (→Title) in this app's connection. SharePoint's Lookup column "additional columns" setting was configured for `EmployeeID` (Email checked) and the `Project_User` data source was refreshed in Studio — `EmployeeID.Email` still doesn't resolve (`PA2108`, "Name isn't valid. 'Email' isn't recognized"). Do not retry this; always use the `LookUp` form for any field beyond `.Id`/`.Value` on this column.
- Choice value sets are duplicated between `Project_List` and `Project_ChangeRequests` and must stay identical, or the apply Patch fails. (`Department` is the exception — plain Text on both, no Choice value set to sync.)
- `Product_Database_SKU_Master` schema and Employee List internal names must be verified in SharePoint before first Studio wiring.
- DatePicker/Toggle modern-control versions are not yet in `pa-yaml-rules.md` — verify in Studio on first paste and add them.
- Manager/Executive need raw `Edit Items` on `Project_ChangeRequests`/`Project_List` for their approve-step `Patch` to work — they could in principle bypass the app's guarded formulas by editing those lists directly in SharePoint. Not closable with list permissions alone (see `docs/sharepoint-schema.md` §SharePoint list permissions); would need moving the apply step into a Power Automate flow with a locked-down connection.
