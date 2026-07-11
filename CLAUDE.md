# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Max Biocare Project List — Canvas App

Power Apps canvas app for managing the company project portfolio on SharePoint, with **2-level approval (Manager → Sponsor) for every data change**. No Create/Update/Delete touches the master list until fully approved.

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
│   └── approval-workflow-plan.md   ← state machine, Project_Notify flow spec, email templates (content spec)
├── flows/Project_Notify/
│   ├── README.md                   ← how to paste the HTML into Power Automate's Send-an-email Body
│   ├── BUILD_GUIDE.md              ← click-by-click steps to build the whole flow from scratch in the portal
│   └── templates/*.html            ← ready-to-paste HTML body per notificationType case (6 files)
└── Src/
    ├── _EditorState.pa.yaml        ← screen order
    ├── App.pa.yaml                 ← App.OnStart global state
    ├── AllProjectsScreen.pa.yaml   ← home: approved projects gallery
    ├── CreateProjectScreen.pa.yaml ← full form → Create change request
    ├── ViewProjectScreen.pa.yaml   ← read-only project details (gallery row tap + View button)
    ├── UpdateProjectScreen.pa.yaml ← Description + Deliverables + End Date only → Update change request (owner-only)
    ├── DeleteProjectScreen.pa.yaml ← confirmation + reason → Delete change request
    └── ApprovalsScreen.pa.yaml     ← shared approval queue (role-filtered) + apply logic
```

## Backend (SharePoint site `maxbiocare.sharepoint.com/sites/Powerapps`)

| List | Role |
|---|---|
| `Project_List` | Master, approved data only — written ONLY by the Sponsor final-approve handler |
| `Project_ChangeRequests` | Staging: every Create/Update/Delete intent + the approval state machine |
| `Project_User` | Roles: Manager / Sponsor only (absent or inactive ⇒ internal `Requester` — not an approver, but can still create change requests). `ProjectSponsor` is picked on the Create form from active `Sponsor` accounts |
| `Project_ApprovalLog` | Decision log — StepNumber **1 = Manager, 2 = Sponsor** |
| `Employee List` | Shared staff directory (existing; internal names `email`/`department` lowercase) |
| `Product_Database_SKU_Master` | Product catalog (existing, read-only) — `RelatedSKU` lookup target (multi-select) |

Flow: `Project_Notify` (PowerAppsV2, 9 positional args) — email on submit / sponsor-assigns-manager / manager-approve / final result. Spec + templates in `docs/approval-workflow-plan.md`.

## Workflow state machine (`Project_ChangeRequests.ApprovalStatus`)

```
(Create only) Pending Sponsor ──Sponsor assigns Manager──▶ Pending Manager Approval ──approve──▶ Pending Sponsor Approval ──approve──▶ APPLY to Project_List ─▶ Approved
                                                                  │ reject (Remark ⚠)                 │ reject (Remark ⚠)           │ apply fails
                                                                  ▼                                   ▼                             ▼
                                                               Rejected                            Rejected           stays Pending Sponsor Approval (retry)
```

The **Sponsor acts at both ends** of a Create CR: first at Step 0 (`Pending Sponsor` — assign a `ProjectManager`), then again at Step 2 (`Pending Sponsor Approval` — final approve + apply). The old separate "Executive/Owner" role was merged into the Sponsor: there is no `ProjectOwner` anymore. The two Sponsor states have distinct names so the "Assign Manager" tab and the final-approval queue never collide.

Update/Delete CRs skip `Pending Sponsor` entirely and start at `Pending Manager Approval` — the target project already has a `ProjectManager` from its own Create flow.

- **`ProjectManager` is never chosen on the Create form.** The requester picks a `ProjectSponsor` instead (owns the project's outcome, not a supervisor); the CR sits at `Pending Sponsor` until that Sponsor opens `ApprovalsScreen`'s "Assign Manager" tab, picks a `ProjectManager`, and submits — which is what flips the CR to `Pending Manager Approval` and starts the approval chain proper. No log row is written for this step (it's an assignment, not an approve/reject decision).
- **Apply-before-flip**: on the Sponsor's Step 2 final approve, the master Patch runs FIRST; the CR becomes `Approved` only if it succeeds. Never mark a CR Approved before its change is applied.
- Apply per type — Create: new master row (+ 2nd Patch for ProjectID) · Update: only `ProjectDescription` + `Deliverables` + `EndDate` · Delete: soft delete `ProjectStatus = "Deleted"`.
- Guards: one pending CR per target project; a level-0 project with active children cannot be deleted; Reject requires Remark.

## ProjectID generation (race-free)

Generated at apply time from the new row's SharePoint ID (two-step Patch):
`"PROJ-" & <MarketCode> & "-" & <DeptCode> & "-" & Text(Year(Today())) & "-" & Text(wNew.ID, "000")` → e.g. `PROJ-AU-GN-2026-002`.
MarketCode = `gSelectedCR.Market.Value` used verbatim — the `Market` Choice values are already the 2-letter codes `AU`/`MY`/`SG`/`VN` (see `docs/sharepoint-schema.md`), so no mapping is needed. `Market` is required on Create, so it is always present.
DeptCode = `Switch(Department, "Quality Assurance","QA", "Regulatory Affairs","RA", "Marketing","MK", "Sales","SL", "Supply Chain","SC", "Finance","FN", "IT","IT", "R&D","RD", "HR","HR", "Operations","OP", "GN")`. `Department` is a single Text value (see `docs/sharepoint-schema.md`), not a Choice column.
The numeric part is a global sequence (not per-department/year) — accepted trade-off; count-based schemes race and hit delegation limits. If the app dies between the two Patches the row has a blank ProjectID — the gallery shows `PROJ-PENDING-<ID>` via `Coalesce`; fix by patching ProjectID manually.

## Global state (App.OnStart)

`gCurrentEmployee` (Employee List row by `User().Email`) · `gCurrentUser`/`gUserRole` (Project_User, fallback Requester) · `gIsApprover` · `gApprovalsPendingCount` (see below) · `gSelectedProject` (gallery → View/Update/Delete screens) · `gSelectedCR` + `gLiveTarget` (Approvals master-detail; `gLiveTarget` = live master row for diff) · `gShowRejectPanel` · `gShowDeleted` · `gStatusFilter` · `gLevelFilter` · `gApprovalsTab` · `gHasPendingCR` · `gAppReady`.

- **`CountRows` against SharePoint is never delegable, full stop** — Studio flags the delegation warning on the formula itself regardless of which property hosts it (`Text`, `OnStart`, `OnVisible`, `OnSelect` all show it identically; moving the call between them does not suppress it). `gApprovalsPendingCount` is computed with `CountRows(Filter(Project_ChangeRequests, ...))` in `App.OnStart` and refreshed in `AllProjectsScreen.OnVisible` purely to avoid re-querying on every render (perf/freshness), **not** to dodge the warning — the warning is expected and accepted, same as the other delegation caveats below. Accurate only within the app's non-delegable query row limit (default 500, raisable to 2000 in Advanced Settings).

## Role-based visibility

Anyone with an email in the tenant can create change requests, including Managers and Sponsors — there is no separate "Requester" role in `Project_User`; it is only the internal default `gUserRole` for anyone with no active `Project_User` row. **`ProjectSponsor` is now a `Project_User` role** (`Role.Value = "Sponsor"`, renamed from the former "Executive") — `cmbSponsor` on `CreateProjectScreen` sources only active Sponsors, so a project's Sponsor is always someone who holds that role, same guarantee as `ProjectManager`. The Sponsor plays **two** parts in a Create CR's lifecycle: Step 0 (assign the Manager) and Step 2 (final approve + apply). The former "Owner/Executive" second-approver was merged into the Sponsor — there is no `ProjectOwner` anymore.

| Role | Sees |
|---|---|
| Requester (default, no `Project_User` row) | All Projects; own change requests (Mine tab); the "Assign Manager" tab and "Approvals (n)" button too, if currently assigned as some CR's `ProjectSponsor` |
| Sponsor | + "Assign Manager" tab filtered `Pending Sponsor` **AND** assigned as that CR's `ProjectSponsor` (Step 0, Create only — picks the `ProjectManager` and submits); + Approvals queue filtered `Pending Sponsor Approval` **AND** assigned as that CR's/project's `ProjectSponsor` (Step 2 — final approve + applies); History tab, scoped to projects where they're `ProjectSponsor`; Show-deleted toggle |
| Manager | + Approvals queue filtered `Pending Manager Approval` **AND** assigned as that CR's `ProjectManager` (acts as Step 1); History tab, scoped to projects where they're `ProjectManager` |

**History tab is per-project too, same as the "To Approve" queue** — not a global audit log. A Manager only sees `Approved`/`Rejected` CRs for projects where they're the `ProjectManager`; a Sponsor only for projects where they're the `ProjectSponsor`. This includes CRs rejected at the *other* step (e.g. a Manager sees their project's CR even if it was later rejected by the Sponsor at Step 2) — the scoping is by project assignment, not by who personally acted on that particular CR.

**Approval authorization is per-project, not just per-role.** A Manager only sees/can act on a CR if they are that project's `ProjectManager`; a Sponsor only if they are that CR's/project's `ProjectSponsor` (for both Step 0 and Step 2). It is not a shared "any Manager can approve any pending-Manager CR" queue. The responsible person is resolved as:
- **Create CR**: `ProjectSponsor`/`ProjectManager` proposed directly on the CR (`gSelectedCR.ProjectSponsor`/`.ProjectManager`) — the project doesn't exist yet, so there's nothing else to check against. `ProjectManager` is blank until the Sponsor assigns it (Step 0).
- **Update/Delete CR**: the *live* target project's `ProjectManager`/`ProjectSponsor` (`gLiveTarget.ProjectManager`/`.ProjectSponsor`, looked up via `TargetItemID`) — the CR itself doesn't carry these fields. Update/Delete CRs have no Step 0 (no reassignment of the Sponsor), but the live project's `ProjectSponsor` is still who does their Step 2 final approve.

**Viewing `Project_List` is public to every user; submitting Update/Delete is not.** `AllProjectsScreen` shows every non-deleted project to everyone regardless of role. The row's `Update`/`Delete` buttons only appear when `ThisItem.RequestedBy.Id = gCurrentEmployee.ID` **and** no CR is already pending for that project — only the original requester who submitted that project's Create request can start an Update or Delete CR on it. This is unrelated to the `ProjectManager`/`ProjectSponsor` approval-authorization rule above — a project's assigned Manager/Sponsor cannot Update/Delete it themselves unless they also happen to be `RequestedBy`.

A **`View`** button is always present on every row (read-only, available to everyone) and navigates to `ViewProjectScreen`, a dedicated read-only details screen (project info grid + description + pending-change banner + a `Close` button — no edit form). Tapping the gallery row itself (`galProjects.OnSelect`) also navigates to `ViewProjectScreen`. `Update`/`Delete` are the extra owner-only actions layered on top (shown only when `ThisItem.RequestedBy.Id = gCurrentEmployee.ID`, project not deleted, and no CR pending). `UpdateProjectScreen` is now **edit-only** — only the owner ever reaches it (via `btnRowUpdate`), so it no longer carries the old `gCanEditProject` read-only-aware branching; all its edit rows are unconditionally visible.

This authorization check is duplicated in 3 places per role (Manager Step 1 / Sponsor Step 2) and must stay in sync if the rule ever changes: the `galChangeRequests` gallery `Items` filter (which CRs appear in "To Approve"), `conActionBar`'s `Visible` (whether Approve/Reject buttons show for the selected CR), and `gApprovalsPendingCount` (the badge count, in both `App.OnStart` and `AllProjectsScreen.OnVisible`). Sponsor's Step 0 duplicates the same check in the equivalent 3 places: the `galChangeRequests` "Assign Manager" case (`gApprovalsTab = "Assign Manager"` in the `Items` `Switch`), `rowAssignManager`'s `Visible`, and `gApprovalsPendingCount`'s `Pending Sponsor` branch. `cmbSponsor` on `CreateProjectScreen` only offers active Sponsors (`Filter(Project_User, IsActive && Role.Value = "Sponsor")`) and `cmbAssignManager` on `ApprovalsScreen`'s "Assign Manager" tab only offers Managers (see `docs/sharepoint-schema.md`), so `ProjectSponsor`/`ProjectManager` are guaranteed to hold someone with the matching role — this authorization model depends on that constraint holding. The "Assign Manager" tab and the "Approvals (n)" button on `AllProjectsScreen` are shown on `gIsApprover || gApprovalsPendingCount > 0`, so a Sponsor (an approver) always sees them.

Email notifications follow the same per-project targeting, not a role broadcast: `SubmittedToSponsor` goes to the CR's own `ProjectSponsor` (Create only), `SubmittedToManager` goes to the CR's/project's specific `ProjectManager` (`Coalesce(LookUp('Employee List', ID = <ProjectManager.Id>).Email, "app.admin@maxbiocare.com")` — fired either on Update/Delete submit or once the Sponsor finishes assigning a Manager), and `ManagerApprovedToSponsor` goes to the CR's/project's specific `ProjectSponsor` the same way.

## Conventions

- **UI text and all source: English only.**
- Guarded writes: `With({wX: Patch(...)}, If(IsBlank(wX.ID), Notify(error), <next step>))` — chain nested `With` for multi-step (log → status → notify).
- Choice: write `{Value: "..."}`; multi-choice: table of `{Value}` records; Lookup/Person: `{Id, Value}`; multi-lookup (`RelatedSKU`): table of `{Id, Value}` records; Number: plain numbers. Exceptions: `Department` is plain Text (options sourced from `Choices('Employee List'.department)` — the Choice column's defined value set, not the distinct values actually in use), not a Choice column itself. `BudgetAmount`/`ActualCost` are **Number**, not SharePoint's Currency type — the actual unit lives in the sibling `Currency` column (plain Text, same convention as `Department`), since a project's budget isn't always in the currency implied by its `Market`. `Currency` is set once at Create (from `CreateProjectScreen`'s `currency` control, itself still derived from the selected `Market`) and never edited afterward — display formulas read it directly (`<record>.Currency`) rather than re-deriving it from `Market` each time.
- Status strings (`Pending Sponsor`, `Pending Manager Approval`, `Pending Sponsor Approval`, `Approved`, `Rejected`, `Deleted`, …) are **literals in every screen** — no shared constant; renaming one means touching every filter, color map, and Patch. Note the two distinct Sponsor states: `Pending Sponsor` = Step 0 (assign Manager), `Pending Sponsor Approval` = Step 2 (final approve).
- Join key for logs: `ChangeRequestIDText = Text(cr.ID)` (delegable), never lookup-column comparison.
- `Project_User.Title` is never populated — always resolve a user's display name/email through its `EmployeeID` lookup into `Employee List`, never read `Project_User.Title` directly.
- `Project_Notify.Run(...)` recipient args are always `Coalesce(Concat(Filter(Project_User, Role.Value = "..." && IsActive), ..., ";"), "app.admin@maxbiocare.com")` — an empty active-approver Filter makes `Concat` return `""`, which fails the flow's "Send an email" action on an empty `To`.
- Inline `RGBA(...)` colors; brand purple `RGBA(83, 74, 183, 1)`; status pill palette mirrors procurement (green `RGBA(15,110,86,1)` / red `RGBA(163,45,45,1)` / amber `RGBA(133,79,11,1)`).
- Control versions and YAML pitfalls: **`.claude/pa-yaml-rules.md` is binding** (`Label@2.5.1`, `Button@0.0.45`, `TextInput@0.0.54`, `Radio@0.0.25` no Default, `ComboBox@0.0.51` DefaultSelectedItems=Filter, `CheckBox@0.0.30`, `Gallery@2.15.0`, `GroupContainer@1.5.0`, `Rectangle@2.3.0`; `|-` block literal for any formula containing `: `; `FillPortions: =0` for AutoLayout children with explicit Height).
- `ActualCost` is display-only everywhere — reserved for the future procurement cost-sync flow (`docs/approval-workflow-plan.md` §Future).

## Project status (last updated: 2026-07-11)

> **Model change (2026-07-11):** The former "Executive/Owner" second approver was **merged into the Sponsor**. `Project_User.Role` value `Executive` → `Sponsor`; `ProjectOwner` is no longer used by the app (the SharePoint column may remain but is ignored); the Sponsor now does both Step 0 (assign Manager) and Step 2 (final approve + apply). Status `Pending Manager` → `Pending Manager Approval`, `Pending Executive` → `Pending Sponsor Approval`. All `Src/*.pa.yaml` and docs below reflect this; SharePoint Choice sets (`Project_User.Role`, `Project_ChangeRequests.ApprovalStatus`) were updated to match.

### Done — repo authoring phase complete
- All docs written: `docs/sharepoint-schema.md` (4 lists + `Project_Notify` 9-arg table), `docs/approval-workflow-plan.md` (state machine, flow build spec, 4 English email templates, future cost-sync placeholder), this CLAUDE.md, `.claude/pa-yaml-rules.md` (copied from procurement-procedure).
- All 8 `Src/*.pa.yaml` files written and YAML-validated: `_EditorState`, `App` (OnStart role resolution), `AllProjectsScreen` (filters, search, pending-change badge, approvals counter), `CreateProjectScreen` (full form, Actual Cost read-only, Root/Child hierarchy picker), `ViewProjectScreen` (read-only project details, gallery row tap + View button target), `UpdateProjectScreen` (Description + Deliverables + End Date only, pending-CR guard, owner-only edit), `DeleteProjectScreen` (reason + confirm required, active-children guard), `ApprovalsScreen` (master-detail, update diff view, manager/sponsor approve chains with apply-before-flip, reject with required remark).
- Cross-checked: column names in formulas match the schema doc; status strings consistent; 13 `Project_Notify.Run` call sites all pass 9 args (includes the `FinalApprovedManagerCopy`/`FinalRejectedManagerCopy` Manager-copy calls at the Sponsor's Step 2, and the `SubmittedToSponsor`/Sponsor-triggered `SubmittedToManager` calls added for the Sponsor-assigns-Manager step — see §Known caveats history).
- Decisions locked with the user: staging list `Project_ChangeRequests`; single shared Approvals screen; Update limited to Description + Deliverables + End Date; soft delete; **Budget Approved column dropped** (Budget Amount only); SKU lookup target = `Product_Database_SKU_Master`; emails on submit / sponsor-assigns-manager / manager-approve / final result; **`ProjectManager` is chosen by a separate `ProjectSponsor` role after Create-submit, not by the requester on the Create form** (Sponsor owns the outcome, Manager supervises — see `docs/approval-workflow-plan.md` §2).

### Next — manual setup outside this repo (in order)
1. Create the 4 SharePoint lists per `docs/sharepoint-schema.md` — Choice value sets on `Project_ChangeRequests` must exactly match `Project_List`. Apply the permission model in `docs/sharepoint-schema.md` §SharePoint list permissions: `Edit` on `Project_ChangeRequests` for Managers + Sponsors (the two writer roles — Sponsor does the Step 0 "Assign Manager" Patch and Step 2 flip); custom "Add Items + Read only" level applies to `Project_ApprovalLog`; Read-only on `Project_List` (except Sponsor: Edit, needed for the apply step).
2. Seed `Project_User` with at least one account per role (Manager/Sponsor).
3. Build the `Project_Notify` flow per `docs/approval-workflow-plan.md` §3 (declare all 9 trigger input types explicitly; Switch with Default → Terminate; pin the `app.admin@maxbiocare.com` connection in Run-only users) — click-by-click steps in `flows/Project_Notify/BUILD_GUIDE.md`; HTML bodies to paste in `flows/Project_Notify/templates/`.
4. Create the canvas app in Power Apps Studio, add the 6 data sources + the flow, then paste the `Src/*.pa.yaml` screens (order: App OnStart, then screens). Verify the exact modern-control version Studio assigns to `DatePicker`/`Toggle` and add it to `.claude/pa-yaml-rules.md`.
5. Verify two unconfirmed schema points before wiring: the display/Title column of `Product_Database_SKU_Master`, and Employee List internal name `email` (lowercase).
6. Run the E2E test checklist: create → **Sponsor assigns Manager (Pending Sponsor → Pending Manager Approval)** → Manager approves (→ Pending Sponsor Approval) → **Sponsor final-approves + applies** → row created with generated ProjectID + 2 log rows; update diff + apply; delete (reason required, soft delete, restore via Show Deleted); reject at each step with remark; duplicate-pending guard; child-project root picker.
7. **Not yet created on SharePoint** — add `RequestedBy` (Lookup→Employee List) to `Project_List`. Until it exists, every existing row has it blank and nobody can Update/Delete any project (see Known caveats).
8. Future phase (spec only, do not build yet): `Project_SyncActualCost` flow — blocked on adding a `ProjectID` join column to the procurement lists; then add an `ActualCostUpdated` case to `Project_Notify` (notify Project Manager).

## Known caveats

- This SharePoint connector delegates neither `<>` nor `Not(...)` — never write `Col.Value <> "X"` or `Not(Col.Value = "X")` against a data source. Instead OR together `=` comparisons against every value you DO want (e.g. exclude `Deleted` by writing `Col.Value = "Not Started" || Col.Value = "In Planning" || ...` for every non-Deleted status). Verbose, but `=`/`&&`/`||` are the only combination confirmed delegable here.
- **`If(...)` and cross-table `LookUp(...)` inside a `Filter(dataSource, ...)` predicate are not delegable either** — and a single non-delegable operator anywhere in the predicate makes Power Apps fall back to fetching only the first N rows (the non-delegable query limit, default 500) and filtering client-side, so rows genuinely matching past that cutoff go silently missing once `Project_ChangeRequests` grows. Mitigated (not fully fixed) by splitting into `Filter(Filter(dataSource, <delegable part — plain `=`/`&&`/`||` on Choice/Lookup.Id>), <non-delegable part — the `If(...)`/cross-table `LookUp(...)`>)`: the inner `Filter` delegates fully at runtime (correct at any table size), and the outer `Filter`'s non-delegable refinement runs client-side against that already-complete, already-narrow result. Applied throughout `gApprovalsPendingCount` (`App.OnStart`, `AllProjectsScreen.OnVisible`), `galChangeRequests.Items`, and the tab-count labels on `ApprovalsScreen`. **Studio's delegation-warning banner still shows anyway** ("the 'If' part of this formula might not work correctly on large data sets") — its checker is a shallow syntax scan that doesn't recognize the inner `Filter` as a delegation boundary, even though the inner call genuinely does delegate. Accepted as a known false-positive-looking warning (same posture as the `CountRows` note above) rather than chasing full silence via `ClearCollect`-based local collections, which would trade the warning for extra state that needs manual refresh triggers to avoid the tab counts and the gallery disagreeing.
- Per-row `LookUp` badges (pending-change on All Projects, manager trail on Approvals) are N+1 queries — fine at portfolio scale; revisit past ~500 rows.
- Recipient email resolution (`Project_Notify` calls) uses `LookUp('Employee List', ID = EmployeeID.Id).Email` per active Manager/Sponsor — another accepted N+1 pattern, and this is final: `Project_User.EmployeeID` only ever exposes `.Id`/`.Value` (→Title) in this app's connection. SharePoint's Lookup column "additional columns" setting was configured for `EmployeeID` (Email checked) and the `Project_User` data source was refreshed in Studio — `EmployeeID.Email` still doesn't resolve (`PA2108`, "Name isn't valid. 'Email' isn't recognized"). Do not retry this; always use the `LookUp` form for any field beyond `.Id`/`.Value` on this column.
- Choice value sets are duplicated between `Project_List` and `Project_ChangeRequests` and must stay identical, or the apply Patch fails. (`Department` is the exception — plain Text on both, no Choice value set to sync.)
- `Project_List.RequestedBy` is blank on any row created before that column existed (or applied before this rule shipped) — `AllProjectsScreen`'s `ThisItem.RequestedBy.Id = gCurrentEmployee.ID` check never matches a blank, so those projects' Update/Delete buttons never show for anyone. Fix per-row by manually setting `RequestedBy` in SharePoint (e.g. to the project's `ApprovedBy` or `ProjectManager` as a reasonable stand-in) once identified.
- `Product_Database_SKU_Master` schema and Employee List internal names must be verified in SharePoint before first Studio wiring.
- DatePicker/Toggle modern-control versions are not yet in `pa-yaml-rules.md` — verify in Studio on first paste and add them.
- **Manager + Sponsor have raw `Edit Items` on `Project_ChangeRequests`** (the two writer roles — Manager writes Step 1 logs/status, Sponsor writes the Step 0 "Assign Manager" Patch and the Step 2 flip); Sponsor additionally has `Edit Items` on `Project_List` for the apply step. Anyone with that access could in principle bypass the app's guarded formulas by editing those lists directly. Not closable with list permissions alone; would need moving the guarded Patches into a Power Automate flow with a locked-down connection. (Historical note: `Project_ChangeRequests` Edit was previously broadened to *everyone* because the old `ProjectSponsor` could be any employee, not a role — now that Sponsor is a `Project_User` role, it can be scoped to the Manager + Sponsor groups.)
- **Manager-copy notifications are app-side done, flow-side pending**: at the Sponsor's Step 2, `ApprovalsScreen.pa.yaml`'s `btnConfirmReject` (guarded by `gSelectedCR.ApprovalStatus.Value = "Pending Sponsor Approval"` — Step 1 rejects stay requester-only) and all 3 `FinalApproved` apply blocks (Create/Update/Delete) now chain a second `Project_Notify.Run("FinalApprovedManagerCopy"/"FinalRejectedManagerCopy", <that project's ProjectManager email>, ...)` alongside the existing requester-facing call — see `docs/approval-workflow-plan.md` §2a/§3 for the full spec and email templates. **Still outstanding**: the `Project_Notify` flow itself needs those 2 new `Switch` cases added in the Power Automate portal (manual setup step 3 in §Next) — until then these 2 call sites will hit the flow's `Default → Terminate (Failed, "Unknown notificationType")` branch.
