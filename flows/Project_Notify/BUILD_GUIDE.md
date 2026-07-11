# `Project_Notify` — click-by-click build guide (Power Automate portal)

Companion to `docs/approval-workflow-plan.md` §3 (the spec) and `README.md` (HTML paste instructions) in this folder. This file is the step-by-step walkthrough for building the flow from an empty canvas.

## 0. Prerequisites

- A Power Automate license/seat on the same tenant as the Power Apps environment (`maxbiocare.sharepoint.com`).
- An Office 365 Outlook connection for `app.admin@maxbiocare.com` (or access to create one) — this is the shared mailbox every notification sends from.
- You do **not** need the Power Apps app published first. The 9 trigger inputs and all 6 `Switch` cases can be built and tested with dummy/manual trigger data. Only `varAppLink` (§4 below) needs the real app URL, and that can be filled in later — leave it blank or with a placeholder until the app is published, then edit that one action.

## 1. Create the flow

1. Go to [make.powerautomate.com](https://make.powerautomate.com) → select the correct environment (top-right environment picker) — must match the Power Apps environment this app will live in.
2. **My flows** (left nav) → **+ New flow** → **Instant cloud flow**.
3. Flow name: `Project_Notify`.
4. Under "Choose how to trigger this flow", search for **PowerApps** → select **"PowerApps (V2)"** (not the older "PowerApps" trigger — V2 is required for typed inputs). Click **Create**.

You land in the flow designer with a single trigger step: `PowerApps (V2)`.

## 2. Add the 9 trigger inputs

On the `PowerApps (V2)` trigger card, click **+ Add an input** nine times, adding these in this exact order (order matters — Power Apps calls `Project_Notify.Run(...)` positionally):

| # | Click "+ Add an input" → | Then rename the input (click the input's name/pencil) to |
|---|---|---|
| 1 | Text | `notificationType` |
| 2 | Text | `recipientEmails` |
| 3 | Text | `requestTitle` |
| 4 | Number | `requestId` |
| 5 | Text | `requestType` |
| 6 | Text | `projectName` |
| 7 | Text | `requesterName` |
| 8 | Text | `actionByName` |
| 9 | Text | `remark` |

**Why rename each one, even though it doesn't change anything functionally**: Power Automate defaults new inputs to `text`, `text_1`, `number`, `text_2`, etc. Renaming only relabels the input's display name in the designer and in the Power Apps formula-bar intellisense hint — **it does NOT change the key `triggerBody()` addresses inside the flow**. Every expression you'll write later in this flow (the Switch's `On` field, the `Set variable` actions, the email templates) still references the *original* default key — `triggerBody()?['text_2']` for the 3rd input, never `triggerBody()?['requestTitle']` even after renaming it to `requestTitle`. Rename anyway, purely so the trigger card and Power Apps' formula bar are readable — but remember the keys used everywhere else in this guide are the defaults from the table above, not the renamed labels.

**Do not leave any input untyped.** If you test-run the flow once with a blank value before explicitly picking Text/Number, Power Automate infers the type as "Any", which then requires reconfiguring. Pick the type explicitly at creation, per the table above.

## 3. Add the two content variables

Click **+ New step** (below the trigger, not inside anything yet):

1. Search **"Initialize variable"** → add it.
   - Name: `varSubject`
   - Type: `String`
   - Value: leave blank.
2. Click **+ New step** again → **"Initialize variable"** → add a second one.
   - Name: `varBody`
   - Type: `String`
   - Value: leave blank.

(Each `Initialize variable` action holds exactly one variable — this is a Power Automate limitation, not a choice. Two variables = two separate actions, run one after the other.)

## 4. Add the app-link variable

Click **+ New step** → **"Initialize variable"** a third time:

- Name: `varAppLink`
- Type: `String`
- Value: the app's deep link. If the app isn't published yet, leave this blank for now and come back to fill it in once you have the real URL:
  `https://apps.powerapps.com/play/e/<ENV_ID>/a/<APP_ID>?tenantId=<TENANT_ID>`
  (Get the real values from Power Apps → the app's **Details** → **Web link**, or **Play** button → copy the address bar URL.)

At this point you have 4 steps total: trigger, `Initialize variable varSubject`, `Initialize variable varBody`, `Initialize variable varAppLink`.

## 5. Add the Switch

1. Click **+ New step** → search **"Switch"** → add the **Control: Switch** action.
2. Click the **On** field → from the dynamic content picker, select the trigger's `notificationType` (shown by its renamed label in the picker). This inserts the correct underlying reference, `@{triggerBody()?['text']}` (`text` being the 1st input's original default key, not `notificationType`) — don't type it manually, use the picker here since this one field is a simple direct reference, not mixed literal+expression text like the HTML bodies later.
3. The Switch starts with one empty **Case** and a **Default** branch. Click **+ Add a case** six more times until you have **7 cases** total.
4. Click each case's header field and type the exact `notificationType` string (case-sensitive, must match what the app sends):
   - Case 1: `SubmittedToSponsor`
   - Case 2: `SubmittedToManager`
   - Case 3: `ManagerApprovedToSponsor`
   - Case 4: `FinalApproved`
   - Case 5: `FinalRejected`
   - Case 6: `FinalApprovedManagerCopy`
   - Case 7: `FinalRejectedManagerCopy`

## 6. Fill in each case

Inside **each** of the 7 case branches, add two actions:

1. **+ Add an action** (inside that case's column) → search **"Set variable"** → configure:
   - Name: `varSubject`
   - Value: the one-line Subject text for this case, from `docs/approval-workflow-plan.md` §3 (e.g. for `SubmittedToManager`: `[Project List] Approval needed: @{triggerBody()?['text_2']}` — `text_2` is `requestTitle`'s original default key, see the mapping table in §3). Type or paste this directly into the Value field — it's a plain expression box, so the mixed literal-text-plus-`@{...}` syntax works as-is.
2. **+ Add an action** again (same case, below the first) → search **"Set variable"** → configure:
   - Name: `varBody`
   - Value: click into the field, then paste the **entire contents** of the matching file from `templates/` (see the table in `README.md` for which file goes with which case). This field is also a plain expression box — pasting raw HTML text here is safe; it becomes the literal string value of `varBody`, tags and all.

Repeat this pair of actions (Set `varSubject`, Set `varBody`) for all 7 cases, each pointing at its own Subject line and its own template file.

## 7. Configure the Default branch

In the **Default** branch (present automatically, to the right of your 7 cases):

1. **+ Add an action** → search **"Terminate"** → add it.
2. Status: `Failed`.
3. Message: `Unknown notificationType`.

This makes a typo'd `notificationType` from the app fail the flow run loudly (visible in Power Automate's run history) instead of silently sending no email.

## 8. Add the single Send-email action, after the Switch

Click **+ New step** **below the entire Switch control** (not inside any case — this action runs once, after whichever case fired):

1. Search **"Send an email (V2)"** (Office 365 Outlook connector) → add it.
2. Sign in / pick the connection for `app.admin@maxbiocare.com` if not already connected.
3. **To**: dynamic content → `recipientEmails`.
4. **Subject**: dynamic content → `varSubject` (the variable, not the trigger input).
5. **Body**: dynamic content → `varBody`.
6. Click **Show advanced options** (bottom of the action card) → find **"Is HTML"** → set to **Yes**. This is the switch that makes Outlook render the `varBody` string as formatted HTML instead of showing the raw `<table>...` tags in the email.

## 9. Save and test

1. Click **Save** (top-right).
2. **Test** (top-right) → **Manually** → **Test**. Since there's no Power Apps caller yet, Power Automate lets you fill in the 9 trigger inputs by hand for a one-off test run.
3. Fill in a `notificationType` (e.g. `SubmittedToManager`) and dummy values for the rest, plus a `recipientEmails` you can actually check (e.g. your own mailbox).
4. Run it → check the run history for a green checkmark on every step, and check the inbox for a correctly formatted HTML email.
5. Repeat with each of the other 6 `notificationType` values to confirm all 7 cases render correctly, then try an unrecognized value (e.g. `Bogus`) to confirm the flow fails via `Terminate` rather than silently doing nothing.

## 10. Wire it into the app

Nothing to do here — `ApprovalsScreen.pa.yaml`, `CreateProjectScreen.pa.yaml`, `UpdateProjectScreen.pa.yaml`, and `DeleteProjectScreen.pa.yaml` already call `Project_Notify.Run(...)` with all 9 args in the right order (see `CLAUDE.md` §Known caveats for the current call-site count). Once this flow exists under the exact name `Project_Notify` in the same environment as the app, Power Apps Studio will resolve those calls automatically the next time the app's data sources are refreshed.

## 11. Fill in the real app link (once published)

Go back to the `Initialize variable varAppLink` action (step 4 above) and replace the placeholder/blank value with the real deep link, then **Save**. No other action needs touching — all 6 templates and the Send-email action already reference `@{variables('varAppLink')}`.
