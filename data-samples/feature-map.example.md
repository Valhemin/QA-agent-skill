# Feature → Path / Tag mapping

The single source AI reads to know which folder a feature lives in and what its tag is.
**When adding a new feature: add one row to this table** (don't edit the rule file).
Kept separate from the rule file to avoid git conflicts when two people add features at once.

| Feature name | Path | Tag |
|---|---|---|
| auth / login | `tests/auth/` | `@auth` |
| checkout | `tests/checkout/` | `@checkout` |
| dashboard | `tests/dashboard/` | `@dashboard` |

Replace these rows with your own project's features. Each feature = one folder under
`tests/<feature>/` and `pages/<feature>/`, with a matching `@<feature>` tag
(see `.claude/rules/writing-tests.md`).
