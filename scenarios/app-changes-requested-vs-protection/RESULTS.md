# app-changes-requested-vs-protection: run records

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
grep -n '^## .* VALID$' scenarios/app-changes-requested-vs-protection/RESULTS.md | tail -1
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
  codeowners-absent: <the HTTP status each of P4's three contents calls returned, naming the three paths>
  actions-can-approve-pull-request-reviews: <the can_approve_pull_request_reviews value from gh api repos/patrickg-unity/agent-workflow-tests/actions/permissions/workflow>
Arms:
  experiment: <PR number, head branch including its run discriminator, author>
  control-protection: <PR number, head branch including its run discriminator, author>
  control-identity: the same pull request as experiment, read once and classified twice
Controls:
  control-protection: EXPECT <the held set> | OBSERVED <the reading, including the reviews field, which this control's held set binds> | <HELD | REFUTED | INDETERMINATE>
  control-identity: EXPECT exactly one review authored by workflow-test-agent with state CHANGES_REQUESTED | OBSERVED <the review count, and each entry's author and state, from the experiment arm's after reading> | <HELD | REFUTED | INDETERMINATE>
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
  <the Requesting changes as line from the run log, verbatim and unedited>
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
gh run list --repo patrickg-unity/agent-workflow-tests --workflow app-changes-requested-vs-protection.yml --limit 200 --json databaseId
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

## 2026-09-15T18:56:21Z VALID

Outcome-row: R1
Verdict-reason: Both controls held, control-protection and control-identity, and exactly one row matched. The experiment arm read reviewDecision CHANGES_REQUESTED with mergeStateStatus BLOCKED, while the control arm, which received nothing, read REVIEW_REQUIRED with BLOCKED. Per row R1 that establishes registration in reviewDecision and does not establish that the review blocks on its own.
Actor: patrickg-unity
Scenario-commit: e2bf65c13281de6eebebc44148eccddbfcb4e121
Subject-pin: none
Workflow-run: 35010548980 https://github.com/patrickg-unity/agent-workflow-tests/actions/runs/35010548980
Workflow-conclusion: success
Preconditions:
  branch-protection-main: {"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection","required_pull_request_reviews":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_pull_request_reviews","dismiss_stale_reviews":false,"require_code_owner_reviews":false,"require_last_push_approval":false,"required_approving_review_count":1},"required_signatures":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_signatures","enabled":false},"enforce_admins":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/enforce_admins","enabled":false},"required_linear_history":{"enabled":false},"allow_force_pushes":{"enabled":false},"allow_deletions":{"enabled":false},"block_creations":{"enabled":false},"required_conversation_resolution":{"enabled":false},"lock_branch":{"enabled":false},"allow_fork_syncing":{"enabled":false}}
  codeowners-absent: CODEOWNERS HTTP 404, .github/CODEOWNERS HTTP 404, docs/CODEOWNERS HTTP 404, each read with gh api repos/patrickg-unity/agent-workflow-tests/contents/<path>. The same call on README.md returned its path, so the three 404s are those files' absence rather than a call that cannot read anything.
  actions-can-approve-pull-request-reviews: false
Arms:
  experiment: PR 5, test/app-changes-a-20260915, patrickg-unity
  control-protection: PR 6, test/app-changes-b-20260915, patrickg-unity
  control-identity: the same pull request as experiment, read once and classified twice
Controls:
  control-protection: EXPECT reviews empty, reviewDecision REVIEW_REQUIRED, and mergeStateStatus BLOCKED | OBSERVED reviews empty, reviewDecision REVIEW_REQUIRED, mergeStateStatus BLOCKED | HELD
  control-identity: EXPECT exactly one review authored by workflow-test-agent with state CHANGES_REQUESTED | OBSERVED one review, author workflow-test-agent, state CHANGES_REQUESTED, on commit c8b6fa9b8bbe5ba148192b319ae64a469c8437ab | HELD
Readings:
  experiment, before, gh pr view 5 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
  ```
  {"mergeStateStatus":"BLOCKED","reviewDecision":"REVIEW_REQUIRED","reviews":[]}
  ```
  experiment, after, gh pr view 5 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
  ```
  {"mergeStateStatus":"BLOCKED","reviewDecision":"CHANGES_REQUESTED","reviews":[{"id":"PRR_kwDOUb-xbs8AAAABNs9srQ","author":{"login":"workflow-test-agent"},"authorAssociation":"NONE","body":"App changes-requested test.","submittedAt":"2026-09-15T18:56:21Z","includesCreatedEdit":false,"reactionGroups":[],"state":"CHANGES_REQUESTED","commit":{"oid":"c8b6fa9b8bbe5ba148192b319ae64a469c8437ab"}}]}
  ```
  control-protection, before, gh pr view 6 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
  ```
  {"mergeStateStatus":"BLOCKED","reviewDecision":"REVIEW_REQUIRED","reviews":[]}
  ```
  control-protection, after, gh pr view 6 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
  ```
  {"mergeStateStatus":"BLOCKED","reviewDecision":"REVIEW_REQUIRED","reviews":[]}
  ```
  acting-identity provenance, after, gh run view 35010548980 --repo patrickg-unity/agent-workflow-tests --log | grep -F 'Requesting changes as'
  ```
  measure	Report the identity the review is submitted as	2026-09-15T18:56:20.3135178Z Requesting changes as workflow-test-agent
  ```
Residue: Both pull requests closed unmerged, per the teardown. Both branches test/app-changes-a-20260915 and test/app-changes-b-20260915 left in place, local and remote, which is what the teardown intends. Branch protection on main unchanged. This record was not landed by the run that produced it, because the measuring job holds contents: read.
Amends: none
Notes: Procedure step 10 re-ran P4's check after the measurement and it returned what it returned at step 1, {"admins":false,"approvals":1,"checks":null,"code_owners":false,"last_push":false,"restrictions":null}. The full protection JSON read after the run is byte-identical to the copy under Preconditions above, compared with diff, and a deliberately altered copy of that same file was passed through the same diff first and reported a difference, so the identical result is a comparison that ran rather than one that could not fail.

The instrument's two fields moved apart on the experiment arm and that is the measurement. reviewDecision went from REVIEW_REQUIRED to CHANGES_REQUESTED while mergeStateStatus stayed BLOCKED across both readings. The control arm's reviewDecision stayed REVIEW_REQUIRED. The arms differ in one thing, so the reviewDecision difference is the App's changes-requested review registering, and the unchanged BLOCKED on the experiment arm carries no information: the control shows an unapproved pull request with no review at all already reads BLOCKED under this rule.

What this run does not establish is whether the changes-requested review blocks a pull request that would otherwise be mergeable. The experiment arm holds no approving review, so its BLOCKED is over-determined. The RUNBOOK section What the control rules out, and what this scenario cannot separate states the limit, and row R1's meaning column repeats it, so a later reader cannot take this record as the blocking answer.

An arm that would settle the blocking question was identified while classifying this run and is not part of this scenario. Have the App submit an approving review first, read the pull request, then have the same App submit a changes-requested review on that same pull request and read it again. If the approval is still counted while the changes-requested review is in force, the second reading isolates the block; if the later review supersedes the earlier one, the approval count falls to zero and the reading is over-determined exactly as this run's is. Either outcome is informative and neither needs a second reviewing identity, so the one-identity route is worth measuring before procuring a second App. That is a new scenario with its own id, because its variable is the order of two reviews rather than the event of one.

The review's author.login reads workflow-test-agent with no [bot] suffix. A cross-check through gh api repos/patrickg-unity/agent-workflow-tests/pulls/5/reviews returns the same review's user.login as workflow-test-agent[bot] with user.type Bot. control-identity is declared against the GraphQL form because gh pr view --json is the instrument.

The acting-identity grep in procedure step 6 matches two lines, not one. The second is the workflow step's own echoed command, which carries terminal escape bytes, and it is omitted from the Readings block above so that no control character enters this file. The line recorded is the step's output.

The run carried two annotations, both reading "Input 'app-id' has been deprecated with message: Use 'client-id' instead." That is the deprecation the RUNBOOK's Notes on the workflow file already names, the run succeeded, and nothing was changed in response to it.
