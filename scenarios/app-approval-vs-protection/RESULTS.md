# app-approval-vs-protection: run records

**Records appear under the `## Records` heading below, oldest first. `## Completeness` carries the
query that reports how many, and the forge audit that checks it.**

Records are appended at the end of this file, oldest first, one per workflow run. Every workflow run
owes exactly one record, whatever its verdict, including a run that failed preflight. Read
[RUNBOOK.md](RUNBOOK.md) for the scenario itself and [../../README.md](../../README.md) for the
standing conventions.

## Verdict vocabulary

| Token | Meaning |
|---|---|
| `VALID` | Every control read HELD and every arm produced a reading. The measurement counts. |
| `REFUTED` | At least one control read REFUTED. The apparatus was not in the assumed state. |
| `INDETERMINATE` | No control read REFUTED, and either a control read its indeterminate set or a reading fell outside every declared set. |
| `NOT-RUN` | The workflow never reached the measurement. Preflight failed, the job errored first, or the run was cancelled. |

A run that proves nothing is called an invalid run elsewhere. That word covers the three tokens
other than `VALID` and is not itself a verdict token. No record here carries it, because `VALID` is
a substring of it and the query below would then return every bad run alongside the good ones.

Find the most recent valid run with:

```
grep -n '^## .* VALID$' scenarios/app-approval-vs-protection/RESULTS.md | tail -1
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
Preconditions:
  branch-protection-main: <the full protection JSON as read back at procedure step 2>
  actions-can-approve-pull-request-reviews: <the can_approve_pull_request_reviews value from gh api repos/patrickg-unity/agent-workflow-tests/actions/permissions/workflow>
Arms:
  experiment: <PR number, head branch including its run discriminator, author>
  control-protection: <PR number, head branch including its run discriminator, author>
  control-identity: the same pull request as experiment, read once and classified twice
Controls:
  control-protection: EXPECT <the held set> | OBSERVED <the reading> | <HELD | REFUTED | INDETERMINATE>
  control-identity: EXPECT exactly one review authored by workflow-test-agent | OBSERVED <the review count and each author, from the experiment arm's after reading> | <HELD | REFUTED | INDETERMINATE>
Readings:
  experiment, before, <the exact instrument command that produced the block below>
  ```
  <the instrument's output, verbatim and unedited>
  ```
  experiment, after, <the exact instrument command>
  ```
  <the instrument's output, verbatim and unedited>
  ```
  control-protection, before, <the exact instrument command>
  ```
  <the instrument's output, verbatim and unedited>
  ```
  control-protection, after, <the exact instrument command>
  ```
  <the instrument's output, verbatim and unedited>
  ```
  acting-identity provenance, after, <the exact gh run view command that produced the block below>
  ```
  <the Approving as line from the run log, verbatim and unedited>
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

`Readings` carries the instrument's output verbatim. Workflow logs are deleted after 90 days and a
public repository cannot extend that ceiling, so every record has a date after which its
`Workflow-run` URL answers nothing, and this file is the only copy that survives it.

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
gh run list --repo patrickg-unity/agent-workflow-tests --workflow app-approval-vs-protection.yml --limit 200 --json databaseId
```

Every run id it returns appears in exactly one `Workflow-run` field, and the number of record
headings in this file equals the number of ids returned. Count the record headings with
`grep -cE '^## [0-9]{4}-[0-9]{2}-[0-9]{2}T'`, which matches a heading only when it opens with a
record's UTC timestamp. A plain `grep -c '^## '` is the wrong instrument here, because it also
counts this file's own section headings and the format block's example heading, so it reports six
more than the record count on every scenario. A dispatch the API rejected before creating a run has
no run id and owes no record. Run metadata is expected to outlive the 90-day log
retention, which has not been confirmed. If it does not, the audit is reliable only inside the
90-day window and the older records are checked against each other.

## Records

## 2026-09-15T15:41:42Z VALID

Outcome-row: R1
Verdict-reason: Both controls held, control-protection and control-identity, and exactly one row matched. The experiment arm read reviewDecision APPROVED with mergeStateStatus CLEAN while the control arm, which received nothing, stayed REVIEW_REQUIRED and BLOCKED.
Actor: patrickg-unity
Scenario-commit: c13b1c2e37c990eda2886bc21a3384e0f10d8f1e
Subject-pin: none
Workflow-run: 34989899690 https://github.com/patrickg-unity/agent-workflow-tests/actions/runs/34989899690
Workflow-conclusion: success
Preconditions:
  branch-protection-main: {"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection","required_pull_request_reviews":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_pull_request_reviews","dismiss_stale_reviews":false,"require_code_owner_reviews":false,"require_last_push_approval":false,"required_approving_review_count":1},"required_signatures":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_signatures","enabled":false},"enforce_admins":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/enforce_admins","enabled":false},"required_linear_history":{"enabled":false},"allow_force_pushes":{"enabled":false},"allow_deletions":{"enabled":false},"block_creations":{"enabled":false},"required_conversation_resolution":{"enabled":false},"lock_branch":{"enabled":false},"allow_fork_syncing":{"enabled":false}}
  actions-can-approve-pull-request-reviews: false
Arms:
  experiment: PR 1, test/app-approval-a, patrickg-unity
  control-protection: PR 2, test/app-approval-b, patrickg-unity
  control-identity: the same pull request as experiment, read once and classified twice
Controls:
  control-protection: EXPECT an unapproved pull request on main reads reviewDecision REVIEW_REQUIRED with mergeStateStatus BLOCKED | OBSERVED reviewDecision REVIEW_REQUIRED, mergeStateStatus BLOCKED, reviews empty | HELD
  control-identity: EXPECT exactly one review authored by workflow-test-agent | OBSERVED one review, author workflow-test-agent, state APPROVED, on commit 93688c0d464227382892fd6f87dacba11da1b165 | HELD
Readings:
  experiment, before, gh pr view 1 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
  ```
  {"mergeStateStatus":"BLOCKED","reviewDecision":"REVIEW_REQUIRED","reviews":[]}
  ```
  experiment, after, gh pr view 1 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
  ```
  {"mergeStateStatus":"CLEAN","reviewDecision":"APPROVED","reviews":[{"id":"PRR_kwDOUb-xbs8AAAABNqyKdA","author":{"login":"workflow-test-agent"},"authorAssociation":"NONE","body":"App approval test.","submittedAt":"2026-09-15T15:39:41Z","includesCreatedEdit":false,"reactionGroups":[],"state":"APPROVED","commit":{"oid":"93688c0d464227382892fd6f87dacba11da1b165"}}]}
  ```
  control-protection, before, gh pr view 2 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
  ```
  {"mergeStateStatus":"BLOCKED","reviewDecision":"REVIEW_REQUIRED","reviews":[]}
  ```
  control-protection, after, gh pr view 2 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
  ```
  {"mergeStateStatus":"BLOCKED","reviewDecision":"REVIEW_REQUIRED","reviews":[]}
  ```
  acting-identity provenance, after, gh run view 34989899690 --repo patrickg-unity/agent-workflow-tests --log | grep -F 'Approving as'
  ```
  measure	Report the identity the review is submitted as	2026-09-15T15:39:40.2244641Z Approving as workflow-test-agent
  ```
Residue: Both pull requests closed unmerged. Both branches test/app-approval-a and test/app-approval-b left in place, local and remote, because branch deletion was outside this run's authorized writes. The RUNBOOK teardown's branch deletion was therefore NOT performed and a second run needs it done first. Branch protection left on, as the charter requires. This record was not landed in the repository by the run that produced it, for the reason under Notes.
Amends: none
Notes: The instrument's two fields moved together. The reviews array carrying an approval and reviewDecision reading APPROVED are separate claims and the RUNBOOK names their conflation as the single most likely misreading of this scenario, so both are recorded and both agree.

The review's author.login reads workflow-test-agent with no [bot] suffix. A cross-check through the REST reviews endpoint returns the same review's user.login as workflow-test-agent[bot]. control-identity is declared against the GraphQL form because gh pr view --json is the instrument. A held set written against the REST form would have read REFUTED here and thrown away a run in which nothing was wrong.

This record could not be appended to RESULTS.md by the session that produced it. Procedure step 9 appends to a file on main, and by step 9 main is protected and requires one approving review, so landing a record needs a branch, a pull request and a merge. None of those is among the writes this run was authorized to make, and merging was explicitly excluded. The scenario's own procedure cannot complete under the protection the scenario itself enables. That is a defect in the procedure rather than in this run, and it is reported rather than worked around.
