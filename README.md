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
- GitHub Issue Forms cannot conditionally apply labels based on dropdown answers. The forms collect testing intent and instruct the author to add `needs-testing` when appropriate.

## Notes

- Repository-specific templates and files take precedence over the defaults stored here.
- Only non-sensitive, organization-safe defaults should be stored in this repository.

## Maintainers

Managed by the iOPSPro organization administrators.
