# iOPSPro GitHub Defaults

Organization-wide GitHub defaults for iOPSPro repositories.

GitHub treats a repository named `.github` as a source of default community health files, issue templates, and reusable workflows. Repositories can still override any default by defining their own local version of the same file.

## Contents

This repository provides:

- Issue forms in `.github/ISSUE_TEMPLATE/`
- Reusable GitHub Actions workflows in `.github/workflows/`
- Project V2 automation conventions for labels, iterations, environment fields, pull request review queues, and testing sub-issues

It does not currently provide pull request templates, contribution guidelines, code of conduct files, or support policy files.

## Issue Templates

Blank issues are disabled. All issue forms add new issues to Project `iOPSPro/5` and include safety language reminding users not to include secrets, credentials, private keys, private customer data, or unredacted logs.

| Template | File | Issue type | Default labels | Purpose |
| --- | --- | --- | --- | --- |
| Bug report | `.github/ISSUE_TEMPLATE/bug.yml` | `Bug` | `bug`, `needs-triage` | Defects, regressions, and broken behavior |
| Feature request | `.github/ISSUE_TEMPLATE/feature-request.yml` | `Feature` | `enhancement`, `needs-triage` | New capabilities and improvements |
| Incident | `.github/ISSUE_TEMPLATE/incident.yml` | `Bug` | `incident`, `high-priority`, `needs-triage`, `needs-testing` | Outages and urgent production-impacting issues |
| Testing | `.github/ISSUE_TEMPLATE/test.yml` | `Test` | `testing`, `needs-triage` | QA validation, regression checks, release verification, and testing child issues |
| Work item | `.github/ISSUE_TEMPLATE/work-item.yml` | `Task` | `work-item`, `needs-triage` | General tasks, migrations, platform work, documentation, governance, and chores |

Shared form fields:

- `Assign to current iteration?` is present on every form. A `Yes` answer is used by automation to add `assign-current-iteration`.
- `Is there a specific environment this is for?` is present on every form and maps to the Project V2 `Environment` field.
- Supported environment values are `V1`, `V2`, `Backend`, `N/A`, and `Both V1 and V2`.
- `Does this issue need testing?` is present on bug, feature, and work item forms. GitHub issue forms cannot apply labels conditionally from dropdown answers by themselves, so repositories need automation if they want this answer to add or remove `needs-testing`.

Security vulnerabilities should not be filed as public issues. The issue template configuration links users to GitHub's private vulnerability reporting documentation.

## Reusable Workflows

The workflows in `.github/workflows/` are reusable workflow building blocks. They do not run automatically in every repository. Participating repositories must add local caller workflows that invoke them with `uses: iOPSPro/.github/.github/workflows/<workflow>.yml@main`.

| Workflow | Purpose | Common trigger |
| --- | --- | --- |
| `.github/workflows/apply-issue-form-iteration-label.yml` | Reads `Assign to current iteration?` and adds `assign-current-iteration` when the answer is `Yes`. | `issues.opened`, `issues.edited` |
| `.github/workflows/sync-project-iteration-from-label.yml` | Finds the current Project V2 iteration, sets `Iteration` and `60 Day Block`, then removes `assign-current-iteration`. | `issues.labeled` for `assign-current-iteration` |
| `.github/workflows/sync-project-application-version-from-form.yml` | Reads the issue form environment answer and sets the Project V2 `Environment` field. | `issues.opened`, `issues.edited` |
| `.github/workflows/create-testing-subissue.yml` | Creates a `Testing: <parent title>` child issue for parent issues labeled `needs-testing`; copies assignees and project fields; marks the parent with `testing-created`. | `issues.opened`, `issues.labeled` gated by `needs-testing` |
| `.github/workflows/copy-parent-project-fields-to-test-child.yml` | For manually created testing child issues, finds the parent issue, copies assignees, adds the child to the project if needed, and copies project fields. | `issues.opened`, `issues.edited`, `issues.labeled`, or `workflow_dispatch` |
| `.github/workflows/move-testing-subissues-to-testing.yml` | Finds testing sub-issues and moves their Project V2 `Status` to `In progress`. | Parent issue lifecycle events |
| `.github/workflows/sync-pr-delivery-board-for-review.yml` | Requests review from the configured reviewer, optionally assigns the reviewer, adds the PR to the Delivery Board, sets current `Iteration` and `60 Day Block`, and sets `Status` to `In progress`. Defaults to `ready-for-dr-review` and `drichard1989`. | `pull_request.labeled` gated by `ready-for-dr-review` |

## Required Labels

These labels act as lightweight contracts between forms, caller workflows, and reusable workflows.

| Label | Meaning |
| --- | --- |
| `needs-triage` | Issue needs initial review and routing. |
| `needs-testing` | Parent issue should have a testing child issue created or maintained. |
| `testing` | Issue is a testing or QA validation issue. |
| `testing-created` | Parent issue already has an automation-created testing child issue. Prevents duplicate child creation. |
| `assign-current-iteration` | Issue should be assigned to the current Project V2 iteration. Removed after successful sync. |
| `ready-for-dr-review` | Pull request is ready for Dylan Richard review and should enter the Delivery Board review queue. |

## Project V2 Requirements

The default project is `iOPSPro/5`. Workflows use GitHub GraphQL Project V2 APIs and require a token with access to read and write the target project fields.

Expected fields:

- `Iteration`: primary Project V2 iteration field.
- `60 Day Block`: secondary Project V2 iteration field used by copy/sync workflows where available.
- `Environment`: single-select field with options `V1`, `V2`, `Backend`, `N/A`, and `Both V1 and V2`.
- `Status`: single-select field used by PR and testing sub-issue automation. It must include `In progress`.

Caller repositories typically pass the project token with:

```yaml
secrets:
  project_token: ${{ secrets.PROJECT_TOKEN }}
```

## Repository Setup

Add caller workflows in each repository that should use these defaults. A typical setup wires issue events to the reusable workflows and passes repository, project, and token inputs.

Example caller for assigning the current iteration from an issue form answer:

```yaml
name: Apply issue-form iteration label

on:
  issues:
    types: [opened, edited]

jobs:
  apply-iteration-label:
    if: ${{ github.event.issue.state == 'open' }}
    uses: iOPSPro/.github/.github/workflows/apply-issue-form-iteration-label.yml@main
    with:
      owner: ${{ github.repository_owner }}
      repo: ${{ github.event.repository.name }}
      issue_number: ${{ github.event.issue.number }}
    secrets: inherit
```

Example caller for syncing the project iteration after the label is applied:

```yaml
name: Sync project iteration

on:
  issues:
    types: [labeled]

jobs:
  sync-iteration:
    if: ${{ github.event.label.name == 'assign-current-iteration' }}
    uses: iOPSPro/.github/.github/workflows/sync-project-iteration-from-label.yml@main
    with:
      owner: ${{ github.repository_owner }}
      repo: ${{ github.event.repository.name }}
      issue_number: ${{ github.event.issue.number }}
      project_owner: iOPSPro
      project_number: 5
    secrets:
      project_token: ${{ secrets.PROJECT_TOKEN }}
```

Use the same pattern for the other reusable workflows:

- Pass `owner`, `repo`, and `issue_number` from the event payload.
- Pass `pull_number` from the pull request payload for PR review queue automation.
- Pass `project_owner: iOPSPro` and `project_number: 5` unless the repository uses a different project.
- Pass `secrets.PROJECT_TOKEN` as `project_token` for workflows that read or write Project V2 data.

Example caller for Dylan Richard PR review queue automation:

```yaml
name: Sync PR to DR review queue

on:
  pull_request:
    types: [labeled, reopened, ready_for_review]

jobs:
  sync-dr-review:
    if: ${{ github.event.pull_request.state == 'open' && contains(github.event.pull_request.labels.*.name, 'ready-for-dr-review') }}
    uses: iOPSPro/.github/.github/workflows/sync-pr-delivery-board-for-review.yml@main
    with:
      owner: ${{ github.repository_owner }}
      repo: ${{ github.event.repository.name }}
      pull_number: ${{ github.event.pull_request.number }}
      project_owner: iOPSPro
      project_number: 5
    secrets:
      project_token: ${{ secrets.PROJECT_TOKEN }}
```

## Testing Child Issues

Testing work supports two paths.

Automation-created path:

1. A parent issue receives `needs-testing`.
2. A caller workflow invokes `create-testing-subissue.yml`.
3. The reusable workflow skips parents that already have `testing-created`.
4. It creates `Testing: <parent title>` with label `testing` and type `Test`.
5. It attaches the child as a sub-issue, copies assignees and project fields, labels the parent `testing-created`, and comments on the parent.

Manually created path:

1. A user creates an issue from the `Testing` form.
2. The issue is attached as a child issue of the parent.
3. A caller workflow invokes `copy-parent-project-fields-to-test-child.yml`.
4. The reusable workflow verifies the `testing` label, finds the parent through the sub-issues API, and copies assignees and project fields.

GitHub Actions does not expose a dedicated documented `issues` event for "issue became a sub-issue." For manually created testing children, use nearby triggers such as `issues.opened`, `issues.edited`, `issues.labeled`, and/or `workflow_dispatch`.

## Maintenance

- Keep issue form labels stable. Several workflows parse rendered issue body headings such as `### Assign to current iteration?`.
- Keep Project V2 field names and option names aligned with the defaults documented here.
- Store only organization-safe, non-sensitive defaults in this repository.
- Put repository-specific intake or automation in the target repository when it differs from these defaults.

## Maintainers

Managed by the iOPSPro organization administrators.
