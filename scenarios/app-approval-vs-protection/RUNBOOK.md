# app-approval-vs-protection

## Question

Does a pull request review submitted by a GitHub App installation token satisfy a branch protection
rule on `main` requiring one approving review, on github.com, today?

The variable under test is the identity that submits the review. Everything else is held fixed: the
same repository, the same protection settings, two pull requests opened by the same human against
the same base branch, and one read command applied to both.

## Subject

GitHub itself. There is no external artifact under test, so every record carries `Subject-pin: none`.

## Prerequisites

Five entries. Each is done before the workflow is dispatched, and each `check` is read-only.

P1
what: Create a GitHub App owned by `patrickg-unity`, granting it `Pull requests: Read and write` and nothing else. Name it so its slug is exactly `workflow-test-agent`, because `control-identity` asserts that the one review on the experiment arm carries that exact slug as its author.
who: operator
where: https://github.com/settings/apps/new
produces: an App with a numeric app id, a slug, and one downloadable private key
lifetime: one-time
check: `curl -s -o /dev/null -w '%{http_code}\n' https://github.com/apps/workflow-test-agent` returns `200` or `301`. It proves existence only, for the reason below.

P2
what: Install that App on `patrickg-unity/agent-workflow-tests`, and on no other repository.
who: operator
where: https://github.com/settings/apps/workflow-test-agent/installations
produces: an installation of the App on this repository
lifetime: one-time
check: open https://github.com/patrickg-unity/agent-workflow-tests/settings/installations and confirm the App is listed. This is the one entry here whose check is a page rather than a command, for the reason below.

P3
what: Set repository secrets `TEST_APP_ID` and `TEST_APP_PRIVATE_KEY`.
who: repository admin
where: https://github.com/patrickg-unity/agent-workflow-tests/settings/secrets/actions
produces: two named repository secrets
lifetime: one-time, until rotation
check: `gh api repos/patrickg-unity/agent-workflow-tests/actions/secrets --jq '.secrets[].name'` lists both names. Secret values are never readable through the API and are never read here.

P4
what: Enable branch protection on `main` requiring one approving review.
who: repository admin
where: `gh api -X PUT repos/patrickg-unity/agent-workflow-tests/branches/main/protection --input - <<<'{"required_pull_request_reviews":{"required_approving_review_count":1,"require_code_owner_reviews":false,"dismiss_stale_reviews":false},"required_status_checks":null,"enforce_admins":false,"restrictions":null}'`
produces: an active protection rule on `main`
lifetime: one-time, and left in place afterwards as part of the repository baseline
check: `gh api repos/patrickg-unity/agent-workflow-tests/branches/main/protection --jq '{approvals: .required_pull_request_reviews.required_approving_review_count, admins: .enforce_admins.enabled}'` returns `{"approvals":1,"admins":false}`.

P5
what: Open PR A on branch `test/app-approval-a-<RUN>` and PR B on branch `test/app-approval-b-<RUN>`, each a different one-line edit, both authored by `patrickg-unity`, both targeting `main`, neither reviewed by anyone. Cut both branches from the current tip of `main`, so that neither reads `mergeStateStatus: BEHIND`, which no substantive row covers. Both branch names carry a per-run discriminator `<RUN>`, so a branch left behind by an earlier run can never block a later one and deleting it is optional hygiene rather than a prerequisite.
who: operator
where: the commands in the block below, with `<clone>` the local checkout path.
produces: two open, unreviewed pull requests
lifetime: per-run
check: `gh pr view <N> --repo patrickg-unity/agent-workflow-tests --json state,author,reviews` returns `OPEN`, author `patrickg-unity`, and an empty `reviews` array, for both.

```
git -C <clone> fetch origin main
git -C <clone> switch --no-track -c test/app-approval-a-<RUN> origin/main
printf '%s\n' "Arm A marker." >> <clone>/README.md
git -C <clone> commit -am "test: arm A marker for app-approval-vs-protection"
git -C <clone> push origin test/app-approval-a-<RUN>
gh pr create --repo patrickg-unity/agent-workflow-tests --base main --head test/app-approval-a-<RUN> --title "app-approval arm A" --body "Experiment arm. Receives the App review."
git -C <clone> switch --no-track -c test/app-approval-b-<RUN> origin/main
printf '%s\n' "Arm B marker." >> <clone>/README.md
git -C <clone> commit -am "test: arm B marker for app-approval-vs-protection"
git -C <clone> push origin test/app-approval-b-<RUN>
gh pr create --repo patrickg-unity/agent-workflow-tests --base main --head test/app-approval-b-<RUN> --title "app-approval arm B" --body "Control arm. Receives no review."
```

### P1 proves existence only

The App slug namespace is global and the page is anonymous, so a `200` proves that an App with that
slug exists somewhere. It does not prove the App belongs to `patrickg-unity`, and it does not prove
the App grants `Pull requests: Read and write`, which is what P1's `what:` requires. Confirm both by
eye on https://github.com/settings/apps/workflow-test-agent, the same way P2 is confirmed. An App
listed on the marketplace redirects rather than returning 200, so the check accepts `301` too.

The API route that would read the owner is not open here. `gh api /apps/workflow-test-agent` returns
HTTP 404 with an ordinary user token, because that endpoint needs an App JWT for an App that is not
publicly listed. The identical call against a listed App, `gh api /apps/dependabot`, returns its
owner, so the 404 is this App's visibility rather than a broken call. Both were run on 2026-09-15 as
`patrickg-unity`.

### P2 has no command form, and it is the expensive one to miss

Two API routes to check the installation were tried from this machine as `patrickg-unity` on
2026-09-15 and neither works with an ordinary user token.
`gh api repos/patrickg-unity/agent-workflow-tests/installation` returns HTTP 401 with "A JSON web
token could not be decoded", because that endpoint expects an App JWT. `gh api /user/installations`
returns HTTP 403 with "You must authenticate with an access token authorized to a GitHub App in
order to list installations".

P2 is also the prerequisite whose absence costs the most. An App that exists with both secrets set
but is not installed on this repository mints no token, so the run dies at the token step with a
message about the installation and nothing about approvals. That is an `RX` run and it costs a whole
dispatch to discover.

### The App must never author the pull request it reviews

GitHub rejects a self-approval, so a variant in which the App opens the pull request measures that
refusal instead of the protection question. The operator authors both pull requests here for that
reason, and any future variant of this scenario keeps the two identities separate.

### What preflight asserts, and what it deliberately does not

The workflow's `preflight` job re-asserts P3 in full, and the part of P5 that concerns the one pull
request named by `pr_number`: that it can be read, that it is `OPEN`, that its author is the
repository owner rather than the App, and that it carries no review yet. A run that starts without
those fails legibly and its measurement job reports as skipped.

It does not cover the rest of P5. PR B is established by the control arm's own reading at
classification time and by nothing in the workflow, and preflight never looks at it.

The review-count assertion is the one that matters most. Dispatch against a PR A already carrying an
approving review and, without it, the run completes normally: PR A reads `APPROVED` with `CLEAN`,
both controls read HELD, and the table selects `R1`, recording a human's review as evidence that App
approvals count. That failure produces well-formed readings throughout and is the third member of
the class `RX` and `RC` exist to catch. It is also why a second dispatch against the same PR A is
refused, since the first run's own App review is a review.

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
least-privilege: `Pull requests: Read and write` on an installation covering `agent-workflow-tests` and no other repository
rotate-when: immediately, if this value is found in any workflow log or any commit. Also on any change in who holds it.
existence-check: `gh api repos/patrickg-unity/agent-workflow-tests/actions/secrets --jq '.secrets[].name'` lists `TEST_APP_PRIVATE_KEY`

## Instrument

```
gh pr view <N> --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
```

Every arm is read with this command and no other. Three fields, read and reported separately:

- `reviewDecision`, one of `APPROVED`, `CHANGES_REQUESTED`, `REVIEW_REQUIRED`, or `null`. The field is nullable, and `null` is what it reads when no review is required at all.
- `mergeStateStatus`, one of `DIRTY`, `UNKNOWN`, `BLOCKED`, `BEHIND`, `UNSTABLE`, `HAS_HOOKS`, `CLEAN`. The field is not nullable.
- `reviews`, the list of reviews on the pull request, each with its author and its state.

The author of an App's review reads differently through the two APIs, and this instrument returns
the shorter form. `gh pr view --json reviews` resolves `author.login` to the bare App slug, while
`gh api repos/OWNER/REPO/pulls/N/reviews` resolves `user.login` to the same slug with `[bot]`
appended. Measured 2026-09-15 on two public pull requests reviewed by Apps: `renovatebot/renovate`
pull request 46230 returned `renovate-approve` through this instrument and `renovate-approve[bot]`
through the REST call, and `actions/checkout` pull request 2560 returned
`copilot-pull-request-reviewer` and `copilot-pull-request-reviewer[bot]` the same way. A human
author reads identically through both. `control-identity` therefore expects `workflow-test-agent`
and a cross-check run through the REST endpoint will show `workflow-test-agent[bot]` without
anything being wrong.

**An approving review appearing in `reviews` is not the same thing as `reviewDecision` reading
`APPROVED`.** A review can be present and still not count toward the protection rule, which is the
entire question this scenario asks. Record both fields verbatim in every record and never infer one
from the other. This is the single most likely place for this scenario to be misread.

One arm's reading is therefore one of the 28 pairs of a `reviewDecision` reading and a
`mergeStateStatus` value. The control sets and the outcome rows below are stated against that space.

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
| `experiment` | PR A | `test/app-approval-a-<RUN>` | `patrickg-unity` | one approving review submitted by the App |
| `control-protection` | PR B | `test/app-approval-b-<RUN>` | `patrickg-unity` | nothing |
| `control-identity` | PR A | `test/app-approval-a-<RUN>` | `patrickg-unity` | the same App review, and the same reading as `experiment` |

`experiment` and `control-identity` are one subject read once. The instrument is run against PR A a
single time per moment and both arms are classified from that one output, which is why the record
carries one `Readings` entry for PR A and two verdicts over it. They differ in what they assert:
`experiment` answers the question from `reviewDecision` and `mergeStateStatus`, while
`control-identity` proves the apparatus from `reviews`, namely that the App is the only thing that
reviewed. Running the instrument twice would be a defect, because the two arms could then read a
pull request that changed between the calls.

Both pull requests target `main`, both are opened by the same human, and neither is reviewed by a
human at any point. The only difference between them is the App review on PR A.

## Control arms

```
control-id: control-protection
asserts: branch protection on main is active and requires one approving review, so an unapproved pull request cannot merge.
instrument: gh pr view <PR B> --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
value-space: reviewDecision is APPROVED, CHANGES_REQUESTED, REVIEW_REQUIRED, or null, and mergeStateStatus is DIRTY, UNKNOWN, BLOCKED, BEHIND, UNSTABLE, HAS_HOOKS, or CLEAN. 28 pairs.
held: reviewDecision REVIEW_REQUIRED and mergeStateStatus BLOCKED.
refuted: mergeStateStatus CLEAN, HAS_HOOKS, or UNSTABLE under any reviewDecision, or reviewDecision null under any mergeStateStatus other than UNKNOWN.
indeterminate: every reading in value-space not listed in held or refuted, which now includes the pair reviewDecision null with mergeStateStatus UNKNOWN, so that a mergeability calculation still in flight is re-read rather than treated as protection being off.
would-fail-if: branch protection on main is deleted, or its required approving review count is set to 0, or the rule is written against a branch other than main. PR B then reads mergeStateStatus CLEAN with reviewDecision null, and nothing can be concluded from PR A.
```

```
control-id: control-identity
asserts: the App under test is the only thing that reviewed the experiment arm, so the reading attributes the approval to the App and to nothing else.
instrument: gh pr view <PR A> --repo patrickg-unity/agent-workflow-tests --json reviewDecision,mergeStateStatus,reviews
value-space: the reviews array of the experiment arm's after reading, of any length from zero upward, each entry carrying an author login that may also be unreadable.
held: the reviews array holds exactly one entry and that entry's author is workflow-test-agent, with no [bot] suffix, for the reason under the instrument above.
refuted: the reviews array holds any number of entries other than one, or holds exactly one entry whose author is a readable login other than workflow-test-agent.
indeterminate: every reading in value-space not listed in held or refuted, which is an entry whose author cannot be read, as when the reviewing account has been deleted. The measure job never running is not classified here, because RX is selected before any arm is read.
would-fail-if: TEST_APP_ID or TEST_APP_PRIVATE_KEY belongs to a different App, in which case the single review carries that App's login. Or any second review lands on the experiment arm, from a human or anything else, at any point including after preflight passed and before the after reading, in which case the array holds two entries and the App's approval can no longer be isolated from the other one.
```

## Procedure

1. Satisfy every prerequisite and run each `check`.
2. Capture the apparatus, which nothing later in this procedure reproduces. Put the full output of `gh api repos/patrickg-unity/agent-workflow-tests/branches/main/protection` into the record's `Preconditions`, and the output of `gh api repos/patrickg-unity/agent-workflow-tests/commits/main --jq .sha` into `Scenario-commit`.
3. Read both pull requests with the instrument and keep both outputs. This is the `before` reading.
4. Dispatch it: `gh workflow run app-approval-vs-protection.yml --repo patrickg-unity/agent-workflow-tests -f pr_number=<PR A>`.
5. Wait for the run to finish, then record its id, URL, and conclusion from `gh run list --repo patrickg-unity/agent-workflow-tests --workflow app-approval-vs-protection.yml --limit 1 --json databaseId,conclusion,url`.
6. Keep the acting identity the workflow reported, with `gh run view <run id> --repo patrickg-unity/agent-workflow-tests --log | grep -F 'Approving as'`, verbatim. This is provenance and not a control reading: it records which identity the workflow minted a token for, it outlives the run log's 90-day retention, and `control-identity` is classified in step 8 from the `reviews` field of the `after` reading rather than from this line. The expected slug is `workflow-test-agent`, and a mismatch here is reported rather than smoothed over even when the reviews field reads HELD.
7. Read both pull requests again with the instrument. This is the `after` reading. Apply the `UNKNOWN` procedure above to either arm that returns it.
8. Classify both controls first, then the experiment, against the outcome table below.
9. Assemble the record. Take the run-side fields from the run's job log, which carries the run id, URL, acting identity, dispatch input and UTC timestamp. Take the readings from steps 3 and 7 and the apparatus from step 2. Fill `Actor` with the login that dispatched the run and `Residue` with what this run actually left behind, which can differ from the teardown's intent when a run failed partway. Hold the assembled record until it is landed in a batch, per `## Landing a record` below. A run never writes `RESULTS.md`.
10. Run the teardown.

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

Never land a record by having the test App approve the landing pull request. The standing rule and
its reasoning are in `scenarios/README.md`.

A record that has not landed yet is not lost, but it is held in one person's scratch rather than in
the repository, and the run log it was derived from is deleted after 90 days. Land batches before
that window closes.

## Outcome table

Exactly one row per run.

| Row | Selected when | Verdict | Meaning |
|---|---|---|---|
| `R1` | Every control HELD, and `experiment` reads `reviewDecision: APPROVED` with `mergeStateStatus: CLEAN` | `VALID` | A review from a GitHub App installation token satisfies the one-approval rule. App approvals count. |
| `R2` | Every control HELD, and `experiment` reads `reviewDecision: REVIEW_REQUIRED` with `mergeStateStatus: BLOCKED` | `VALID` | The App review does not satisfy the rule. App approvals do not count. |
| `R3` | Every control HELD, and `experiment` reads `reviewDecision: APPROVED` with `mergeStateStatus: BLOCKED` | `VALID` | The App review counts toward the review requirement, and something other than that requirement still blocks the merge. This reading does not by itself identify what else is blocking. Go next to the recorded `reviews` field and to the protection JSON under `Preconditions` in the run's record. |
| `RC` | Any control read its `refuted` set | `REFUTED` | The apparatus was not in the assumed state. The run measured nothing about the question. Read the `Controls` block to see which control refuted and why, then fix the setup and run again. |
| `RI` | No control read its `refuted` set, and either a control read its `indeterminate` set, or the `experiment` reading matches no substantive row, or two substantive rows match | `INDETERMINATE` | The readings cannot be classified. Re-read without changing anything. Where two rows matched, name both in `Notes`, and the table itself needs a fix. |
| `RX` | Preflight failed, or the job errored or was cancelled before the review was submitted. No arm produced a reading, so this row is selected and no other row is evaluated | `NOT-RUN` | The instrument never ran. Record the failure verbatim. |

`RC`, `RI`, and `RX` are reserved ids and are never used for a substantive outcome.

Both `RX` and `RC` exist because each catches a failure that looks like a finding. Without `RX`, a
workflow that dies before submitting the review leaves PR A reading `REVIEW_REQUIRED` and `BLOCKED`,
which is exactly row `R2`, and this repository would record a confident wrong answer. Without `RC`,
a run taken while protection is off leaves PR A reading `CLEAN`, which is exactly row `R1`, and that
is the answer a reader most wants to accept. Both failures produce well-formed readings inside the
expected value set, so nothing about them looks anomalous at classification time.

## Teardown

Creates: branches `test/app-approval-a-<RUN>` and `test/app-approval-b-<RUN>`, and their two pull requests.

Restores: nothing that a later run depends on. Close both pull requests without merging:

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

Deliberately left: branch protection on `main` stays enabled, because later scenarios assume it as
part of the repository baseline. The minted installation token is revoked by the action in its own
post step and needs no teardown.

This scenario does not change `enforce_admins`, and a later scenario that does must restore it to
`false`.

## Notes on the workflow file

`preflight` and `measure` are separate jobs rather than steps of one job. When a needed job fails,
its dependents report as skipped rather than failed, so a run that never set up reads differently in
the run's own job data from a run whose measurement failed. That distinction is the `RX` row, taken
from structure instead of from parsing a log.

`measure` declares `permissions: contents: read` and nothing else. That is the point of the
experiment: with that permission set the default `GITHUB_TOKEN` cannot submit a review, so a review
that does land can only have come from the minted App token.

Preflight tests secret presence through the expressions `${{ secrets.TEST_APP_ID != '' }}` and
`${{ secrets.TEST_APP_PRIVATE_KEY != '' }}`, which put a boolean into the step environment. Putting
the secret itself into an environment variable and testing that would answer the same question. This
form never places the private key in the runner environment at all, which is worth the slightly odd
shape on a public repository where log redaction is best-effort and degrades on multi-line
structured data.

The input guard tests `pr_number` with a parameter expansion rather than a `grep` pipeline. `grep -q`
succeeds when any single line matches, and an environment value can carry newlines, so a pipeline
accepts a multi-line input whose first or last line happens to be an integer. Stripping every digit
and testing what is left rejects that, because a newline survives the strip. The guard was run
against a bare integer, an integer with a leading zero, a non-numeric string, an integer followed by
a shell metacharacter and a second command, a command substitution, a two-line value opening with an
integer, and a two-line value closing with one. It accepts only the bare positive integer.

One trap for whoever re-runs that verification. Command substitution strips trailing newlines, so
building a test value with `$(printf ...)` silently removes the very newline the trailing-newline
case exists to exercise, and the guard then reads as accepting a value it would in fact reject.
Build multi-line and trailing-newline cases with a `$'...'` literal instead.

The action is pinned to a commit sha and carries no version comment. The sha is the anchor, and a
comment naming a version is a second claim that drifts from it. `bcd2ba49218906704ab6c1aa796996da409d3eb1`
is the commit that the tag `v3.2.0` points at, and `app-slug` is one of the three outputs that
action declares at that sha, alongside `token` and `installation-id`. Both were read from the GitHub
API on 2026-09-15.

At that sha the action's `app-id` input carries a deprecation notice pointing at `client-id`.
`app-id` still works and is what this scenario uses, because `TEST_APP_ID` holds the numeric app id.
An edit that switches to `client-id` has to change the secret's contents as well as the input name.
