# SparkDiff report

Report a CI result to SparkDiff, keyed by the commit the run is on, so it
shows on the Spark that commit is a revision of. It runs the
`sparkdiff/report` image, pinned by version.

## The whole workflow

SparkDiff pushes each revision of a Spark to your remote as a branch named
`spark/<change>/<rev>`. A workflow that runs on every branch push runs on
those too; one that filters branches adds `spark/**`. Paste this, change
`my-repo` to the repository's name in SparkDiff, and put the test command
where `go test ./...` is:

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
        uses: sparkdiff/report@v0.1.0
        with:
          server: ${{ vars.SPARK_SERVER }}
          token: ${{ secrets.SPARK_TOKEN }}
          repo: my-repo
          status: ${{ needs.test.result == 'success' && 'success' || needs.test.result == 'failure' && 'failure' || 'neutral' }}
          summary: go test ./...
```
<!-- spark-report:example:end -->

**Why two jobs.** The report runs in a job of its own that checks out
nothing and runs none of the repository's code. A step that runs earlier in
the same job can reach a later step's secret (it can set the environment the
later step runs in), and on a `spark/**` branch the code is whatever a
member or an agent submitted for review. So the job that runs the code holds
no secret, and the job that holds the token runs only this Action.

`if: always()` reports a failure too. `needs.test.result` is `success`,
`failure`, `cancelled` or `skipped`; SparkDiff has no `cancelled` or
`skipped`, so the line above reports those as `neutral`.

## The token

Set up once, in the repository's or the organization's settings on GitHub:

- the variable `SPARK_SERVER`: your SparkDiff server's address, `https://...`;
- the secret `SPARK_TOKEN`: the token of an agent you create on your
  SparkDiff organization's Agents tab for CI and nothing else.

That token can do whatever an agent can do in your organization today, not
only report: give CI an agent of its own, sponsored by someone who lands
nothing through it, and never an agent that reviews. A report-only
credential is planned.

## Inputs

| Input | Required | Default | What it is |
|---|---|---|---|
| `server` | yes | | The SparkDiff server's address. |
| `token` | yes | | A SparkDiff API token. It reaches the program as `SPARK_TOKEN`, never on a command line. |
| `repo` | yes | | The repository's name in SparkDiff. |
| `sha` | no | `GITHUB_SHA` | The 40-hex commit the result is for. |
| `kind` | no | `test_run` | `test_run`, `ci_check`, `coverage`, `review_verdict`, `recording` or `custom`. |
| `status` | no | `neutral` | `success`, `failure`, `pending`, `neutral` or `error`. |
| `source` | no | `github-actions` | Who ran it. |
| `url` | no | this run's page | A link to the run. |
| `summary` | no | | One status line, at most 4096 bytes. A log belongs behind `payload`. |
| `payload` | no | | A pointer to a blob holding the log or the report. |
| `timeout` | no | `60s` | The most the whole report may take, retries included. |

There is no input for `--insecure`: from GitHub's runners the server is
reached over https.

## Pull requests

Report from the push run. On a `pull_request` run `GITHUB_SHA` is a merge
commit GitHub made, and on a `pull_request_target` run it is the tip of the
base branch; neither is a revision's commit, so the result is recorded and
no Spark shows it. The step warns (an annotation, not a failure) when that
is the commit it would report. To report from a pull request run anyway,
pass `sha: ${{ github.event.pull_request.head.sha }}`.

## What it prints

On success, one line naming the evidence item and the Spark revision it is
on, or saying it matches no revision yet. On a refusal, the server's
sentence whole, on one line, and the step fails. A report is tried up to
three times when the server cannot be reached or answers 502, 503 or 504; a
report retried after the server had in fact recorded it is recorded twice
(two rows with the same result). The secret check refuses a token pasted
into the summary or the link, as visible text; a URL-encoded or
base64-encoded one is not found, so keep tokens out of what you report. The
sentences are listed in SparkDiff's `docs/04-architecture/evidence-api.md`,
"What a refusal says".

## Maintenance (SparkDiff's)

This directory is the root of the public repository `sparkdiff/report`.
`action.yml` names the image by version tag in SparkDiff's source; after the
image is first pushed, the published copy pins it to the image's digest
(`docker://sparkdiff/report@sha256:...`), and the Action is tagged from that
commit.
