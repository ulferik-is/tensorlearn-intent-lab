# Intent

## Goal
A support coordinator needs to preview the tickets contained in a CSV file before
anyone approves a later import. The outcome is a pure, side-effect-free preview
function: given CSV text, it either returns every ticket it would create, in input
order, or a single error code explaining why the file is not acceptable. Nothing is
imported, saved or sent anywhere; the preview only reports what an import would do.

## Scope
The implementation may add exactly two new files: `src/import-tickets.mjs`, which
exports the preview function (proposed name `previewTickets(csvText)`), and
`tests/import-tickets.test.mjs`, which covers the acceptance cases below with
`node:test`. Deliberately excluded: any UI, database, network calls, file reads or
writes, CLI wrappers, new package dependencies, and the actual import step. Do not
edit `src/tickets.mjs`, its tests, `.github/`, `CLAUDE.md`, `package.json` or this
document. If the work seems to need anything outside these two files, stop and ask
before expanding scope.

## Constraints
- `src/tickets.mjs` is the stable ticket representation. It may be imported and
  reused (for example `ticket(title, priority)`), but never modified. Its errors must
  be translated into the `{"error": code}` result rather than thrown to the caller.
- The function is preview-only: it takes a string, returns a plain object, and has
  no side effects. No file system, network, environment or global state access.
- Use only Node.js 22+ built-ins; write a small CSV parser rather than adding a
  dependency. It must respect double-quoted fields, including commas inside quotes
  and doubled quotes (`""`) as an escaped quote.
- Columns: `title` is required and `priority` is optional. Any other column name, a
  missing `title`, a duplicate column name, an empty input with no header, or a data
  row whose field count differs from the header all fail with `unknown-column`.
- Values are trimmed. A blank title fails with `empty-title`. Priority must be `low`,
  `normal` or `urgent`; a missing column or blank value becomes `normal`; anything
  else fails with `invalid-priority`.
- At most 100 data rows. The row count is checked after the header is validated but
  before any row values are validated, so 101 rows fail with `row-limit` even when
  some rows also contain invalid values.
- Success is exactly `{"tickets":[{"title":"...","priority":"..."}]}`. Failure is
  exactly `{"error":"<code>"}`. There is never a partial success.

## Plan
1. Before writing code, restate the scope and the open assumptions listed under
   Verification, and wait for the user to approve implementation.
2. Write `tests/import-tickets.test.mjs` first, with one test per acceptance case in
   the JSON contract (expanding `repeat` into that many copies of the data row).
3. In `src/import-tickets.mjs`, parse the CSV text into rows of fields, honouring
   quotes and ignoring a single trailing newline.
4. Validate the header (required, allowed, unique column names); otherwise return
   `unknown-column`.
5. Count data rows and return `row-limit` when there are more than 100.
6. Walk rows in input order: check the field count, trim values, default a blank
   priority to `normal`, then build each ticket with `ticket()` from
   `src/tickets.mjs`. Return the first error found; title is checked before priority.
7. Return `{ tickets }` only when every row is valid. Run the verification commands
   and report the results.

## Acceptance
The preview is complete when every case in the contract below produces exactly its
expected object, deep-equal, with no extra keys. Positive cases include explicit and
default priority, a quoted title containing a comma, trimming, and order preservation.
Negative cases cover each error code: empty title, unknown, missing and duplicate
columns, an empty file, invalid priority and too many rows. The existing
`tests/tickets.test.mjs` must keep passing unchanged.

```json
{
  "schema": 1,
  "change": "ticket-csv-preview",
  "allowed_changes": ["src/import-tickets.mjs", "tests/import-tickets.test.mjs"],
  "protected_paths": ["src/tickets.mjs", ".github/", "CLAUDE.md"],
  "non_goals": ["ui", "database", "network"],
  "requirements": {
    "columns": ["title", "priority"],
    "default_priority": "normal",
    "valid_priorities": ["low", "normal", "urgent"],
    "reject_unknown_columns": true,
    "reject_empty_title": true,
    "max_rows": 100,
    "side_effects": "preview-only"
  },
  "verification": ["npm test", "git diff --check"],
  "approval": "ask-before-implementation",
  "acceptance_cases": [
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "expected": {"tickets":[{"title":"Printer issue","priority":"urgent"}]}
    },
    {
      "csv": "title\nPassword reset\n",
      "expected": {"tickets":[{"title":"Password reset","priority":"normal"}]}
    },
    {
      "csv": "title,priority\n  Password reset  ,\nPrinter issue,low\n",
      "expected": {"tickets":[{"title":"Password reset","priority":"normal"},{"title":"Printer issue","priority":"low"}]}
    },
    {
      "csv": "title,priority\n\"Question about billing, October\",normal\n",
      "expected": {"tickets":[{"title":"Question about billing, October","priority":"normal"}]}
    },
    {
      "csv": "title,priority\n   ,urgent\n",
      "expected": {"error":"empty-title"}
    },
    {
      "csv": "title,owner\nPrinter issue,Sam\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "priority\nurgent\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "title,title\nPrinter issue,Password reset\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "title,priority\nPrinter issue,critical\n",
      "expected": {"error":"invalid-priority"}
    },
    {
      "csv": "title\nPrinter issue\n",
      "repeat": 101,
      "expected": {"error":"row-limit"}
    }
  ]
}
```

## Verification
Run `npm test` and `git diff --check`; both must succeed, and `git status` must show
changes only in the two allowed files. Ask the user for approval before starting the
implementation, and again before touching any other file, adding a dependency, or
resolving one of these open assumptions differently: header names match exactly and
case-sensitively; a header with no data rows returns `{"tickets":[]}`; priority values
are case-sensitive, so `Urgent` is invalid; blank lines in the middle of the file are
malformed rows. Do not commit, push or open a PR unless the user asks.
