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
| `Project_User` | Roles: Requester / Manager / Executive / Admin (absent ⇒ Requester) |
| `Project_ApprovalLog` | Decision log — StepNumber **1 = Manager, 2 = Executive** |
| `Employee List` | Shared staff directory (existing; internal names `email`/`department` lowercase) |
| `Product_Database_SKU_Master` | Product catalog (existing, read-only) — `RelatedSKU` lookup target |

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
DeptCode = `Switch(First(Department).Value, "Quality Assurance","QA", "Regulatory Affairs","RA", "Marketing","MK", "Sales","SL", "Supply Chain","SC", "Finance","FN", "IT","IT", "R&D","RD", "HR","HR", "Operations","OP", "GN")`.
The numeric part is a global sequence (not per-department/year) — accepted trade-off; count-based schemes race and hit delegation limits. If the app dies between the two Patches the row has a blank ProjectID — the gallery shows `PROJ-PENDING-<ID>` via `Coalesce`; fix by patching ProjectID manually.

## Global state (App.OnStart)

`gCurrentEmployee` (Employee List row by `User().Email`) · `gCurrentUser`/`gUserRole` (Project_User, fallback Requester) · `gIsApprover` · `gSelectedProject` (gallery → Update/Delete screens) · `gSelectedCR` + `gLiveTarget` (Approvals master-detail; `gLiveTarget` = live master row for diff) · `gShowRejectPanel` · `gShowDeleted` · `gStatusFilter` · `gLevelFilter` · `gApprovalsTab` · `gHasPendingCR` · `gAppReady`.

## Role-based visibility

| Role | Sees |
|---|---|
| Requester (default) | All Projects; own change requests (Mine tab) |
| Manager | + Approvals queue filtered `Pending Manager` (acts as Step 1) |
| Executive | + Approvals queue filtered `Pending Executive` (acts as Step 2 + applies) |
| Admin | + Both queues, can act at either step; History tab; Show-deleted toggle |

## Conventions

- **UI text and all source: English only.**
- Guarded writes: `With({wX: Patch(...)}, If(IsBlank(wX.ID), Notify(error), <next step>))` — chain nested `With` for multi-step (log → status → notify).
- Choice: write `{Value: "..."}`; multi-choice: table of `{Value}` records; Lookup/Person: `{Id, Value}`; Currency/Number: plain numbers.
- Status strings (`Pending Manager`, `Pending Executive`, `Approved`, `Rejected`, `Deleted`, …) are **literals in every screen** — no shared constant; renaming one means touching every filter, color map, and Patch.
- Join key for logs: `ChangeRequestIDText = Text(cr.ID)` (delegable), never lookup-column comparison.
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
1. Create the 4 SharePoint lists per `docs/sharepoint-schema.md` — Choice value sets on `Project_ChangeRequests` must exactly match `Project_List`.
2. Seed `Project_User` with at least one account per role (Requester/Manager/Executive/Admin).
3. Build the `Project_Notify` flow per `docs/approval-workflow-plan.md` §3 (declare all 9 trigger input types explicitly; Switch with Default → Terminate; pin the `app.admin@maxbiocare.com` connection in Run-only users).
4. Create the canvas app in Power Apps Studio, add the 6 data sources + the flow, then paste the `Src/*.pa.yaml` screens (order: App OnStart, then screens). Verify the exact modern-control version Studio assigns to `DatePicker`/`Toggle` and add it to `.claude/pa-yaml-rules.md`.
5. Verify two unconfirmed schema points before wiring: the display/Title column of `Product_Database_SKU_Master`, and Employee List internal name `email` (lowercase).
6. Run the E2E test checklist: create → 2-step approve → row created with generated ProjectID + 2 log rows; update diff + apply; delete (reason required, soft delete, restore via Show Deleted); reject at each step with remark; duplicate-pending guard; child-project root picker.
7. Future phase (spec only, do not build yet): `Project_SyncActualCost` flow — blocked on adding a `ProjectID` join column to the procurement lists; then add an `ActualCostUpdated` case to `Project_Notify` (notify Project Manager).

## Known caveats

- `ProjectStatus.Value <> "Deleted"` raises a delegation warning on SharePoint choice columns — acceptable while the list stays well under 2000 rows.
- Per-row `LookUp` badges (pending-change on All Projects, manager trail on Approvals) are N+1 queries — fine at portfolio scale; revisit past ~500 rows.
- Choice value sets are duplicated between `Project_List` and `Project_ChangeRequests` and must stay identical, or the apply Patch fails.
- `Product_Database_SKU_Master` schema and Employee List internal names must be verified in SharePoint before first Studio wiring.
- DatePicker/Toggle modern-control versions are not yet in `pa-yaml-rules.md` — verify in Studio on first paste and add them.
