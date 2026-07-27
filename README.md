# Copilot usage report

Dependency-free Node CLI for collecting GitHub Copilot billing and usage data.
The command writes exactly one JSON document to stdout. Errors and diagnostics
are written to stderr so stdout can be redirected safely.

## Local usage

```bash
GITHUB_TOKEN="$(gh auth token)" node src/copilot-usage.mjs \
  --scope organization \
  --slug ORGANIZATION_LOGIN \
  --pretty
```

By default, the CLI downloads the latest rolling 28-day user report, keeps only
records whose `day` belongs to the selected month, and requests individual daily
reports only for target dates outside the rolling window. It then aggregates the
records into month-to-date user totals. The current UTC day is excluded because
GitHub only publishes reports for completed days.

Fetch one day instead for diagnostics:

```bash
GITHUB_TOKEN="$(gh auth token)" node src/copilot-usage.mjs \
  --scope organization \
  --slug ORGANIZATION_LOGIN \
  --user-report-day 2026-07-25 \
  --pretty
```

Redirect stdout when a file is useful:

```bash
GITHUB_TOKEN="$(gh auth token)" node src/copilot-usage.mjs \
  --scope organization \
  --slug ORGANIZATION_LOGIN > copilot-usage.json
```

## Enterprise testing

Use the same CLI with an enterprise-authorized token:

```bash
GITHUB_TOKEN="..." node src/copilot-usage.mjs \
  --scope enterprise \
  --slug ENTERPRISE_SLUG
```

Organization and enterprise APIs use different endpoint paths. The CLI maps
those paths internally and emits the same normalized output shape.

The enterprise seat API can return more assignment records than unique billed
seats when a user receives Copilot through multiple organizations or enterprise
teams. `copilot.totalSeats` is GitHub's unique billed-seat count, while
`copilot.assignmentRecordCount` is the number of returned assignment records.

## Output semantics

- `credits.grossUsed` is month-to-date AI-credit consumption.
- `models[].percentageOfGrossCredits` is a credit-based model percentage from
  the billing API.
- `userMetrics.users[].aiCreditsUsed` is the user's aggregated total across the
  reports with content in `userMetrics.period`.
- `userMetrics.summary.analyticsCreditsUsed` is an analytics total, not a
  billing total. GitHub does not expect it to reconcile exactly with
  `credits.grossUsed`, and the output reports the observed difference rather
  than hiding it.
- `userMetrics.period.noContentDays` lists dates for which GitHub returned HTTP
  204 instead of a downloadable user report.
- User-level model percentages are activity-based because GitHub's daily user
  report does not attribute AI credits to individual models. Both interaction
  and code-generation percentage bases are named explicitly in the output.
- Included pool and effective limit remain `null` unless a reliable billing-
  entity-level source is added. Current organization seats are not treated as
  the enterprise's shared pool.

Month-to-date reports are currently recalculated on each run, although the
rolling report substantially reduces API calls. A `TODO` in the report-fetching
code marks the intended extension point for persisting daily aggregates in S3
(or another store) so GitHub Actions can fetch only missing days.

## Tests

```bash
npm test
```
