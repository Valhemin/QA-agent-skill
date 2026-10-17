# .claude/templates/ — standard templates for all outputs

Each output type in the repo has exactly one template here — skills and other docs **point here**,
they don't copy the template, so editing one place keeps everything in sync.

The `.example.` suffix marks these as blank templates, not real outputs — same convention as
`data/test-users.example.json`.

| File | When to use | Used by skill |
|---|---|---|
| `requirement.example.md` | Writing `requirements/<system>/<feature>.md` | `plan-testcases` |
| `testcase.example.md` | Writing `testcases/<system>/<feature>.md` (must have all 3 columns `TC-ID`/`Result`/`Notes`) | `plan-testcases` |
| `bug-report.example.md` | Drafting a bug report to paste into a ticket | `report-bug` |

MR description has no separate file here — GitLab auto-fills it from
`.gitlab/merge_request_templates/Default.md` when opening a new MR, no need to copy manually.
