# Pixi in this repository

This repository uses **Pixi** to manage a reproducible toolchain (Vinca, rattler-build, rattler-index, etc.) and to provide a consistent workflow for:

- Building ROS 2 kilted packages as `.conda` artifacts
- Indexing a local file-based conda channel
- Uploading artifacts to prefix.dev

Pixi website and docs:

- https://pixi.sh/

## Key concepts (as used here)

- **Workspace**: a folder containing `pixi.toml` (and optionally `pixi.lock`).
- **Channels**: conda package sources used by the solver.
  - In this repo we commonly use:
    - `https://prefix.dev/conda-forge`
    - `https://prefix.dev/robostack-kilted`
    - `https://prefix.dev/irl-kilted`
    - A local file-based channel: `file:///.../ros-kilted/output`
- **Tasks**: named commands in `pixi.toml` under `[tasks]`.
- **Activation scripts**: shell snippets configured via `[activation]` that Pixi runs when activating the environment.

## Common commands

### For users (consumer workspaces)

- Install / solve environment:
  - `pixi install`
- Run a task (e.g., `build`):
  - `pixi run build`
- Start an interactive shell with the Pixi env:
  - `pixi shell`

### For package creators (buildfarm)

From within `ros-kilted/`:

- Install build tooling:
  - `pixi install`
- Generate recipes (via Vinca):
  - `pixi run generate-recipes`
- Build all generated recipes:
  - `pixi run build`
- Index the local channel:
  - `pixi run rattler-index fs ./output --force`

## Troubleshooting

### "No candidates found" after publishing new packages

Pixi may use cached `repodata.json`. If a consumer workspace cannot see a package that you just published, clear repodata cache:

- `pixi clean cache --repodata -y`

Then retry `pixi install`.

### File-based channel paths: use the channel root

A conda channel is expected to contain subdirs like `linux-64/` and `noarch/`.

- Correct: `file:///.../ros-kilted/output`
- Incorrect: `file:///.../ros-kilted/output/linux-64`

Pointing to a subdir breaks lookups for `noarch/repodata.json` and can make solving fail.

### Running tasks from a different directory

If you need to run tasks without `cd`-ing into the workspace directory, use:

- `pixi run -m /path/to/workspace <task>`

Example:

- `pixi run -m /path/to/ros-kilted plansys2-all`
