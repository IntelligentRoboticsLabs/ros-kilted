# EasyNav (EasyNavigation) on ROS 2 kilted

This documentation covers two related workflows:

1. **For users**: how to set up a Pixi-based ROS workspace that consumes EasyNav/NavMap binaries and builds your own code with `colcon`.
2. **For package creators**: how to build EasyNav/NavMap packages in this repository (`ros-kilted/`) and publish them to prefix.dev.

Project websites:

- EasyNavigation (EasyNav): https://easynavigation.github.io/
- NavMap: https://easynavigation.github.io/ (NavMap is part of the EasyNavigation ecosystem)

Pixi background and common troubleshooting lives in:

- `irl-docs/pixi.md`

---

## 1) What is this?

EasyNav is a navigation stack developed by IRL (and collaborators) for ROS 2.

In this repo we build EasyNav-related ROS packages as Conda artifacts (`.conda`) and publish them to prefix.dev (channel `irl-kilted`).

---

## 2) For users: workspace with Pixi + colcon

This folder contains a ready-to-use workspace template:

- `irl-docs/easynav/pixi.toml`
- `irl-docs/easynav/activate.sh`

It is aligned with the workspace used in this repo (`easynav_plugins_ws/`).

### 2.1 Create a new workspace

Create a new folder for your workspace and copy the template files:

```bash
mkdir -p ~/easynav_plugins_ws
cd ~/easynav_plugins_ws

cp /path/to/pixi-buildfarm/ros-kilted/irl-docs/easynav/pixi.toml ./pixi.toml
cp /path/to/pixi-buildfarm/ros-kilted/irl-docs/easynav/activate.sh ./activate.sh
chmod +x ./activate.sh
```

Then update the local file-based channel path if needed:

- `file:///.../ros-kilted/output`

Important: use the **channel root**, not `.../output/linux-64`. See `irl-docs/pixi.md`.

What the template does by default:

- Prefers Zenoh (`RMW_IMPLEMENTATION=rmw_zenoh_cpp`).
- Pins `eigen < 5` to avoid CMake falling back to system Eigen via PCL.
- Enables bash completion for `ros2` (via `argcomplete`) in interactive shells.

### 2.2 Install dependencies

```bash
pixi install
```

### 2.3 Build your ROS workspace

Add your ROS packages under `src/` (as in a normal colcon workspace) and build:

```bash
pixi run build
```

After the first build, the activation script will source `install/setup.bash` automatically.

### 2.4 Running EasyNav

Exact commands depend on your workspace and launch files.

As a rule of thumb:

- Use `ros2 launch ...` for launching systems.
- Use `ros2 run ...` for single executables.

If you need a curated set of binaries from `irl-kilted`, keep the channel order in `pixi.toml` so local artifacts override remote ones.

---

## 3) For package creators: buildfarm in this repo

All buildfarm work happens under `ros-kilted/`.

### 3.1 Quick path (Pixi tasks)

```bash
cd ros-kilted
pixi install
pixi run easynav-all
```

This runs, in order: clean → generate recipes → build → index.

### 3.2 How the subset is defined

The EasyNav subset is defined in:

- `ros-kilted/easynav_subset/vinca.yaml`

Notes:

- Vinca reads `vinca.yaml` from the directory passed via `-d`.
- Generated recipes always go into `ros-kilted/recipes/`.

### 3.3 Manual flow (if you need it)

Generate recipes:

```bash
cd ros-kilted
pixi run remove-recipes
pixi run -v vinca -d ./easynav_subset --platform linux-64 -m
```

Build `.conda` artifacts:

```bash
pixi run rattler-build build \
  --recipe-dir ./recipes \
  --target-platform linux-64 \
  -m ./conda_build_config.yaml \
  -c https://prefix.dev/robostack-kilted \
  -c https://repo.prefix.dev/conda-forge \
  --skip-existing
```

Index the local channel:

```bash
pixi run rattler-index fs ./output --force
```

### 3.4 Upload to prefix.dev

Typical upload to `irl-kilted`:

```bash
cd ros-kilted
export PREFIX_API_KEY="..."

pixi run rattler-build upload prefix \
  --channel irl-kilted \
  --force \
  output/linux-64/*.conda
```

If consumers can't see newly-published packages, clear repodata cache on the consumer side:

```bash
pixi clean cache --repodata -y
```

1) Reintenta el mismo comando (suele funcionar tras unos intentos).
2) Si quieres evitar “burst”, sube 1 fichero por petición y con backoff:

```bash
cd ros-kilted
export PREFIX_API_KEY="..."

for pkg in output/linux-64/*.conda; do
  for attempt in 1 2 3 4 5; do
    if pixi run rattler-build upload prefix --channel irl-kilted --force "$pkg"; then
      break
    fi
    sleep $((attempt * 2))
  done
done
```


---

## Real incident (debug): Eigen from system vs Pixi

We hit a recurring issue when building consumer workspaces (for example `easynav_plugins_ws`): the build was mixing system Eigen headers (`/usr/include/eigen3`) with the Pixi environment Eigen headers (`$CONDA_PREFIX/include/eigen3`).

### Symptoms

- Eigen compilation errors like `EIGEN_NOEXCEPT does not name a type`, redefinitions, etc.
- `build/**/flags.make` explicitly contains `-isystem /usr/include/eigen3`.

### Root causes and applied fixes

1) **Contaminated CMake exports (producer-side)**

Some packages (observed in `ros-kilted-navmap-ros` and `ros-kilted-easynav-common`) were exporting `INTERFACE_INCLUDE_DIRECTORIES` containing `;/usr/include/eigen3` in installed `*.cmake` files.

Fix applied in the recipes:

- In [recipes/ros-kilted-navmap-ros/recipe.yaml](../recipes/ros-kilted-navmap-ros/recipe.yaml) and [recipes/ros-kilted-easynav-common/recipe.yaml](../recipes/ros-kilted-easynav-common/recipe.yaml):
  - bumped `build:number` to `19`
  - added explicit `eigen` dependency in `host` and `run`
  - added a `build:post_process` step removing the exact occurrence `;/usr/include/eigen3` from installed `*.cmake`

2) **PCL forces `find_package(Eigen3 3.3 ...)` (consumer-side)**

Even after fixing EasyNav/NavMap exports, `PCLConfig.cmake` can force `find_package(Eigen3 3.3 REQUIRED NO_MODULE)`.

If the solver installs Eigen 5.x, Eigen 5's `Eigen3ConfigVersion.cmake` is not considered compatible with the `3.3` request, and CMake may fall back to system Eigen — reintroducing `/usr/include/eigen3`.

Recommended consumer fix:

- Pin in Pixi: `eigen = "<5"` (for example 3.4.0).

### Quick verification

- Check `build/**/flags.make` and confirm `/usr/include/eigen3` is **not** present.
- Alternatively, create a small CMake project that does `find_package(PCL CONFIG REQUIRED)` and verify `Eigen3::Eigen` points at `$CONDA_PREFIX/include/eigen3`.

---

## Execution log (this session)

Verified results (linux-64):

- `recipe.yaml` generated: 13
- `.conda` produced under `ros-kilted/output/linux-64/`: 13
- `build_gap_report.py`: 0 gaps (13/13)

Built packages (EasyNav):
- `ros-kilted-easynav`
- `ros-kilted-easynav-common`
- `ros-kilted-easynav-controller`
- `ros-kilted-easynav-core`
- `ros-kilted-easynav-interfaces`
- `ros-kilted-easynav-localizer`
- `ros-kilted-easynav-maps-manager`
- `ros-kilted-easynav-planner`
- `ros-kilted-easynav-sensors`
- `ros-kilted-easynav-support-py`
- `ros-kilted-easynav-system`
- `ros-kilted-easynav-tools`
- `ros-kilted-yaets` (dependency)

Verified results (NavMap, linux-64):

- `recipe.yaml` generated: 3
- `.conda` produced under `ros-kilted/output/linux-64/`: 3
- `build_gap_report.py`: 0 gaps (3/3)

Built packages (NavMap):
- `ros-kilted-navmap-core`
- `ros-kilted-navmap-ros`
- `ros-kilted-navmap-ros-interfaces`

Artifacts (linux-64) listed under `output/linux-64/`:
- (Example) `ros-kilted-navmap-core-0.4.0-*_19.conda`
- (Example) `ros-kilted-navmap-ros-0.4.0-*_19.conda`
- (Example) `ros-kilted-navmap-ros-interfaces-0.4.0-*_19.conda`
