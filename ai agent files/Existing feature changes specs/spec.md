# /spec

Generate a `SPEC.md` from a Jira ticket and create a feature branch automatically.

Supports two modes:
- **New feature:** `/spec <TICKET-ID>`
- **Change request:** `/spec <TICKET-ID> --cr`

---

## Usage

```
/spec APP-123          ← new feature
/spec APP-123 --cr     ← change request on existing feature
```

---

## Instructions

Follow these steps in order. Do not skip any step.

### Step 1 — Validate input

- Read `$ARGUMENTS`
- Extract the ticket ID (e.g. `APP-123`)
- Check if `--cr` flag is present in `$ARGUMENTS`
  - If `--cr` is present → set mode to **change request**
  - If not → set mode to **new feature**
- If no ticket ID is provided, stop and ask: *"Please provide a Jira ticket number. Usage: /spec APP-123 or /spec APP-123 --cr"*

---

### Step 2 — Fetch the Jira ticket

Use the Atlassian MCP to call `get_issue` with the ticket ID.

Extract the following fields:
- `summary` → ticket title
- `description` → full description
- `priority.name` → ticket priority
- `labels` → comma-separated list
- `status.name` → current status
- `assignee.displayName` → assignee name
- `acceptanceCriteria` or any custom field containing acceptance criteria
- Any field related to UX notes, edge cases, or requirements
- The ticket URL (format: `https://<your-jira-domain>/browse/<TICKET-ID>`)

If the ticket is not found, stop and tell the user: *"Ticket <ID> was not found in Jira. Please check the ticket number and try again."*

---

### Step 3 — Derive the branch short title

- Take the `summary` field from Jira
- Convert to lowercase
- Remove special characters (keep only letters, numbers, hyphens)
- Truncate to a maximum of 5 words
- Replace spaces with hyphens

**Example:**
- Summary: `"Update car listing filter behaviour"`
- Short title: `update-car-listing-filter`
- Full branch name: `feature/APP-123-update-car-listing-filter`

---

### Step 4 — Create the git branch

Run the following shell commands in order:

```bash
git checkout develop
git pull origin develop
git checkout -b feature/<TICKET-ID>-<short-title>
```

If `develop` branch does not exist, try `main` instead.
If git commands fail, report the error clearly and stop — do not proceed further.

---

### Step 5 — Branch based on mode

---

## IF MODE = NEW FEATURE

### Step 5A — Read the new feature template

Read the file at `.claude/templates/spec_template.md` exactly as-is.

### Step 6A — Map Jira data to template

Replace placeholders with data fetched from Jira:

| Placeholder | Source |
|---|---|
| `{{TICKET_ID}}` | Ticket ID (e.g. APP-123) |
| `{{TITLE}}` | `summary` field |
| `{{SHORT_TITLE}}` | Derived short title from Step 3 |
| `{{PRIORITY}}` | `priority.name` |
| `{{LABELS}}` | `labels` joined with `, ` |
| `{{TICKET_URL}}` | Full Jira URL to the ticket |
| `{{SUMMARY}}` | `description` — clean up Jira markup to readable markdown |
| `{{REQUIREMENTS}}` | Extract from description or dedicated requirements field |
| `{{ACCEPTANCE_CRITERIA}}` | Acceptance criteria field or section from description |
| `{{UX_NOTES}}` | Any UX/design notes from the ticket |
| `{{EDGE_CASES}}` | Any edge cases or error handling notes |
| `{{GOALS}}` | Infer from summary + description if not explicit |
| `{{DATE}}` | Today's date in YYYY-MM-DD format |

For any placeholder where no Jira data is available, replace with:
`<!-- No data found — fill in manually -->`

Do not remove any sections. Every section must be present in the output.

### Step 7A — Write the spec file

Write to: `.claude/specs/<TICKET-ID>-spec.md`

### Step 8A — Confirm completion

```
✅ Spec created successfully

  Ticket:   APP-123 — Add biometric login for iOS users
  Branch:   feature/APP-123-add-biometric-login-ios
  Spec:     .claude/specs/APP-123-spec.md

Next steps:
  1. Review and fill in any sections marked <!-- No data found -->
  2. Add Figma link under UX & Design
  3. Commit the spec: git add .claude/specs/APP-123-spec.md && git commit -m "docs: add spec for APP-123"
```

---

## IF MODE = CHANGE REQUEST (--cr)

### Step 5B — Extract keywords from ticket

From the Jira `summary` and `description`, extract 4–8 meaningful keywords that relate to the feature being changed. Ignore generic words like "update", "fix", "change", "improve".

**Example:**
- Summary: `"Update filter behaviour on car listing screen"`
- Keywords: `filter`, `listing`, `car`, `search`

---

### Step 6B — Phase 1: Auto-scan the codebase

Search the Flutter project for files related to the extracted keywords.

Run these searches:

```bash
# Search for keyword matches in lib/ directory
grep -rl "<keyword1>" lib/ --include="*.dart"
grep -rl "<keyword2>" lib/ --include="*.dart"

# Also search by filename pattern
find lib/ -name "*<keyword1>*" -o -name "*<keyword2>*"
```

Repeat for each keyword. Collect all unique matching file paths.

Then group the findings into:
- **Screens** — files ending in `_screen.dart` or `_page.dart`
- **BLoCs / Controllers** — files ending in `_bloc.dart`, `_cubit.dart`, `_controller.dart`
- **Models** — files ending in `_model.dart`, `_entity.dart`
- **Repositories / Services** — files ending in `_repository.dart`, `_service.dart`
- **Widgets** — files ending in `_widget.dart` or inside a `widgets/` folder
- **Other** — anything else relevant

---

### Step 7B — Phase 2: Show findings and ask for confirmation

Present the findings to the user clearly before reading any file:

```
📂 Files found related to APP-123

  Screens:
    lib/features/listing/filter_screen.dart
    lib/features/listing/listing_screen.dart

  BLoCs / Controllers:
    lib/features/listing/bloc/filter_bloc.dart
    lib/features/listing/bloc/filter_event.dart
    lib/features/listing/bloc/filter_state.dart

  Models:
    lib/models/filter_model.dart

  Other:
    lib/core/constants/filter_constants.dart

Are these the right files?
  • Type YES to proceed with these files
  • Type ADD <filepath> to include a file I missed
  • Type REMOVE <filepath> to exclude a file
  • Type DONE when the list is finalised
```

Wait for the user to respond before proceeding.

Process their input:
- `YES` or `DONE` → proceed with current list
- `ADD lib/some/file.dart` → add that file to the list, confirm addition
- `REMOVE lib/some/file.dart` → remove from list, confirm removal
- After any ADD or REMOVE, re-display the updated list and ask again until user says YES or DONE

---

### Step 8B — Read confirmed files

Read every file in the confirmed list. For each file, understand:
- What the file currently does
- What state, logic, or UI it manages
- How it connects to other files in the list

Do not write the spec yet. Build a full mental model of the current implementation first.

---

### Step 9B — Read the change request template

Read the file at `.claude/templates/spec_cr_template.md` exactly as-is.

---

### Step 10B — Map Jira data + codebase findings to CR template

Replace placeholders using both Jira data and what was learned from reading the files:

| Placeholder | Source |
|---|---|
| `{{TICKET_ID}}` | Ticket ID |
| `{{TITLE}}` | `summary` field |
| `{{SHORT_TITLE}}` | Derived short title |
| `{{PRIORITY}}` | `priority.name` |
| `{{LABELS}}` | `labels` joined with `, ` |
| `{{TICKET_URL}}` | Full Jira URL |
| `{{CHANGE_TYPE}}` | Infer from ticket: UI / Logic / API / Mixed |
| `{{SUMMARY}}` | `description` cleaned to markdown |
| `{{CURRENT_BEHAVIOUR}}` | Described from reading actual code files |
| `{{REQUESTED_CHANGE}}` | What Jira is asking to change |
| `{{WHAT_STAYS}}` | Parts of existing code that must NOT change |
| `{{AFFECTED_FILES}}` | Confirmed file list from Phase 2 |
| `{{BEFORE_AFTER}}` | Side-by-side or descriptive before/after comparison |
| `{{IMPLEMENTATION_PLAN}}` | Step-by-step what to change in which file |
| `{{REGRESSION_RISK}}` | What existing behaviour could break |
| `{{ACCEPTANCE_CRITERIA}}` | From Jira |
| `{{EDGE_CASES}}` | From Jira + inferred from code |
| `{{DATE}}` | Today's date in YYYY-MM-DD format |

For `{{CURRENT_BEHAVIOUR}}` and `{{IMPLEMENTATION_PLAN}}` — these must be based on the actual code read, not guesses.

For any placeholder where no data is available, replace with:
`<!-- No data found — fill in manually -->`

---

### Step 11B — Write the spec file

Write to: `.claude/specs/<TICKET-ID>-spec.md`

---

### Step 12B — Confirm completion

```
✅ Change request spec created successfully

  Ticket:   APP-123 — Update filter behaviour on car listing screen
  Branch:   feature/APP-123-update-filter-behaviour
  Mode:     Change Request
  Files read: 6 files
  Spec:     .claude/specs/APP-123-spec.md

Next steps:
  1. Review "Current Behaviour" section — confirm it matches your understanding
  2. Review "What Stays the Same" — add anything Claude may have missed
  3. Review "Regression Risk" — flag anything sensitive to your team
  4. Commit the spec: git add .claude/specs/APP-123-spec.md && git commit -m "docs: add CR spec for APP-123"
```
