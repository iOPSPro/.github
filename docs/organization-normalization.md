# Organization Normalization

## Purpose

GitHub Issue Fields now separate business segment classification from customer or organization identity.

- `Segment` is a single-select Issue Field used for business segment reporting.
- `Organization` is a text Issue Field used for the canonical customer or organization name.

This avoids the 100-option limit on GitHub single-select fields while keeping reporting consistent through a source-controlled canonical registry.

## Field Model

Expected Organization Issue Fields:

| Field | Type | Current iOPSPro ID | Values |
| --- | --- | --- | --- |
| `Segment` | Single-select | `42612296` | `GSE`, `Gate Systems`, `Both`, `N/A` |
| `Organization` | Text | `42656518` | Canonical name from `data/organization-registry.json` |

`Segment` replaces the previous overloaded `Customer` use for business segment reporting. Organization names should not be added as `Segment` options.

## Registry

Canonical organization names live in:

`data/organization-registry.json`

Each entry must include:

```json
{
  "id": "delta-air-lines",
  "name": "Delta Air Lines",
  "aliases": ["DAL", "Delta", "Delta Airlines"]
}
```

Rules:

- `id` is stable, lowercase, and hyphenated.
- `name` is the canonical reporting display value.
- `aliases` includes known abbreviations, alternate spellings, and legacy values.
- Aliases must not point to more than one canonical organization.

## Automation

Use `.github/workflows/sync-issue-segment-and-organization.yml` from participating repositories.

The workflow:

- reads `Segment` and `Organization` answers from issue forms
- reads current Issue Field values when triggered by Issue Field add/remove activity
- writes `Segment` to the Segment Issue Field
- normalizes known organization aliases to the canonical registry name
- writes the canonical organization to the Organization Issue Field when its field ID is provided
- comments on unknown organizations with a likely suggestion when available
- fails on unknown organizations so invalid metadata is visible during issue creation or update

Example caller:

```yaml
name: Sync segment and organization

on:
  issues:
    types: [opened, edited, field_added, field_removed]

jobs:
  sync-segment-organization:
    if: ${{ github.event.issue.state == 'open' }}
    uses: iOPSPro/.github/.github/workflows/sync-issue-segment-and-organization.yml@main
    with:
      owner: ${{ github.repository_owner }}
      repo: ${{ github.event.repository.name }}
      issue_number: ${{ github.event.issue.number }}
      issue_field_owner: iOPSPro
      segment_issue_field_id: "42612296"
      organization_issue_field_id: "42656518"
    secrets:
      project_token: ${{ secrets.PROJECT_TOKEN }}
```

## Migration Strategy

1. Confirm the `Segment` Issue Field exists with the supported values.
2. Create the `Organization` text Issue Field at the iOPSPro organization level.
3. Update caller workflows to invoke `sync-issue-segment-and-organization.yml`.
4. Backfill existing issues:
   - map old `Customer` values `GSE`, `Gate Systems`, and `Both` into `Segment`
   - map old customer or tenant names into `Organization`
   - add common variants to the registry as aliases before backfill
5. Update reporting to group customer identity by `Organization` and business classification by `Segment`.
6. Stop adding organization/customer names as single-select options.

## Maintenance

When a workflow reports an unknown organization:

1. If the value is a typo or alias for an existing organization, add it to that organization's `aliases`.
2. If it is a new organization, add a new registry entry.
3. Re-run or edit the issue so automation writes the canonical value.

Keep the registry non-sensitive. Do not store private customer data, tenant identifiers that should not be public, credentials, URLs, or contract details.
