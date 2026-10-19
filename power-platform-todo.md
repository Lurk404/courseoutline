# Power Platform Prototype — To-Do

**Course Outline & Syllabus Builder** · Power Apps + Power Automate + SharePoint
Started 2026-10-18 · Base form: `coursesyllabus2.html` · Policy: F1003 (amended 2026-06-23)

> **Goal:** a canvas app that fills in a syllabus or outline, saves each submission as a new SharePoint list row, prefills from the last row next term (flagging fields to review), produces F1003-compliant, inline-styled, WCAG AA HTML, and gives instructors a **formatted** copy.

## How to use this file

- Each `##` section is a GitHub milestone. Each checkbox is an issue, or a sub-task in a tracking issue.
- Suggested labels: `sharepoint` `power-apps` `power-automate` `dataverse` `it-question` `decision` `test` `a11y` `f1003` `docs`
- Commit exports at the end of every milestone (see section 2).
- Use synthetic test data only. No real instructor details in the repo.

---

## 0. Decisions

- [ ] **Scope:** syllabus only first, or outline + syllabus from the start (`DocType`)?
- [ ] **Course code shape:** one field (`ENGL 1127`) or subject + number as two fields? This affects the record key and filtering.
- [ ] **Record key:** course code + instructor email, with term used for ordering. Do multiple sections need separate records?
- [ ] **Submitting on someone's behalf:** can a department assistant or chair fill one in for an instructor? If yes, instructor email becomes an editable field instead of the signed-in user.
- [ ] **First-time course:** offer to start from another instructor's latest record? This has privacy and permission implications.
- [ ] **Drafts:** allow save-as-draft, or submit-only? Each submission is a new row either way.
- [ ] **Flow failure after the row is saved:** keep the row and retry the flow, or roll back?
- [ ] **HTML generator location:** Power Fx named formulas in the app (default) or an Office Script called from the flow (lets the generator code be shared with the GitHub tool).
- [ ] **Copy route** (after the section 6 tests): code component / rich text box / file upload / source-view fallback.
- [ ] **Security model:** item-level permissions with direct writes from the app (prototype default), or no list access for users with all reads and writes going through flows.
- [ ] **Repo layout:** a `power-platform/` folder in `Lurk404/courseoutline`, or a separate repo.

---

## 1. Questions for IT

Send these early; several answers gate later milestones.

**Licensing**
- [ ] Which Microsoft 365 licence do faculty and staff have? Does it include Power Apps and Power Automate standard-connector use rights for everyone?
- [ ] Are there DLP policies in the default environment that block or separate SharePoint, Office 365 Outlook, Office 365 Users, or Excel Online (Business)?

**Environment**
- [ ] Which environment should this live in: default or dedicated? Who administers it?
- [ ] Does that environment have a Dataverse database? (Needed only to deploy a code component.)
- [ ] Is "Power Apps component framework for canvas apps" turned on there, or can it be?
- [ ] Is a custom code component acceptable under security policy (TypeScript, clipboard only, no external calls)? Does it need a security review?
- [ ] What role do I need to import a solution (Environment Maker / System Customizer)?
- [ ] Can a service account own the production app, flows and connections? Does it need its own licence?

**SharePoint and records**
- [ ] Where should the production list and library live, and who owns that site?
- [ ] How long must outline and syllabus records be kept? Is there a retention label to apply?
- [ ] Who needs read access to all records (chairs, deans, curriculum office, registrar)?
- [ ] Are item-level list permissions acceptable, or is flow-mediated access preferred?

**Privacy**
- [ ] Does this need a Privacy Impact Assessment? It collects instructor name, email, phone and office hours.
- [ ] What is the data residency of SharePoint and Power Platform data in Langara's tenant?

**Email**
- [ ] Can a shared mailbox receive copies, and can flows send from it?

**Embedding** (pinned: ask now, build later)
- [ ] Can a Power Apps web link be embedded in Brightspace (iframe or widget)? Does that iframe allow clipboard write?
- [ ] Do browser policies on managed devices restrict clipboard access?
- [ ] Are Office Scripts enabled in the tenant? (Only if the Office Script generator is chosen.)

---

## 2. GitHub setup

- [ ] Create the folder structure:

```
power-platform/
  README.md
  todo.md                  ← this file
  schema/
    formdata.v1.json       ← JSON Schema for the payload
    sharepoint-list.md     ← list and library columns, indexes, permissions
    migrations.md          ← how old schema versions load
  powerfx/                 ← named formulas, HTML generator, screen YAML
  flows/                   ← exported flow packages (.zip)
  app/                     ← .msapp exports
  pcf/                     ← code component source
  tests/                   ← sample outputs, test logs, copy-fidelity results
```

- [ ] Create milestones (sections 3–10) and the labels.
- [ ] Export convention after each milestone:
  - App: Studio → File → Save as → This computer (`.msapp`)
  - Flow: My flows → Export → Package (`.zip`)
- [ ] Copy named formulas and each screen's code (YAML) into `powerfx/` so changes diff readably.
- [ ] Keep connection details and tenant-specific URLs out of commits.

---

## 3. Data contract (JSON)

- [ ] Inventory every field in `coursesyllabus2.html`: id, label, type, required/optional, repeating or not.
- [ ] Write `formdata.v1.json` (JSON Schema). It needs to cover:
  - `schemaVersion` and `docType`
  - scalar fields
  - `outcomes[]`
  - `assessments[]` {name, weight, due}
  - `schedule[]` {week, topic, reading, deadlines}
  - textbooks / materials
- [ ] Map each field to its F1003 clause: 3.2 for outlines, 3.4 for syllabi. Keep the mapping as a table in the schema doc.
- [ ] Carry over the required rules from `coursesyllabus.html`:
  - Everything is required except grading guidelines and departmental/course guidelines.
  - Assessment due dates and instructor phone are optional.
- [ ] Tag term-sensitive fields with `reviewOnCarryOver: true`: term, office hours, schedule dates and deadlines, assessment due dates, textbook edition, and others.
- [ ] Write `migrations.md`: how a v1 record loads into v2 (renamed, removed and new fields).
- [ ] Keep fixed wording out of the JSON. The FIPPA note, policy list, Student Services link and Musqueam acknowledgement are generator constants, versioned with the app.
- [ ] *(Later, optional)* Have the GitHub tool emit the same JSON so both front ends share one contract.

---

## 4. SharePoint (prototype on a site you own)

### List: `CourseOutlines`

- [ ] Create the list with these columns:

| Column | Type | Notes |
|---|---|---|
| Title | Single line | Auto: `CourseCode Term email` |
| CourseCode | Single line | Indexed |
| Term | Single line | `YYYY#0`, indexed |
| InstructorEmail | Single line (**not** Person) | Indexed; keeps filters delegable |
| InstructorName | Single line | |
| DocType | Choice | Outline / Syllabus |
| SchemaVersion | Number | |
| Revision | Number | Increments per key per term |
| FormDataJson | Multiple lines, **plain text** | Full payload |
| HtmlFile | Hyperlink | Set by the flow |
| FlowStatus | Choice | Pending / Done / Failed |

- [ ] Index CourseCode, Term and InstructorEmail (List settings → Indexed columns).
- [ ] Turn on item-level permissions (List settings → Advanced settings):
  - Read access: items created by the user
  - Create and edit access: items created by the user
- [ ] Confirm version history is on.
- [ ] Create a "By term" view grouped by Term, with counts. This replaces the bulk-inbox usage tally.

### Library: `OutlineFiles`

- [ ] Create the library with columns: CourseCode, Term, InstructorEmail, DocType, ListItemId.
- [ ] Decide how files are protected. Libraries don't offer the list's item-level setting, so choose one:
  - the flow sets unique permissions on each file,
  - a folder per instructor, or
  - the library is readable only by owners and reviewers, and instructors get their file by email or through the app.
- [ ] File naming: `COURSECODE_TERM_DOCTYPE_rN.html`.

### Permissions

- [ ] Create groups: Submitters (Contribute on the list), Reviewers (read all), Owners.
- [ ] Test which permission level lets Reviewers see all items despite the item-level settings, and document it in `sharepoint-list.md`.

---

## 5. Power Apps — canvas app

### Setup
- [ ] Create a blank canvas app (tablet, responsive layout) in the agreed environment.
- [ ] Settings: review modern controls and accessibility-related settings; turn on named formulas if needed.
- [ ] Add data sources: `CourseOutlines`, `OutlineFiles` (if needed), Office 365 Users.
- [ ] Set `varMe` = `Office365Users.MyProfileV2().mail`. Show the signed-in name and email on screen.

### Start screen
- [ ] Inputs: course code, term (dropdown of valid `YYYY#0` codes), doc type.
- [ ] "Load previous" gets the newest row for course code + `varMe`: Filter → SortByColumns(Created, descending) → First.
- [ ] Show what was loaded (term and date), or "No previous record — starting blank".
- [ ] Rehydrate the form:
  - `ParseJSON(FormDataJson)` into variables
  - `ClearCollect` for each repeating section
  - apply migrations based on `schemaVersion`

### Form screens
- [ ] Put sections in syllabus order, matching the template or the open section-order decision.
- [ ] Build repeating sections (outcomes, assessments, schedule) as a collection plus an editable gallery. Add, remove, and move up/down buttons must be labelled and keyboard-operable.
- [ ] Learning outcomes list/paragraph toggle.
- [ ] Show the assessment weight total live. Flag it when it isn't 100%; don't block submission.
- [ ] Paste normalisation for list-bearing multi-line inputs: strip bullets and numbering, split into lines.
- [ ] Required-field validation matching section 3. Put an error summary at the top that moves focus to each field.
- [ ] Carry-over review for term-sensitive fields:
  - show "Carried over from {term} — review" as text plus an icon
  - keep it until the field is edited or confirmed
  - show a count at the top of the form

### HTML generator
- [ ] Named formula(s) that produce the fragment and update as the user types.
- [ ] Constants:
  - FIPPA note
  - policy list: E1003, F1004, E2008, E2006, F1002, E2011, E1005, plus the F1008 attendance fallback
  - Student Services URL, Grades link, BC Transfer Guide link
  - Musqueam acknowledgement
- [ ] Wrap every user value in `EncodeHTML()`. Validate URLs before putting them in `href`.
- [ ] Markup rules: inline styles only, no classes; `<div>`/`<p>` instead of `<dl>`; single-quoted attributes.
- [ ] Orange header band (outline mode). White on `#F15A22` needs at least 1.5rem text, or 1.25rem bold.
- [ ] Two outputs: the fragment for pasting, and a full-document version (doctype, `lang="en"`, `<title>`) for the archive and Brightspace upload.
- [ ] Preview screen using an HTML text control.

### Save
- [ ] `Patch` a new row with the key columns, the `JSON(...)` payload, SchemaVersion and Revision.
- [ ] Call the flow with the item ID and HTML. Wrap the call in `IfError` and show a visible, announced status message. Display the returned file link.

### Form accessibility
- [ ] `AccessibleLabel` on every input and icon button, plus visible labels.
- [ ] Logical tab order: use containers, no stray TabIndex values.
- [ ] Visible focus on every control.
- [ ] Status and error labels use `Live` = Polite or Assertive.
- [ ] Reflow: usable at 320 px width / 400% zoom (responsive containers).
- [ ] AA contrast on all colours.
- [ ] Run App checker → Accessibility and fix every item.

---

## 6. Formatted copy

### 6a. Native tests (no admin needed — do first)
- [ ] Load the generated HTML into a Rich text editor control, then Ctrl+A, Ctrl+C, and paste into Brightspace and Drupal. Do the inline styles survive? Record which don't in `tests/`.
- [ ] Run the same test on the HTML text preview. Is it selectable at all?
- [ ] Run the same test with the rich text editor in View mode. Can users still select inside it?

### 6b. Code component (after IT approves)
- [ ] Install Power Platform CLI (`pac`) and confirm the Node version.
- [ ] Run `pac pcf init` (field template, no framework), e.g. `LangaraCopyFormatted`.
- [ ] Properties: `Html` and `ButtonLabel` as inputs; a `CopyResult` output (ok/failed) so the app can announce the result.
- [ ] Implement the copy, porting from `coursesyllabus2.html`:
  - `navigator.clipboard.write` with a `ClipboardItem` containing `text/html` and `text/plain`
  - fallback: `execCommand('copy')` on a hidden contenteditable element
- [ ] Render a real `<button>` with an accessible name and visible focus. No external calls.
- [ ] Test in the local harness, then deploy to the environment (push or solution import).
- [ ] Add it to the app. Announce the result, and show a fallback message if the copy fails.
- [ ] Check the app's licence designation still says Standard (app Details).

### 6c. File routes
- [ ] Brightspace: upload the archived `.html` as a content topic. Check rendering, inline styles and editability.
- [ ] "Copy helper" file: the same content plus a small Copy formatted button. Open it from Downloads (`file://`) and test.

### 6d. Fallback
- [ ] Write short instructor steps for pasting raw HTML into Brightspace and Drupal source views.

---

## 7. Power Automate

### Flow: "Course outline – archive and notify"
- [ ] Trigger: Power Apps (V2). Inputs: ItemId, CourseCode, Term, InstructorEmail, DocType, HtmlFragment, HtmlDocument.
- [ ] Create the file in `OutlineFiles` (naming from section 4) and set its metadata.
- [ ] Apply file permissions per the section 4 decision.
- [ ] Update the list item: set HtmlFile to the link and FlowStatus = Done.
- [ ] Send email (V2) to the instructor, plus the shared mailbox or chair if decided. Include the HTML body and the file link or an attachment.
- [ ] Respond to the app with the file URL and status.
- [ ] Error handling: Try/Catch scopes using run-after on failure. On failure, set FlowStatus = Failed, respond with the error text, and notify the owner.
- [ ] Confirm every action uses a standard connector (no Premium badge).
- [ ] Add a co-owner now; plan the move to a service account.
- [ ] Export the package to `flows/`.

### Optional
- [ ] Scheduled per-term summary email (counts by term and doc type), or rely on the grouped list view.
- [ ] Office Script generator (only if chosen in section 0): host workbook, script, Run script action. Limit is 1,600 Run script calls per user per day.

---

## 8. Dataverse (only for the code component)

- [ ] Confirm the environment has Dataverse (Power Platform admin center → environment details).
- [ ] Create a publisher and a solution to hold the component.
- [ ] Confirm no app data goes to Dataverse; it all stays in SharePoint.
- [ ] Document how to update the component: bump the version in the manifest before rebuilding.

---

## 9. Tests

### Data round trip
- [ ] Save, then reload: every field is identical, including repeating rows and their order.
- [ ] Special characters survive saving, reloading and HTML output: `& < > " '`, accents, smart quotes, em dashes, and `snəw̓eyəɬ leləm̓`.
- [ ] The largest realistic syllabus (long schedule, many assessments) saves and reloads without truncation.
- [ ] A record saved under an old schema loads correctly into a new app version (migration test).

### Prefill
- [ ] Loads the newest record when several terms exist.
- [ ] Two sections, same course, same instructor: behaves as decided in section 0.
- [ ] Same course, different instructor: does not load the other person's record (unless decided otherwise).
- [ ] Carry-over flags appear on term-sensitive fields and clear when the field is edited or confirmed.
- [ ] Delegation: seed 2,500+ dummy rows. The lookup still finds the right record, and there are no delegation warnings.

### Permissions
- [ ] Test user A can't see user B's rows, in the app or in the list.
- [ ] A Reviewer sees all rows.
- [ ] File visibility matches the section 4 decision.
- [ ] The flow works when a non-owner triggers it. Note whose connection it used.

### Licensing
- [ ] An ordinary faculty test account opens and uses the app with no trial or licence prompt.

### Output HTML
- [ ] Save sample outputs to `tests/` and run `node axe-run.js` and `node contrast.js` on them.
- [ ] F1003 checklist: every 3.2 item (outline) and every 3.4 item (syllabus) is present.
- [ ] WAVE check after pasting into Brightspace and into Drupal.

### Copy fidelity matrix
- [ ] Test every combination. Record results in `tests/copy-matrix.md`.

| | Brightspace editor | Brightspace source | Drupal editor | Drupal source |
|---|---|---|---|---|
| Code component | | | | |
| Rich text box | | | | |
| File / upload | | | | |
| Raw HTML (`Copy()`) | | | | |

- [ ] Repeat in Chrome, Edge, Firefox and Safari.
- [ ] Repeat in each host: Power Apps web player, Teams (if used), Brightspace iframe (later).

### Form accessibility
- [ ] Keyboard only: complete and submit a full syllabus.
- [ ] Screen readers (NVDA + Chrome; VoiceOver + Safari): check labels, errors, status messages and gallery rows.
- [ ] 400% zoom and Windows contrast themes.

### Failure paths
- [ ] Flow fails: the user sees a message, the row is marked Failed, and a retry works.
- [ ] Network drops during the save: clear error, no duplicate rows.

### Email
- [ ] The email renders acceptably in Outlook desktop, web and mobile.

---

## 10. Pilot and handover

- [ ] Pilot with 3–5 instructors in one department; log their feedback as issues.
- [ ] One-page instructor guide, including the source-view fallback.
- [ ] Runbook in the repo covering:
  - owners, environment and connections
  - how to update fixed wording when a policy changes
  - how to bump the schema version
- [ ] Move ownership to the service account and add co-owners.
- [ ] Brightspace embedding (pinned): iframe or widget, clipboard permission, SSO check.
