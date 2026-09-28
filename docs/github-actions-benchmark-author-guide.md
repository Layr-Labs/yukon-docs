# GitHub Actions Benchmark Author Guide

## Examples

1. https://github.com/Layr-Labs/mlxfast-challenge-dev
2. https://github.com/Layr-Labs/eip8200-challenges
3. https://github.com/Layr-Labs/mlxfast-bonsai2-27b-engine
4. https://github.com/Lulu-Zhou-EigenLabs/ssi-ordering-challenge

## 1. What Is Yukon?

Yukon turns a GitHub repository into an optimization benchmark. Benchmark authors define
what solvers may edit, how the score is produced, and which direction is better. Solvers
submit candidate archives; Yukon validates them, records the official score, and promotes
only submissions that improve the current best.

For GitHub Actions benchmarks, your repository's workflow is the executor. Yukon owns
orchestration, score comparison, editable-path enforcement, and promotion decisions.
Yukon also provides the benchmark CLI and database-backed submission system for your
benchmark.

## 2. `benchmark.json`

Put `benchmark.json` at the benchmark root. For a repo-root benchmark, this is the
repository root. For a leaf benchmark imported with `rootDir`, it is the leaf directory.

Minimal GitHub Actions-routed manifest:

```json
{
  "schemaVersion": 1,
  "name": "example-speedup",
  "description": "Optimize submission/solution.py while preserving evaluator correctness.",
  "category": "python",
  "direction": "-",
  "editablePaths": ["submission"],
  "setupCommand": ["bash", "-lc", ".yukon/setup.sh"],
  "benchmarkCommand": ["bash", "-lc", ".yukon/run.sh"],
  "scorePath": ".yukon/score.json",
  "runner": {
    "provider": "github-actions",
    "workflow": "benchmark.yml"
  }
}
```

Multi-track GitHub Actions manifest:

```json
{
  "schemaVersion": 2,
  "name": "example-suite",
  "tracks": [
    {
      "name": "fast-path",
      "description": "Optimize the fast-path implementation.",
      "category": "go",
      "direction": "+",
      "editablePaths": ["tracks/fast-path/submission"],
      "setupCommand": ["bash", "-lc", "./setup.sh fast-path"],
      "benchmarkCommand": ["bash", "-lc", "./benchmark.sh fast-path"],
      "scorePath": "benchmark-results/fast-path/score.json",
      "runner": {
        "provider": "github-actions",
        "workflow": "benchmark-fast-path.yml"
      }
    },
    {
      "name": "small-path",
      "description": "Optimize the small-path implementation.",
      "category": "go",
      "direction": "-",
      "editablePaths": ["tracks/small-path/submission"],
      "setupCommand": ["bash", "-lc", "./setup.sh small-path"],
      "benchmarkCommand": ["bash", "-lc", "./benchmark.sh small-path"],
      "scorePath": "benchmark-results/small-path/score.json",
      "runner": {
        "provider": "github-actions",
        "workflow": "benchmark-small-path.yml"
      }
    }
  ]
}
```

Use schema v1 for one benchmark track. Use schema v2 when one repository hosts
multiple independently scored tracks. In schema v2, each entry in `tracks` has the
same core fields as a schema-v1 manifest, and each track must have its own
non-overlapping editable surface and score path. Yukon can then validate, score,
and promote a track without replacing sibling-track submissions.

Schema:

| Field | Required | Explanation |
| --- | --- | --- |
| `schemaVersion` | yes | Use `1` for a single-track manifest or `2` for a multi-track manifest. |
| `name` | yes | Short benchmark name. |
| `description` | yes | Human-readable description of the task. |
| `category` | yes | Grouping label shown in clients. |
| `direction` | yes | `"+"` means higher is better. `"-"` means lower is better. |
| `editablePaths` | yes | Files or directories solvers may replace, relative to the benchmark root. |
| `setupCommand` | yes | Local/default setup command array. Keep it aligned with the workflow setup step. |
| `benchmarkCommand` | yes | Local/default scoring command array. It must write `scorePath`. |
| `scorePath` | yes | JSON score file path, relative to the benchmark root. Must not be inside `editablePaths`. |
| `runner` | yes | Use `{ "provider": "github-actions", "workflow": "<file>.yml" }`. |
| `minScoreImprovementBips` | optional | Minimum relative improvement in basis points. `100` means 1%. |
| `tracks` | schema v2 | Array of track manifests. Only valid when `schemaVersion` is `2`. |
| `maxSubmissionBytes` | optional | Per-track archive byte limit before a candidate is built. |
| `optionalEditablePaths` | optional | Editable paths that may be absent from a submission archive. Use sparingly and document why absence is valid. |
| `editableSurfaceByteBudget` | optional | Fine-grained byte budget for editable files, used by larger challenge repos to reject oversized submissions before measurement. |
| `preSubmitCommand` | optional | Local/server preflight command that can fail a submission before the ranked benchmark is dispatched. |
| `scoreArtifact` | optional | Artifact path template when the uploaded artifact layout is not just `scorePath`. Keep it aligned with the workflow upload step. |
| `leaderboard` | optional | Leaderboard metadata, such as `{ "namespace": "<track-id>" }`, for public score grouping. |

Path rules:

- Paths are relative POSIX paths: use `submission`, not `/submission`, `../submission`,
  or `submission\\x`.
- `editablePaths` must not include `benchmark.json`, `.git`, duplicate paths, or
  overlapping paths like `["submission", "submission/src"]`.
- In schema v2, sibling tracks should have disjoint `editablePaths` unless the tracks
  are intentionally coupled and cannot be promoted independently.
- `scorePath` must be outside `editablePaths`.
- `runner.workflow` is only the filename inside `.github/workflows`, for example
  `"benchmark.yml"`, not `.github/workflows/benchmark.yml`.

Score file:

```json
{
  "score": 12.345,
  "metrics": {
    "correct": true,
    "latencyMs": 12.345
  }
}
```

Only `score` is required. It must be a finite number. `metrics` is optional JSON and is
stored with the submission. The benchmark should exit nonzero instead of writing a fake
score when correctness, setup, or measurement fails.

## 3. Workflow Spec

The workflow named by `runner.workflow` is the production benchmark runner. Yukon
dispatches it for baseline validation and submission validation.

Required behavior:

- Define `workflow_dispatch`.
- Check out the dispatched commit or ref, not a pull request merge ref.
- Run setup and benchmark from a clean checkout.
- Upload at least one artifact containing the file at `scorePath`. The artifact name is
  not significant.
- Fail the workflow, or omit/produce an invalid score artifact, when the benchmark cannot
  produce a trusted score.
- Use clear step names containing `Setup` and `Benchmark` if you want Yukon to classify
  failures precisely.

Minimal workflow:

```yaml
name: Benchmark

on:
  workflow_dispatch:

permissions:
  contents: read
  actions: read

concurrency:
  group: benchmark-${{ github.ref }}
  cancel-in-progress: true

jobs:
  benchmark:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          ref: ${{ github.sha }}
          persist-credentials: false

      - name: Setup
        run: bash .yukon/setup.sh

      - name: Benchmark
        run: bash .yukon/run.sh

      - name: Upload score
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: benchmark-score
          path: .yukon/score.json
          if-no-files-found: error
```

Workflow rules that matter in production:

- Use `workflow_dispatch` as the submission entry point. Do not add `pull_request`,
  `push`, or `schedule` triggers to a workflow that executes untrusted submissions.
- Check out `${{ github.sha }}`. Do not check out a branch name that may move while a
  run is queued.
- Set `persist-credentials: false` unless a trusted step explicitly needs a checkout
  token. Ranked workflows should usually need read-only `contents` permission.
- Pin third-party actions by full commit SHA for self-hosted or secret-bearing jobs.
  Version tags are acceptable for minimal public examples, but not for private or
  high-value benchmark runners.
- Treat `editablePaths` as untrusted input. If the workflow runs on self-hosted
  hardware, add an early hosted job that lints the manifest and performs static review
  or surface checks before the GPU/TEE/private runner is reached.
- Re-verify the editable surface at runtime for `submissions/*` refs. The dispatched
  candidate should differ from the trusted base only inside manifest-declared
  editable paths.
- Keep private prompts, goldens, baselines, calibration data, and runner credentials
  outside the editable surface. Prefer content-addressed pins and fail closed when a
  staged artifact is absent or hash-mismatched.
- Upload the score only after validating its schema and artifact path. For private
  benchmarks, redact failure details and deny-list prompt/golden/model artifacts before
  uploading anything from a failed run.
- Use concurrency deliberately. Hosted validation can cancel superseded runs; expensive
  self-hosted measurements often should not be cancelled mid-window because that can
  waste the runner slot and leave hardware in a bad state for the next run.

Artifact conditions:

- Yukon lists all artifacts on the workflow run; it does not require the name
  `benchmark-score`.
- Each candidate artifact must be at most 50 MiB compressed and 50 MiB expanded.
- The artifact zip must contain the exact manifest `scorePath`, for example
  `.yukon/score.json`.
- If the exact path is not present, Yukon falls back to a unique basename match, for
  example a single `score.json`. Do not rely on this if the artifact can contain multiple
  files with the same basename.
- The score file must be valid JSON with a finite numeric `score`; optional `metrics`
  must be JSON-serializable.

If the benchmark is imported with `rootDir`, GitHub Actions still starts in the repository
root. Set `defaults.run.working-directory` or `cd` into the leaf before running setup and
benchmark commands.

Avoid using `pull_request` as the benchmark trigger for Yukon submission branches. Yukon
opens a PR as the public record, then dispatches the workflow explicitly. A `pull_request`
trigger can double-run submissions.

Your workflow is responsible for runner choice, dependencies, caches, timeouts, secrets,
network isolation, and hardware. Treat code under `editablePaths` as untrusted.

## 4. Submissions And Promotions

Before Yukon can import the benchmark, install the correct GitHub App on the benchmark
repository:

- Dev: https://github.com/apps/yukon-eigen/installations/new
- Prod: https://github.com/apps/yukon-autoresearch/installations/new

Yukon's GitHub Actions flow is PR-centric:

1. A solver submits an archive containing replacement files for `editablePaths`.
2. Yukon validates the archive server-side and builds a candidate commit from the current
   benchmark baseline plus only those editable files.
3. Yukon pushes `submissions/<submissionId>` and opens a PR into the default branch.
4. Yukon dispatches `runner.workflow` on that submission branch.
5. Yukon waits for the workflow run, downloads the score artifact, parses `scorePath`, and
   compares the score against the current best using `direction` and
   `minScoreImprovementBips`.
6. If the submission does not improve, Yukon comments with the score and closes the PR.
7. If the submission improves, Yukon leaves the PR open for promotion.
8. Promotion fast-forwards the default branch to the validated candidate commit. GitHub
   then marks the PR merged.

For schema-v2 benchmarks, this flow applies per track. `yukon switch <track>` changes
the local track selection without changing Git branches, `HEAD`, the index, or the
worktree; `yukon tracks` lists the available track selections.

The PR is the durable public record: diff, workflow run, score comment, and final
closed/merged state live together.

Yukon never trusts solver-provided Git history. It promotes only the commit it scored, or
a content-identical rebase when another Yukon promotion moved the default branch first.
Humans should not merge submission PRs manually, because GitHub merge/squash/rebase
buttons create commits Yukon did not score.

Once your repository has `benchmark.json`, the workflow, and the correct GitHub App
installation set up, contact us. We will import the benchmark and coordinate the Yukon
CLI, database, and frontend pieces.
