# Buildfarm EasyNav + NavMap (ROS kilted) → recetas + build de paquetes

Este documento es un runbook para generar **solo** los paquetes de EasyNavigation (EasyNav) y el mínimo de dependencias necesarias (las que no existan ya en `robostack-kilted`), usando el proyecto `ros-kilted/` de este workspace.

Incluye también un flujo equivalente para **NavMap** (paquetes `navmap_*`), que suele ser una dependencia frecuente de workspaces que consumen EasyNav.

Repositorio de referencia:
- https://github.com/EasyNavigation/EasyNavigation.git (rama `kilted`)

> Nota: En kilted, los paquetes `easynav*` aparecen en `ros-kilted/rosdistro_snapshot.yaml` (release repo `EasyNavigation-release`). Por eso podemos tratarlos como parte del rosdistro snapshot.

---

## 1) Preparación

Desde la raíz del workspace:

```bash
cd ros-kilted
pixi install
```

---

## 2) Selección “solo EasyNav”

`vinca` no acepta un flag tipo `--config`; usa el `vinca.yaml` del directorio que se le pasa con `-d`.

Por eso creamos un directorio dedicado con un `vinca.yaml` mínimo:

- `ros-kilted/easynav_subset/vinca.yaml`

Ahí se define:
- `packages_select_by_deps`: solo `easynav` + `easynav_*`
- `skip_existing`: `https://conda.anaconda.org/robostack-kilted/` (para no regenerar recetas de paquetes ya construidos)
- `rosdistro_snapshot`: `../rosdistro_snapshot.yaml`
- `patch_dir`: `../patch`

Importante (comportamiento observado):
- `-d ./easynav_subset` indica a `vinca` **dónde leer** el `vinca.yaml`.
- Las recetas generadas se escriben en el directorio estándar `ros-kilted/recipes/`.

---

## 2b) Selección “solo NavMap”

Si tu build falla por `find_package(navmap_core)` (o similares), puedes construir primero NavMap como subset, igual que EasyNav.

Perfil mínimo:

- `ros-kilted/navmap_subset/vinca.yaml`

Ahí se define:
- `packages_select_by_deps`: `navmap_core`, `navmap_ros`, `navmap_ros_interfaces`
- `skip_existing`: `https://conda.anaconda.org/robostack-kilted/` (para no regenerar recetas de paquetes ya construidos)
- `rosdistro_snapshot`: `../rosdistro_snapshot.yaml`
- `patch_dir`: `../patch`

Generación de recetas (tras limpiar):

```bash
cd ros-kilted
pixi run remove-recipes
pixi run -v vinca -d ./navmap_subset --platform linux-64 -m -n
```

Verificación rápida:

```bash
find recipes -maxdepth 2 -name recipe.yaml | wc -l
ls -1 recipes | sort
```

---

## 3) Limpiar recetas previas

```bash
cd ros-kilted
pixi run remove-recipes
```

---

## 4) Generar recetas (linux-64)

```bash
cd ros-kilted
pixi run -v vinca -d ./easynav_subset --platform linux-64 -m -n
```

Verificación rápida:

```bash
find recipes -maxdepth 2 -name recipe.yaml | wc -l
ls -1 recipes | head
```

---

## 5) (Opcional) Comprobar parches

El script de este repo `check_patches_clean_apply.py` asume que las recetas están en `./recipes` (directorio raíz), que es justo donde las escribe `vinca`.

```bash
cd ros-kilted
pixi run python check_patches_clean_apply.py
```

Si no hay recetas con `patches:`, el script dirá que no hay nada que testear.

---

## 6) Build con rattler-build (solo subset)

Construir todas las recetas generadas para `linux-64`:

```bash
cd ros-kilted
pixi run rattler-build build \
  --recipe-dir ./recipes \
  --target-platform linux-64 \
  -m ./conda_build_config.yaml \
  -c https://prefix.dev/robostack-kilted \
  -c https://repo.prefix.dev/conda-forge \
  --skip-existing
```

Verificar artefactos producidos:

```bash
ls -1 output/linux-64/*.conda | head
ls -1 output/linux-64/*.conda | wc -l
```

---

## 7) Validación “recetas vs artefactos”

```bash
cd ros-kilted
python build_gap_report.py --platform linux-64 --output-dir output --recipes-dir recipes
```

---

## 8) Publicación a prefix.dev (opcional)

Si quieres subir lo construido a tu canal:

```bash
export PREFIX_API_KEY="..."
CHANNEL="tu-org/tu-canal"

pixi run rattler-build upload prefix --channel "${CHANNEL}" --skip-existing output/linux-64/*.conda
```

---

## Registro de ejecución (esta sesión)

Resultados verificados (linux-64):

- `recipe.yaml` generados: 13
- `.conda` producidos en `ros-kilted/output/linux-64/`: 13
- `build_gap_report.py`: 0 gaps (13/13)

Paquetes construidos (EasyNav):
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
- `ros-kilted-yaets` (dependencia)

Resultados verificados (NavMap, linux-64):

- `recipe.yaml` generados: 3
- `.conda` producidos en `ros-kilted/output/linux-64/`: 3
- `build_gap_report.py`: 0 gaps (3/3)

Paquetes construidos (NavMap):
- `ros-kilted-navmap-core`
- `ros-kilted-navmap-ros`
- `ros-kilted-navmap-ros-interfaces`

Artefactos (linux-64) listados en `output/linux-64/`:
- `ros-kilted-navmap-core-0.4.0-np2py312h2ed9cc7_15.conda`
- `ros-kilted-navmap-ros-0.4.0-np2py312h2ed9cc7_15.conda`
- `ros-kilted-navmap-ros-interfaces-0.4.0-np2py312h2ed9cc7_15.conda`
