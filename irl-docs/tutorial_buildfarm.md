# Tutorial: run the buildfarm and publish to prefix.dev

This workspace contains a buildfarm based on **Pixi** + **Vinca** + **rattler-build**.
It generates Conda recipes (for example under `ros-kilted/recipes/`) and produces **`.conda`** artifacts that can be indexed into a local file-based channel and optionally uploaded to **https://prefix.dev/**.

Scope: this is the shared, repo-level runbook. Software-specific details live in:

- `irl-docs/easynav/buildfarm_easynav.md`
- `irl-docs/plansys2/buildfarm_plansys2.md`

Pixi background and common troubleshooting lives in:

- `irl-docs/pixi.md`

---

## 0) Requirements

- A native build host for your target platform:
  - `linux-64` builds on Linux x86_64.
  - `osx-64` / `osx-arm64` build on macOS.
  - `win-64` builds on Windows.
  - `linux-aarch64` builds on Linux ARM64.
- Internet access (to fetch dependencies from upstream channels).
- Optional: a prefix.dev account + channel and an API key.

This repo uses `rattler-build upload prefix` which reads `PREFIX_API_KEY`.

---

## 1) Where the build project lives

The main build project is:

- `ros-kilted/`

Key files/folders:

- `pixi.toml`: environment + tasks.
- `vinca.yaml`, `robostack.yaml`, `rosdistro_snapshot.yaml`: inputs for recipe generation.
- `patch/`: patch files.
- `conda_build_config.yaml`: variant/pinning config (important for reproducible builds).

---

## 2) Create the Pixi environment

```bash
cd ros-kilted
pixi install
```

Useful version checks:

```bash
pixi --version
pixi run rattler-build --version
```

---

## 3) Generate recipes (Vinca → `recipes/`)

Generate recipes for Linux x86_64:

```bash
cd ros-kilted
pixi run remove-recipes
pixi run vinca --platform linux-64 -m -n
```

Notes:

- `--platform` controls the platform the recipes target.
- `-m` (multiple) generates multiple recipes.
- `-n` matches what CI typically uses for PR builds.

---

## 4) (Recommended) Check patches apply

```bash
cd ros-kilted
pixi run check-patches
```

---

## 5) Build `.conda` artifacts (rattler-build)

Build everything under `recipes/`:

```bash
cd ros-kilted
pixi run rattler-build build \
  --recipe-dir recipes \
  --target-platform linux-64 \
  -m ./conda_build_config.yaml \
  -c https://prefix.dev/robostack-kilted \
  -c https://repo.prefix.dev/conda-forge \
  --skip-existing
```

Artifacts end up in `output/<platform>/` (for example `output/linux-64/`).

Build a single recipe manually:

```bash
cd ros-kilted
PACKAGE=ros-kilted-ros-workspace
pixi run rattler-build build \
  --recipe ./recipes/${PACKAGE}/recipe.yaml \
  --target-platform linux-64 \
  -m ./conda_build_config.yaml \
  -c https://prefix.dev/robostack-kilted \
  -c https://repo.prefix.dev/conda-forge
```

---

## 6) Index a local file-based channel (optional, but recommended)

If you want consumer workspaces to install from your local output folder, index it:

```bash
cd ros-kilted
pixi run rattler-index fs ./output --force
```

Important: consumer environments must point to the **channel root** (the folder that contains both `linux-64/` and `noarch/`).

---

## 7) Upload artifacts to prefix.dev

Authentication:

```bash
export PREFIX_API_KEY="...your_api_key..."
```

Upload `.conda` files from a platform subdir:

```bash
cd ros-kilted
CHANNEL="your-org/your-channel"

pixi run rattler-build upload prefix \
  --channel "${CHANNEL}" \
  --skip-existing \
  output/linux-64/*.conda
```

Publish to `irl-kilted` (overwrite enabled):

```bash
cd ros-kilted
export PREFIX_API_KEY="...your_api_key..."

pixi run rattler-build upload prefix \
  --channel irl-kilted \
  --force \
  output/linux-64/*.conda
```

If consumers can't see the new packages, the common fix is repodata cache cleanup on the consumer side:

```bash
pixi clean cache --repodata -y
```

---

## 8) What CI does (high level)

Per platform, CI typically runs:

1. `pixi run vinca --platform <platform> -m -n`
2. `pixi run check-patches`
3. `pixi run rattler-build build --recipe-dir recipes --target-platform <platform> ... --skip-existing`
4. `pixi run rattler-index fs ./output --force`

Publishing is usually a separate job/pipeline because it needs secrets (`PREFIX_API_KEY`).

---

## 9) Debugging tips

- Inspect produced artifacts:
  - `ls -1 output/linux-64/*.conda | head`
- Compare “recipes vs built artifacts”:
  - `python build_gap_report.py --platform linux-64 --output-dir output --recipes-dir recipes`
- If patches fail:
  - check `patch/` and the generated recipe `source: ... patches:`.

### Eigen: detect and avoid system contamination

If a consumer workspace fails with Eigen-related errors, a common cause is mixing include trees:

- System Eigen: `/usr/include/eigen3`
- Pixi/conda Eigen: `$CONDA_PREFIX/include/eigen3`

Checklist:

- Search for `/usr/include/eigen3` in `build/**/flags.make`.
- If present, identify which dependency injects `-isystem /usr/include/eigen3`.

---

## 10) Quick checklist (Linux)

```bash
cd ros-kilted
pixi install
pixi run remove-recipes
pixi run vinca --platform linux-64 -m -n
pixi run check-patches
pixi run rattler-build build --recipe-dir recipes --target-platform linux-64 -m ./conda_build_config.yaml -c https://prefix.dev/robostack-kilted -c https://repo.prefix.dev/conda-forge --skip-existing
pixi run rattler-index fs ./output --force

export PREFIX_API_KEY="..."
pixi run rattler-build upload prefix --channel your-org/your-channel --skip-existing output/linux-64/*.conda
```

Platform note: platform selection happens in the commands that support it (`vinca --platform ...` and `rattler-build --target-platform ...`).
