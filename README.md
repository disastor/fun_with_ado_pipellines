# .cloudbees/workflows

Three CloudBees Unify workflows, all built around the same 
action (`ghcr.io/shozab-unify-examples/cloudbees-ado-actions:v1`).
Each workflow is manually triggered (`workflow_dispatch`).

## Shared prerequisites

Every workflow here needs these configured on the component (or inherited
from the org level) before any of them will run:

| Name | Type | Purpose |
|---|---|---|
| `vars.ADO_ORGANIZATION` | var | Azure DevOps organization name |
| `vars.ADO_PROJECT` | var | Azure DevOps project name |
| `secrets.ADO_PAT` | secret | Azure DevOps PAT (Build: Read & execute, Work Items: Read) |

---

## `ado-pipeline.yaml` — Run Azure DevOps Pipeline

The simplest of the three. Triggers a specific Azure Pipelines run and waits
for it to finish.

**Inputs (typed in manually at run time):**
- `pipeline_id` — the numeric Azure DevOps pipeline ID (free text, no
  validation — you need to already know the number)
- `branch` — full git ref, defaults to `refs/heads/main`

**What it does:**
1. Calls the action in `pipeline` mode, passing org/project/PAT plus the
   pipeline ID and branch straight through.
2. Waits for completion (`ADO_WAIT_FOR_COMPLETION: true`), polling every 10
   seconds, up to a 30-minute timeout (`ADO_WAIT_TIMEOUT_SECONDS: 1800`).
3. Fails the step if the ADO run doesn't succeed
   (`ADO_FAIL_ON_UNSUCCESSFUL: true`).
4. Prints the result and run URL to the log.

**Use this when:** you already know the pipeline ID and don't need a
friendlier picker — quickest path to "just run pipeline X."

---

## `ado-pipeline-choice.yaml` — Run Azure DevOps Pipeline (curated picker)
### presenting solely as an option to show another way to get the pipeline started

Same end result as `ado-pipeline.yaml`, but nobody has to know or type a raw
pipeline ID.

**Inputs:**
- `pipeline` — a dropdown (`type: choice`) with three curated options:
  `team-a-build`, `team-a-deploy`, `team-b-build`
- `branch` — same as above

**What it does differently:**
1. An extra first step (`resolve-pipeline`) runs a small `case` statement
   that maps the chosen friendly name to its real numeric pipeline ID
   (`team-a-build` → `1`, `team-a-deploy` → `42`, `team-b-build` → `7`).
2. Everything after that is identical to `ado-pipeline.yaml` — the resolved
   ID is fed into the same action, same wait/fail/timeout behavior.

**Use this when:** other people who don't know (or shouldn't need to know)
numeric pipeline IDs need to trigger a run themselves.

**Maintenance note:** adding a new pipeline means editing this file in two
places — the `options:` list *and* the `case` statement — and keeping the
spelling exactly in sync between them (and the `default:`, if that one
changes too).

---

## `ado-work-item.yaml` — Test Azure DevOps Work Item Action

A different job entirely — this one doesn't touch pipelines at all. It
queries Azure Boards directly and produces a formatted evidence report.

**Inputs:** none — it's a fixed query, not parameterized.

**What it does:**
1. Calls the same action, but in `work-item` mode via a WIQL query
   (`ADO_OPERATION: query-wiql`) that pulls the **5 most recently changed
   work items** in the whole project (`ADO_TOP: 5`), across every type,
   state, and assignee — not scoped to anything specific like a tag or
   sprint.
2. A second step spins up a plain `python:3.12-slim` container and
   reformats the raw JSON response into two markdown tables:
   - **Work Item Summary** — ID (linked), type, title, state, assignee,
     last-changed date
   - **Work Item Details** — area path, iteration path, tags, created
     by/date, changed by, and a plain-text version of the description
     (HTML stripped, truncated to 300 characters)
3. Publishes that markdown as a formal evidence item on the run
   (`cloudbees-io/publish-evidence-item@v1`), so it shows up as a readable
   report rather than raw log output.

**Use this when:** you want a quick "what's recently moved on the board"
snapshot surfaced inside a Unify run — more of a reporting/visibility tool
than an automation trigger.

**Worth knowing before relying on this one:**
- It always returns the 5 *most recently changed* items project-wide, with
  no filter — if you want it scoped (e.g. only items tagged for a specific
  release, or only ones in a specific state), that WIQL query needs editing
  directly in this file.
- Descriptions get published as evidence readable by anyone who can view the
  run. If any work items in this project contain sensitive details in their
  description field, that content is now surfaced outside Azure Boards —
  worth checking who has access to Unify run evidence before pointing this
  at a project with anything sensitive in ticket descriptions.
