# /spec

Generate a `SPEC.md` from a Jira ticket and create a feature branch automatically.

---

## Usage

```
/spec <TICKET-ID>
```

**Example:**
```
/spec APP-123
```

---

## What this command does

1. Takes the Jira ticket ID from `$ARGUMENTS`
2. Creates a new git branch from `develop` using the convention `feature/<TICKET-ID>-<short-title>`
3. Fetches all ticket data from Jira via the Atlassian MCP
4. Reads the spec template from `.claude/templates/spec_template.md`
5. Maps Jira fields to the template sections
6. Writes the final file to `.claude/specs/<TICKET-ID>-spec.md`

---

## Instructions

Follow these steps in order. Do not skip any step.

### Step 1 — Validate input

- Read `$ARGUMENTS` and extract the ticket ID (e.g. `APP-123`)
- If no ticket ID is provided, stop and ask the user: *"Please provide a Jira ticket number. Usage: /spec APP-123"*

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

### Step 3 — Derive the branch short title

- Take the `summary` field from Jira
- Convert to lowercase
- Remove special characters (keep only letters, numbers, hyphens)
- Truncate to a maximum of 5 words
- Replace spaces with hyphens

**Example:**
- Summary: `"Add biometric login for iOS users"`
- Short title: `add-biometric-login-ios`
- Full branch name: `feature/APP-123-add-biometric-login-ios`

### Step 4 — Create the git branch

Run the following shell commands in order:

```bash
git checkout develop
git pull origin develop
git checkout -b feature/<TICKET-ID>-<short-title>
```

If `develop` branch does not exist, try `main` instead.
If git commands fail, report the error clearly and stop — do not proceed to create the spec file.

### Step 5 — Read the spec template

Read the file at `.claude/templates/spec_template.md` exactly as-is.

### Step 6 — Map Jira data to the template

Replace placeholders with the data fetched from Jira:

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

For any placeholder where no Jira data is available, replace it with:
`<!-- No data found — fill in manually -->`

Do not remove any sections from the template. Every section must be present in the output.

### Step 7 — Write the spec file

Write the final content to:
```
.claude/specs/<TICKET-ID>-spec.md
```

**Example output path:**
```
.claude/specs/APP-123-spec.md
```

### Step 8 — Confirm completion

Report back to the user with a summary like this:

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
