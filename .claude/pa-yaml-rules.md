# Power Apps Canvas YAML — Lessons Learned

Rules derived from mistakes made while editing `.pa.yaml` files in this project.

---

## 1. Form control: version and Layout placement

**Wrong:**
```yaml
- myForm:
    Control: Form@2.1.1          # ← old version, missing Layout
    Properties:
      Layout: Vertical           # ← WRONG: inside Properties
      DataSource: =MyList
```

**Correct:**
```yaml
- myForm:
    Control: Form@2.4.4          # ← use 2.4.4
    Layout: Vertical             # ← at control level, sibling of Control/Properties/Children
    Properties:
      DataSource: =MyList
      Item: =gMyRecord
      SnapToColumns: =false
```

**Rule:** `Layout` is a metadata key of the control definition (like `Variant:`), not a Power Fx property inside `Properties:`. Putting it inside `Properties:` causes `PA1001: Power Fx expressions must start with '='`.

---

## 2. YAML plain scalar cannot contain `: ` (colon + space)

**Wrong:**
```yaml
OnFailure: =Notify("Please contact IT with reference: " & x, NotificationType.Warning)
#                                            ^^^^^^^^^^^ colon-space breaks YAML parsing
```

**Correct:**
```yaml
OnFailure: |-
  =Notify("Please contact IT with reference: " & x, NotificationType.Warning)
```

**Rule:** Any property value that contains `: ` (colon followed by space) inside a string must use the `|-` block literal syntax.

---

## 2b. AutoLayout children with explicit Height must set FillPortions: =0

In a Canvas App AutoLayout container, if a child does NOT declare `FillPortions`, the parent may distribute remaining space proportionally and override the child's `Height` property.

**Rule:** Any child GroupContainer that has an explicit or dynamic `Height` formula (e.g. `Height: =If(...)`, `Height: =68`) must also declare `FillPortions: =0` to opt out of proportional fill and honour its own `Height`. Applies to `OnSelect`, `OnFailure`, `OnSuccess`, `Text`, or any other property. Common triggers: error messages with `"reference: "`, `"error: "`, URLs with `https://`, etc.

**Also applies to `LayoutMinHeight` rows, not just fixed `Height` ones.** A row/column sized with `LayoutMinHeight` (meant to auto-size to its content, with a floor) is just as stretch-eligible as one with a hardcoded `Height` — if its siblings in a `Vertical` AutoLayout parent have unused space (e.g. a short screen like `UpdateProjectScreen`/`DeleteProjectScreen` where `containerScroll`'s fixed `Height` exceeds total content height), the parent inflates the `LayoutMinHeight` row far past its natural content height. Symptom actually seen in Studio: a `Horizontal` + `LayoutWrap: =true` row of label/value info-columns (`rowProjectInfo`) got stretched tall, so its second wrapped line rendered near the bottom of the oversized box and the next sibling row butted up against it with no gap. **Rule: every row/column that should hug its content — whether via `Height` or `LayoutMinHeight` — must declare `FillPortions: =0`.** Only rows that are genuinely meant to grow and fill leftover space should omit it.

---

## 3. Form control belongs INSIDE the scrollable container

When adding a Form (e.g., for attachment upload) to a screen that has a scrollable `conScrollable_*` container, the Form must be a **child of that container**, not a sibling at screen level.

**Wrong:** Add Form at screen level with `Y: =Parent.Height - 200`, reduce container Height.

**Correct:** Add Form as the last child inside `conScrollable_*`. No `Y:` needed — AutoLayout manages positioning. Leave container `Height: =Parent.Height - 220` unchanged.

The indentation depth increases by 6 spaces for each nesting level. A Form inside `conScrollable_*` sits at 12-space indent (`            - frmXxx:`), its Properties at 16 spaces, its DataCard at 18 spaces, and DataCard children at 24+ spaces.

---

## 4. Attachment upload pattern (Form + SubmitForm)

Canvas Apps cannot write attachments via `Patch()`. The only way is `Form` control + `SubmitForm()`.

### Required Form structure
```yaml
- frmMyLog:
    Control: Form@2.4.4
    Layout: Vertical
    Properties:
      DataSource: =Procurement_ExecutionLog   # or Procurement_Requests
      Item: =gMyLogEntry                      # global var pointing to the record
      OnFailure: |-
        =Notify("Saved but attachment failed...", NotificationType.Warning)
      SnapToColumns: =false
      Width: =Parent.Width
      Height: =140
    Children:
      - MyAttachments_DataCard:
          Control: TypedDataCard@1.0.7
          Variant: ClassicAttachmentsEdit
          Properties:
            DataField: ="{Attachments}"
            Default: =ThisItem.Attachments
            Required: =false
            Update: =attMyPhotos.Attachments
            Width: =Parent.Width
          Children:
            - DataCardKeyMyPhotos:
                Control: Label@2.5.1
                MetadataKey: FieldName          # required
                Properties:
                  Text: ="Upload Photos *"
                  Y: =8
            - attMyPhotos:
                Control: Attachments@2.3.0
                MetadataKey: FieldValue         # required
                Properties:
                  Items: =If(IsBlank(Parent.Default), gMyPendingAttachments, Parent.Default)
                  MaxAttachments: =5
            - ErrorMessageMyPhotos:
                Control: Label@2.5.1
                MetadataKey: ErrorMessage       # required
                Properties:
                  Text: =Parent.Error
                  Visible: =Parent.DisplayMode = DisplayMode.Edit
```

### Submit sequence (in button OnSelect, after Patch succeeds)
```
// MUST save attachments BEFORE EditForm — EditForm resets the control
Set(gMyPendingAttachments, attMyPhotos.Attachments);
// Point form to the newly created log entry
Set(gMyLogEntry, wLog);
// Switch form to edit mode for that record
EditForm(frmMyLog);
// Write attachments to the record
SubmitForm(frmMyLog);
// Optimistic navigate (OnFailure handles upload failure)
Notify("Submitted successfully", NotificationType.Success);
Navigate(HomeScreen)
```

**Why `Set(gMyPendingAttachments, ...)` before `EditForm`:**  
`EditForm()` resets all data card values back to `Parent.Default`. If `Parent.Default` is empty (new record), the Attachments control would lose the user's selected files. Saving to a global variable first, then using `Items = If(IsBlank(Parent.Default), gMyPendingAttachments, Parent.Default)` ensures the files survive the reset.

---

## 5. Radio control — correct version and default selection

**Wrong:**
```yaml
- myRadio:
    Control: Radio@2.3.1        # ← version does not exist / too new
    Properties:
      Default: ="Option A"      # ← Default is NOT a valid property for Radio
      Items: =["Option A", "Option B"]
```

**Correct:**
```yaml
- myRadio:
    Control: Radio@0.0.25       # ← correct version
    Properties:
      Items: =If(condition, ["Option A", "Option B"], ["Option B", "Option A"])
      # Radio selects the FIRST item by default — reorder Items to control default
```

**Rule:** `Radio@0.0.25` is the correct version. The control has no `Default` property (`PA2108` error). Drive the default selection by ordering `Items` so the desired default appears first. When the condition changes (e.g., after patching a global var), `Items` reorders and the radio resets to the new first item.

To make radio options horizontal, use the namespaced enum:
```yaml
Layout: ='RadioGroupCanvas.Layout'.Horizontal
```
**Not** `Layout.Horizontal` — that enum does not exist for Radio and will cause a PA error.

---

## 6. ComboBox — no `Default` property, use `DefaultSelectedItems`

**Wrong:**
```yaml
- myComboBox:
    Control: ComboBox@0.0.51
    Properties:
      Default: =LookUp(MyList, ID = someID)    # ← Default is NOT a valid property
```

**Correct:**
```yaml
- myComboBox:
    Control: ComboBox@0.0.51
    Properties:
      DefaultSelectedItems: =Filter(MyList, ID = someID)  # ← takes a TABLE (Filter), not a single record (LookUp)
```

**Rule:** `ComboBox@0.0.51` does not have a `Default` property (`PA2108`). Use `DefaultSelectedItems` which expects a **table** — use `Filter(...)` not `LookUp(...)`. When the target ID is blank, `Filter` returns an empty table and nothing is pre-selected.

---

## 6b. ComboBox on a full-record data source: author it as `ModernCombobox@1.1.1` with `ItemDisplayText`

When `Items` is a raw table of records (e.g. `Sort('Employee List', Title, ...)`, `Filter(Project_List, ...)`) rather than `Choices(...)`, a plain `ComboBox@0.0.51` does **not** reliably infer which column to display — observed in Studio rendering the numeric `ID` instead of the name/title in every dropdown option. Studio also silently auto-upgrades any `ComboBox@0.0.51` you paste in to `ModernCombobox@1.1.1` on its own, which doesn't even have a `DisplayFields` property — so author it as `ModernCombobox@1.1.1` directly instead of round-tripping through the legacy control.

**Correct:**
```yaml
- cmbOwner:
    Control: ModernCombobox@1.1.1
    Properties:
      AllowExternalSelectedItems: =false
      FontWeight: =""
      Height: =40
      InputTextPlaceholder: ="Choose an employee"
      ItemDisplayText: =ThisItem.Title   # ← the field that actually exists on the source table
      Items: =Sort('Employee List', Title, SortOrder.Ascending)
      SelectMultiple: =false
      Size: =0
      Width: =Parent.Width
```

**Rule:** for `ModernCombobox@1.1.1` on a full-record source, set `ItemDisplayText` to the record's display field (`ThisItem.Title`, `ThisItem.Value` for `Choices(...)`-derived tables, etc.). `.Selected`/`.SelectedItems` still expose every field on the underlying record (e.g. `.Selected.ID`, `.Selected.ProjectID`) regardless of what `ItemDisplayText` renders — it only controls what's shown in the box, not what's queryable.

---

## 6c. Multiline + pre-filled text input: `ModernTextInput@1.1.1` with `Type`, not `TextInput@0.0.54` with `Mode`

Two different "multiline" incantations exist for two different controls in this app family, and mixing them up trips a `PA2108`:

- `TextInput@0.0.54` (legacy control, used for fresh/blank remark-style fields like reject remarks, delete reasons): multiline via `Mode: ='TextInputCanvas.Mode'.Multiline`. **Has no `Default` property at all** — confirmed by Studio error (`PA2108: Unknown property 'Default' for control type 'TextInput@0.0.54'`) and by grepping the sibling `procurement-procedure`/`invoice-batch-app` repos, where every usage of this control is always a fresh/empty field, never pre-filled. Use this control only when the field starts blank every time.
- `ModernTextInput@1.1.1` (used everywhere in this app for pre-filled/editable fields, e.g. `txtNewDescription`, `txtDetailInvoiceNumber` in the sibling invoice-batch-app): **does** support `Default`, and multiline is `Type: =TextInputType.Multiline` — **not** `Mode: ='TextInputCanvas.Mode'.Multiline` (that's the legacy control's enum; pasting it onto `ModernTextInput` is a different `PA2108`).

**Correct — multiline field that also needs to show/edit an existing value:**
```yaml
- txtNewDeliverables:
    Control: ModernTextInput@1.1.1
    Properties:
      Default: =gSelectedProject.Deliverables
      FontWeight: =""
      Height: =100
      MaxLength: =-1
      Placeholder: ="List the expected deliverables for this project"
      Size: =0
      TriggerOutput: =TriggerOutput.FocusOut
      Type: =TextInputType.Multiline
      Width: =Parent.Width - 40
```

**Rule:** need `Default` (pre-filled/editable) → `ModernTextInput@1.1.1` + `Type: =TextInputType.Multiline` for multiline. Need only a fresh/blank field (remark, reason) → `TextInput@0.0.54` + `Mode: ='TextInputCanvas.Mode'.Multiline` is fine and simpler. Never combine a control with the other control's enum.

---

## 4. Global var initialisation

Declare all attachment globals in `App.OnStart`:
```
Set(gMyLogEntry, Blank());
Set(gMyPendingAttachments, Blank())
```

Not declaring them doesn't crash the app but causes stale data if the user visits the screen twice in one session.
