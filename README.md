# GitHub Actions — reusable-workflow mechanics probe

A small set of **live, executable probes** answering questions about
`workflow_call` (reusable workflows), dynamic `runs-on`, `workflow_dispatch`
inputs, and `concurrency` — the ones GitHub's documentation either leaves
implicit or only demonstrates for a different trigger.

Every claim below was produced by dispatching the workflow in this repo and
reading the result. Nothing here is recalled from memory or inferred from
documentation prose.

> **Stamp:** probes executed **2026-07-31** against github.com (hosted
> `ubuntu-latest`, runner image build 2026-07-07). GitHub Actions behaviour
> changes; re-run before trusting any of this. Reproduction is one command per
> probe — see [Reproducing](#reproducing).

---

## Table of contents

- [Why this exists](#why-this-exists)
- [Results at a glance](#results-at-a-glance)
- [Reproducing](#reproducing)
- [The probes](#the-probes)
  - [01 — Can `runs-on` read `inputs` under `workflow_call`?](#01--can-runs-on-read-inputs-under-workflow_call)
  - [02 — Does the `[self-hosted, "<expr>"]` array form parse?](#02--does-the-self-hosted-expr-array-form-parse)
  - [03 — Is a `type: choice` input legal with no `default`?](#03--is-a-type-choice-input-legal-with-no-default)
  - [04 — Can workflow-level `concurrency` read `inputs`?](#04--can-workflow-level-concurrency-read-inputs)
  - [05 — Is job-level `concurrency` allowed on a `uses:` job?](#05--is-job-level-concurrency-allowed-on-a-uses-job)
  - [99 — The flow-mapping collision (deliberately broken)](#99--the-flow-mapping-collision-deliberately-broken)
- [Reading the signals](#reading-the-signals)
- [File map](#file-map)
- [What this repo is not](#what-this-repo-is-not)

---

## Why this exists

Reusable workflows are the sanctioned way to share one pipeline across several
targets without duplicating it. The moment you try to build that, you hit
questions the docs don't answer head-on:

- The documented example of a dynamic `runs-on` uses `workflow_dispatch`
  inputs. Does the same work from `workflow_call`? If not, a reusable workflow
  can't choose its own runner and the whole shape collapses.
- Can a manual trigger be made *mandatory* — no default, no accidental
  invocation — for something you should never fire by reflex?
- Where does `concurrency` belong when the real work lives in a called
  workflow?

Each probe isolates exactly one of those so a single parse failure can't mask
the others.

## Results at a glance

| # | Question | Answer |
|---|---|---|
| 01 | `runs-on` reads the `inputs` context under `workflow_call` | ✅ **Yes** |
| 01 | Local `uses: ./.github/workflows/<file>.yml` resolves | ✅ **Yes** |
| 01 | `github` context inside a called workflow | ⚠️ **The caller's** (`github.workflow` = caller; `github.job` = called job's id) |
| 01 | `github.token` usable inside a called workflow | ✅ **Yes**, including as an HTTP auth header for `git` |
| 02 | `runs-on: [self-hosted, "${{ inputs.x }}"]` under `workflow_call` | ✅ **Parses and dispatches** |
| 03 | `type: choice` with `required: true` and **no** `default` | ✅ **Legal, and genuinely mandatory** — omitting it is an API error |
| 04 | Workflow-level `concurrency.group` reads `inputs` | ✅ **Yes** (block style) |
| 05 | Job-level `concurrency` on a job whose body is `uses:` | ✅ **Yes**, expressions included |
| 99 | `${{ … }}` inside a YAML **flow mapping** `{ … }` | ❌ **Never parses**, and fails with a misleading error |

## Reproducing

```bash
R=<owner>/<this-repo>

gh workflow run 01-runs-on-from-workflow-call.yml   --repo $R
gh workflow run 02-self-hosted-label-expression.yml --repo $R   # will sit QUEUED — cancel it
gh workflow run 03-choice-input-no-default.yml      --repo $R -f box=bravo
gh workflow run 03-choice-input-no-default.yml      --repo $R   # expect HTTP 422
gh workflow run 04-concurrency-workflow-level.yml   --repo $R
gh workflow run 05-concurrency-job-level.yml        --repo $R
gh workflow run 99-BROKEN-flow-mapping-collision.yml --repo $R  # expect HTTP 422

gh workflow list --repo $R      # the fastest parse check — see "Reading the signals"
```

Workflows must be on the **default branch** to be dispatchable at all; a
`workflow_dispatch` workflow living only on a feature branch returns
`HTTP 404: not found on the default branch`.

---

## The probes

### 01 — Can `runs-on` read `inputs` under `workflow_call`?

**Files:** [`01-runs-on-from-workflow-call.yml`](.github/workflows/01-runs-on-from-workflow-call.yml) → [`_reusable.yml`](.github/workflows/_reusable.yml)

The reusable workflow declares a `runner_label` input and uses it directly:

```yaml
on:
  workflow_call:
    inputs:
      runner_label:
        required: true
        type: string
jobs:
  probe:
    runs-on: "${{ inputs.runner_label }}"
```

**Result: ✅ works.** The caller passed `ubuntu-latest`, the job ran on it, and
finished in 3 seconds.

This is the load-bearing one. It means a *single* reusable workflow can hold a
pipeline while each caller decides which machine runs it — you don't need
per-target copies of the pipeline, and you don't need conditionals to pick a
runner.

Three other things fall out of the same run:

**Local `uses:` resolves.** `uses: ./.github/workflows/_reusable.yml` works for
same-repository calls, at the caller's commit — no `owner/repo@ref` needed.

**The `github` context belongs to the caller.** Observed inside the *called*
workflow:

```
github.workflow     = 01-runs-on-from-workflow-call   <- the CALLER's name
github.job          = probe                           <- the CALLED workflow's job id
github.event_name   = workflow_dispatch
github.sha          = 9dbe6c45…
github.ref          = refs/heads/main
```

Worth internalising: `github.workflow` is the caller's, but `github.job` is the
called workflow's job id. If you pin anything to `github.sha` — say, to force a
job to operate on the exact commit that supplied the workflow — that still
resolves correctly through the call.

**`github.token` survives the call, and works as real credentials.** The probe
doesn't just check the token is non-empty; it uses it:

```bash
AUTH=$(printf 'x-access-token:%s' "$GH_TOKEN" | base64 | tr -d '\n')
git -c http.https://github.com/.extraheader="AUTHORIZATION: basic $AUTH" \
    ls-remote "https://github.com/${{ github.repository }}.git" HEAD
```

That succeeded. So the ephemeral-token pattern — authenticating `git` against a
private repo with no stored PAT and no deploy key on the machine — keeps
working after a refactor into a reusable workflow.

---

### 02 — Does the `[self-hosted, "<expr>"]` array form parse?

**Files:** [`02-self-hosted-label-expression.yml`](.github/workflows/02-self-hosted-label-expression.yml) → [`_reusable-selfhosted.yml`](.github/workflows/_reusable-selfhosted.yml)

```yaml
runs-on: [self-hosted, "${{ inputs.runner_label }}"]
```

**Result: ✅ parses and dispatches.** No runner in this repo carries the label,
so the job sits in `queued` forever — which *is* the pass signal. Compare:

| Observed | Means |
|---|---|
| Run created, job `queued` | Syntax accepted; no runner matches the labels |
| Dispatch returns `HTTP 422` | The file didn't parse (see probe 99) |

Remember to cancel it: `gh run cancel <id> --repo $R`.

A job listing multiple labels only runs on a runner carrying **every** label,
so `[self-hosted, alpha]` and `[self-hosted, bravo]` can't poach each other's
jobs. A bare `runs-on: self-hosted`, by contrast, matches *any* self-hosted
runner — worth avoiding once more than one exists.

---

### 03 — Is a `type: choice` input legal with no `default`?

**File:** [`03-choice-input-no-default.yml`](.github/workflows/03-choice-input-no-default.yml)

```yaml
on:
  workflow_dispatch:
    inputs:
      box:
        required: true
        type: choice
        options:
          - alpha
          - bravo
        # deliberately NO default
```

**Result: ✅ legal, and genuinely enforced.**

```
$ gh workflow run 03-choice-input-no-default.yml --repo $R -f box=bravo
https://github.com/.../actions/runs/30615422981          # runs

$ gh workflow run 03-choice-input-no-default.yml --repo $R
HTTP 422: Required input 'box' not provided              # refused
```

Useful whenever there is no *safe* value to fall back to. `type: choice` also
constrains the value to the option list, so an expression consuming it — a
`runs-on` label, an environment name, a target identifier — can't be fed
arbitrary text by whoever clicks the button.

---

### 04 — Can workflow-level `concurrency` read `inputs`?

**File:** [`04-concurrency-workflow-level.yml`](.github/workflows/04-concurrency-workflow-level.yml)

```yaml
concurrency:
  group: probe-04-${{ inputs.box }}
  cancel-in-progress: false
```

**Result: ✅ works** — provided it's written in **block style**. Probe 99 is the
same construct in flow style, and it never parses.

So one workflow can serialise runs *per target* rather than globally: two
different targets proceed in parallel, two runs against the same target queue.

---

### 05 — Is job-level `concurrency` allowed on a `uses:` job?

**File:** [`05-concurrency-job-level.yml`](.github/workflows/05-concurrency-job-level.yml)

A job that calls a reusable workflow accepts only a restricted set of keywords,
so this needed checking rather than assuming.

```yaml
jobs:
  call:
    concurrency:
      group: probe-05-${{ inputs.box }}
      cancel-in-progress: false
    uses: ./.github/workflows/_reusable.yml
    with: …
```

**Result: ✅ supported, expressions included.**

One caveat straight from GitHub's docs, worth repeating because it is easy to
trip over: a called workflow **inherits the caller's workflow name**, so using
the workflow context as a concurrency group identifier in *both* caller and
called workflow — with `cancel-in-progress: true` — can make the called
workflow cancel its own caller. Keep the groups distinct, or keep
`cancel-in-progress: false`.

---

### 99 — The flow-mapping collision (deliberately broken)

**File:** [`99-BROKEN-flow-mapping-collision.yml`](.github/workflows/99-BROKEN-flow-mapping-collision.yml) — **do not fix it, it's the exhibit**

This is the most useful thing in the repo, because the failure mode actively
lies to you.

```yaml
# BROKEN — the {{ }} collides with the flow mapping's braces
concurrency: {group: probe-99-${{ inputs.box }}, cancel-in-progress: false}

# FINE — same expression, block style
concurrency:
  group: probe-99-${{ inputs.box }}
  cancel-in-progress: false
```

In YAML **flow mappings**, `{` and `}` are structural. An embedded `${{ … }}`
brings its own braces, the mapping's structure breaks, and the file never
parses.

The reason this costs real time is the diagnosis:

```
$ gh workflow run 99-BROKEN-flow-mapping-collision.yml --repo $R
HTTP 422: Workflow does not have 'workflow_dispatch' trigger
```

The trigger is plainly there in the file. The error is a downstream symptom of
a document that never parsed, and it points you at the wrong line entirely.
There is no error annotation anywhere in the Actions UI, because there is no
run to annotate.

Flow style is fine for expression-free YAML — `{required: true, type: string}`
parses happily. It's only the `${{ … }}` that collides.

---

## Reading the signals

`gh workflow list` is the fastest parse check available, and it's not
documented as one:

```
01-runs-on-from-workflow-call                              active   324307191   <- parsed: shows `name:`
.github/workflows/99-BROKEN-flow-mapping-collision.yml     active   324307202   <- FAILED: shows PATH
```

A workflow GitHub could parse is listed by its `name:` field. One it couldn't
is listed by **file path** — because `name:` was never readable. Both show
`active`, so state is no help.

Summary of the three signals worth knowing:

| Signal | Meaning |
|---|---|
| `gh workflow list` shows a **path** instead of a name | The file failed to parse |
| Dispatch → `HTTP 422: Workflow does not have 'workflow_dispatch' trigger` | Almost always a parse failure, not a missing trigger |
| Dispatch → `HTTP 404: not found on the default branch` | The workflow isn't on the default branch |
| Dispatch → `HTTP 422: Required input '<x>' not provided` | A `required: true` input with no default — working as intended |
| Run created, job stays `queued` | Syntax fine; no runner matches the labels |

## File map

```
.github/workflows/
  _reusable.yml                          shared reusable workflow (all probes call it)
  _reusable-selfhosted.yml               same, but targets a self-hosted label array
  01-runs-on-from-workflow-call.yml      runs-on from inputs; local uses:; github ctx; token
  02-self-hosted-label-expression.yml    [self-hosted, "<expr>"] array form
  03-choice-input-no-default.yml         mandatory choice input with no default
  04-concurrency-workflow-level.yml      workflow-level concurrency reading inputs
  05-concurrency-job-level.yml           job-level concurrency on a uses: job
  99-BROKEN-flow-mapping-collision.yml   deliberate negative case — leave broken
```

## What this repo is not

Not a library, not a template, not maintained. It is a **dated record of
observed behaviour** — useful exactly as long as you treat the stamp at the top
as a shelf life and re-run the probes rather than trusting the table.
