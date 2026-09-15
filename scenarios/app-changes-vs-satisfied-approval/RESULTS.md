# app-changes-vs-satisfied-approval: run records

**Records appear under the `## Records` heading below, oldest first. `## Completeness` carries the
query that reports how many, and the forge audit that checks it.**

Records are appended at the end of this file, oldest first, one per workflow run. Every workflow run
owes exactly one record, whatever its verdict, including a run that failed preflight and a run whose
dispatched phase is setup rather than measurement. Read [RUNBOOK.md](RUNBOOK.md) for the scenario
itself and [../../README.md](../../README.md) for the standing conventions.

## Verdict vocabulary

| Token | Meaning |
|---|---|
| `VALID` | Every control read HELD and every arm produced a reading. The measurement counts. |
| `REFUTED` | At least one control read REFUTED. The apparatus was not in the assumed state. |
| `INDETERMINATE` | No control read REFUTED, and either a control read its indeterminate set or a reading fell outside every declared set. |
| `NOT-RUN` | The workflow never reached the measurement. Preflight failed, the job errored first, the run was cancelled, or the dispatched phase reaches no measurement by design. |

A run that proves nothing is called an invalid run elsewhere. That word covers the three tokens
other than `VALID` and is not itself a verdict token. No record here carries it, because `VALID` is
a substring of it and the query below would then return every bad run alongside the good ones.

Find the most recent valid run with:

```
grep -n '^## .* VALID$' scenarios/app-changes-vs-satisfied-approval/RESULTS.md | tail -1
```

The `$` anchor is why a record's verdict is the last token on its heading line, with no trailing
whitespace. Substituting `REFUTED`, `INDETERMINATE`, or `NOT-RUN` answers the matching question, and
dropping `| tail -1` lists every run of that class with its line number.

## Record format

The block below is the format. It holds no data. Every value in it is written inside angle brackets
and describes what goes there.

````
## <UTC timestamp, as 2026-01-31T14:32:07Z> <VERDICT>

Outcome-row: <a row id from the RUNBOOK outcome table, or NONE>
Verdict-reason: <one line. For VALID, which controls held and that exactly one row matched. Otherwise the specific cause, naming a control id or a prerequisite id.>
Actor: <the GitHub login that dispatched the run>
Scenario-commit: <the full 40-character sha of this repository at the moment of dispatch>
Subject-pin: none
Workflow-run: <numeric run id> <run URL>
Workflow-conclusion: <the conclusion GitHub reports for the run, verbatim. success, failure, cancelled, skipped, startup_failure, timed_out, neutral, stale and action_required are the values seen so far, and any other value the API returns is written as it came rather than mapped onto one of these.>
Dispatched-phase: <open-pr, approve, or request-changes, as the phase input carried it>
Dispatched-against: <the pull request number or the head branch the dispatch named>
Preconditions:
  branch-protection-main: <the full protection JSON as read back at procedure step 2>
  codeowners-absent: <the HTTP status each of P4's three contents calls returned, naming the three paths>
  actions-can-approve-pull-request-reviews: <the can_approve_pull_request_reviews value from gh api repos/patrickg-unity/agent-workflow-tests/actions/permissions/workflow, as it stood while this run executed, which is true for a run taken under P6 and not this repository's standing false>
Arms:
  experiment-app-approver: <PR number, head branch including its run discriminator, author, approver>
  experiment-other-approver: <PR number, head branch including its run discriminator, author, approver>
  control-protection: <PR number, head branch including its run discriminator, author>
  control-approval-in-force: the before reading of the pull request this run dispatched against, read once and classified twice
  control-identity: the after reading of the pull request this run dispatched against, read once and classified twice
Controls:
  control-protection: EXPECT <the held set> | OBSERVED <the reading, including the reviews and latestReviews fields, which this control's held set binds> | <HELD | REFUTED | INDETERMINATE>
  control-approval-in-force: EXPECT <the held set> | OBSERVED <the before reading of the dispatched pull request> | <HELD | REFUTED | INDETERMINATE | NOT-APPLICABLE>
  control-identity: EXPECT <the held set for the dispatched phase> | OBSERVED <the entry count, and each entry's author and state, in reviews and in latestReviews, from the after reading> | <HELD | REFUTED | INDETERMINATE | NOT-APPLICABLE>
Readings:
  <arm id>, <before | after>, <the exact instrument command that produced the block below>
  ```
  <the instrument's output, verbatim and unedited>
  ```
  acting-identity provenance, after, <the exact gh run view command that produced the block below>
  ```
  <the Reviewing as line from the run log, verbatim and unedited, or the Opened as line for an open-pr dispatch>
  ```
Residue: <what this run actually left behind, which can differ from the RUNBOOK teardown when a run failed partway>
Amends: <the UTC timestamp of the record this one corrects, or none>
Notes: <anything else, or none>
````

Fields appear in that order. A field with no value is written with an explicit `none` rather than
left out, so a reader can tell an empty field from a forgotten one.

`Outcome-row` is first because it is the answer. `Verdict-reason` is required for every verdict,
`VALID` included. `Scenario-commit` pins the version of the outcome table the row id was read
against, without which a row id cites a moving target, because the RUNBOOK is mutable. `Controls`
carries the expected set and the observed reading on one line, so the verdict token can be checked
against its own evidence without opening the RUNBOOK.

`Dispatched-phase` is a field here and not in the other scenarios on this surface because this
workflow performs one of three operations per dispatch and the outcome table's rows are selected by
which one. A record without it cannot be classified by a later reader, and it is placed immediately
after the run fields because it is a property of the dispatch rather than of the apparatus.

A control that the dispatched phase does not reach is written `NOT-APPLICABLE` rather than omitted.
`control-approval-in-force` and `control-identity` are both `NOT-APPLICABLE` on an `open-pr`
dispatch, which submits no review and reads no experiment arm, and `control-approval-in-force` is
`NOT-APPLICABLE` on an `approve` dispatch, whose whole purpose is to create the state that control
asserts. A run with a `NOT-APPLICABLE` control is not a run with a control that held.

`Readings` carries one entry per arm and moment this run produced, with the arm id written out, and
carries the instrument's output verbatim. Workflow logs are deleted after 90 days and a public
repository cannot extend that ceiling, so every record has a date after which its `Workflow-run` URL
answers nothing, and this file is the only copy that survives it.

## Corrections

A record is never edited to change what it says. Append a new record carrying
`Amends: <the corrected record's timestamp>` and enough of the corrected record to stand alone.

Then make the one permitted in-place edit and no other: append the literal token ` SUPERSEDED` to
the end of the corrected record's heading line, turning `## <timestamp> VALID` into
`## <timestamp> VALID SUPERSEDED`. The `$` anchor in the query above then stops matching that line,
so a superseded record is excluded with no change to the query.

## Completeness

The record set is auditable against the forge:

```
gh run list --repo patrickg-unity/agent-workflow-tests --workflow app-changes-vs-satisfied-approval.yml --limit 200 --json databaseId
```

Every run id it returns appears in exactly one `Workflow-run` field, and the number of record
headings in this file equals the number of ids returned. Count the record headings with
`grep -cE '^## [0-9]{4}-[0-9]{2}-[0-9]{2}T'`, which matches a heading only when it opens with a
record's UTC timestamp. A plain `grep -c '^## '` is the wrong instrument here, because it also
counts this file's own section headings and the format block's example heading, so it reports six
more than the record count on every scenario. A dispatch the API rejected before creating a run has
no run id and owes no record. Run metadata is expected to outlive the 90-day log retention, which
has not been confirmed. If it does not, the audit is reliable only inside the 90-day window and the
older records are checked against each other.

## Records

None.
