# iOPSPro GitHub Defaults

Organization-wide default GitHub configuration for iOPSPro repositories. This repository currently standardizes issue intake and reusable Project V2 automation for issues, testing sub-issues, iteration assignment, and environment assignment.

## What This Repository Provides

GitHub applies files from a special `.github` repository as organization defaults when a repository does not define its own local version. Repository-specific files still take precedence.

This repository currently contains:

- Issue forms in `.github/ISSUE_TEMPLATE/`
- Reusable workflows in `.github/workflows/`
- This repository-level README

It does not currently contain pull request templates, contribution guidelines, support files, or other community health files.

## Issue Forms

All issue forms add new issues to project `iOPSPro/5` and disable blank issues. The forms also include public-repository safety language reminding authors not to include secrets, credentials, private keys, private customer data, or unredacted logs.

| Form | File | Type | Default labels | Primary use |
| --- | --- | --- | --- | --- |
| Bug report | `.github/ISSUE_TEMPLATE/bug.yml` | `Bug` | `bug`, `needs-triage` | Defects, regressions, and broken behavior |
| Feature request | `.github/ISSUE_TEMPLATE/feature-request.yml` | `Feature` | `enhancement`, `needs-triage` | New capabilities and improvements |
| Incident | `.github/ISSUE_TEMPLATE/incident.yml` | `Bug` | `incident`, `high-priority`, `needs-triage`, `needs-testing` | Outages and urgent production-impacting issues |
| Testing | `.github/ISSUE_TEMPLATE/test.yml` | `Test` | `testing`, `needs-triage` | QA validation, regression checks, release verification, and testing child issues |
| Work item | `.github/ISSUE_TEMPLATE/work-item.yml` | `Task` | `work-item`, `needs-triage` | General tasks, migrations, platform work, documentation, governance, and chores |

Shared form fields:

- `Assign to current iteration?` appears on every form. A `Yes` answer is used by automation to add `assign-current-iteration`.
- `Is there a specific environment this is for?` appears on every form and maps to the Project V2 `Environment` field.
- Environment options are `V1`, `V2`, `Backend`, `N/A`, and `Both V1 and V2`.
- `Does this issue need testing?` appears on bug, feature, and work item forms. GitHub Issue Forms cannot conditionally apply labels from dropdown answers, so participating repositories need workflow automation to translate this intent into labels when required.

`config.yml` disables blank issues and provides a contact link for security vulnerabilities. Security vulnerabilities should not be filed as public issues.

## Label Contracts

The workflows use labels as simple signals between issue forms, per-repository caller workflows, and Project V2 updates.

| Label | Meaning |
| --- | --- |
| `needs-triage` | Issue needs initial review and routing. |
| `needs-testing` | Parent issue should have a testing child issue created or maintained. |
| `testing` | Issue is a testing or QA validation issue. |
| `testing-created` | Parent issue already had an automation-created testing child issue. Prevents duplicate child creation. |
| `assign-current-iteration` | Issue should be assigned to the current Project V2 iteration. Removed after successful iteration sync. |

## Project Field Contracts

The reusable workflows assume a GitHub Projects V2 project, defaulting to `iOPSPro/5` where defaults are provided.

Expected fields:

- `Iteration`: primary Project V2 iteration field.
- `60 Day Block`: secondary Project V2 iteration field. Some workflows skip this field if it is absent; the iteration sync workflow requires only `Iteration`.
- `Environment`: single-select field with options `V1`, `V2`, `Backend`, `N/A`, and `Both V1 and V2`.
- `Status`: single-select field used by the testing sub-issue status workflow. It must include an `In progress` option for that workflow to update items.

Workflows use GraphQL Project V2 APIs and require a token with access to read and write the relevant project fields. Caller repositories should pass that token as `project_token`, usually from `secrets.PROJECT_TOKEN`.

## Reusable Workflows

Workflows in this repository are reusable workflow scaffolds. They do not automatically run for every repository in the organization. Each participating repository needs a small caller workflow that invokes the relevant file with `uses: iOPSPro/.github/.github/workflows/<workflow>.yml@main`.

| Workflow | Purpose | Typical caller trigger |
| --- | --- | --- |
| `.github/workflows/apply-issue-form-iteration-label.yml` | Reads the `Assign to current iteration?` answer and adds `assign-current-iteration` when the answer is `Yes`. | `issues.opened`, `issues.edited` |
| `.github/workflows/sync-project-iteration-from-label.yml` | Finds the current iteration for `Iteration` and, when available, `60 Day Block`; sets those project fields; removes `assign-current-iteration` after success. | `issues.labeled` for `assign-current-iteration` |
| `.github/workflows/sync-project-application-version-from-form.yml` | Reads the environment answer from the issue form and sets the Project V2 `Environment` single-select field. | `issues.opened`, `issues.edited` |
| `.github/workflows/create-testing-subissue.yml` | Creates a `Testing: <parent title>` child issue for parents labeled `needs-testing`; copies parent assignees; attaches it as a sub-issue; adds/copies project fields; labels the parent `testing-created`; comments with the child issue number. | `issues.opened` or `issues.labeled`, gated by `needs-testing` |
| `.github/workflows/copy-parent-project-fields-to-test-child.yml` | For manually created testing child issues, finds the parent via the sub-issues API, copies parent assignees, adds the child to the project if needed, and copies `Iteration`, `60 Day Block`, and `Environment`. | `issues.opened`, `issues.edited`, `issues.labeled`, and/or `workflow_dispatch` |
| `.github/workflows/move-testing-subissues-to-testing.yml` | Finds sub-issues with `Testing:` in the title and moves their Project V2 `Status` to `In progress`. | Parent issue lifecycle events, commonly after parent issue closure or readiness transitions |

The workflow files include commented example caller workflows at the bottom where applicable.

## Testing Child Issue Model

There are two supported paths for testing child issues.

Automation-created path:

1. A parent issue receives `needs-testing`.
2. A caller workflow invokes `create-testing-subissue.yml`.
3. The reusable workflow checks that the parent has `needs-testing` and does not already have `testing-created`.
4. It creates a child issue titled `Testing: <parent title>` with label `testing` and type `Test`.
5. It attaches the child as a GitHub sub-issue, copies parent assignees, adds/copies Project V2 field values, labels the parent `testing-created`, and comments on the parent.

Manually created path:

1. A user creates an issue with the `Testing` form.
2. The issue is attached as a child issue of the parent.
3. A caller workflow invokes `copy-parent-project-fields-to-test-child.yml`.
4. The reusable workflow verifies the child has the `testing` label, finds the parent through the sub-issues API, and copies assignees plus project field values.

GitHub Actions does not expose a dedicated documented `issues` activity type for "issue became a sub-issue." For manually created testing children, caller repositories should trigger the copy workflow from nearby events such as `issues.opened`, `issues.edited`, `issues.labeled`, and/or `workflow_dispatch`.

## Recommended Per-Repository Setup

Participating repositories should add caller workflows for the pieces they need:

- Apply the issue-form iteration label on issue open/edit.
- Sync current iteration when `assign-current-iteration` is applied.
- Sync `Environment` from issue forms on issue open/edit.
- Create testing sub-issues when `needs-testing` is present.
- Copy parent fields for manually created testing children.
- Move testing sub-issues to `In progress` when the parent workflow needs that handoff.

Caller workflows should pass:

- `owner: ${{ github.repository_owner }}`
- `repo: ${{ github.event.repository.name }}`
- `issue_number` from the issue event or `workflow_dispatch` input
- `project_owner: iOPSPro`
- `project_number: 5`
- `project_token: ${{ secrets.PROJECT_TOKEN }}`

## Maintenance Notes

- Keep issue form question labels stable unless the corresponding workflow inputs are updated. The workflows parse rendered issue body headings such as `### Assign to current iteration?`.
- Keep Project V2 field names and single-select option names aligned with the defaults documented above.
- Keep only non-sensitive, organization-safe defaults in this repository.
- If repository-specific intake or automation differs from these defaults, define local templates or caller workflow inputs in that repository.

## Maintainers

Managed by the iOPSPro organization administrators.
