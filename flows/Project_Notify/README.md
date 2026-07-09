# `Project_Notify` — email HTML templates

Ready-to-paste HTML bodies for the `Project_Notify` Power Automate flow's **"Send an email (V2)"** action. Full flow build spec (trigger inputs, `Switch` cases, recipient resolution) is in `docs/approval-workflow-plan.md` §3 — this folder only holds the HTML source referenced from there.

## How to use

This flow has exactly **one** `Send an email (V2)` action, placed after the `Switch` — not one per case. Each `Switch` case only sets the `varSubject`/`varBody` variables (via `Set variable`); the single `Send an email (V2)` action at the end reads whatever those variables ended up holding. So the HTML from this folder is pasted into a `Set variable` action's **Value** field, not into `Send an email`'s Body field. Full click-by-click build steps: [`BUILD_GUIDE.md`](BUILD_GUIDE.md).

Quick version, per case:

1. One-time setup: add an `Initialize variable` action named **`varAppLink`** (String), value = the app deep link once the app is published: `https://apps.powerapps.com/play/e/<ENV_ID>/a/<APP_ID>?tenantId=<TENANT_ID>`. Every template's link points at `@{variables('varAppLink')}` — set it once and every email picks it up; there is nothing to find-and-replace per template.
2. Inside the matching `Switch` case, on the `Set variable` action targeting `varBody`: click its **Value** field (a plain expression box, not a rich-text editor — paste directly, no Code View needed here).
3. Open the matching file below and paste its **entire contents** as-is.
4. Leave every `@{triggerBody()?['text_N']}` and `@{variables('varAppLink')}` expression exactly as written — Power Automate string-interpolates `@{...}` tokens embedded in literal text at runtime, resolving them against the flow's trigger inputs / variables. Do not retype them through the dynamic-content picker; pasting the file as one block preserves the syntax exactly. **Note these reference the trigger inputs' original default keys (`text`, `text_1`, `text_2`, ...), not the renamed display names** (`notificationType`, `recipientEmails`, `requestTitle`, ...) — renaming an input only relabels it in the designer, it doesn't change the key `triggerBody()` addresses. See the mapping table in `docs/approval-workflow-plan.md` §3.
5. On that same case's `Set variable` action targeting `varSubject`, paste the matching Subject line from `docs/approval-workflow-plan.md` §3 (not stored here — subjects are one-line plain text with no HTML).
6. On the flow's single `Send an email (V2)` action (after the Switch, run once regardless of which case fired): To = `recipientEmails`, Subject = `varSubject`, Body = `varBody`, and toggle **Is HTML = Yes** under Advanced options — this is what makes the Outlook connector render the pasted markup as HTML instead of showing raw tags.

| `notificationType` (Switch case) | Template file | Recipient |
|---|---|---|
| `SubmittedToManager` | [`templates/submitted-to-manager.html`](templates/submitted-to-manager.html) | that project's `ProjectManager` |
| `ManagerApprovedToExecutive` | [`templates/manager-approved-to-executive.html`](templates/manager-approved-to-executive.html) | that project's `ProjectOwner` |
| `FinalApproved` | [`templates/final-approved.html`](templates/final-approved.html) | the requester |
| `FinalRejected` | [`templates/final-rejected.html`](templates/final-rejected.html) | the requester |
| `FinalApprovedManagerCopy` | [`templates/final-approved-manager-copy.html`](templates/final-approved-manager-copy.html) | that project's `ProjectManager` (Step 2/Executive approve only) |
| `FinalRejectedManagerCopy` | [`templates/final-rejected-manager-copy.html`](templates/final-rejected-manager-copy.html) | that project's `ProjectManager` (Step 2/Executive reject only) |

## Design notes

- **Table-based layout, inline styles only** — no `<style>` block, no CSS classes, no flex/grid. Outlook desktop renders HTML email through the Word engine, which ignores most of that; tables + inline styles is the one layout method that survives across Outlook desktop/web, Gmail, and mobile mail clients.
- **600px fixed-width card** centered on a light background — the standard safe email width; anything wider clips on some clients.
- Colors reuse the app's existing brand palette (`CLAUDE.md` §Conventions): brand purple `#534AB7` (`RGBA(83,74,183,1)`) for the header bar, status pill green `#0F6E56` / red `#A32D2D` / amber `#854F0B` matching the in-app status pills.
- Each file is fully self-contained (no shared partial/include — Power Automate's Code View has no include mechanism, so every template repeats its own header/footer markup verbatim).
- `remark` only appears in the two reject templates; the other four never reference it (the flow always receives `""` for `remark` on those calls per `docs/approval-workflow-plan.md`, so nothing would render anyway, but the row is omitted rather than left blank for a cleaner layout).
- **Every template links to the app** using the same solid purple button style — `SubmittedToManager`/`ManagerApprovedToExecutive` label theirs "Open Approvals" (action required), the other 4 label theirs "Open the Project List app →" (FYI-only). All 6 point at `@{variables('varAppLink')}`.
