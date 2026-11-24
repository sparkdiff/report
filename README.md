# SparkDiff report

Report a CI result to SparkDiff. The report attaches to the Spark whose
revision is the tested commit.

## The workflow

Once revision refs are turned on for the repository, SparkDiff pushes each
revision of a Spark as a branch named `spark/<change>/<rev>`, so a workflow
that runs on every branch push runs on it. Paste this, change `my-repo` to
the repository's name in SparkDiff, and put your test command where
`go test ./...` is:

<!-- spark-report:example:begin -->
```yaml
name: CI
on:
  push:
    branches: ["**"]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: go test ./...

  report:
    needs: test
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Report to SparkDiff
        uses: sparkdiff/report@v0.1.1
        with:
          token: ${{ secrets.SPARKDIFF_TOKEN }}
          repo: my-repo
          status: ${{ needs.test.result == 'success' && 'success' || needs.test.result == 'failure' && 'failure' || 'skipped' }}
          summary: go test ./...
```
<!-- spark-report:example:end -->

The report runs in a job of its own that checks out nothing, so the job
holding the token never runs your repository's code. A test job that was
cancelled or skipped is reported as `skipped`.

## The token

<!-- spark-report:setup:begin -->
In SparkDiff, open Your account and the tab Your agents, create an agent
for CI, and copy its token. Store it on GitHub as the repository or
organization secret `SPARKDIFF_TOKEN`. Make that agent for CI and nothing
else, and never use an agent that reviews: whatever holds the token can act
as that agent.
<!-- spark-report:setup:end -->

## Inputs

| Input | Required | What it is |
|---|---|---|
| `token` | yes | The CI agent's token: `${{ secrets.SPARKDIFF_TOKEN }}`. |
| `repo` | yes | The repository's name in SparkDiff. |
| `status` | yes | `success`, `failure`, `error`, `skipped` or `pending`. |
| `summary` | no | One line saying what ran and what it said. |
| `url` | no | A link to the run. Defaults to this workflow run's page. |
| `sha` | no | The commit the result is for. Defaults to the commit this run is on. |
