# app-changes-requested-vs-protection

## Question

Does a pull request review submitted by a GitHub App installation token with the `REQUEST_CHANGES`
event block a merge to `main` under a branch protection rule requiring one approving review with
code-owner review off, on github.com, today?

The variable under test is whether the App submits a changes-requested review. Everything else is
held fixed: the same repository, the same protection settings, two pull requests opened by the same
human against the same base branch, and one read command applied to both.

## Subject

GitHub itself. There is no external artifact under test, so every record carries `Subject-pin: none`.

## What the control rules out, and what this scenario cannot separate

Read this before the outcome table. It is the reason the table's rows say what they say.

Two causes can produce `mergeStateStatus: BLOCKED` on the experiment arm: the changes-requested
review, and the one-approval requirement that the experiment arm does not satisfy. The experiment
arm carries both at once, so its `BLOCKED` reading on its own attributes nothing.

`control-protection` measures the second cause alone. It is a pull request against the same branch,
under the same rule, carrying no review of any kind. Its `held` reading is `REVIEW_REQUIRED` with
`BLOCKED`.

**What that rules out.** It rules out reading `BLOCKED` on the experiment arm as evidence about the
changes-requested review. The control demonstrates that the unmet approval requirement is by itself
sufficient to produce `BLOCKED` under this exact rule, so the experiment arm's `BLOCKED` is already
fully explained without reference to any review, and it carries no information about whether the
review blocks. A scenario that read only the experiment arm would report `BLOCKED` as the answer to
the question and would be reporting the approval requirement.

**What the two arms do separate.** They separate whether the App's changes-requested review
registers in `reviewDecision` at all. The arms differ in exactly one thing, the App review, so a
`reviewDecision` reading that differs between them is attributable to it and to nothing else. That
is the discriminator this scenario reads, and it is why `reviewDecision` and `mergeStateStatus` are
recorded as separate fields and never collapsed into one verdict.

**What no arm here can separate.** Whether the changes-requested review blocks a pull request that
would otherwise be mergeable. An arm answering that needs one approving review and one
changes-requested review in force on the same pull request at the same time. Whether that state is
reachable with a single reviewing identity is unmeasured here, and this repository has only one
reviewing identity available in any case: its one human authors the pull requests and GitHub bars a
pull request author from approving or requesting changes on their own pull request, which leaves the
App. Building that arm therefore starts by measuring whether one identity can hold both, which is
what the sequence of an approving review followed by a changes-requested review from the same App
would answer, and only then with a second reviewing identity. Either way it is a separate scenario
with its own id, because its variable is the order of two reviews rather than the event of one, and
it is not a fix to this one.

## Prerequisites

Five entries. Each is done before the workflow is dispatched, and each `check` is read-only.

P1
what: A GitHub App owned by `patrickg-unity` exists, granting it `Pull requests: Read and write` and nothing else, with its slug exactly `workflow-test-agent`, because `control-identity` asserts that the one review on the experiment arm carries that exact slug as its author.
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
what: Branch protection on `main` is in force requiring one approving review, with code-owner review off, last-push approval off, no required status checks, no push restrictions, and no CODEOWNERS file in the repository. This scenario does not establish that state and does not change it; it is part of the repository baseline. The absent CODEOWNERS file is part of this entry rather than a separate one, because `require_code_owner_reviews: false` and an absent CODEOWNERS file are two independent ways the code-owner path stays out of this measurement and the question names both.
who: repository admin
where: no action, where the check already passes. Where it does not, the repository baseline has drifted and is restored before this scenario is dispatched.
produces: an active protection rule on `main`
lifetime: standing
check: `gh api repos/patrickg-unity/agent-workflow-tests/branches/main/protection --jq '{approvals: .required_pull_request_reviews.required_approving_review_count, code_owners: .required_pull_request_reviews.require_code_owner_reviews, last_push: .required_pull_request_reviews.require_last_push_approval, checks: .required_status_checks, restrictions: .restrictions, admins: .enforce_admins.enabled}'` returns `{"admins":false,"approvals":1,"checks":null,"code_owners":false,"last_push":false,"restrictions":null}`, and `gh api repos/patrickg-unity/agent-workflow-tests/contents/CODEOWNERS`, the same call on `contents/.github/CODEOWNERS`, and the same call on `contents/docs/CODEOWNERS` each return HTTP 404.

P5
what: Open PR A on branch `test/app-changes-a-<RUN>` and PR B on branch `test/app-changes-b-<RUN>`, each a different one-line edit, both authored by `patrickg-unity`, both targeting `main`, neither reviewed by anyone. Cut both branches from the current tip of `main`, so that neither reads `mergeStateStatus: BEHIND`, which no substantive row covers. Both branch names carry a per-run discriminator `<RUN>`, so a branch left behind by an earlier run can never block a later one and deleting it is optional hygiene rather than a prerequisite.
who: operator
where: the commands in the block below, with `<clone>` the local checkout path.
produces: two open, unreviewed pull requests
lifetime: per-run
check: `gh pr view <N> --repo patrickg-unity/agent-workflow-tests --json state,author,reviews` returns `OPEN`, author `patrickg-unity`, and an empty `reviews` array, for both.

```
git -C <clone> fetch origin main
git -C <clone> switch --no-track -c test/app-changes-a-<RUN> origin/main
printf '%s\n' "Arm A marker." >> <clone>/README.md
git -C <clone> commit -am "test: arm A marker for app-changes-requested-vs-protection"
git -C <clone> push origin test/app-changes-a-<RUN>
gh pr create --repo patrickg-unity/agent-workflow-tests --base main --head test/app-changes-a-<RUN> --title "app-changes arm A" --body "Experiment arm. Receives the App changes-requested review."
git -C <clone> switch --no-track -c test/app-changes-b-<RUN> origin/main
printf '%s\n' "Arm B marker." >> <clone>/README.md
git -C <clone> commit -am "test: arm B marker for app-changes-requested-vs-protection"
git -C <clone> push origin test/app-changes-b-<RUN>
gh pr create --repo patrickg-unity/agent-workflow-tests --base main --head test/app-changes-b-<RUN> --title "app-changes arm B" --body "Control arm. Receives no review."
```

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

### The App must never author the pull request it reviews

GitHub bars a pull request author from requesting changes on their own pull request, so a variant in
which the App opens the pull request measures that refusal instead of the protection question. The
operator authors both pull requests here for that reason.

### What preflight asserts, and what it deliberately does not

The workflow's `preflight` job re-asserts P3 in full, and the part of P5 that concerns the one pull
request named by `pr_number`: that it can be read, that it is `OPEN`, that its author is the
repository owner rather than the App, and that it carries no review yet. A run that starts without
those fails legibly and its measurement job reports as skipped.

It does not cover the rest of P5. PR B is established by the control arm's own reading at
classification time and by nothing in the workflow, and preflight never looks at it.

The review-count assertion is the one that matters most, and it fails in a direction worth naming.
Dispatch against a PR A that already carries a changes-requested review and, without that
assertion, the run completes normally: PR A reads `CHANGES_REQUESTED` with `BLOCKED`, both controls
read HELD, and the table selects `R1`, recording an earlier review as evidence about this run's App
review. That failure produces well-formed readings throughout and is the third member of the class
`RX` and `RC` exist to catch. It is also why a second dispatch against the same PR A is refused,
since the first run's own App review is a review.

It does not assert P4, for two reasons. The control arm is what proves protection at measurement
time, and a preflight that proved it as well would make a setup failure and a control failure
indistinguishable. The default `GITHUB_TOKEN` also has no route to it: the `permissions` key a
workflow can set has no `administration` entry, so no job can grant itself the repository
administration access the branch-protection read needs.

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
least-privilege: `Pull requests: Read and write` on an installation covering `agent-workflow-tests` and no other repository. A changes-requested review needs the same grant an approving review needs.
rotate-when: immediately, if this value is found in any workflow log or any commit. Also on any change in who holds it.
existence-check: `gh api repos/patrickg-unity/agent-workflow-tests/actions/secrets --jq '.secrets[].name'` lists `TEST_APP_PRIVATE_KEY`

## Instrument

```
gh pr view <N> --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
```

Every arm is read with this command and no other. Three fields, read and reported separately:

- `reviewDecision`, one of `CHANGES_REQUESTED`, `APPROVED`, `REVIEW_REQUIRED`, or `null`. The three
  named values are the complete `PullRequestReviewDecision` enum, read from the GitHub GraphQL
  schema on 2026-09-15 by introspecting `__type(name: "PullRequestReviewDecision")`. The field is
  nullable. This runbook makes no claim about which configuration produces `null`; a `null` reading
  is handled by the control sets and by `RI`, and is never interpreted.
- `mergeStateStatus`, one of `DIRTY`, `UNKNOWN`, `BLOCKED`, `BEHIND`, `UNSTABLE`, `HAS_HOOKS`, or
  `CLEAN`. That is the complete `MergeStateStatus` enum, read from the same schema in the same call.
  The field is not nullable.
- `reviews`, the list of reviews on the pull request, each with its author and its state. A review
  state is one of `PENDING`, `COMMENTED`, `APPROVED`, `CHANGES_REQUESTED`, or `DISMISSED`, the
  complete `PullRequestReviewState` enum from the same schema.

The author of an App's review reads differently through the two APIs, and this instrument returns
the shorter form. Measured on 2026-09-15 against pull request 1 of this repository, which carries
one review from this same App. This instrument returned `author.login` as `workflow-test-agent`,
while `gh api repos/patrickg-unity/agent-workflow-tests/pulls/1/reviews` returned `user.login` as
`workflow-test-agent[bot]` with `user.type` as `Bot`. `control-identity` therefore expects
`workflow-test-agent`, and a cross-check run through the REST endpoint will show
`workflow-test-agent[bot]` without anything being wrong.

**A changes-requested review appearing in `reviews` is not the same thing as `reviewDecision`
reading `CHANGES_REQUESTED`.** A review can be present and still not move the decision field, and
which of those happened is half of what this scenario asks. Record both fields verbatim in every
record and never infer one from the other.

**`mergeStateStatus: BLOCKED` on the experiment arm is not evidence that the changes-requested
review blocks.** That arm holds no approving review either, and `control-protection` shows the
missing approval alone produces `BLOCKED`. The section `## What the control rules out, and what this
scenario cannot separate` is the long form. These two paragraphs are the two most likely places for
this scenario to be misread.

One arm's `reviewDecision` and `mergeStateStatus` reading is therefore one of 28 pairs. The control
sets and the outcome rows below are stated against that space, and `control-identity` is stated
against the `reviews` field instead.

### A reading of `UNKNOWN`

`mergeStateStatus: UNKNOWN` means the state cannot currently be determined. GitHub computes
mergeability asynchronously, so a pull request read soon after it is opened or soon after a review
lands legitimately returns `UNKNOWN`. It is a timing artifact and not a result.

Re-read instead of recording it. Wait 15 seconds and run the same command again, up to 4 further
attempts, so at most 5 readings across roughly a minute. Take the first reading that is not
`UNKNOWN`. If the fifth reading is still `UNKNOWN`, record that reading verbatim and classify the
run `RI`.

Never classify a bounded-out `UNKNOWN` as `RC`. `UNKNOWN` is not evidence that protection is off,
and `RC` sends the next person to fix a configuration that was never broken.

The wait and the attempt count are a first bound chosen against the known behavior rather than
against measured data. Revise them once this scenario has real runs, and say so in the record that
motivates the change.

## Arms

| Arm | Pull request | Branch | Author | What it receives |
|---|---|---|---|---|
| `experiment` | PR A | `test/app-changes-a-<RUN>` | `patrickg-unity` | one changes-requested review submitted by the App |
| `control-protection` | PR B | `test/app-changes-b-<RUN>` | `patrickg-unity` | nothing |
| `control-identity` | PR A | `test/app-changes-a-<RUN>` | `patrickg-unity` | the same App review, and the same reading as `experiment` |

`experiment` and `control-identity` are one subject read once. The instrument is run against PR A a
single time per moment and both arms are classified from that one output, which is why the record
carries one `Readings` entry for PR A and two verdicts over it. They differ in what they assert:
`experiment` answers the question from `reviewDecision` and `mergeStateStatus`, while
`control-identity` proves the apparatus from `reviews`, namely that the App is the only thing that
reviewed and that what it submitted was a changes-requested review rather than an approval. Running
the instrument twice would be a defect, because the two arms could then read a pull request that
changed between the calls.

Both pull requests target `main`, both are opened by the same human, and neither is reviewed by a
human at any point. The only difference between them is the App review on PR A.

## Control arms

```
control-id: control-protection
asserts: branch protection on main is active and requires one approving review, so an unreviewed pull request cannot merge, and the reading an unreviewed pull request produces under that rule is established by this arm rather than assumed.
instrument: gh pr view <PR B> --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
value-space: the whole instrument output. reviewDecision is CHANGES_REQUESTED, APPROVED, REVIEW_REQUIRED, or null, mergeStateStatus is DIRTY, UNKNOWN, BLOCKED, BEHIND, UNSTABLE, HAS_HOOKS, or CLEAN, which is 28 pairs, and reviews is a list of any length from zero upward.
held: reviews empty, reviewDecision REVIEW_REQUIRED, and mergeStateStatus BLOCKED.
refuted: reviews non-empty under any pair, or reviews empty with mergeStateStatus CLEAN, HAS_HOOKS, or UNSTABLE under any reviewDecision, or reviews empty with reviewDecision null, APPROVED, or CHANGES_REQUESTED under any mergeStateStatus other than UNKNOWN.
indeterminate: every reading in value-space not listed in held or refuted, which is an empty reviews list paired with mergeStateStatus DIRTY, BEHIND, or UNKNOWN, and so includes reviewDecision null with mergeStateStatus UNKNOWN, so that a mergeability calculation still in flight is re-read rather than treated as protection being off.
would-fail-if: branch protection on main is deleted, or its required approving review count is set to 0, or the rule is written against a branch other than main, in which case PR B reads mergeStateStatus CLEAN and nothing can be concluded from PR A. Or any review at all lands on PR B, from a human or anything else, at any point up to the after reading, in which case this arm is no longer the zero-review baseline the experiment arm is compared against and the reviewDecision comparison the scenario rests on is gone.
```

```
control-id: control-identity
asserts: the App under test is the only thing that reviewed the experiment arm, and what it submitted was a changes-requested review, so the reading attributes the result to an App changes-requested review and to nothing else.
instrument: gh pr view <PR A> --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
value-space: the reviews array of the experiment arm's after reading, of any length from zero upward, each entry carrying an author login that may also be unreadable and a state that is PENDING, COMMENTED, APPROVED, CHANGES_REQUESTED, or DISMISSED and may also be unreadable.
held: the reviews array holds exactly one entry, that entry's author is workflow-test-agent with no [bot] suffix for the reason under the instrument above, and that entry's state is CHANGES_REQUESTED.
refuted: the reviews array holds any number of entries other than one, or holds exactly one entry whose author is a readable login other than workflow-test-agent, or holds exactly one entry authored by workflow-test-agent whose state is a readable value other than CHANGES_REQUESTED.
indeterminate: every reading in value-space not listed in held or refuted, which is an entry whose author or whose state cannot be read, as when the reviewing account has been deleted. The measure job never running is not classified here, because RX is selected before any arm is read.
would-fail-if: TEST_APP_ID or TEST_APP_PRIVATE_KEY belongs to a different App, in which case the single review carries that App's login. Or the workflow submits an approval instead of a changes-requested review, as an edit to the review step's event flag would do, in which case the single entry reads APPROVED and the run is thrown away rather than recorded as a finding about changes-requested reviews. Or any second review lands on the experiment arm, from a human or anything else, at any point including after preflight passed and before the after reading, in which case the array holds two entries and the App's review can no longer be isolated from the other one.
```

## Procedure

1. Satisfy every prerequisite and run each `check`.
2. Capture the apparatus, which nothing later in this procedure reproduces. Put the full output of `gh api repos/patrickg-unity/agent-workflow-tests/branches/main/protection` into the record's `Preconditions`, and the output of `gh api repos/patrickg-unity/agent-workflow-tests/commits/main --jq .sha` into `Scenario-commit`.
3. Read both pull requests with the instrument and keep both outputs. This is the `before` reading.
4. Dispatch it: `gh workflow run app-changes-requested-vs-protection.yml --repo patrickg-unity/agent-workflow-tests -f pr_number=<PR A>`.
5. Wait for the run to finish, then record its id, URL, and conclusion from `gh run list --repo patrickg-unity/agent-workflow-tests --workflow app-changes-requested-vs-protection.yml --limit 1 --json databaseId,conclusion,url`.
6. Keep the acting identity the workflow reported, with `gh run view <run id> --repo patrickg-unity/agent-workflow-tests --log | grep -F 'Requesting changes as'`, verbatim. This is provenance and not a control reading: it records which identity the workflow minted a token for, it outlives the run log's 90-day retention, and `control-identity` is classified in step 8 from the `reviews` field of the `after` reading rather than from this line. The expected slug is `workflow-test-agent`, and a mismatch here is reported rather than smoothed over even when the reviews field reads HELD.
7. Read both pull requests again with the instrument. This is the `after` reading. Apply the `UNKNOWN` procedure above to either arm that returns it.
8. Classify both controls first, then the experiment, against the outcome table below.
9. Assemble the record. Take the run-side fields from the run's job log, which carries the run id, URL, acting identity, dispatch input and UTC timestamp. Take the readings from steps 3 and 7 and the apparatus from step 2. Fill `Actor` with the login that dispatched the run and `Residue` with what this run actually left behind, which can differ from the teardown's intent when a run failed partway. Hold the assembled record until it is landed in a batch, per `## Landing a record` below. A run never writes `RESULTS.md`.
10. Re-run P4's `check` and confirm it returns what it returned at step 1. This scenario changes no protection setting, and this step is what makes that a measurement rather than an intention. Put the second reading in the record under `Notes`.
11. Run the teardown.

Step 8 orders the classifications on purpose. Once the experiment reading is in view it is hard to
read a control as anything but confirmation of it.

## Landing a record

A run never writes to this repository. The measuring job holds `contents: read` and nothing more,
which is what makes the measurement unambiguous, and that grant is not traded for bookkeeping.

Records reach `RESULTS.md` in batches, by the maintainer, in one act per batch rather than one per
run:

1. Collect the assembled records for every run since the last landing, oldest first.
2. Cut a branch from the current tip of `main`.
3. Append each record to the end of its scenario's `RESULTS.md`, in run order.
4. Open a pull request and merge it under the admin bypass, which `enforce_admins: false` provides
   and which the repository `README.md` `## Baseline` already names as the route by which anything
   lands here.

Never land a record by having the test App review the landing pull request. The standing rule and
its reasoning are in `scenarios/README.md`.

A record that has not landed yet is not lost, but it is held in one person's scratch rather than in
the repository, and the run log it was derived from is deleted after 90 days. Land batches before
that window closes.

## Outcome table

Exactly one row per run. Read `## What the control rules out, and what this scenario cannot
separate` before reading the `Meaning` column, because both substantive rows are bounded by it.

| Row | Selected when | Verdict | Meaning |
|---|---|---|---|
| `R1` | Every control HELD, and `experiment` reads `reviewDecision: CHANGES_REQUESTED` with `mergeStateStatus: BLOCKED` | `VALID` | An App installation token's changes-requested review registers in `reviewDecision`, and the pull request cannot merge. The block is over-determined and this row does not establish that the review blocks on its own: the experiment arm holds no approving review either, and `control-protection` shows that alone produces `BLOCKED`. What this row establishes is registration. |
| `R2` | Every control HELD, and `experiment` reads `reviewDecision: REVIEW_REQUIRED` with `mergeStateStatus: BLOCKED` | `VALID` | An App installation token's changes-requested review does not register in `reviewDecision` at all. The experiment reading is identical to the control's on both fields, and `control-identity` is what proves a review is nevertheless present, which is what makes this row a finding rather than a review that failed to land. |
| `RC` | Any control read its `refuted` set | `REFUTED` | The apparatus was not in the assumed state. The run measured nothing about the question. Read the `Controls` block to see which control refuted and why, then fix the setup and run again. |
| `RI` | No control read its `refuted` set, and either a control read its `indeterminate` set, or the `experiment` reading matches no substantive row, or two substantive rows match | `INDETERMINATE` | The readings cannot be classified. Re-read without changing anything. Where two rows matched, name both in `Notes`, and the table itself needs a fix. |
| `RX` | Preflight failed, or the job errored or was cancelled before the review was submitted. No arm produced a reading, so this row is selected and no other row is evaluated | `NOT-RUN` | The instrument never ran. Record the failure verbatim. |

`RC`, `RI`, and `RX` are reserved ids and are never used for a substantive outcome.

There are two substantive rows and not three. A reading of `reviewDecision: CHANGES_REQUESTED` with
a mergeable `mergeStateStatus` would be the direct negative answer to the question, and it has no
row on purpose: the experiment arm holds no approving review, so that reading also contradicts
`control-protection`'s `held` set, and a run producing it is anomalous rather than informative. It
falls to `RI` and is re-read. A reading of `reviewDecision: APPROVED` likewise has no row, because
it means the workflow submitted the wrong review event, which `control-identity` catches as `RC`.

Both `RX` and `RC` exist because each catches a failure that looks like a finding. Without `RX`, a
workflow that dies before submitting the review leaves PR A reading `REVIEW_REQUIRED` and `BLOCKED`,
which is exactly row `R2`, and this repository would record "an App's changes-requested review does
not register" on the strength of a review that was never submitted. That is the most dangerous
single failure in this scenario, because `R2` is a plausible answer and nothing about the reading
looks wrong. `control-identity`'s review-count assertion is the second guard on the same failure,
and it is why that control is not optional here.

## Teardown

Creates: branches `test/app-changes-a-<RUN>` and `test/app-changes-b-<RUN>`, and their two pull
requests.

Restores: nothing that a later run depends on, because this scenario changes no repository setting.
Close both pull requests without merging:

```
gh pr close <PR A> --repo patrickg-unity/agent-workflow-tests
gh pr close <PR B> --repo patrickg-unity/agent-workflow-tests
```

Deleting the branches is optional hygiene and is deliberately not part of the teardown. Because both
branch names carry a per-run discriminator, a branch left in place blocks nothing, and the next run
cuts fresh names from the current tip of `main`.

Two reasons to leave them. They are the run's artifacts, and the closed pull requests reference them,
so deleting them makes those pull requests' diffs unreadable and destroys the evidence the record's
`Readings` were taken against. And branch deletion is separately gated in some of the environments
this runbook is executed from, so a teardown that requires it fails for reasons unrelated to the
scenario.

Where someone does delete them, `gh pr close` may already have removed the remote ref if the
repository has automatic head-branch deletion enabled, in which case the matching
`gh api -X DELETE repos/patrickg-unity/agent-workflow-tests/git/refs/heads/<branch>` returns 422 with
"Reference does not exist". That is the already-done case rather than a failure.

Deliberately left: branch protection on `main` stays exactly as it was found, because it is the
repository baseline and this scenario reads it without writing it. Procedure step 10 is the
measurement that says so. The minted installation token is revoked by the action in its own post
step and needs no teardown.

This scenario does not change `enforce_admins`.

## Notes on the workflow file

This scenario has its own workflow file rather than a review-event input on an existing one. The
review event is the operation under test, so fixing it in the file means no dispatch can submit the
wrong one, and `control-identity`'s `held` set has one value to assert rather than one that varies
per dispatch. It also leaves the workflow behind an already-recorded scenario untouched, so that
scenario's records keep citing a file that has not changed under them.

`preflight` and `measure` are separate jobs rather than steps of one job. When a needed job fails,
its dependents report as skipped rather than failed, so a run that never set up reads differently in
the run's own job data from a run whose measurement failed. That distinction is the `RX` row, taken
from structure instead of from parsing a log.

`measure` declares `permissions: contents: read` and nothing else. That is the point of the
experiment: with that permission set the default `GITHUB_TOKEN` cannot submit a review, so a review
that does land can only have come from the minted App token.

`gh pr review --request-changes` requires a body, which is why the review step passes one. An
approving review does not, so a workflow adapted from one that approves and stripped of its `--body`
fails at the review step.

Preflight tests secret presence through the expressions `${{ secrets.TEST_APP_ID != '' }}` and
`${{ secrets.TEST_APP_PRIVATE_KEY != '' }}`, which put a boolean into the step environment. Putting
the secret itself into an environment variable and testing that would answer the same question. This
form never places the private key in the runner environment at all, which is worth the slightly odd
shape on a public repository where log redaction is best-effort and degrades on multi-line
structured data.

The input guard tests `pr_number` with a parameter expansion rather than a `grep` pipeline. `grep -q`
succeeds when any single line matches, and an environment value can carry newlines, so a pipeline
accepts a multi-line input whose first or last line happens to be an integer. Stripping every digit
and testing what is left rejects that, because a newline survives the strip.

One trap for whoever verifies that guard. Command substitution strips trailing newlines, so building
a test value with a `$(printf ...)` substitution silently removes the very newline the
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
