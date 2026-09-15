# Adding a scenario

A scenario is three files at three paths plus one row in the [repository index](../README.md). Copy
the skeletons below, replace every `@@TOKEN@@`, then work the landing checklist at the bottom.

Read [../README.md](../README.md) first. It holds the conventions these skeletons assume: the id
grammar, the control-arm rule, the verdict vocabulary, the outcome-row ids, and the safety rules.

`_template` is a reserved id and is not a usable scenario id.

## Known gap: a scenario whose subject is a trigger cannot be built here

Every per-scenario workflow is `workflow_dispatch` only. A scenario that measures a `pull_request`
event, a push hook, a schedule, or a `workflow_run` chain therefore has nowhere to put the thing it
measures, because the rule forbids exactly the trigger under test. A label-trigger test is the
clearest case: its whole subject is a `pull_request: types: [labeled]` trigger.

This is a documented boundary rather than an oversight. Resolving it is an open question and nothing
in this repository depends on the answer. Do not work around it by adding a trigger to a scenario
workflow.

## Standing rule: a scenario never routes its own bookkeeping through its own subject

A test surface must not route its own bookkeeping through its own subject.

The concrete case that produced this rule: `app-approval-vs-protection` established that a GitHub
App installation token's review satisfies a branch protection rule requiring one approval. That
makes the test App a working approver, and a landing pull request needs an approval, so reaching for
the App to approve it is the obvious move and it is forbidden.

The reason is what the repository would lose. The App's approval power is the thing under test. A
record-landing path that depends on it converts any future change in that behavior from a test
result into a broken bookkeeping pipeline, and the repository loses the ability to record the very
finding that broke it. The result would be indistinguishable from the tooling being down.

This generalizes past approvals. Whatever a scenario measures, the machinery that records its results
does not depend on that thing working. Records land under the admin bypass, by a human, per
`RUNBOOK.md` `## Landing a record`.

## Skeleton 1, to `scenarios/@@ID@@/RUNBOOK.md`

````
# @@ID@@

## Question

@@THE QUESTION, AS ONE SENTENCE ENDING IN A QUESTION MARK@@

The variable under test is @@THE ONE THING THAT DIFFERS BETWEEN THE ARMS@@. Everything else is held
fixed: @@WHAT IS HELD FIXED@@.

## Subject

@@WHAT IS UNDER TEST. WRITE "GitHub itself" WHERE THERE IS NO EXTERNAL ARTIFACT, IN WHICH CASE EVERY
RECORD CARRIES Subject-pin: none. OTHERWISE NAME THE ARTIFACT AND THE FIELD THAT PINS ITS VERSION.@@

## Prerequisites

@@Write `None.` where there are none. Never omit this section.@@

P1
what: @@ONE IMPERATIVE LINE@@
who: @@operator | repository admin | any actor with write access@@
where: @@THE EXACT UI PATH OR THE EXACT API CALL@@
produces: @@THE OBSERVABLE ARTIFACT THIS ACT CREATES@@
lifetime: @@one-time | per-run | expires WHEN@@
check: @@A READ-ONLY COMMAND RETURNING A DEFINITE YES OR NO@@

@@A `check` never requires the credential it is checking for, and never performs the operation the
outcome table discriminates on. Where no credential-free command exists, the check is a settings
page and the entry says so.@@

## Credentials

@@Write `None.` where the scenario needs none.@@

name: @@THE EXACT REPOSITORY SECRET NAME@@
scope: @@repository | environment NAME@@
shape: @@A WORD DESCRIPTION, NEVER A SAMPLE@@
obtained-from: @@THE EXACT PLACE THE OPERATOR CREATES OR COPIES IT@@
least-privilege: @@THE MINIMUM PERMISSION SET THIS SCENARIO NEEDS@@
rotate-when: immediately, if this value is found in any workflow log or any commit. @@ANY OTHER CONDITION@@
existence-check: @@A READ-ONLY COMMAND PROVING IT IS SET, WITHOUT READING IT@@

## Instrument

```
@@THE EXACT READ COMMAND, IDENTICAL FOR EVERY ARM@@
```

@@EACH FIELD IT RETURNS, WITH THE COMPLETE SET OF VALUES THAT FIELD CAN TAKE AND WHETHER IT IS
NULLABLE. STATE THE FULL VALUE SPACE OF ONE ARM'S READING HERE, BECAUSE THE CONTROL SETS AND THE
OUTCOME ROWS ARE STATED AGAINST IT.@@

## Arms

| Arm | Subject | What it receives |
|---|---|---|
| `experiment` | @@WHAT IS POINTED AT@@ | @@THE OPERATION UNDER TEST@@ |
| @@CONTROL-ID@@ | @@WHAT IS POINTED AT@@ | @@NOTHING, OR THE DELIBERATELY DIFFERENT TREATMENT@@ |

## Control arms

@@Write `None.` plus one sentence naming the apparatus assumption you are choosing not to prove, and
why that is acceptable. Never leave this section out.@@

```
control-id: @@KEBAB-CASE TOKEN, UNIQUE WITHIN THE SCENARIO@@
asserts: @@ONE SENTENCE NAMING THE APPARATUS PROPERTY THIS ARM PROVES@@
instrument: @@THE EXACT READ COMMAND, IDENTICAL TO THE EXPERIMENT ARM'S@@
value-space: @@THE COMPLETE SET OF VALUES THE INSTRUMENT CAN RETURN@@
held: @@THE SUBSET MEANING THE APPARATUS IS IN THE ASSUMED STATE@@
refuted: @@THE SUBSET MEANING THE APPARATUS IS NOT IN THE ASSUMED STATE@@
indeterminate: every reading in value-space not listed in held or refuted.
would-fail-if: @@A CONCRETE, NAMEABLE CONDITION UNDER WHICH THIS ARM READS REFUTED@@
```

## Procedure

1. @@SATISFY EVERY PREREQUISITE AND RUN EACH check@@
2. @@READ EVERY ARM WITH THE INSTRUMENT. THIS IS THE before READING.@@
3. Dispatch it: `gh workflow run @@ID@@.yml --repo patrickg-unity/agent-workflow-tests -f @@INPUT_NAME@@=@@VALUE@@`
4. @@WAIT FOR THE RUN, RECORD ITS ID, URL, AND CONCLUSION@@
5. @@READ EVERY ARM AGAIN. THIS IS THE after READING.@@
6. @@CLASSIFY THE CONTROLS FIRST, THEN THE EXPERIMENT@@
7. @@APPEND THE RECORD TO RESULTS.md, THEN RUN THE TEARDOWN@@

## Outcome table

Exactly one row per run.

@@Substantive rows are stated against the value space above, and `RI` closes every case they and
`RC` leave, so the table is total by construction. Check that before landing.@@

| Row | Selected when | Verdict | Meaning |
|---|---|---|---|
| `R1` | @@EVERY CONTROL HELD, AND THE EXPERIMENT READS ...@@ | `VALID` | @@WHAT THAT ANSWERS@@ |
| `R2` | @@EVERY CONTROL HELD, AND THE EXPERIMENT READS ...@@ | `VALID` | @@WHAT THAT ANSWERS@@ |
| `RC` | Any control read its `refuted` set | `REFUTED` | The apparatus was not in the assumed state. Fix the setup and run again. |
| `RI` | No control read its `refuted` set, and either a control read its `indeterminate` set, or the `experiment` reading matches no substantive row, or two substantive rows match | `INDETERMINATE` | Re-read without changing anything. Where two rows matched, name both in `Notes`. |
| `RX` | Preflight failed, or the job errored or was cancelled before the instrument ran | `NOT-RUN` | Record the failure verbatim and do not classify. |

## Teardown

Creates and leaves in place: @@BRANCHES, PULL REQUESTS, LABELS, CHECK RUNS, VARIABLES, PROTECTION
CHANGES@@

Restores: @@WHAT, AND BY WHAT COMMAND@@

Deliberately left: @@WHAT, AND WHY@@

This scenario @@does | does not@@ change `enforce_admins`. Where it does, it restores it to `false`.

## Notes on the workflow file

@@THE DESIGNATED HOME FOR ANY WHY ABOUT THE WORKFLOW. THE ZERO-COMMENT RULE DISPLACES IT HERE, SO A
CHOICE THAT NEEDS EXPLAINING IS EXPLAINED IN THIS SECTION AND NEVER IN A YAML COMMENT.@@
````

The number of substantive rows is whatever the scenario's own value space needs, and the two in the
skeleton above are an illustration rather than a prescribed count. The live
[`app-approval-vs-protection`](app-approval-vs-protection/RUNBOOK.md) scenario carries three.

## Skeleton 2, to `scenarios/@@ID@@/RESULTS.md`

Copy [`app-approval-vs-protection/RESULTS.md`](app-approval-vs-protection/RESULTS.md) down to but
not including its `## Records` heading, then write that heading into the new file with `None.` under
it. Replace the scenario id everywhere it appears. Everything above `## Records` is the canonical
header: the verdict vocabulary, the query, the record format, the correction rule, and the
completeness audit.

The records below that heading are never copied. They are one scenario's evidence for runs that
happened, so a record carried into a new scenario would assert a run that scenario never had.

Four blocks inside its record format are specific to that scenario and are rewritten rather than
copied. Everything else in the file is standing convention and is kept as it is.

- `Subject-pin`, which reads `none` there because that scenario's subject is GitHub itself. A
  scenario measuring an external artifact names it and pins its version.
- `Preconditions`, whose `branch-protection-main` and `actions-can-approve-pull-request-reviews`
  lines are the apparatus that scenario assumes. Replace them with the apparatus yours assumes.
- `Arms`, which names that scenario's two pull requests.
- `Controls`, and the `acting-identity provenance` entry under `Readings`, which name that
  scenario's own control ids and its own provenance capture. Replace them with yours, one line per
  control you declared.

Do not write a record by hand from memory. The format block in that file names every field and the
order they appear in.

## Skeleton 3, to `.github/workflows/@@ID@@.yml`

````
name: @@ID@@

on:
  workflow_dispatch:
    inputs:
      @@INPUT_NAME@@:
        description: @@WHAT IT IS@@
        type: string
        required: true

jobs:
  preflight:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      @@DECLARE ONLY WHAT PREFLIGHT ACTUALLY READS. KEEP pull-requests: read ONLY IF A CHECK READS A PULL REQUEST, AND ADD NOTHING THE MEASURING JOB NEEDS.@@
    steps:
      - name: Assert the dispatch input is well formed
        env:
          INPUT_VALUE: ${{ inputs.@@INPUT_NAME@@ }}
        run: |
          set -euo pipefail
          if [ -z "$INPUT_VALUE" ] || [ -n "${INPUT_VALUE//[0-9]/}" ] || [ "${INPUT_VALUE:0:1}" = "0" ]
          then
            echo "Prerequisite @@PN@@ not satisfied: input @@INPUT_NAME@@ must be one line holding a positive integer with no leading zero. See scenarios/@@ID@@/RUNBOOK.md, section Prerequisites, entry @@PN@@."
            exit 1
          fi
          @@THE POSITIVE-INTEGER SHAPE ABOVE IS THE COMMON CASE. REPLACE THE TEST WHERE THE INPUT IS SOMETHING ELSE, AND KEEP A CONSTRUCTION THAT REJECTS A MULTI-LINE VALUE. A grep PIPELINE DOES NOT, BECAUSE grep MATCHES ANY SINGLE LINE OF A MULTI-LINE VALUE.@@

      - name: @@ASSERT ONE PREREQUISITE, NAMING ITS ENTRY ID IN THE FAILURE MESSAGE@@
        env:
          @@NAME@@: @@EXPRESSION@@
        run: |
          set -euo pipefail
          if @@THE CONDITION THAT MEANS THE PREREQUISITE IS NOT SATISFIED@@
          then
            echo "Prerequisite @@PN@@ not satisfied: @@THE ARTIFACT BY NAME, NEVER BY VALUE@@. See scenarios/@@ID@@/RUNBOOK.md, section Prerequisites, entry @@PN@@."
            exit 1
          fi

  measure:
    needs: preflight
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: @@THE OPERATION UNDER TEST@@
        env:
          @@THE CREDENTIAL, IN THIS STEP AND NEVER AT JOB OR WORKFLOW LEVEL@@
        run: |
          set -euo pipefail
          @@THE COMMAND@@
````

`preflight` is a separate job and not a first step, so that a setup failure shows in the run's own
job data as a skipped measurement instead of as a failed one. That distinction is the `RX` row, read
from structure rather than from a log. Preflight steps never carry `continue-on-error`, and a
preflight that can pass while failing is worse than none.

## Landing checklist

1. The id matches the grammar in the repository README and is not reserved.
2. The three files exist at the three paths, and the workflow file stem equals the directory name.
3. `grep -rn '@@' scenarios/<new-id>/ .github/workflows/<new-id>.yml` returns nothing.
4. The workflow's only trigger is `workflow_dispatch`.
5. The workflow has a `preflight` job and every measuring job declares `needs: preflight`.
6. Every prerequisite entry has a read-only `check` that never requires the credential it is checking for and never performs the operation the outcome table discriminates on. Where no credential-free command exists the check may be a settings page instead, and the entry says so.
7. Every control arm declares `value-space`, `held`, `refuted`, `indeterminate` as the complement, and `would-fail-if`.
8. The outcome table carries `RC`, `RI`, and `RX`, and its substantive rows plus `RC` are stated against the declared value space.
9. `## Teardown` names everything the scenario creates and what it restores.
10. A scenario that changes `enforce_admins` restores it to `false` in `## Teardown`. Leaving it `true` makes this repository unextendable by one person, for the reasons in the repository README's `## Baseline` section.
11. No credential value, sample value, or redacted-looking value appears anywhere in the three files.
12. `grep -rn 'app-approval-vs-protection' scenarios/<new-id>/ .github/workflows/<new-id>.yml` returns nothing, which catches an id the RESULTS.md copy above left behind.
13. `grep -cE '^## [0-9]{4}-[0-9]{2}-[0-9]{2}T' scenarios/<new-id>/RESULTS.md` returns 0, which catches a record copied in along with the header. Item 12 cannot catch one, because a record body carries no scenario id. This is the instrument `## Completeness` already uses to count records, so the checklist and that section cannot drift apart on what a record heading is.
14. No comment line appears in any of the three files, the workflow included. A why goes into the runbook prose.
15. The scenario has a row in the index table in the repository README.
