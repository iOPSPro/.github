# .github
Organization-wide default GitHub configuration, issue forms, pull request templates, and community health files for iOPSPro repositories.


# iOPSPro GitHub Defaults

This repository contains organization-wide default GitHub community health files for repositories in the iOPSPro organization.

## Purpose

The files in this repository are used to standardize and improve repository setup across iOPSPro, including:

- Issue forms and issue templates
- Pull request templates
- Shared contribution guidance
- Other default community health files supported by GitHub

## How it works

Repositories in the iOPSPro organization can inherit these default files when they do not define their own repository-specific versions.

This helps us:
- Create a more consistent issue intake process
- Improve work item quality and triage
- Standardize pull request expectations
- Reduce repeated setup across repositories

## Contents

Typical files in this repository may include:

- `.github/ISSUE_TEMPLATE/`
- `.github/workflows/`
- `PULL_REQUEST_TEMPLATE.md`
- `CONTRIBUTING.md`
- `SUPPORT.md`
- Other supported default GitHub community health files

## Issue forms

Organization-wide issue forms live in `.github/ISSUE_TEMPLATE/`.

Repositories in the iOPSPro organization inherit these defaults only if they do not define their own local issue templates/forms.

## Future testing sub-issue model (automation)

We are standardizing a simple, human-readable signal for when an issue should spawn a testing child issue:

- Parent trigger label: `needs-testing`
- Child label: `testing`
- Optional future marker on parent: `testing-created`

Important limitations and expectations:

- This `.github` repository does not automatically receive issue events from every repository in the organization.
- Reusable workflows in `.github/workflows/` are meant to be called by small per-repo caller workflows in participating repositories.
- Reusable workflow scaffold: `.github/workflows/create-testing-subissue.yml`.
- Reusable workflow for manually created testing child issues: `.github/workflows/copy-parent-project-fields-to-test-child.yml`.
- GitHub Issue Forms cannot conditionally apply labels based on dropdown answers. The forms collect testing intent and instruct the author to add `needs-testing` when appropriate.

There are two supported testing child issue paths:

- Automation-created child issue: a parent issue with `needs-testing` calls `create-testing-subissue.yml`. That workflow creates the child, attaches it as a sub-issue, adds it to the configured project when needed, and copies `Iteration`, `60 Day Block`, and `Environment` from the parent.
- Manually created child issue: a user creates an issue with the `Test` template, attaches it as a child of a parent issue, and a per-repo caller invokes `copy-parent-project-fields-to-test-child.yml`. That workflow finds the child's parent through the sub-issues API and copies assignees plus the same project fields.

GitHub Actions does not currently expose a dedicated documented `issues` activity type for "issue became a sub-issue." Per-repo callers should trigger the manual-copy workflow on practical nearby events such as `issues.opened`, `issues.edited`, `issues.labeled`, and/or `workflow_dispatch`.

## Conditional iteration label model (automation)

To support project iteration assignment flows based on issue form answers:

- Issue forms include a required dropdown: `Assign to current iteration?`
- If the response is `Yes`, automation can add label: `assign-current-iteration`
- Reusable workflow scaffold: `.github/workflows/apply-issue-form-iteration-label.yml`

This provides a stable signal that downstream workflows can use to assign project fields (for example, current iteration) after issue creation.

## Environment project field model (automation)

Environment should be stored in the GitHub Project custom field instead of repository labels:

- Project field: `Environment`
- Field type: single select
- Supported options: `V1`, `V2`, `Backend`, `N/A`, `Both V1 and V2`
- Issue forms include a required dropdown: `Is there a specific environment this is for?`
- Reusable workflow scaffold: `.github/workflows/sync-project-application-version-from-form.yml`

The reusable workflow reads the issue form answer on issue open/edit and sets the Project V2 single-select field directly. This avoids relying on repository-scoped version labels that can be lost when issues move between repositories.

## Notes

- Repository-specific templates and files take precedence over the defaults stored here.
- Only non-sensitive, organization-safe defaults should be stored in this repository.

## Maintainers

Managed by the iOPSPro organization administrators.
