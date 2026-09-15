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

A record's heading timestamp is the run's own `createdAt` from
`gh run list --json createdAt`, truncated to whole seconds. It is that field and not the review's
`submittedAt` because two of this scenario's three phases submit no review, so `submittedAt` has no
value for them and the headings would not be comparable.

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

## 2026-09-15T19:44:21Z NOT-RUN

Outcome-row: RX
Verdict-reason: The dispatched phase was open-pr and the creation step was refused HTTP 403 because P6 was not satisfied, so no review was submitted and no arm produced a reading about the question. RX is selected on the run's job structure before any arm is read, so all three controls are NOT-APPLICABLE.
Actor: patrickg-unity
Scenario-commit: ce13cbedb5a49a34594dee10511c5e9c96dbc298
Subject-pin: none
Workflow-run: 35015406772 https://github.com/patrickg-unity/agent-workflow-tests/actions/runs/35015406772
Workflow-conclusion: failure
Dispatched-phase: open-pr
Dispatched-against: test/satisfied-approval-b-20260915
Preconditions:
  branch-protection-main: {"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection","required_pull_request_reviews":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_pull_request_reviews","dismiss_stale_reviews":false,"require_code_owner_reviews":false,"require_last_push_approval":false,"required_approving_review_count":1},"required_signatures":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_signatures","enabled":false},"enforce_admins":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/enforce_admins","enabled":false},"required_linear_history":{"enabled":false},"allow_force_pushes":{"enabled":false},"allow_deletions":{"enabled":false},"block_creations":{"enabled":false},"required_conversation_resolution":{"enabled":false},"lock_branch":{"enabled":false},"allow_fork_syncing":{"enabled":false}}
  codeowners-absent: CODEOWNERS HTTP 404, .github/CODEOWNERS HTTP 404, docs/CODEOWNERS HTTP 404, each read with gh api repos/patrickg-unity/agent-workflow-tests/contents/<path>. The same call on README.md returned HTTP 200, so the three 404s are those files' absence rather than a call that cannot read anything.
  actions-can-approve-pull-request-reviews: false
Arms:
  experiment-app-approver: PR 9, test/satisfied-approval-a-20260915, patrickg-unity, approver not yet supplied at this run
  experiment-other-approver: not yet opened. This run is the attempt to open it.
  control-protection: PR 10, test/satisfied-approval-c-20260915, patrickg-unity
  control-approval-in-force: the before reading of the pull request this run dispatched against, read once and classified twice
  control-identity: the after reading of the pull request this run dispatched against, read once and classified twice
Controls:
  control-protection: EXPECT reviews empty, latestReviews empty, reviewDecision REVIEW_REQUIRED, and mergeStateStatus BLOCKED | OBSERVED not classified for this run | NOT-APPLICABLE
  control-approval-in-force: EXPECT reviewDecision APPROVED with mergeStateStatus CLEAN and one APPROVED review | OBSERVED this run dispatched against a branch and read no pull request | NOT-APPLICABLE
  control-identity: EXPECT the held set for the dispatched phase | OBSERVED no review was submitted | NOT-APPLICABLE
Readings:
  acting-identity provenance, after, gh run view 35015406772 --repo patrickg-unity/agent-workflow-tests --log | grep -F 'not permitted to create or approve'
  ```
  open-pull-request	Open the pull request as the default token identity	2026-09-15T19:44:31.7930683Z gh: GitHub Actions is not permitted to create or approve pull requests. (HTTP 403)
  ```
Residue: Nothing. No pull request was created, no review was submitted, and no repository setting was changed by this run. Branch test/satisfied-approval-b-20260915 was left as P5 pushed it, with no pull request.
Amends: none
Notes: This run is the evidence for P6 and was dispatched deliberately before P6 was satisfied, so that the prerequisite rests on a measured refusal rather than on a claim about what the setting does. The refusal text names the setting in GitHub's own words and is quoted verbatim above. Preflight passed and open-pull-request failed, which is the correct structure: the prerequisite this run tests is not one preflight asserts, for the reason in RUNBOOK.md, section What preflight asserts, and what it deliberately does not.

The refusal establishes that on this repository, with default_workflow_permissions read and can_approve_pull_request_reviews false, the default GITHUB_TOKEN cannot open a pull request even in a job holding pull-requests: write. The toggle and the job permission are two independent gates and the toggle is the outer one. The 403 came from gh api -X POST repos/OWNER/REPO/pulls, not from gh pr create, so it is the API's refusal and not a CLI precondition.

## 2026-09-15T19:45:05Z VALID

Outcome-row: R2
Verdict-reason: control-protection and control-identity held, control-approval-in-force is NOT-APPLICABLE because this phase creates the state that control asserts, and exactly one row matched. The arm read reviewDecision APPROVED with mergeStateStatus CLEAN, which is the baseline a request-changes dispatch against PR 9 is read against and answers nothing about blocking on its own.
Actor: patrickg-unity
Scenario-commit: ce13cbedb5a49a34594dee10511c5e9c96dbc298
Subject-pin: none
Workflow-run: 35015480708 https://github.com/patrickg-unity/agent-workflow-tests/actions/runs/35015480708
Workflow-conclusion: success
Dispatched-phase: approve
Dispatched-against: 9
Preconditions:
  branch-protection-main: {"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection","required_pull_request_reviews":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_pull_request_reviews","dismiss_stale_reviews":false,"require_code_owner_reviews":false,"require_last_push_approval":false,"required_approving_review_count":1},"required_signatures":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_signatures","enabled":false},"enforce_admins":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/enforce_admins","enabled":false},"required_linear_history":{"enabled":false},"allow_force_pushes":{"enabled":false},"allow_deletions":{"enabled":false},"block_creations":{"enabled":false},"required_conversation_resolution":{"enabled":false},"lock_branch":{"enabled":false},"allow_fork_syncing":{"enabled":false}}
  codeowners-absent: CODEOWNERS HTTP 404, .github/CODEOWNERS HTTP 404, docs/CODEOWNERS HTTP 404, each read with gh api repos/patrickg-unity/agent-workflow-tests/contents/<path>. The same call on README.md returned HTTP 200.
  actions-can-approve-pull-request-reviews: false
Arms:
  experiment-app-approver: PR 9, test/satisfied-approval-a-20260915, patrickg-unity, approver workflow-test-agent
  experiment-other-approver: not yet opened
  control-protection: PR 10, test/satisfied-approval-c-20260915, patrickg-unity
  control-approval-in-force: the before reading of the pull request this run dispatched against, read once and classified twice
  control-identity: the after reading of the pull request this run dispatched against, read once and classified twice
Controls:
  control-protection: EXPECT reviews empty, latestReviews empty, reviewDecision REVIEW_REQUIRED, and mergeStateStatus BLOCKED | OBSERVED reviews empty, latestReviews empty, reviewDecision REVIEW_REQUIRED, mergeStateStatus BLOCKED | HELD
  control-approval-in-force: EXPECT reviewDecision APPROVED with mergeStateStatus CLEAN and one APPROVED review | OBSERVED this phase creates that state rather than reading it | NOT-APPLICABLE
  control-identity: EXPECT after an approve dispatch, reviews holds exactly one entry authored by workflow-test-agent with state APPROVED, and latestReviews holds that same single entry | OBSERVED reviews one entry, author workflow-test-agent, state APPROVED, on commit ddf727d6a5dbf57ec0137b95dc12db78e8565cde; latestReviews one entry, author workflow-test-agent, state APPROVED | HELD
Readings:
  experiment-app-approver, before, gh pr view 9 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
  ```
  {"latestReviews":[],"mergeStateStatus":"BLOCKED","reviewDecision":"REVIEW_REQUIRED","reviews":[]}
  ```
  experiment-app-approver, after, gh pr view 9 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
  ```
  {"latestReviews":[{"id":"","author":{"login":"workflow-test-agent"},"authorAssociation":"NONE","body":"App approval, the baseline this scenario reads the changes-requested review against.","submittedAt":"2026-09-15T19:45:15Z","includesCreatedEdit":false,"reactionGroups":[],"state":"APPROVED","commit":{"oid":""}}],"mergeStateStatus":"CLEAN","reviewDecision":"APPROVED","reviews":[{"id":"PRR_kwDOUb-xbs8AAAABNtbihQ","author":{"login":"workflow-test-agent"},"authorAssociation":"NONE","body":"App approval, the baseline this scenario reads the changes-requested review against.","submittedAt":"2026-09-15T19:45:15Z","includesCreatedEdit":false,"reactionGroups":[],"state":"APPROVED","commit":{"oid":"ddf727d6a5dbf57ec0137b95dc12db78e8565cde"}}]}
  ```
  control-protection, before, gh pr view 10 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
  ```
  {"latestReviews":[],"mergeStateStatus":"BLOCKED","reviewDecision":"REVIEW_REQUIRED","reviews":[]}
  ```
  acting-identity provenance, after, gh run view 35015480708 --repo patrickg-unity/agent-workflow-tests --log | grep -F 'Reviewing as'
  ```
  submit-review	Report the identity the review is submitted as	2026-09-15T19:45:14.4330764Z Reviewing as workflow-test-agent with event approve
  ```
Residue: PR 9 carries one approving review from workflow-test-agent. No repository setting was changed by this run.
Amends: none
Notes: The two instrument fields moved together on this arm. reviewDecision went from REVIEW_REQUIRED to APPROVED and mergeStateStatus from BLOCKED to CLEAN, on a pull request whose only review is the App's. That is row R2 and it is the state the next dispatch against PR 9 is read against.

This reading also reproduces, incidentally, what a scenario already on this surface established about App approvals satisfying the one-approval rule. It is recorded here because this scenario's own arm needs the baseline measured rather than cited, not as a second finding.

latestReviews and reviews agree entry for entry here, because one identity has reviewed once. The two fields are recorded separately anyway, because the run that follows is where they come apart.

## 2026-09-15T19:45:58Z VALID

Outcome-row: R3
Verdict-reason: All three controls held, control-protection, control-approval-in-force, and control-identity, and exactly one row matched. The arm read reviewDecision CHANGES_REQUESTED with mergeStateStatus BLOCKED and no APPROVED entry in latestReviews, so the approval was replaced rather than joined and the block is over-determined. Per row R3 this arm is non-separable and answers nothing about whether the changes-requested review blocks.
Actor: patrickg-unity
Scenario-commit: ce13cbedb5a49a34594dee10511c5e9c96dbc298
Subject-pin: none
Workflow-run: 35015573960 https://github.com/patrickg-unity/agent-workflow-tests/actions/runs/35015573960
Workflow-conclusion: success
Dispatched-phase: request-changes
Dispatched-against: 9
Preconditions:
  branch-protection-main: {"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection","required_pull_request_reviews":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_pull_request_reviews","dismiss_stale_reviews":false,"require_code_owner_reviews":false,"require_last_push_approval":false,"required_approving_review_count":1},"required_signatures":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_signatures","enabled":false},"enforce_admins":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/enforce_admins","enabled":false},"required_linear_history":{"enabled":false},"allow_force_pushes":{"enabled":false},"allow_deletions":{"enabled":false},"block_creations":{"enabled":false},"required_conversation_resolution":{"enabled":false},"lock_branch":{"enabled":false},"allow_fork_syncing":{"enabled":false}}
  codeowners-absent: CODEOWNERS HTTP 404, .github/CODEOWNERS HTTP 404, docs/CODEOWNERS HTTP 404, each read with gh api repos/patrickg-unity/agent-workflow-tests/contents/<path>. The same call on README.md returned HTTP 200.
  actions-can-approve-pull-request-reviews: false
Arms:
  experiment-app-approver: PR 9, test/satisfied-approval-a-20260915, patrickg-unity, approver workflow-test-agent
  experiment-other-approver: not yet opened
  control-protection: PR 10, test/satisfied-approval-c-20260915, patrickg-unity
  control-approval-in-force: the before reading of the pull request this run dispatched against, read once and classified twice
  control-identity: the after reading of the pull request this run dispatched against, read once and classified twice
Controls:
  control-protection: EXPECT reviews empty, latestReviews empty, reviewDecision REVIEW_REQUIRED, and mergeStateStatus BLOCKED | OBSERVED reviews empty, latestReviews empty, reviewDecision REVIEW_REQUIRED, mergeStateStatus BLOCKED | HELD
  control-approval-in-force: EXPECT reviewDecision APPROVED, mergeStateStatus CLEAN, reviews one entry with state APPROVED, latestReviews one entry with state APPROVED | OBSERVED reviewDecision APPROVED, mergeStateStatus CLEAN, reviews one entry state APPROVED author workflow-test-agent, latestReviews one entry state APPROVED author workflow-test-agent | HELD
  control-identity: EXPECT after a request-changes dispatch, reviews holds exactly two entries, the later authored by workflow-test-agent with state CHANGES_REQUESTED and the earlier authored by the approver named in Arms with state APPROVED | OBSERVED reviews two entries, first author workflow-test-agent state APPROVED, second author workflow-test-agent state CHANGES_REQUESTED; latestReviews one entry, author workflow-test-agent, state CHANGES_REQUESTED | HELD
Readings:
  experiment-app-approver, before, gh pr view 9 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
  ```
  {"latestReviews":[{"id":"","author":{"login":"workflow-test-agent"},"authorAssociation":"NONE","body":"App approval, the baseline this scenario reads the changes-requested review against.","submittedAt":"2026-09-15T19:45:15Z","includesCreatedEdit":false,"reactionGroups":[],"state":"APPROVED","commit":{"oid":""}}],"mergeStateStatus":"CLEAN","reviewDecision":"APPROVED","reviews":[{"id":"PRR_kwDOUb-xbs8AAAABNtbihQ","author":{"login":"workflow-test-agent"},"authorAssociation":"NONE","body":"App approval, the baseline this scenario reads the changes-requested review against.","submittedAt":"2026-09-15T19:45:15Z","includesCreatedEdit":false,"reactionGroups":[],"state":"APPROVED","commit":{"oid":"ddf727d6a5dbf57ec0137b95dc12db78e8565cde"}}]}
  ```
  experiment-app-approver, after, gh pr view 9 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
  ```
  {"latestReviews":[{"id":"","author":{"login":"workflow-test-agent"},"authorAssociation":"NONE","body":"App changes-requested test, submitted while the approval requirement is already satisfied.","submittedAt":"2026-09-15T19:46:10Z","includesCreatedEdit":false,"reactionGroups":[],"state":"CHANGES_REQUESTED","commit":{"oid":""}}],"mergeStateStatus":"BLOCKED","reviewDecision":"CHANGES_REQUESTED","reviews":[{"id":"PRR_kwDOUb-xbs8AAAABNtbihQ","author":{"login":"workflow-test-agent"},"authorAssociation":"NONE","body":"App approval, the baseline this scenario reads the changes-requested review against.","submittedAt":"2026-09-15T19:45:15Z","includesCreatedEdit":false,"reactionGroups":[],"state":"APPROVED","commit":{"oid":"ddf727d6a5dbf57ec0137b95dc12db78e8565cde"}},{"id":"PRR_kwDOUb-xbs8AAAABNtcCqA","author":{"login":"workflow-test-agent"},"authorAssociation":"NONE","body":"App changes-requested test, submitted while the approval requirement is already satisfied.","submittedAt":"2026-09-15T19:46:10Z","includesCreatedEdit":false,"reactionGroups":[],"state":"CHANGES_REQUESTED","commit":{"oid":"ddf727d6a5dbf57ec0137b95dc12db78e8565cde"}}]}
  ```
  control-protection, after, gh pr view 10 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
  ```
  {"latestReviews":[],"mergeStateStatus":"BLOCKED","reviewDecision":"REVIEW_REQUIRED","reviews":[]}
  ```
  acting-identity provenance, after, gh run view 35015573960 --repo patrickg-unity/agent-workflow-tests --log | grep -F 'Reviewing as'
  ```
  submit-review	Report the identity the review is submitted as	2026-09-15T19:46:09.1130683Z Reviewing as workflow-test-agent with event request-changes
  ```
Residue: PR 9 carries two reviews from workflow-test-agent, one APPROVED and one CHANGES_REQUESTED. No repository setting was changed by this run.
Amends: none
Notes: This is the arm the RUNBOOK predicted would be non-separable, and the prediction is confirmed by a field rather than by an argument. reviews still lists the App's approval, latestReviews does not. GitHub counts the most recent review from each reviewer, so after this run the pull request has no approval in force, the one-approval requirement is unmet again, and mergeStateStatus BLOCKED is fully explained without reference to the changes-requested review.

Read against the previous record, the pair is the whole point of the arm. PR 9 went APPROVED with CLEAN, then CHANGES_REQUESTED with BLOCKED, and it would be easy to report that transition as the changes-requested review blocking a mergeable pull request. It is not, because the same review that arrived also removed the approval. A scenario reading only reviewDecision and mergeStateStatus would have recorded the wrong answer here with well-formed readings throughout, which is why latestReviews is in the instrument.

The consequence for the question is that one identity cannot supply both the approval and the objection. The blocking question needs two identities, which is what the arm recorded two records below does.

## 2026-09-15T19:49:15Z NOT-RUN

Outcome-row: RX
Verdict-reason: The dispatched phase was open-pr, which submits no review and by design reaches no measurement, so RX is selected on the run's job structure and all three controls are NOT-APPLICABLE. The run created PR 11, which is what the phase exists to do and which the record carries under Residue.
Actor: patrickg-unity
Scenario-commit: ce13cbedb5a49a34594dee10511c5e9c96dbc298
Subject-pin: none
Workflow-run: 35015899952 https://github.com/patrickg-unity/agent-workflow-tests/actions/runs/35015899952
Workflow-conclusion: success
Dispatched-phase: open-pr
Dispatched-against: test/satisfied-approval-b-20260915
Preconditions:
  branch-protection-main: {"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection","required_pull_request_reviews":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_pull_request_reviews","dismiss_stale_reviews":false,"require_code_owner_reviews":false,"require_last_push_approval":false,"required_approving_review_count":1},"required_signatures":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_signatures","enabled":false},"enforce_admins":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/enforce_admins","enabled":false},"required_linear_history":{"enabled":false},"allow_force_pushes":{"enabled":false},"allow_deletions":{"enabled":false},"block_creations":{"enabled":false},"required_conversation_resolution":{"enabled":false},"lock_branch":{"enabled":false},"allow_fork_syncing":{"enabled":false}}
  codeowners-absent: CODEOWNERS HTTP 404, .github/CODEOWNERS HTTP 404, docs/CODEOWNERS HTTP 404, each read with gh api repos/patrickg-unity/agent-workflow-tests/contents/<path>. The same call on README.md returned HTTP 200.
  actions-can-approve-pull-request-reviews: true
Arms:
  experiment-app-approver: PR 9, test/satisfied-approval-a-20260915, patrickg-unity, approver workflow-test-agent
  experiment-other-approver: PR 11, test/satisfied-approval-b-20260915, github-actions[bot], approver not yet supplied at this run
  control-protection: PR 10, test/satisfied-approval-c-20260915, patrickg-unity
  control-approval-in-force: the before reading of the pull request this run dispatched against, read once and classified twice
  control-identity: the after reading of the pull request this run dispatched against, read once and classified twice
Controls:
  control-protection: EXPECT reviews empty, latestReviews empty, reviewDecision REVIEW_REQUIRED, and mergeStateStatus BLOCKED | OBSERVED not classified for this run | NOT-APPLICABLE
  control-approval-in-force: EXPECT reviewDecision APPROVED with mergeStateStatus CLEAN and one APPROVED review | OBSERVED this run dispatched against a branch and read no pull request | NOT-APPLICABLE
  control-identity: EXPECT the held set for the dispatched phase | OBSERVED no review was submitted | NOT-APPLICABLE
Readings:
  experiment-other-approver, before, gh pr view 11 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
  ```
  {"latestReviews":[],"mergeStateStatus":"BLOCKED","reviewDecision":"REVIEW_REQUIRED","reviews":[]}
  ```
  acting-identity provenance, after, gh run view 35015899952 --repo patrickg-unity/agent-workflow-tests --log | grep -E 'Opened as|Opened-pull-request: '
  ```
  open-pull-request	Open the pull request as the default token identity	2026-09-15T19:49:30.7333698Z Opened-pull-request: 11
  open-pull-request	Report the identity the pull request was opened as	2026-09-15T19:49:31.1535770Z Opened as app/github-actions
  open-pull-request	Emit the run-side record fields	2026-09-15T19:49:31.1656226Z Opened-pull-request: 11
  ```
Residue: PR 11 exists, open, authored by github-actions[bot], head test/satisfied-approval-b-20260915, base main, carrying no review. The repository setting P6 turns on was true while this run executed and was restored to false later in the same session, per the teardown.
Amends: none
Notes: This is the same dispatch as the first record in this file, re-run with P6 satisfied, and the pair is the measurement P6 rests on. The same phase against the same branch was refused HTTP 403 before the setting was turned on and succeeded after, with nothing else changed between them.

The reading of PR 11 above is recorded for the run's context and is not interpreted, because RX means no arm produced a reading about the question. It does establish the pull request opened unreviewed, which is what the next dispatch against it assumes.

The identity line reads app/github-actions rather than github-actions[bot]. That is the same identity through a different API surface: gh pr view --json author returns app/github-actions with is_bot true, and gh api repos/patrickg-unity/agent-workflow-tests/pulls/11 returns user.login github-actions[bot] with user.type Bot. Both were read on 2026-09-15 against this pull request. The Arms block above writes the REST spelling, and the next record is where the difference cost a dispatch.

## 2026-09-15T19:50:36Z NOT-RUN

Outcome-row: RX
Verdict-reason: Preflight refused the dispatch at the permitted-author assertion, because that assertion listed github-actions[bot] and the author it read was app/github-actions, so submit-review reported skipped and no review was submitted. RX is selected on the run's job structure and all three controls are NOT-APPLICABLE.
Actor: patrickg-unity
Scenario-commit: ce13cbedb5a49a34594dee10511c5e9c96dbc298
Subject-pin: none
Workflow-run: 35016034878 https://github.com/patrickg-unity/agent-workflow-tests/actions/runs/35016034878
Workflow-conclusion: failure
Dispatched-phase: request-changes
Dispatched-against: 11
Preconditions:
  branch-protection-main: {"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection","required_pull_request_reviews":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_pull_request_reviews","dismiss_stale_reviews":false,"require_code_owner_reviews":false,"require_last_push_approval":false,"required_approving_review_count":1},"required_signatures":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_signatures","enabled":false},"enforce_admins":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/enforce_admins","enabled":false},"required_linear_history":{"enabled":false},"allow_force_pushes":{"enabled":false},"allow_deletions":{"enabled":false},"block_creations":{"enabled":false},"required_conversation_resolution":{"enabled":false},"lock_branch":{"enabled":false},"allow_fork_syncing":{"enabled":false}}
  codeowners-absent: CODEOWNERS HTTP 404, .github/CODEOWNERS HTTP 404, docs/CODEOWNERS HTTP 404, each read with gh api repos/patrickg-unity/agent-workflow-tests/contents/<path>. The same call on README.md returned HTTP 200.
  actions-can-approve-pull-request-reviews: true
Arms:
  experiment-app-approver: PR 9, test/satisfied-approval-a-20260915, patrickg-unity, approver workflow-test-agent
  experiment-other-approver: PR 11, test/satisfied-approval-b-20260915, github-actions[bot], approver not yet supplied at this run
  control-protection: PR 10, test/satisfied-approval-c-20260915, patrickg-unity
  control-approval-in-force: the before reading of the pull request this run dispatched against, read once and classified twice
  control-identity: the after reading of the pull request this run dispatched against, read once and classified twice
Controls:
  control-protection: EXPECT reviews empty, latestReviews empty, reviewDecision REVIEW_REQUIRED, and mergeStateStatus BLOCKED | OBSERVED not classified for this run | NOT-APPLICABLE
  control-approval-in-force: EXPECT reviewDecision APPROVED with mergeStateStatus CLEAN and one APPROVED review | OBSERVED preflight refused before any arm was read | NOT-APPLICABLE
  control-identity: EXPECT the held set for the dispatched phase | OBSERVED no review was submitted | NOT-APPLICABLE
Readings:
  acting-identity provenance, after, gh run view 35016034878 --repo patrickg-unity/agent-workflow-tests --log | grep -F 'Prerequisite P5 not satisfied: pull request 11'
  ```
  preflight	Assert the target pull request is open, authored by a permitted identity, and in the state this phase requires	2026-09-15T19:50:43.8976426Z Prerequisite P5 not satisfied: pull request 11 is authored by app/github-actions, which is neither patrickg-unity nor github-actions[bot], so the App could be reviewing its own pull request. See scenarios/app-changes-vs-satisfied-approval/RUNBOOK.md, section Prerequisites, entry P5.
  ```
Residue: Nothing. PR 11 still carried no review after this run and no repository setting was changed by it.
Amends: none
Notes: This run was dispatched deliberately, to read what the permitted-author assertion does when handed an author spelling it does not know rather than to reason about it. The assertion is an allowlist, so it refused the dispatch and submit-review reported skipped, which is the RX row taken from the run's own job data and not from parsing a log. A denylist in the same position would have passed the unrecognized spelling through to the review step.

The refusal was a false negative and not a real one. PR 11's author is github-actions[bot], which is a permitted author for this scenario, and the assertion carried only the spelling gh api returns while reading the spelling gh pr view returns. The workflow was fixed after this run, in the commit merged as pull request 12, and the assertion now carries both spellings. The run that follows this one is the same dispatch against the same pull request with the fixed assertion, and its Scenario-commit differs from this one's for that reason.

Nothing about the preflight message was wrong except the set it compared against. It named the author it read, quoted the prerequisite by id, and pointed at the section of the RUNBOOK that owns it, which is what made the cause readable without opening the workflow.

## 2026-09-15T19:57:10Z VALID

Outcome-row: R4
Verdict-reason: All three controls held, control-protection, control-approval-in-force, and control-identity, and exactly one row matched. The arm read reviewDecision CHANGES_REQUESTED with mergeStateStatus BLOCKED and an APPROVED entry in latestReviews authored by patrickg-unity, an identity other than workflow-test-agent. Per row R4 the App's changes-requested review blocks a merge whose approval requirement is still satisfied by another reviewer.
Actor: patrickg-unity
Scenario-commit: 5437c4ca7842319f158db8f39148f8c36b3991b2
Subject-pin: none
Workflow-run: 35016627202 https://github.com/patrickg-unity/agent-workflow-tests/actions/runs/35016627202
Workflow-conclusion: success
Dispatched-phase: request-changes
Dispatched-against: 11
Preconditions:
  branch-protection-main: {"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection","required_pull_request_reviews":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_pull_request_reviews","dismiss_stale_reviews":false,"require_code_owner_reviews":false,"require_last_push_approval":false,"required_approving_review_count":1},"required_signatures":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/required_signatures","enabled":false},"enforce_admins":{"url":"https://api.github.com/repos/patrickg-unity/agent-workflow-tests/branches/main/protection/enforce_admins","enabled":false},"required_linear_history":{"enabled":false},"allow_force_pushes":{"enabled":false},"allow_deletions":{"enabled":false},"block_creations":{"enabled":false},"required_conversation_resolution":{"enabled":false},"lock_branch":{"enabled":false},"allow_fork_syncing":{"enabled":false}}
  codeowners-absent: CODEOWNERS HTTP 404, .github/CODEOWNERS HTTP 404, docs/CODEOWNERS HTTP 404, each read with gh api repos/patrickg-unity/agent-workflow-tests/contents/<path>. The same call on README.md returned HTTP 200.
  actions-can-approve-pull-request-reviews: true
Arms:
  experiment-app-approver: PR 9, test/satisfied-approval-a-20260915, patrickg-unity, approver workflow-test-agent
  experiment-other-approver: PR 11, test/satisfied-approval-b-20260915, github-actions[bot], approver patrickg-unity
  control-protection: PR 10, test/satisfied-approval-c-20260915, patrickg-unity
  control-approval-in-force: the before reading of the pull request this run dispatched against, read once and classified twice
  control-identity: the after reading of the pull request this run dispatched against, read once and classified twice
Controls:
  control-protection: EXPECT reviews empty, latestReviews empty, reviewDecision REVIEW_REQUIRED, and mergeStateStatus BLOCKED | OBSERVED reviews empty, latestReviews empty, reviewDecision REVIEW_REQUIRED, mergeStateStatus BLOCKED | HELD
  control-approval-in-force: EXPECT reviewDecision APPROVED, mergeStateStatus CLEAN, reviews one entry with state APPROVED, latestReviews one entry with state APPROVED | OBSERVED reviewDecision APPROVED, mergeStateStatus CLEAN, reviews one entry state APPROVED author patrickg-unity authorAssociation OWNER, latestReviews one entry state APPROVED author patrickg-unity | HELD
  control-identity: EXPECT after a request-changes dispatch, reviews holds exactly two entries, the later authored by workflow-test-agent with state CHANGES_REQUESTED and the earlier authored by the approver named in Arms with state APPROVED | OBSERVED reviews two entries, first author patrickg-unity state APPROVED, second author workflow-test-agent state CHANGES_REQUESTED; latestReviews two entries, workflow-test-agent CHANGES_REQUESTED and patrickg-unity APPROVED | HELD
Readings:
  experiment-other-approver, before, gh pr view 11 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
  ```
  {"latestReviews":[{"id":"","author":{"login":"patrickg-unity"},"authorAssociation":"OWNER","body":"Human approval, arm B baseline.","submittedAt":"2026-09-15T19:56:30Z","includesCreatedEdit":false,"reactionGroups":[],"state":"APPROVED","commit":{"oid":""}}],"mergeStateStatus":"CLEAN","reviewDecision":"APPROVED","reviews":[{"id":"PRR_kwDOUb-xbs8AAAABNthmMg","author":{"login":"patrickg-unity"},"authorAssociation":"OWNER","body":"Human approval, arm B baseline.","submittedAt":"2026-09-15T19:56:30Z","includesCreatedEdit":false,"reactionGroups":[],"state":"APPROVED","commit":{"oid":"f9ebc1679f4b8010028fc1f69c36b842a207a949"}}]}
  ```
  experiment-other-approver, after, gh pr view 11 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
  ```
  {"latestReviews":[{"id":"","author":{"login":"workflow-test-agent"},"authorAssociation":"NONE","body":"App changes-requested test, submitted while the approval requirement is already satisfied.","submittedAt":"2026-09-15T19:57:25Z","includesCreatedEdit":false,"reactionGroups":[],"state":"CHANGES_REQUESTED","commit":{"oid":""}},{"id":"","author":{"login":"patrickg-unity"},"authorAssociation":"OWNER","body":"Human approval, arm B baseline.","submittedAt":"2026-09-15T19:56:30Z","includesCreatedEdit":false,"reactionGroups":[],"state":"APPROVED","commit":{"oid":""}}],"mergeStateStatus":"BLOCKED","reviewDecision":"CHANGES_REQUESTED","reviews":[{"id":"PRR_kwDOUb-xbs8AAAABNthmMg","author":{"login":"patrickg-unity"},"authorAssociation":"OWNER","body":"Human approval, arm B baseline.","submittedAt":"2026-09-15T19:56:30Z","includesCreatedEdit":false,"reactionGroups":[],"state":"APPROVED","commit":{"oid":"f9ebc1679f4b8010028fc1f69c36b842a207a949"}},{"id":"PRR_kwDOUb-xbs8AAAABNtiDnQ","author":{"login":"workflow-test-agent"},"authorAssociation":"NONE","body":"App changes-requested test, submitted while the approval requirement is already satisfied.","submittedAt":"2026-09-15T19:57:25Z","includesCreatedEdit":false,"reactionGroups":[],"state":"CHANGES_REQUESTED","commit":{"oid":"f9ebc1679f4b8010028fc1f69c36b842a207a949"}}]}
  ```
  control-protection, after, gh pr view 10 --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
  ```
  {"latestReviews":[],"mergeStateStatus":"BLOCKED","reviewDecision":"REVIEW_REQUIRED","reviews":[]}
  ```
  acting-identity provenance, after, gh run view 35016627202 --repo patrickg-unity/agent-workflow-tests --log | grep -F 'Reviewing as'
  ```
  submit-review	Report the identity the review is submitted as	2026-09-15T19:57:24.4982385Z Reviewing as workflow-test-agent with event request-changes
  ```
Residue: PR 11 carries one APPROVED review from patrickg-unity and one CHANGES_REQUESTED review from workflow-test-agent. The repository setting P6 turns on was true while this run executed and was restored to false afterwards, per the teardown.
Amends: none
Notes: This is the record that answers the question, and it answers it because of the reading taken before the dispatch rather than the one taken after. PR 11 read reviewDecision APPROVED with mergeStateStatus CLEAN while its only review was a human approval, so at that moment the one-approval requirement was satisfied and nothing else blocked the merge. The single thing that changed afterwards is the App's changes-requested review, and the pull request read BLOCKED.

latestReviews holds two entries after the dispatch, workflow-test-agent CHANGES_REQUESTED and patrickg-unity APPROVED. The approval is therefore still in force, which is exactly what distinguishes this arm from the one recorded three records above, where the App's own approval was replaced by its own objection and BLOCKED became over-determined again. An App installation token's changes-requested review overrides an already-satisfied approval requirement on this repository, under a rule requiring one approving review with code-owner review off and last-push approval off.

What this record does not establish is which mechanism GitHub applies, a changes-requested review that suppresses merge on its own or one that withdraws an approval it did not cast. Both present as BLOCKED with the approval surviving in latestReviews, and the instrument has no field that tells them apart. The RUNBOOK section Why this scenario exists and what makes it separable states that limit.

The approving review this arm rests on was submitted by patrickg-unity in their own terminal, with gh pr review 11 --repo patrickg-unity/agent-workflow-tests --approve --body "Human approval, arm B baseline.", and reads authorAssociation OWNER. No bot was substituted into the approver slot. The author slot holds github-actions[bot] because the approver cannot also be the author, and the RUNBOOK section Why P6 exists, and why the three identities cannot be reduced to two is the reasoning for that assignment.

Two readings returned mergeStateStatus UNKNOWN and were re-read under the RUNBOOK's UNKNOWN procedure, which is the first time that bound has been exercised on this surface. PR 11's before reading returned UNKNOWN on the first attempt and CLEAN on the second. PR 10's after reading returned UNKNOWN on the first attempt, and BLOCKED on the next, which is the reading recorded above. Both resolved inside the first of the four further attempts the procedure allows, so the bound of five readings across roughly a minute is not contradicted by these runs and is not revised.

Procedure step 15 ran after the measurement. P4's check returned what it returned at step 1, {"admins":false,"approvals":1,"checks":null,"code_owners":false,"last_push":false,"restrictions":null}, and the full protection JSON read afterwards is byte-identical to the copy under Preconditions above, compared with diff. A deliberately altered copy of that same file, with required_approving_review_count set to 2, was passed through the same diff and reported a difference, so the identical result is a comparison that ran rather than one that could not fail. The three CODEOWNERS paths returned HTTP 404 again and the same call on README.md returned HTTP 200. P6 was restored and gh api repos/patrickg-unity/agent-workflow-tests/actions/permissions/workflow returned {"default_workflow_permissions":"read","can_approve_pull_request_reviews":false}.

P2 has no command form and its check is a settings page, which was not opened during this session. It is carried by the token mint succeeding in this run and in the two before it, which an uninstalled App cannot do. That is weaker than the page and is recorded as such rather than as a satisfied check.
