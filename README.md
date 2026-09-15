# agent-workflow-tests

A test surface for questions about GitHub agent workflows. Each question is one scenario: a workflow
that performs a single operation, a runbook saying how to run it and how to read it, and an
append-only file holding what every run actually returned.

The repository is public and holds real credentials in repository secrets. Read the safety rules
below before adding anything.

## Scenarios

| Id | Question | Added |
|---|---|---|
| [`app-approval-vs-protection`](scenarios/app-approval-vs-protection/RUNBOOK.md) | Does a review submitted by a GitHub App installation token satisfy a branch protection rule requiring one approval? | 2026-09-15 |

## Layout

| Path | Job |
|---|---|
| `README.md` | What the repository is, the standing conventions, the scenario index, the safety rules |
| `scenarios/README.md` | The copyable skeletons and the checklist for adding a scenario |
| `scenarios/<id>/RUNBOOK.md` | One scenario's question, prerequisites, arms, commands, outcome table, teardown |
| `scenarios/<id>/RESULTS.md` | One scenario's run records, append-only |
| `.github/workflows/<id>.yml` | One scenario's instrument |

A fact is stated in one of these files and pointed at from the others, with one deliberate
exception. The standing conventions are here, so a runbook never restates the control rule.
`RESULTS.md` is the exception and repeats the verdict vocabulary and the queries on purpose, because
it is copied wholesale into every new scenario and has to stand alone once it lands there.

## The convention

**Words.** The *instrument* is the exact read command whose output is the measurement, and every arm
is read with it. An *arm* is one configured subject the instrument is pointed at: an experiment arm
answers the question, a control arm proves the *apparatus*, which is the state of the world the
scenario assumes but does not measure. A *reading* is one arm's verbatim output at one moment, and a
*record* is one entry in a `RESULTS.md` covering exactly one workflow run.

**Ids.** A scenario id is lowercase kebab-case ASCII matching `^[a-z][a-z0-9]*(-[a-z0-9]+)*$`,
3 to 40 characters, naming the variable under test and never the expected answer. `README` and any
token starting with `_` or `.` are reserved. One token is the directory name, the workflow file
stem, and the cross-reference key in every record, which is why the grammar is narrow. An id is
frozen once its directory has a commit on `main`, because the dispatch key, badge URLs, historical
runs, and every existing record key on it. A scenario whose id turns out to be wrong is superseded
by a new id rather than renamed, and the old runbook gains one line naming the successor.

**Triggers.** Every scenario workflow is `workflow_dispatch` only, with no push trigger and no
`pull_request` trigger, so a scenario never fires by accident. Dispatching needs write access, so on
a public repository this also stops an outside party spending the repository's credentials.

**Control arms.** A control arm is a declared, falsifiable claim about the apparatus, read with the
same instrument as the experiment arm. Its declaration partitions the instrument's whole value space
into three named sets, `held`, `refuted`, and `indeterminate`, with `indeterminate` always written
as the complement instead of as a list, so a value GitHub adds later cannot fall through all three.
Each control also declares `would-fail-if`, a concrete condition under which it reads refuted. A
control with no such condition proves nothing and is not a control.

Two controls may share a subject and a reading. One instrument call then serves both arms, and they
differ in what each asserts about the same output, which is cheaper than a second call and removes
any chance of the two arms reading a subject that changed between them.

**A scenario with a control arm is invalid when the control does not read as expected.** The
experiment arm's readings are then not interpreted, not reported as a finding, and not recorded as
any outcome row other than `RC` or `RI`.

**Outcome tables.** Substantive rows are `R1`, `R2`, and so on. Three reserved ids appear in every
table: `RC` when a control read its refuted set, `RI` when no control refuted and either a control
read its indeterminate set or a reading falls outside every declared set or two substantive rows
match, and `RX` when the instrument never ran. Row ids are never
renumbered and never reused for a different condition, because records cite them by id.

## Verdict vocabulary

| Token | Meaning |
|---|---|
| `VALID` | Every control read HELD and every arm produced a reading. The measurement counts. |
| `REFUTED` | At least one control read REFUTED. The apparatus was not in the assumed state. |
| `INDETERMINATE` | No control read REFUTED, and either a control read its indeterminate set or a reading fell outside every declared set. |
| `NOT-RUN` | The workflow never reached the measurement. Preflight failed, the job errored first, or the run was cancelled. |

A run that proves nothing is called an **invalid** run. That word covers the three tokens other than
`VALID` and is not itself a verdict token. The string `INVALID` is deliberately absent: `VALID` is a
substring of it, so the query below would silently return every bad run alongside the good ones.

## Running a scenario

1. Read the scenario's `RUNBOOK.md` end to end and satisfy every entry in its `## Prerequisites` section. Each entry carries a read-only `check` that never requires the credential it is checking for, written as a command where one exists and as a settings page where none does.
2. Dispatch it with `gh workflow run <id>.yml --repo patrickg-unity/agent-workflow-tests -f <input>=<value>`.
3. Wait for the run to finish, then keep its id, URL, and conclusion from `gh run list --repo patrickg-unity/agent-workflow-tests --workflow <id>.yml --limit 1 --json databaseId,conclusion,url`.
4. Read every arm with the scenario's instrument, written exactly as the runbook writes it, and keep the output verbatim.
5. Classify against the runbook's outcome table. Exactly one row is selected. Where no row matches, the answer is `RI` and never the nearest row.
6. Append a record to the scenario's `RESULTS.md`, then run the teardown the runbook names.

A new scenario's workflow has to be on `main` before it can be dispatched at all. GitHub documents
that once a workflow has run at least once, it can then be dispatched against any branch or tag
through the API or the CLI, which is how a scenario gets tested on a feature branch. That is written
as what becomes available after the first run rather than as a bar on dispatching before it, so this
file claims no more than that. Source:
`content/actions/reference/workflows-and-actions/events-that-trigger-workflows.md` in `github/docs`,
read 2026-09-15. Land the workflow on `main` early either way. With `workflow_dispatch` as its only
trigger, an unfinished workflow sitting on `main` is inert.

## Recording a result

Records are appended at the end of a `RESULTS.md`, oldest first, one per workflow run. That ordering
is the only one whose write cannot touch a byte already in the file.

Every workflow run owes exactly one record, whatever its verdict. A run that failed preflight earns
a `NOT-RUN` record. The failures are most of what a runbook is for, so a file holding only the
successes teaches the next person nothing.

Find the most recent valid run with:

```
grep -n '^## .* VALID$' scenarios/<id>/RESULTS.md | tail -1
```

The `$` anchor is why a record's verdict is the last token on its heading line. Substituting
`REFUTED`, `INDETERMINATE`, or `NOT-RUN` answers the matching question.

A record is never edited to change what it says. A correction is a new record carrying
`Amends: <timestamp>`. One in-place edit is permitted and no other: appending the literal token
` SUPERSEDED` to the corrected record's heading, which takes that line out of the query above.

## Baseline

The standing state of this repository. A scenario that changes any of it says so in its
`## Teardown` section and restores it. The protection entries are established at prerequisite P4 of
`app-approval-vs-protection` and are not in force before it, because branch protection cannot be set
on `main` until a commit creates the branch.

- `github.com/patrickg-unity/agent-workflow-tests`, public, default branch `main`.
- Branch protection on `main` requiring one approving review, with `enforce_admins: false`, set by `app-approval-vs-protection` at prerequisite P4 and deliberately left on afterwards.
- No required status checks and no push restrictions, set by that same call.
- `patrickg-unity` holds admin and is the only human.

`enforce_admins: false` is a standing invariant of this repository, not a parameter of any one
scenario. The chain that makes it so:

- A `workflow_dispatch` workflow must be on `main` to be dispatchable at all.
- `main` requires one approving review to merge.
- A pull request author cannot approve their own pull request.
- There is one human here, so nobody else can supply that review.

The admin bypass is therefore the only route by which any future scenario lands. Setting
`enforce_admins: true` would make this surface unextendable by one person, and it would present as a
merge refusal with no stated cause instead of as a policy change.

## Safety on a public repository

1. Treat every workflow log line as published. Log redaction is best-effort by GitHub's own statement and degrades on structured data, which a PEM private key is.
2. A credential lives in a repository secret and nowhere else. Not in a file, not in a workflow expression default, not in a runbook, not in a message.
3. Every credential here is dedicated to this repository and carries the least permission its scenario needs. Anyone with write access to a repository can read all of its secrets, so the secret set is sized to what one throwaway public repository can afford to lose.
4. If a credential reaches a log, delete that run's logs and rotate the credential before anything else, then record it under `Notes` in the scenario's `RESULTS.md`.

Never committed: a private key, a credential file, an environment file, any credential fixture, or
any test double holding a real value. Two mechanical guards stand behind that rule. `.gitignore`
covers the key and environment file patterns, and secret scanning push protection is enabled on this
repository, which refuses a push carrying a recognized credential rather than reporting it after it
lands. Measured on 2026-09-15 with
`gh api repos/patrickg-unity/agent-workflow-tests --jq '.security_and_analysis'`, which reported
`secret_scanning` enabled and `secret_scanning_push_protection` enabled. Neither guard recognizes
every secret shape, so the rule stands on its own and the guards are the backstop.

No sample value and no redacted-looking sample either. State the shape in words, as in "the numeric
app id" or "a PKCS8 PEM block", because a value-shaped example teaches the next reader that a value
belongs at that spot and they fill it in with a real one. A credential committed here is not
remediated by a later commit: git history is public and permanent, so the credential is rotated, the
exposure is recorded under `Notes`, and removing it from history is a separate decision.

## Adding a scenario

Copy the three skeletons in [scenarios/README.md](scenarios/README.md) and work the landing
checklist there.

One thing about landing is not obvious. Adding or updating a file under `.github/workflows/` through
the GitHub Contents API needs the `workflow` OAuth scope, and the `gh` token in use on the machine
this repository was built from carries `admin:public_key`, `gist`, `read:org`, and `repo` without
it. Land workflow files with `git push` over SSH, which is how they are landed anyway and involves
no OAuth token. A push over HTTPS refused with a scope message is this, and not a permissions
problem on the repository.
Arm B marker.
