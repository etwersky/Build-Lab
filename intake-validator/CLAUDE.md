# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```
npm install                          # install dependencies (none currently — zero runtime deps)
npm start                            # run the CLI against data/sample-records.json
node index.js <path-to-file>         # run the CLI against an arbitrary JSON file
npm test                             # run the full test suite
node --test tests/processor.test.js  # same, invoked directly
node --test --test-name-pattern="averageScore is correct for two records"  # run a single test by name
```

There is no build step, linter, or bundler configured — this is plain CommonJS Node.js.

## Architecture

This is a small CLI tool with a strict one-way pipeline, wired together in [index.js](index.js):

1. **Read + parse** — `index.js` reads the file path from `argv`, reads it, and `JSON.parse`s it. It exits with a non-zero code and a `console.error` message for every failure mode (missing arg, missing file, bad JSON, non-array root) rather than throwing.
2. **Validate** — [src/processor.js](src/processor.js) `validateRecords()` runs each record through `validateRecord()`, which checks fields in a fixed order (`id` → `name` → `email` → `status` → `score` → `submittedAt`) and returns on the first failure. Records are partitioned into `{ valid, invalid }`, where each invalid entry keeps the original `record` plus a human-readable `reason` string. The set of allowed `status` values lives in the `VALID_STATUSES` constant at the top of the file.
3. **Summarise** — `summarise(valid)` (also in processor.js) computes aggregate stats (count by status, average score, oldest/newest `submittedAt`) over the *valid* records only. It returns an all-null/zero shape when given an empty array — callers don't need a separate empty-check.
4. **Report** — [src/reporter.js](src/reporter.js) `formatReport(summary, invalid)` turns the two pieces of data above into the final human-readable text block (the boxed report with the status bar chart and the list of invalid records with reasons). This is the only place output formatting lives.

Key conventions to preserve when extending this:
- Validation failures never throw — they're modeled as data (`{ ok: false, reason }` / `{ record, reason }`), and `index.js` is the only place that calls `process.exit`.
- `processor.js` has no knowledge of output formatting, and `reporter.js` has no knowledge of validation rules — keep that split when adding fields.
- Adding a new required field means updating: the check in `validateRecord`, the JSDoc field list at the top of processor.js, and likely `summarise`/`formatReport` if it should appear in the aggregate output.

## Known issue

`summarise()` in [src/processor.js](src/processor.js) divides `scoreTotal` by `records.length + 1` instead of `records.length` (see the `BUG:` comment at line 113-114), so `averageScore` is always reported slightly low. This is intentionally left in place — do not silently fix it as a drive-by change.
