# Tutorial: reproducir la buildfarm de RoboStack y publicar a prefix.dev

Este workspace contiene una buildfarm basada en **pixi** + **vinca** + **rattler-build** para generar recetas Conda para ROS (ej. `ros-kilted/`) y construir artefactos **`.conda`** listos para subir a un canal en **https://prefix.dev/**.

> Alcance: este tutorial describe el flujo práctico para **reproducir builds** y **subir paquetes** a prefix.dev. Está basado en los comandos definidos en `ros-kilted/pixi.toml` y en los scripts/CI del repo.

---

## 0) Requisitos

- Linux/macOS/Windows (para build nativa por plataforma). En general:
  - `linux-64` se puede construir en Linux x86_64.
  - `osx-64` / `osx-arm64` se construyen en macOS.
  - `win-64` se construye en Windows.
  - `linux-aarch64` se construye en Linux ARM64.
- Acceso a internet (para descargar dependencias desde `conda-forge` en prefix.dev).
- Una cuenta en **prefix.dev** y un **channel** (por ejemplo: `tu-org/tu-canal`) con un **API key**.

> Nota: este repo usa `rattler-build upload prefix`, que soporta `PREFIX_API_KEY`.

---

## 1) Dónde está el “proyecto” de build

En este workspace, el proyecto está en:

- `ros-kilted/`

Ahí encontrarás:

- `pixi.toml`: define dependencias y tareas (`generate-recipes`, `build`, `check-patches`, etc.).
- `vinca.yaml`, `robostack.yaml`, `rosdistro_snapshot.yaml`: inputs para que `vinca` genere recetas.
- `patch/`: parches que se aplican durante el build.
- `conda_build_config.yaml`: configuración/pinnings de conda-build/rattler-build.

---

## 2) Crear el entorno reproducible con pixi

Desde el directorio del proyecto:

```bash
cd ros-kilted
pixi install
```

Esto crea un entorno con (entre otros) `vinca`, `rattler-build` y `rattler-index`.

Consejo: si quieres ver versiones:

```bash
pixi --version
pixi run rattler-build --version
```

---

## 3) Generar recetas (vinca → `recipes/`)

`vinca` genera un árbol `recipes/<paquete>/recipe.yaml` a partir del rosdistro/config del repo.

Ejemplo para Linux x86_64:

```bash
pixi run vinca --platform linux-64 -m -n
```

- `--platform` controla para qué plataforma se generan las recetas.
- `-m` (multiple) genera múltiples recetas.
- `-n` suele evitar algunos pasos de “normalización”/re-resolución (depende de vinca), y es lo que usa CI en PR.

Si quieres empezar limpio:

```bash
pixi run remove-recipes
```

---

## 4) (Recomendado) Comprobar que los parches aplican

Antes de compilar todo, conviene detectar pronto si un patch ya no aplica.

```bash
pixi run check-patches
```

Esto crea una carpeta `recipes_only_patch/` con recetas mínimas y ejecuta la fase de parches con `rattler-build`.

---

## 5) Construir paquetes `.conda` con rattler-build

### Build de todo lo generado

```bash
pixi run rattler-build build \
  --recipe-dir recipes \
  --target-platform linux-64 \
  -m ./conda_build_config.yaml \
  -c https://repo.prefix.dev/conda-forge \
  --skip-existing
```

- `--skip-existing` evita reconstruir paquetes ya presentes en los channels.
- Los artefactos suelen quedar en `output/<platform>/` (por ejemplo `output/linux-64/`).

### Build de un paquete concreto

El repo define una tarea `build-one`, pero **ojo**: es una “task template” y no está pensada para pasar `--help` (pixi intenta ejecutarla literalmente).

Para construir uno a mano (recomendado):

```bash
PACKAGE=ros-kilted-ros-workspace
pixi run rattler-build build \
  --recipe ./recipes/${PACKAGE}/recipe.yaml \
  --target-platform linux-64 \
  -m ./conda_build_config.yaml \
  -c https://repo.prefix.dev/conda-forge
```

Si el paquete tiene parches locales en `patch/`, asegúrate de que la receta los referencia o de copiarlos al lugar esperado por la receta (según el layout que genere `vinca`).

---

## 6) Subir artefactos a prefix.dev

`rattler-build` soporta directamente prefix.dev:

```bash
pixi run rattler-build upload prefix --help
```

### Autenticación

Configura el API key (en tu shell local o en CI):

```bash
export PREFIX_API_KEY="...tu_api_key..."
```

### Subida

Supongamos que has construido para `linux-64` y tienes archivos en `output/linux-64/`:

```bash
CHANNEL="tu-org/tu-canal"

pixi run rattler-build upload prefix \
  --channel "${CHANNEL}" \
  --skip-existing \
  output/linux-64/*.conda
```

Si quieres sobreescribir lo existente (no recomendado salvo que sepas por qué):

```bash
pixi run rattler-build upload prefix --channel "${CHANNEL}" --force output/linux-64/*.conda
```

---

## 7) Reproducir lo que hace CI (visión general)

En PRs, el workflow hace (por plataforma):

1. `pixi run vinca --platform <platform> -m -n`
2. `pixi run check-patches`
3. (Opcional) restaura un cache de `output/<platform>`
4. `pixi run rattler-build build --recipe-dir recipes --target-platform <platform> ... --skip-existing`
5. Actualiza índice local (cache) con `rattler-index fs <folder>/.. --force`

La publicación a un channel se hace típicamente **en un job separado** (o en otra pipeline) porque requiere secretos (`PREFIX_API_KEY`).

---

## 8) Tips de depuración

- Ver qué se ha construido:
  - `ls output/linux-64 | head`
- Comparar “recetas vs artefactos construidos”:
  - `python build_gap_report.py --output-dir output --recipes-dir recipes`
- Si fallan parches:
  - revisa `patch/` y las secciones `source: ... patches:` de la receta generada.

---

## 9) Checklist rápido (Linux)

```bash
cd ros-kilted
pixi install
pixi run remove-recipes
pixi run vinca --platform linux-64 -m -n
pixi run check-patches
pixi run rattler-build build --recipe-dir recipes --target-platform linux-64 -m ./conda_build_config.yaml -c https://repo.prefix.dev/conda-forge --skip-existing
export PREFIX_API_KEY="..."
pixi run rattler-build upload prefix --channel tu-org/tu-canal --skip-existing output/linux-64/*.conda
```

---

## 10) Próximos ajustes (para esta sesión)

Dime:

1) ¿Qué **channel** quieres usar en prefix.dev (nombre exacto)?
2) ¿Qué **plataforma(s)** vas a construir ahora mismo (linux-64, win-64, etc.)?
3) ¿Quieres **construir todo** o solo un subconjunto de paquetes (y cuál)?

Con eso adapto este tutorial a tu caso con comandos exactos y un “runbook” mínimo.
