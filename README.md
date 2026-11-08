# Denpex GitHub Action

Diagnose distributed training failures in CI, or gate a pipeline before it burns GPU-hours.

Instead of reading 10,000 lines of PyTorch, NCCL and SLURM output to find out why a job died,
add this action to the end of your workflow. When the training step fails, Denpex reads the log,
identifies the root cause. CUDA OOM, an InfiniBand flap, a straggler node, a hidden dead rank -
and prints the fix in your Actions console.

It runs the same single-file agent you install on a training node, downloaded and
checksum-verified at each run. Nothing to install, no package to add.

## Quickstart, diagnose a crash

```yaml
name: Nightly Training
on: [push]

jobs:
  train:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run PyTorch distributed training
        run: python -m torch.distributed.run --nproc_per_node=8 train.py 2>&1 | tee training.log

      - name: Diagnose with Denpex
        uses: Denpex/denpex-action@main
        if: failure()
        with:
          # Optional for one frictionless no-key diagnosis. Later CI diagnoses need a key
          # because CI cannot complete the browser check. With a key, diagnosis is attributed
          # to your account and alerts route to Slack / PagerDuty / webhook channels.
          api-key: ${{ secrets.DENPEX_API_KEY }}
```

Note the `| tee training.log`, the action needs a log on disk to read. It looks for
`training.log`, then `train.log`, then `nohup.out`, or whatever you set as `logs-file`.

## Gate a pipeline before launch

```yaml
      - name: Denpex preflight
        uses: Denpex/denpex-action@main
        with:
          check: preflight            # or preflight-deep, check-versions, validate-checkpoint
          checkpoint-dir: ./checkpoints
```

Preflight checks rendezvous, fabric, GPU/driver uniformity, ulimits and clock skew, and exits
non-zero on a blocker so the job fails before the allocation is spent. `preflight-deep` adds the
NCCL topology lint (GPU-to-NIC affinity, PCIe ACS, NCCL env against the real fabric, memlock) and
needs a runner that can see the GPUs.

## Inputs

| Input | Default | What it does |
|---|---|---|
| `check` | `auto` | `auto` diagnoses when a log is present, otherwise runs preflight. Or set `diagnose`, `preflight`, `preflight-deep`, `check-versions`, `validate-checkpoint` explicitly. |
| `api-key` | *(empty)* | Optional for the first no-key diagnosis. Later CI diagnoses require a key because CI cannot complete the browser check. |
| `logs-file` | `training.log` | Path to the training log, relative to `working-directory`. |
| `fail-on-diagnosis` | `false` | Whether a successful diagnosis fails the step. Off by default: the job has already failed, and failing again buries the diagnosis in a collapsed red step. |
| `checkpoint-dir` | *(empty)* | Checkpoint directory. Required for `validate-checkpoint`. |
| `agent-url` | `https://denpex.com/agent/denpex.py` | Override only for an air-gapped or self-hosted mirror. |
| `verify-checksum` | `true` | Verify the agent against the published `SHA256SUMS` before running it. |
| `python-version` | `3.x` | Python for the agent (needs 3.8+). |
| `working-directory` | `.` | Directory the check runs in. |

## Outputs

| Output | What it means |
|---|---|
| `diagnosed` | `true` when a log was found and diagnosed, `false` when the run was a preflight gate or no log was found. |

## What it does not do

- It does not upload your source, your weights, or your dataset. It sends the log text you point
  it at, and nothing else.
- It does not diagnose a placeholder. If the log is missing or under 6 bytes, it says so
  rather than inventing an answer.
- It does not run unverified code. The agent is checked against the published `SHA256SUMS`
  before execution; set `verify-checksum: false` only for a mirror without a manifest, and know
  that you are then executing unverified code on a runner that holds your secrets.

## Privacy

Diagnosis sends log text to the Denpex API. To keep logs on your own infrastructure entirely,
run the agent directly with `DENPEX_LOCAL=1`, which diagnoses on-host with zero egress, see
[the agent docs](https://denpex.com/docs/agent). That mode is not available through this action,
because a GitHub-hosted runner is not your infrastructure.

## Links

| Resource | Description | Link |
| :--- | :--- | :--- |
| **Denpex Platform** | Deterministic GPU and ML training failure diagnostics | [denpex.com](https://denpex.com) |
| **CI & GitHub Action Guide** | Full documentation for CI gates and workflow setup | [denpex.com/docs/ci](https://denpex.com/docs/ci) |
| **Python Agent & CLI** | On-host monitoring, evidence capture, and zero-egress CLI | [denpex.com/docs/agent](https://denpex.com/docs/agent) |
| **REST API Reference** | Programmatic `/api/diagnose` and preflight endpoints | [denpex.com/docs/api](https://denpex.com/docs/api) |
| **VS Code Extension** | Offline GPU crash diagnostics in VS Code | [denpex.com/vscode](https://denpex.com/vscode) |
| **Open VSX Extension** | Extension for Cursor, Windsurf, VSCodium, and Gitpod | [denpex.com/openvsx](https://denpex.com/openvsx) |
| **Failure Encyclopedia** | 500+ documented CUDA, NCCL, Xid, Slurm, and vLLM failures | [denpex.com/failures](https://denpex.com/failures) |
| **Free GPU Tools** | VRAM, NCCL, DeepSpeed ZeRO, and Slurm calculators | [denpex.com/tools](https://denpex.com/tools) |
| **NVIDIA Xid Reference** | Hardware vs software Xid fault routing table | [denpex.com/xid](https://denpex.com/xid) |
| **Release Changelog** | Platform, agent, SDK, and extension release notes | [denpex.com/changelog](https://denpex.com/changelog) |
