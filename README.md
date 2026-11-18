# Denpex GPU Crash Diagnostics GitHub Action

Runs preflight GPU/NCCL topology validation or diagnoses failed PyTorch, CUDA OOM, Xid, and NCCL CI runs and posts a sticky PR comment plus `$GITHUB_STEP_SUMMARY` report.

## Example Workflow

```yaml
name: GPU Training CI
on: [pull_request]

permissions:
  contents: read
  pull-requests: write

jobs:
  train:
    runs-on: gpu-runner
    steps:
      - uses: actions/checkout@v4
      - name: Run training
        id: train
        run: torchrun --nproc_per_node=8 train.py 2>&1 | tee training.log
      - name: Diagnose failed GPU CI run
        if: failure()
        uses: denpex/denpex-gpu-action@v1
        with:
          check: diagnose
          logs-file: training.log
          comment-on-pr: 'true'
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

## PR Funnel & B2B Team Upgrade

- Writes `$GITHUB_STEP_SUMMARY` and upserts sticky PR comments marked with `<!-- denpex-gpu-ci-diagnosis -->`.
- Links directly to the 30-day B2B Team Trial (`https://denpex.com/pricing?utm_source=github_action_pr`).
- Synchronized automatically with `L:\Saas Web App Fix For GPU servers\action.yml` via `npm run sync:all-surfaces`.
