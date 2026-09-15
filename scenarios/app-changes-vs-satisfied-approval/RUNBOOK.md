# app-changes-vs-satisfied-approval

## Question

Does a pull request review submitted by a GitHub App installation token with the `REQUEST_CHANGES`
event block a merge to `main` when the one-approving-review requirement on `main` is already
satisfied at the moment the App submits it, on github.com, today?

The variable under test is the identity that supplied the approving review the App's
changes-requested review is added beside, either the same App or a different identity. Everything
else is held fixed: the same repository, the same protection settings, three pull requests against
the same base branch, and one read command applied to all of them.

## Subject

GitHub itself. There is no external artifact under test, so every record carries `Subject-pin: none`.

## Why this scenario exists and what makes it separable

Read this before the outcome table. It is the reason the table's rows say what they say.

A pull request under a rule requiring one approving review and carrying only a changes-requested
review is blocked twice over. The changes-requested review is one candidate cause and the unmet
approval requirement is the other, and a `mergeStateStatus: BLOCKED` reading taken in that state
attributes nothing, because the unmet requirement alone is already sufficient to produce it. No
control arm repairs that: a control can show the unmet requirement is sufficient, which is exactly
what makes the experiment reading uninformative rather than what rescues it.

The fix is in the order of operations rather than in the arms. Satisfy the approval requirement
first, read the pull request and confirm it reads `APPROVED` with `CLEAN`, and only then have the
App submit its changes-requested review. A `BLOCKED` reading taken after that point has one
remaining candidate cause, because the reading immediately before it showed that nothing else
blocked. That before reading is the load-bearing part of this scenario and not a formality: without
it there is no evidence the requirement was ever satisfied, and the arm collapses back into the
overdetermined state described above.

The standing form of that requirement is in [`../README.md`](../README.md) under `## Standing rule:
a scenario asking whether something blocks satisfies the approval requirement first`, because it
binds every future scenario on this surface and not only this one.

**What one identity cannot do.** GitHub counts the most recent review from each reviewer, so an
approval and a changes-requested review from the same identity are not two reviews in force at once.
The later one replaces the earlier one in that count. The arm in which the App supplies both is
therefore expected to fall back into the overdetermined state, and `latestReviews` is in the
instrument so that this is read rather than argued: an arm whose `latestReviews` holds no `APPROVED`
entry no longer satisfies the requirement, whatever its `reviews` field still lists. That arm
reports non-separable and reports no answer to the question.

**What no arm here can separate.** Which of two blocking mechanisms GitHub applies, a
changes-requested review that suppresses merge on its own, or one that withdraws an approval it did
not cast. Both present as `BLOCKED` with an `APPROVED` entry surviving in `latestReviews`, and the
instrument has no field that tells them apart. This scenario answers whether the merge is blocked
and not how.

## Prerequisites

Seven entries. Each is done before the workflow is dispatched, and each `check` is read-only.

P1
what: A GitHub App owned by `patrickg-unity` exists, granting it `Pull requests: Read and write` and nothing else, with its slug exactly `workflow-test-agent`, because every `control-identity` verdict asserts that the changes-requested review carries that exact slug as its author.
who: operator
where: https://github.com/settings/apps/workflow-test-agent, or https://github.com/settings/apps/new where it does not exist yet
produces: an App with a numeric app id, a slug, and one downloadable private key
lifetime: one-time
check: `curl -s -o /dev/null -w '%{http_code}\n' https://github.com/apps/workflow-test-agent` returns `200` or `301`. It proves existence only, for the reason below.

P2
what: That App is installed on `patrickg-unity/agent-workflow-tests`, and on no other repository.
who: operator
where: https://github.com/settings/apps/workflow-test-agent/installations
produces: an installation of the App on this repository
lifetime: one-time
check: open https://github.com/patrickg-unity/agent-workflow-tests/settings/installations and confirm the App is listed. This is the one entry here whose check is a page rather than a command, for the reason below.

P3
what: Repository secrets `TEST_APP_ID` and `TEST_APP_PRIVATE_KEY` are set, holding that App's credentials.
who: repository admin
where: https://github.com/patrickg-unity/agent-workflow-tests/settings/secrets/actions
produces: two named repository secrets
lifetime: one-time, until rotation
check: `gh api repos/patrickg-unity/agent-workflow-tests/actions/secrets --jq '.secrets[].name'` lists both names. Secret values are never readable through the API and are never read here.

P4
what: Branch protection on `main` is in force requiring one approving review, with code-owner review off, last-push approval off, no required status checks, no push restrictions, and no CODEOWNERS file in the repository. This scenario does not establish that state and does not change it; it is part of the repository baseline. The absent CODEOWNERS file is part of this entry rather than a separate one, because `require_code_owner_reviews: false` and an absent CODEOWNERS file are two independent ways the code-owner path stays out of this measurement.
who: repository admin
where: no action, where the check already passes. Where it does not, the repository baseline has drifted and is restored before this scenario is dispatched.
produces: an active protection rule on `main`
lifetime: standing
check: `gh api repos/patrickg-unity/agent-workflow-tests/branches/main/protection --jq '{approvals: .required_pull_request_reviews.required_approving_review_count, code_owners: .required_pull_request_reviews.require_code_owner_reviews, last_push: .required_pull_request_reviews.require_last_push_approval, checks: .required_status_checks, restrictions: .restrictions, admins: .enforce_admins.enabled}'` returns `{"admins":false,"approvals":1,"checks":null,"code_owners":false,"last_push":false,"restrictions":null}`, and `gh api repos/patrickg-unity/agent-workflow-tests/contents/CODEOWNERS`, the same call on `contents/.github/CODEOWNERS`, and the same call on `contents/docs/CODEOWNERS` each return HTTP 404.

P5
what: Open PR A on branch `test/satisfied-approval-a-<RUN>` and PR C on branch `test/satisfied-approval-c-<RUN>`, each a different one-line edit, both authored by `patrickg-unity`, both targeting `main`, neither reviewed by anyone. Push branch `test/satisfied-approval-b-<RUN>` and open no pull request for it, because PR B is opened by the `open-pr` phase of this scenario's own workflow and its author has to be an identity that is neither the App nor the approver. Cut all three branches from the current tip of `main`, so that none reads `mergeStateStatus: BEHIND`, which no substantive row covers. All three branch names carry a per-run discriminator `<RUN>`, so a branch left behind by an earlier run can never block a later one and deleting it is optional hygiene rather than a prerequisite.
who: operator
where: the commands in the block below, with `<clone>` the local checkout path.
produces: two open, unreviewed pull requests and one pushed branch with no pull request
lifetime: per-run
check: `gh pr view <N> --repo patrickg-unity/agent-workflow-tests --json state,author,reviews` returns `OPEN`, author `patrickg-unity`, and an empty `reviews` array, for PR A and PR C. `gh api repos/patrickg-unity/agent-workflow-tests/git/ref/heads/test/satisfied-approval-b-<RUN> --jq .object.sha` returns a sha, and `gh pr list --repo patrickg-unity/agent-workflow-tests --head test/satisfied-approval-b-<RUN> --state all --json number` returns an empty array.

```
git -C <clone> fetch origin main
git -C <clone> switch --no-track -c test/satisfied-approval-a-<RUN> origin/main
printf '%s\n' "Arm A marker." >> <clone>/README.md
git -C <clone> commit -am "test: arm A marker for app-changes-vs-satisfied-approval"
git -C <clone> push origin test/satisfied-approval-a-<RUN>
gh pr create --repo patrickg-unity/agent-workflow-tests --base main --head test/satisfied-approval-a-<RUN> --title "satisfied-approval arm A" --body "Experiment arm. The App approves, then the same App requests changes."
git -C <clone> switch --no-track -c test/satisfied-approval-b-<RUN> origin/main
printf '%s\n' "Arm B marker." >> <clone>/README.md
git -C <clone> commit -am "test: arm B marker for app-changes-vs-satisfied-approval"
git -C <clone> push origin test/satisfied-approval-b-<RUN>
git -C <clone> switch --no-track -c test/satisfied-approval-c-<RUN> origin/main
printf '%s\n' "Arm C marker." >> <clone>/README.md
git -C <clone> commit -am "test: arm C marker for app-changes-vs-satisfied-approval"
git -C <clone> push origin test/satisfied-approval-c-<RUN>
gh pr create --repo patrickg-unity/agent-workflow-tests --base main --head test/satisfied-approval-c-<RUN> --title "satisfied-approval arm C" --body "Control arm. Receives no review."
```

P6
what: The repository setting "Allow GitHub Actions to create and approve pull requests" is on, so that the `open-pr` phase can open PR B as `github-actions[bot]`. This scenario changes this setting and restores it, and `## Teardown` is where the restore is stated.
who: repository admin
where: `gh api -X PUT repos/patrickg-unity/agent-workflow-tests/actions/permissions/workflow -f default_workflow_permissions=read -F can_approve_pull_request_reviews=true`
produces: `can_approve_pull_request_reviews: true` on this repository
lifetime: per-run, restored to `false` by the teardown
check: `gh api repos/patrickg-unity/agent-workflow-tests/actions/permissions/workflow --jq .can_approve_pull_request_reviews` returns `true`. This check reads the setting and does not perform the creation the setting enables, so it does not perform an operation the outcome table discriminates on.

P7
what: The pull request named by a `request-changes` dispatch already carries exactly one review, and that review is an approval, so that the approval requirement on `main` is satisfied at the moment the App submits its changes-requested review. For PR A the approval comes from the `approve` phase of this workflow. For PR B it comes from `patrickg-unity`, who is not its author and can therefore review it, with `gh pr review <PR B> --repo patrickg-unity/agent-workflow-tests --approve --body "Human approval, arm B baseline."`.
who: the `approve` dispatch, for PR A. operator, for PR B.
where: procedure step 6 for PR A, and the `gh pr review` command above for PR B
produces: one approving review on the pull request that is about to receive the changes-requested review
lifetime: per-run, per pull request
check: `gh pr view <N> --repo patrickg-unity/agent-workflow-tests --json reviewDecision,reviews --jq '{decision: .reviewDecision, count: (.reviews | length), states: [.reviews[].state]}'` returns `{"count":1,"decision":"APPROVED","states":["APPROVED"]}`. The workflow's `preflight` job asserts the same thing, for the reason below.

### P1 proves existence only

The App slug namespace is global and the page is anonymous, so a `200` proves that an App with that
slug exists somewhere. It does not prove the App belongs to `patrickg-unity`, and it does not prove
the App grants `Pull requests: Read and write`, which is what P1's `what:` requires. Confirm both by
eye on https://github.com/settings/apps/workflow-test-agent, the same way P2 is confirmed. An App
listed on the marketplace redirects rather than returning 200, so the check accepts `301` too.

The API route that would read the owner is not open here. `gh api /apps/workflow-test-agent` returns
HTTP 404 with an ordinary user token, because that endpoint needs an App JWT for an App that is not
publicly listed.

### P2 has no command form, and it is the expensive one to miss

Two API routes that would check the installation need an App JWT or an App-authorized token and do
not work with an ordinary user token. `gh api repos/patrickg-unity/agent-workflow-tests/installation`
returns HTTP 401 and `gh api /user/installations` returns HTTP 403.

P2 is also the prerequisite whose absence costs the most. An App that exists with both secrets set
but is not installed on this repository mints no token, so the run dies at the token step with a
message about the installation and nothing about reviews. That is an `RX` run and it costs a whole
dispatch to discover.

### Why P6 exists, and why the three identities cannot be reduced to two

GitHub refuses a review on the reviewer's own pull request, for `APPROVE` and `REQUEST_CHANGES`
alike. The arm in which a different identity supplies the approval therefore needs three distinct
identities: an author, an approver, and the App. This repository has one human and one App, which is
two, so the third identity has to be `github-actions[bot]`, the identity the default `GITHUB_TOKEN`
acts as.

That identity cannot open a pull request while `can_approve_pull_request_reviews` is `false`, which
is this repository's standing state and is what the two other scenarios on this surface record as
part of their apparatus. The setting is one toggle covering creation and approval together, labelled
"Allow GitHub Actions to create and approve pull requests" in the settings UI and exposed as the
single field `can_approve_pull_request_reviews` on `actions/permissions/workflow`. P6 turns it on
for the run and the teardown turns it off, and every record carries the value that was in force when
its own measurement was taken rather than the standing one.

Putting `github-actions[bot]` in the author slot rather than the approver slot is deliberate. The
approval is then a human's, submitted by `patrickg-unity` against a pull request that identity did
not open, which is the strongest form of "a different reviewer already satisfied the requirement"
available here. The alternative, `patrickg-unity` authoring and `github-actions[bot]` approving,
needs the same setting and yields a weaker baseline: the approval would come from another bot
identity, and whether a bot approval satisfies the rule would become a second unmeasured assumption
inside the arm that exists to remove one.

### What preflight asserts, and what it deliberately does not

The workflow's `preflight` job re-asserts P3 in full, the input shape for the dispatched phase, and
the part of P5 and P7 that concerns the one pull request or branch the dispatch names. For an
`approve` dispatch it asserts the pull request is `OPEN`, that its author is an identity permitted to
author an arm here, and that it carries no review yet. For a `request-changes` dispatch it asserts
the same two things about the author and the state, and asserts P7: exactly one review, that review
`APPROVED`, and `reviewDecision` reading `APPROVED`.

That last assertion is this scenario's premise turned into a gate. A `request-changes` dispatch
against a pull request whose approval requirement is not satisfied produces exactly the
uninterpretable reading described at the top of this file, and it produces it while looking
well-formed. Asserting it in preflight means that failure reports as `RX` and never as a substantive
row, and it means the assertion cannot be skipped by an operator who read the runbook quickly. The
cost is that an unsatisfied approval requirement can no longer present as `RC`. It presents as `RX`,
and the `RX` row says so.

It does not assert P4, for two reasons. `control-protection` is what proves protection at
measurement time, and a preflight that proved it as well would make a setup failure and a control
failure indistinguishable. The default `GITHUB_TOKEN` also has no route to it: the `permissions` key
a workflow can set has no `administration` entry, so no job can grant itself the repository
administration access the branch-protection read needs.

It does not assert P6 either. The `open-pr` phase's own outcome is the evidence for P6. The pull
request is created, or the creation is refused and GitHub's error text names the setting. A preflight
assertion would replace that measured refusal with a generic setup failure.

## Credentials

name: TEST_APP_ID
scope: repository
shape: the numeric app id from the App settings page, decimal digits only. The slug is not a valid value for this secret.
obtained-from: https://github.com/settings/apps/workflow-test-agent, the App ID field
least-privilege: not a credential on its own and useless without the private key
rotate-when: it is an identifier and does not rotate. It changes only if the App is recreated.
existence-check: `gh api repos/patrickg-unity/agent-workflow-tests/actions/secrets --jq '.secrets[].name'` lists `TEST_APP_ID`

name: TEST_APP_PRIVATE_KEY
scope: repository
shape: a PKCS8 PEM block, multi-line, pasted whole including its header and footer lines
obtained-from: the one-time download from https://github.com/settings/apps/workflow-test-agent under Private keys
least-privilege: `Pull requests: Read and write` on an installation covering `agent-workflow-tests` and no other repository. Both review events this scenario submits need that one grant.
rotate-when: immediately, if this value is found in any workflow log or any commit. Also on any change in who holds it.
existence-check: `gh api repos/patrickg-unity/agent-workflow-tests/actions/secrets --jq '.secrets[].name'` lists `TEST_APP_PRIVATE_KEY`

## Instrument

```
gh pr view <N> --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
```

Every arm is read with this command and no other. Four fields, read and reported separately:

- `reviewDecision`, one of `CHANGES_REQUESTED`, `APPROVED`, `REVIEW_REQUIRED`, or `null`. The three
  named values are the complete `PullRequestReviewDecision` enum, read from the GitHub GraphQL
  schema on 2026-09-15 by introspecting `__type(name: "PullRequestReviewDecision")`. The field is
  nullable. This runbook makes no claim about which configuration produces `null`; a `null` reading
  is handled by the control sets and by `RI`, and is never interpreted.
- `mergeStateStatus`, one of `DIRTY`, `UNKNOWN`, `BLOCKED`, `BEHIND`, `UNSTABLE`, `HAS_HOOKS`, or
  `CLEAN`. That is the complete `MergeStateStatus` enum, read from the same schema in the same call.
  The field is not nullable.
- `reviews`, every review on the pull request in submission order. A review state is one of
  `PENDING`, `COMMENTED`, `APPROVED`, `CHANGES_REQUESTED`, or `DISMISSED`, the complete
  `PullRequestReviewState` enum from the same schema.
- `latestReviews`, the most recent review from each reviewing identity, one entry per identity, with
  the same per-entry shape and the same state enum.

`latestReviews` is in the instrument because it is the field that says which reviews are still in
force, and because the two experiment arms differ in exactly the way that field exposes. An identity
that approves and then requests changes appears twice in `reviews` and once in `latestReviews`, so an
arm whose `latestReviews` holds no `APPROVED` entry has no approval in force regardless of what
`reviews` still lists. Reading only `reviews` would show an approval that no longer counts and would
read as a satisfied requirement.

Two measured details about `latestReviews` through this instrument, taken on 2026-09-15 against pull
request 5 of this repository, which carries one App review. Its entries carry an empty string in
`id` and an empty string in `commit.oid`, where the same review read through `reviews` carries both.
Identity and state are therefore read from `author.login` and `state`, which are populated in both
fields, and no control here rests on an id or a commit oid taken from `latestReviews`.

The author of an App's review reads differently through the two APIs, and this instrument returns
the shorter form. Measured on 2026-09-15 against pull request 5 of this repository. This instrument
returned `author.login` as `workflow-test-agent`, while
`gh api repos/patrickg-unity/agent-workflow-tests/pulls/5/reviews` returned `user.login` as
`workflow-test-agent[bot]` with `user.type` as `Bot`. Every `control-identity` verdict therefore
expects `workflow-test-agent`, and a cross-check run through the REST endpoint will show
`workflow-test-agent[bot]` without anything being wrong.

**`mergeStateStatus: BLOCKED` is evidence about the changes-requested review only where the reading
taken immediately before it was `APPROVED` with `CLEAN` on the same pull request.** That before
reading is what removes the unmet approval requirement as a candidate cause, and nothing else in the
record does it. A `BLOCKED` reading presented without its paired before reading is the single most
likely way this scenario gets misread, and `## Why this scenario exists and what makes it separable`
is the long form.

One arm's `reviewDecision` and `mergeStateStatus` reading is one of 28 pairs. The substantive outcome
rows are stated against that space together with the presence or absence of an `APPROVED` entry in
`latestReviews`, and the `control-identity` sets are stated against `reviews` and `latestReviews`
instead.

### A reading of `UNKNOWN`

`mergeStateStatus: UNKNOWN` means the state cannot currently be determined. GitHub computes
mergeability asynchronously, so a pull request read soon after it is opened or soon after a review
lands legitimately returns `UNKNOWN`. It is a timing artifact and not a result.

Re-read instead of recording it. Wait 15 seconds and run the same command again, up to 4 further
attempts, so at most 5 readings across roughly a minute. Take the first reading that is not
`UNKNOWN`. If the fifth reading is still `UNKNOWN`, record that reading verbatim and classify the
run `RI`.

Never classify a bounded-out `UNKNOWN` as `RC`. `UNKNOWN` is not evidence that protection is off, and
`RC` sends the next person to fix a configuration that was never broken.

The wait and the attempt count are carried over from the two scenarios already on this surface and
are a bound chosen against the known behavior rather than against measured data. Revise them once
this scenario has runs that exercise them, and say so in the record that motivates the change.

## Arms

| Arm | Pull request | Branch | Author | Approver | What it receives |
|---|---|---|---|---|---|
| `experiment-app-approver` | PR A | `test/satisfied-approval-a-<RUN>` | `patrickg-unity` | `workflow-test-agent` | one App approval, then one changes-requested review from that same App |
| `experiment-other-approver` | PR B | `test/satisfied-approval-b-<RUN>` | `github-actions[bot]` | `patrickg-unity` | one human approval, then one changes-requested review from the App |
| `control-protection` | PR C | `test/satisfied-approval-c-<RUN>` | `patrickg-unity` | none | nothing |
| `control-approval-in-force` | PR A or PR B | the arm's own branch | as above | as above | the same before reading as its experiment arm |
| `control-identity` | PR A or PR B | the arm's own branch | as above | as above | the same after reading as its experiment arm |

`control-approval-in-force` and `control-identity` are not separate subjects. Each experiment arm's
before reading and after reading are taken once and classified twice, once for the experiment and
once for the control that reads the same output for a different property. Running the instrument a
second time against the same pull request at the same moment would be a defect, because the two arms
could then read a pull request that changed between the calls.

`control-protection` is authored by `patrickg-unity` rather than by `github-actions[bot]`, so its
author does not match the arm it is compared against. Branch protection applies to the base branch
and not to the author, and `require_last_push_approval` is `false` under P4, so there is no path by
which the author of an unreviewed pull request changes what protection does to it. Opening it as the
operator costs no dispatch and leaves the `open-pr` phase with one job.

The two experiment arms are read as two independent before-and-after pairs and not against each
other. They differ in the approver's identity, which is the variable, and also in the author's
identity, which is forced: `patrickg-unity` supplies the approval on PR B and so cannot have opened
it. Nothing in the outcome table compares a reading on PR A against a reading on PR B, so that
second difference costs the scenario nothing.

## Control arms

```
control-id: control-protection
asserts: branch protection on main is active and requires one approving review, so a pull request with no review cannot merge, and the reading an unreviewed pull request produces under that rule is established by this arm rather than assumed.
instrument: gh pr view <PR C> --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
value-space: the whole instrument output. reviewDecision is CHANGES_REQUESTED, APPROVED, REVIEW_REQUIRED, or null, mergeStateStatus is DIRTY, UNKNOWN, BLOCKED, BEHIND, UNSTABLE, HAS_HOOKS, or CLEAN, which is 28 pairs, and reviews and latestReviews are each a list of any length from zero upward.
held: reviews empty, latestReviews empty, reviewDecision REVIEW_REQUIRED, and mergeStateStatus BLOCKED.
refuted: reviews non-empty under any pair, or reviews empty with mergeStateStatus CLEAN, HAS_HOOKS, or UNSTABLE under any reviewDecision, or reviews empty with reviewDecision null, APPROVED, or CHANGES_REQUESTED under any mergeStateStatus other than UNKNOWN.
indeterminate: every reading in value-space not listed in held or refuted, which is an empty reviews list paired with mergeStateStatus DIRTY, BEHIND, or UNKNOWN, and so includes reviewDecision null with mergeStateStatus UNKNOWN, so that a mergeability calculation still in flight is re-read rather than treated as protection being off.
would-fail-if: branch protection on main is deleted, or its required approving review count is set to 0, or the rule is written against a branch other than main, in which case PR C reads mergeStateStatus CLEAN. This arm is the one that stops a CLEAN after reading on an experiment arm being reported as the App having no veto when in fact there was no rule to veto. Or any review at all lands on PR C, from a human or anything else, at any point up to the last after reading of the run, in which case this arm is no longer the zero-review baseline.
```

```
control-id: control-approval-in-force
asserts: the approval requirement on main was satisfied and nothing else blocked the merge at the moment immediately before the App submitted its changes-requested review, so a BLOCKED reading afterwards has the changes-requested review as its only remaining candidate cause.
instrument: gh pr view <the experiment arm's pull request> --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
value-space: the before reading of the experiment arm, which is one of the same 28 reviewDecision and mergeStateStatus pairs, with reviews and latestReviews each a list of any length from zero upward.
held: reviewDecision APPROVED, mergeStateStatus CLEAN, reviews holding exactly one entry whose state is APPROVED, and latestReviews holding exactly one entry whose state is APPROVED.
refuted: mergeStateStatus BLOCKED, DIRTY, BEHIND, UNSTABLE, or HAS_HOOKS under any reviewDecision, or reviewDecision REVIEW_REQUIRED, CHANGES_REQUESTED, or null under any mergeStateStatus, or reviews holding any number of entries other than one, or the single entry's state being a readable value other than APPROVED.
indeterminate: every reading in value-space not listed in held or refuted, which is reviewDecision APPROVED with mergeStateStatus UNKNOWN and exactly one APPROVED review, so that a mergeability calculation still in flight is re-read rather than recorded as a satisfied or an unsatisfied baseline.
would-fail-if: the approving review is dismissed, or never landed, or landed on a different pull request, or a second review lands before the before reading is taken, in which case this arm reads refuted and the run's after reading is not interpreted. A reading of mergeStateStatus BLOCKED here is the specific failure this arm exists to catch, because it means something other than the changes-requested review was already blocking and the whole point of the ordering is gone.
```

```
control-id: control-identity
asserts: the App under test submitted the changes-requested review on the experiment arm, the approving review the arm was supposed to carry is the one that is there, and no third review joined them, so the after reading attributes the result to an App changes-requested review added beside that approval and to nothing else.
instrument: gh pr view <the experiment arm's pull request> --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews,latestReviews
value-space: the reviews and latestReviews arrays of the experiment arm's after reading, each of any length from zero upward, each entry carrying an author login that may also be unreadable and a state that is PENDING, COMMENTED, APPROVED, CHANGES_REQUESTED, or DISMISSED and may also be unreadable.
held: after an approve dispatch, reviews holds exactly one entry, authored by workflow-test-agent with no [bot] suffix for the reason under the instrument above, with state APPROVED, and latestReviews holds that same single entry. After a request-changes dispatch, reviews holds exactly two entries, the later of which is authored by workflow-test-agent with state CHANGES_REQUESTED, and the earlier of which is authored by the approver named in the record's Arms block with state APPROVED.
refuted: reviews holds any number of entries other than the number the held set names for the dispatched phase, or an entry's readable author is an identity other than the one the held set names for its position, or an entry's readable state is a value other than the one the held set names for its position.
indeterminate: every reading in value-space not listed in held or refuted, which is an entry whose author or whose state cannot be read, as when the reviewing account has been deleted. The review job never running is not classified here, because RX is selected before any arm is read.
would-fail-if: TEST_APP_ID or TEST_APP_PRIVATE_KEY belongs to a different App, in which case the changes-requested review carries that App's login. Or the workflow submits the wrong review event for the dispatched phase, in which case the entry's state is not the one the held set names and the run is thrown away rather than recorded as a finding. Or any third review lands on the experiment arm, from a human or anything else, at any point including after preflight passed and before the after reading, in which case the array holds three entries and the App's review can no longer be isolated from the other one.
```

## Procedure

The `open-pr` and `approve` dispatches are setup for the `request-changes` dispatches, and each of
them is still a run of this workflow and still owes its own record. Steps 4 through 7 therefore
produce records that classify `RX`, `R1`, and no substantive answer to the question. The question is
answered at steps 9 and 11.

1. Satisfy P1 through P5 and run each `check`.
2. Capture the apparatus, which nothing later in this procedure reproduces. Put the full output of `gh api repos/patrickg-unity/agent-workflow-tests/branches/main/protection` into each record's `Preconditions`, and the output of `gh api repos/patrickg-unity/agent-workflow-tests/commits/main --jq .sha` into `Scenario-commit`.
3. Read PR A and PR C with the instrument and keep both outputs.
4. Dispatch the `open-pr` phase against branch B: `gh workflow run app-changes-vs-satisfied-approval.yml --repo patrickg-unity/agent-workflow-tests -f phase=open-pr -f head_branch=test/satisfied-approval-b-<RUN>`. Where P6 has not been satisfied, this dispatch is expected to fail at the creation step and its error text is the evidence for P6. Record it, satisfy P6, and dispatch again.
5. Read the pull request number the `open-pr` run reported. This is PR B. Read it with the instrument.
6. Dispatch the `approve` phase against PR A: `gh workflow run app-changes-vs-satisfied-approval.yml --repo patrickg-unity/agent-workflow-tests -f phase=approve -f pr_number=<PR A>`.
7. Read PR A again with the instrument. This is PR A's `before` reading, and it is what `control-approval-in-force` is classified from. Apply the `UNKNOWN` procedure where it returns `UNKNOWN`.
8. Satisfy P7 for PR B with the `gh pr review` command in that entry, then read PR B again with the instrument. This is PR B's `before` reading.
9. Dispatch the `request-changes` phase against PR A: `gh workflow run app-changes-vs-satisfied-approval.yml --repo patrickg-unity/agent-workflow-tests -f phase=request-changes -f pr_number=<PR A>`. Then read PR A with the instrument. This is PR A's `after` reading.
10. Read PR C again with the instrument, so that `control-protection` is classified from a reading taken in the same window as the experiment readings it licenses.
11. Dispatch the `request-changes` phase against PR B the same way, then read PR B with the instrument. This is PR B's `after` reading.
12. For each dispatch, record its id, URL, and conclusion from `gh run list --repo patrickg-unity/agent-workflow-tests --workflow app-changes-vs-satisfied-approval.yml --limit 20 --json databaseId,conclusion,url`, and keep the acting identity each review run reported with `gh run view <run id> --repo patrickg-unity/agent-workflow-tests --log | grep -F 'Reviewing as'`, verbatim. That line is provenance and not a control reading: it records which identity the workflow minted a token for, it outlives the run log's 90-day retention, and `control-identity` is classified from the `reviews` and `latestReviews` fields of the after reading rather than from it. The expected slug is `workflow-test-agent`, and a mismatch is reported rather than smoothed over even where the reviews field reads HELD.
13. Classify every control first, then the experiment, against the outcome table below, one row per run.
14. Assemble one record per run. Take the run-side fields from the run's own job log, which carries the run id, URL, acting identity, dispatched phase and UTC timestamp. Take the readings from steps 3, 5, 7, 8, 9, 10, and 11 and the apparatus from step 2. Fill `Actor` with the login that dispatched the run and `Residue` with what that run actually left behind. Hold the assembled records until they are landed in a batch, per `## Landing a record` below. A run never writes `RESULTS.md`.
15. Restore P6 and re-run its `check`, confirming it now returns `false`. Re-run P4's `check` and confirm it returns what it returned at step 1. This scenario changes no protection setting, and this step is what makes that a measurement rather than an intention. Put both readings in the records under `Notes`.
16. Run the teardown.

Step 13 orders the classifications on purpose. Once the experiment reading is in view it is hard to
read a control as anything but confirmation of it.

## Landing a record

A run never writes to this repository. The review job holds `contents: read` and nothing more, which
is what makes the measurement unambiguous, and that grant is not traded for bookkeeping.

Records reach `RESULTS.md` in batches, by the maintainer, in one act per batch rather than one per
run:

1. Collect the assembled records for every run since the last landing, oldest first.
2. Cut a branch from the current tip of `main`.
3. Append each record to the end of its scenario's `RESULTS.md`, in run order.
4. Open a pull request and merge it under the admin bypass, which `enforce_admins: false` provides
   and which the repository `README.md` `## Baseline` already names as the route by which anything
   lands here.

Never land a record by having the test App review the landing pull request. The standing rule and its
reasoning are in `scenarios/README.md`.

A record that has not landed yet is not lost, but it is held in one person's scratch rather than in
the repository, and the run log it was derived from is deleted after 90 days. Land batches before
that window closes.

## Outcome table

Exactly one row per run. Read `## Why this scenario exists and what makes it separable` before
reading the `Meaning` column, because every substantive row is bounded by it.

| Row | Selected when | Verdict | Meaning |
|---|---|---|---|
| `R1` | The dispatched phase is `approve`, every control HELD, and the arm reads `reviewDecision: APPROVED` with `mergeStateStatus: BLOCKED` | `VALID` | The App's approving review satisfies the requirement in `reviewDecision` and something else still blocks the merge. The arm is not a usable baseline in this state: a later `BLOCKED` reading would be over-determined again. Read the recorded protection JSON under `Preconditions` before dispatching `request-changes` against this pull request. |
| `R2` | The dispatched phase is `approve`, every control HELD, and the arm reads `reviewDecision: APPROVED` with `mergeStateStatus: CLEAN` | `VALID` | The approval requirement is satisfied and nothing else blocks. This is the baseline a `request-changes` dispatch against this pull request is read against, and it is what makes a later `BLOCKED` attributable. It answers nothing about blocking on its own. |
| `R3` | The dispatched phase is `request-changes`, every control HELD, and the arm reads `reviewDecision: CHANGES_REQUESTED` with `mergeStateStatus: BLOCKED`, with no `APPROVED` entry in `latestReviews` | `VALID` | The changes-requested review replaced the approval it was added beside, so the approval requirement is no longer satisfied and `BLOCKED` is over-determined. This arm does not separate its causes and answers nothing about whether the changes-requested review blocks. Report it as non-separable. |
| `R4` | The dispatched phase is `request-changes`, every control HELD, and the arm reads `reviewDecision: CHANGES_REQUESTED` with `mergeStateStatus: BLOCKED`, with an `APPROVED` entry in `latestReviews` authored by an identity other than `workflow-test-agent` | `VALID` | The App's changes-requested review blocks a merge whose approval requirement is still satisfied by another reviewer. The App's objection overrides an already-satisfied approval requirement, so the agent holds a real veto. |
| `R5` | The dispatched phase is `request-changes`, every control HELD, and the arm reads `reviewDecision: CHANGES_REQUESTED` with `mergeStateStatus: CLEAN`, with an `APPROVED` entry in `latestReviews` authored by an identity other than `workflow-test-agent` | `VALID` | The App's changes-requested review registers in `reviewDecision` and does not block. Any approver can merge straight past the App, so the agent holds no veto. |
| `R6` | The dispatched phase is `request-changes`, every control HELD, and the arm reads `reviewDecision: APPROVED` with `mergeStateStatus: CLEAN`, with the App's `CHANGES_REQUESTED` entry present in `reviews` | `VALID` | The App's changes-requested review does not register in `reviewDecision` at all once the approval requirement is satisfied, and does not block. `control-identity` is what proves the review is nevertheless present, which is what makes this row a finding rather than a review that failed to land. |
| `RC` | Any control read its `refuted` set | `REFUTED` | The apparatus was not in the assumed state. The run measured nothing about the question. Read the `Controls` block to see which control refuted and why, then fix the setup and run again. |
| `RI` | No control read its `refuted` set, and either a control read its `indeterminate` set, or the arm's reading matches no substantive row for the dispatched phase, or two substantive rows match | `INDETERMINATE` | The readings cannot be classified. Re-read without changing anything. Where two rows matched, name both in `Notes`, and the table itself needs a fix. |
| `RX` | The dispatched phase is `open-pr`, which submits no review and by design reaches no measurement, or preflight failed, or the review job errored or was cancelled before the review was submitted | `NOT-RUN` | The instrument measured nothing about the question. Record the run verbatim. For an `open-pr` dispatch, record the pull request it created or the refusal it returned, because that is the evidence for P6. For a preflight failure, record which assertion failed, and note that an unsatisfied P7 arrives here rather than as `RC`. |

`RC`, `RI`, and `RX` are reserved ids and are never used for a substantive outcome.

There is no row for `reviewDecision: CHANGES_REQUESTED` with a mergeable `mergeStateStatus` and no
`APPROVED` entry in `latestReviews`. That reading would mean a pull request with no approval in force
is mergeable under a rule requiring one, which contradicts `control-protection`'s `held` set, so a
run producing it is anomalous rather than informative. It falls to `RI` and is re-read. `HAS_HOOKS`
and `UNSTABLE` likewise have no row: this repository has no required status checks and no
pre-receive hooks under P4, so either value is a change in the apparatus rather than a result, and
`RI` is where it goes.

`R1` and `R2` are separated rather than collapsed because only one of them licenses the dispatch that
follows it. An `approve` run that lands `R1` has produced a pull request that is approved and still
blocked, and dispatching `request-changes` against it would produce a reading with two candidate
causes and no way to tell them apart. That is the failure this whole scenario exists to avoid, and
`R1` is where it would first be visible.

`RX` carries two unlike cases on purpose, a phase that never measures and a run that failed to. Both
are runs that produced no reading about the question, which is what `NOT-RUN` means, and the record's
`Verdict-reason` is where the two are told apart. Adding a verdict token for the first would put a
fifth token into a vocabulary three files share.

## Teardown

Creates: branches `test/satisfied-approval-a-<RUN>`, `test/satisfied-approval-b-<RUN>`, and
`test/satisfied-approval-c-<RUN>`, and their three pull requests. PR B is created by the workflow
rather than by the operator.

Restores: the repository setting P6 turns on, back to `false`:

```
gh api -X PUT repos/patrickg-unity/agent-workflow-tests/actions/permissions/workflow -f default_workflow_permissions=read -F can_approve_pull_request_reviews=false
gh api repos/patrickg-unity/agent-workflow-tests/actions/permissions/workflow
```

The second call is the read-back, and procedure step 15 is where its output is confirmed and
recorded. That `PUT` sends `default_workflow_permissions` as well as the field being restored,
because the endpoint takes both fields together and omitting one is not the same as leaving it
alone. `read` is this repository's standing value for it and is what P6 also sends, so neither call
changes it.

Then close all three pull requests without merging:

```
gh pr close <PR A> --repo patrickg-unity/agent-workflow-tests
gh pr close <PR B> --repo patrickg-unity/agent-workflow-tests
gh pr close <PR C> --repo patrickg-unity/agent-workflow-tests
```

Deleting the branches is optional hygiene and is deliberately not part of the teardown. Because all
three branch names carry a per-run discriminator, a branch left in place blocks nothing, and the next
run cuts fresh names from the current tip of `main`.

Two reasons to leave them. They are the run's artifacts, and the closed pull requests reference them,
so deleting them makes those pull requests' diffs unreadable and destroys the evidence the records'
`Readings` were taken against. And branch deletion is separately gated in some of the environments
this runbook is executed from, so a teardown that requires it fails for reasons unrelated to the
scenario.

Where someone does delete them, `gh pr close` may already have removed the remote ref if the
repository has automatic head-branch deletion enabled, in which case the matching
`gh api -X DELETE repos/patrickg-unity/agent-workflow-tests/git/refs/heads/<branch>` returns 422 with
"Reference does not exist". That is the already-done case rather than a failure.

Deliberately left: branch protection on `main` stays exactly as it was found, because it is the
repository baseline and this scenario reads it without writing it. Procedure step 15 is the
measurement that says so. The minted installation tokens are revoked by the action in its own post
step and need no teardown.

This scenario does not change `enforce_admins`.

## Notes on the workflow file

The workflow takes a `phase` input and submits a different review event per phase. The scenario
already on this surface that submits one changes-requested review argues the opposite way, that
fixing the event in the file means no dispatch can submit the wrong one. That argument holds where
the scenario measures one event. Here the sequence is the subject: an approval has to be in force
before the changes-requested review is submitted, and the operator has to read the pull request in
between, so the two events cannot be one dispatch and cannot be one fixed event either. What the
earlier argument protected is instead protected by `preflight`, which asserts a state per phase that
only the intended phase can satisfy: `approve` requires zero reviews, and `request-changes` requires
exactly one review and that it is an approval. A dispatch carrying the wrong phase fails there rather
than producing a well-formed reading of the wrong thing.

`phase` is declared `type: choice`, so GitHub rejects any value outside the three named options
before a run starts. `preflight` re-tests it anyway with a `case` statement, because a later edit
that adds an option to the list would otherwise reach the review step with no guard in front of it.

`pr_number` and `head_branch` are both optional, and `preflight` requires exactly the one the
dispatched phase uses and requires the other to be empty. A dispatch that sets both is refused rather
than having one silently ignored, because the ignored one is the field an operator would have
believed named the subject.

The two jobs are `open-pull-request` and `submit-review`, each with `needs: preflight` and an `if`
keyed to the phase, so exactly one of them runs per dispatch. They are separate jobs rather than
branches inside one job so that their permissions differ. `open-pull-request` holds
`pull-requests: write`, which is what opening a pull request as `github-actions[bot]` needs.
`submit-review` holds `contents: read` and nothing else, which is the point of the experiment: with
that permission set the default `GITHUB_TOKEN` cannot submit a review, so a review that does land can
only have come from the minted App token. A single job holding the union of the two grants would give
the `GITHUB_TOKEN` review-submitting access during the measurement and cost the scenario that
argument.

`preflight` holds `pull-requests: read` because it reads `reviewDecision` and the review list. That
grant cannot submit a review either.

The permitted-author test lists two identities, the repository owner taken from `GITHUB_REPOSITORY`
and the literal `github-actions[bot]`. The owner is derived rather than written down. The bot login is
written down because it is the fixed login of the default `GITHUB_TOKEN` and there is no expression
that yields it, and the test exists to rule out the App having authored the pull request it is about
to review, which GitHub would refuse.

`preflight` and the measuring jobs are separate jobs rather than steps of one job. When a needed job
fails, its dependents report as skipped rather than failed, so a run that never set up reads
differently in the run's own job data from a run whose measurement failed. That distinction is the
`RX` row, taken from structure instead of from parsing a log.

`gh pr review --request-changes` requires a body, which is why the review step passes one for both
events. An approving review does not require one, and passing it anyway keeps the two branches of the
`case` statement the same shape.

The identity line the review step prints reads `Reviewing as <slug> with event <phase>` for both
phases, so procedure step 12 has one string to grep for. That grep matches two lines in a run log,
not one. The second is the step's own echoed script text, which carries terminal escape bytes, and
the record keeps the step's output line and not that one.

Preflight tests secret presence through the expressions `${{ secrets.TEST_APP_ID != '' }}` and
`${{ secrets.TEST_APP_PRIVATE_KEY != '' }}`, which put a boolean into the step environment. Putting
the secret itself into an environment variable and testing that would answer the same question. This
form never places the private key in the runner environment at all, which is worth the slightly odd
shape on a public repository where log redaction is best-effort and degrades on multi-line structured
data.

The input guard tests `pr_number` with a parameter expansion rather than a `grep` pipeline. `grep -q`
succeeds when any single line matches, and an environment value can carry newlines, so a pipeline
accepts a multi-line input whose first or last line happens to be an integer. Stripping every digit
and testing what is left rejects that, because a newline survives the strip. `head_branch` is tested
the same way against the set of characters a branch name here is allowed to hold, so a value carrying
a newline, a space, or a shell metacharacter is refused.

One trap for whoever verifies those guards. Command substitution strips trailing newlines, so
building a test value with a `$(printf ...)` substitution silently removes the very newline the
trailing-newline case exists to exercise, and the guard then reads as accepting a value it would in
fact reject. Build multi-line and trailing-newline cases with a `$'...'` literal instead.

The action is pinned to a commit sha and carries no version comment. The sha is the anchor, and a
comment naming a version is a second claim that drifts from it.
`bcd2ba49218906704ab6c1aa796996da409d3eb1` is the commit that the tag `v3.2.0` points at, and
`app-slug` is one of the three outputs that action declares at that sha, alongside `token` and
`installation-id`. Both were read from the GitHub API on 2026-09-15.

At that sha the action's `app-id` input carries a deprecation notice pointing at `client-id`.
`app-id` still works and is what this scenario uses, because `TEST_APP_ID` holds the numeric app id.
An edit that switches to `client-id` has to change the secret's contents as well as the input name.
