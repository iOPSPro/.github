# Site Normalization

## Purpose

GitHub Issue Field `Site` stores the canonical airport, facility, or operating location for reporting.

`Site` is a text Issue Field, so consistency is enforced by a source-controlled registry and GitHub Actions automation instead of single-select options.

## Field Model

Expected Organization Issue Field:

| Field | Type | Current iOPSPro ID | Values |
| --- | --- | --- | --- |
| `Site` | Text | `42617022` | Canonical name from `data/site-registry.json` |

## Registry

Canonical site names live in:

`data/site-registry.json`

Each entry must include:

```json
{
  "id": "salt-lake-city-international-airport",
  "code": "SLC",
  "name": "Salt Lake City International Airport",
  "aliases": ["SLC", "Salt Lake City"]
}
```

Rules:

- `id` is stable, lowercase, and hyphenated.
- `code` stores the common site or airport code when one exists.
- `name` is the canonical reporting display value.
- `aliases` includes alternate names, abbreviations, and common variants.
- Aliases must not point to more than one canonical site.

## Automation

Use `.github/workflows/sync-issue-site.yml` from participating repositories.

The workflow:

- reads the `Site` answer from issue forms
- reads current Issue Field values when triggered by Issue Field add/remove activity
- normalizes known site aliases to the canonical registry name
- writes the canonical site to the `Site` Issue Field
- comments on unknown sites with a likely suggestion when available
- avoids no-op rewrites when the Issue Field already contains the canonical value

Example caller:

```yaml
name: Sync site

on:
  issues:
    types: [opened, edited, field_added, field_removed]

jobs:
  sync-site:
    if: ${{ github.event.issue.state == 'open' }}
    uses: iOPSPro/.github/.github/workflows/sync-issue-site.yml@main
    with:
      owner: ${{ github.repository_owner }}
      repo: ${{ github.event.repository.name }}
      issue_number: ${{ github.event.issue.number }}
      issue_field_owner: iOPSPro
      site_issue_field_id: "42617022"
    secrets:
      project_token: ${{ secrets.PROJECT_TOKEN }}
```

## Migration Strategy

1. Populate `data/site-registry.json` with canonical sites and aliases.
2. Update caller workflows to invoke `sync-issue-site.yml`.
3. Backfill existing issues by running the caller workflow manually or editing the `Site` Issue Field.
4. Update reporting to group by canonical `Site`.
5. Stop relying on ad hoc site spellings in issue bodies or Issue Field values.

## Maintenance

When a workflow reports an unknown site:

1. If the value is a typo or alias for an existing site, add it to that site's `aliases`.
2. If it is a new site, add a new registry entry.
3. Re-run or edit the issue so automation writes the canonical value.

Keep the registry non-sensitive. Do not store private tenant identifiers, credentials, URLs, or site-specific security details.
