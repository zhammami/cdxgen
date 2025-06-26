# Python SBOM generation modes

This document summarises how `cdxgen` creates a Software Bill of Materials for Python projects. Behaviour varies depending on available manifest files and command line options such as `--install-deps`.

## Manifest detection

The tool recognises many Python manifests including:

- `pyproject.toml`
- `setup.py`
- `requirements.txt` or `requirements/*.txt`
- `poetry.lock`
- `pdm.lock`
- `uv.lock`
- `Pipfile` / `Pipfile.lock`
- `bdist_wheel` metadata (`METADATA`, `.whl`, `.egg-info`)
- Pixi's `pixi.toml` and `pixi.lock`

If `pixi.toml` or `pixi.lock` is found, SBOM generation is delegated to **createPixiBom** which parses or generates the `pixi.lock` file.

## Summary of behaviours

The table below outlines how the main combinations are handled.

| Primary manifest | Lock file present | `--install-deps` | Behaviour |
|------------------|------------------|-----------------|-----------|
| `pyproject.toml` | `poetry.lock`/`pdm.lock`/`uv.lock` | either | lock file parsed via `parsePyLockData`. In `--deep` mode or when the lock file lacks dependency information, packages are installed and `pip freeze` is executed. |
| `pyproject.toml` | none and `uv` mode | true | `uv sync` is run to create `uv.lock`, which is then parsed. |
| `pyproject.toml` | none | true | packages installed with `pip` into a temporary virtual environment and frozen; AST analysis via `getPyModules` used in deep mode or when needed. |
| `pyproject.toml` | none | false | only direct imports detected by `getPyModules` (if `--deep`); no transitive dependencies. |
| `setup.py` | n/a | true | project installed in a virtual environment and frozen to obtain dependencies. |
| `setup.py` | n/a | false | direct dependencies detected via AST if `--deep`. |
| `requirements*.txt` | n/a | true | each file installed and frozen to capture transitive dependencies. Fallback to manual parsing when installation fails. |
| `requirements*.txt` | n/a | false | files parsed directly which yields only direct dependencies. |
| `Pipfile` | `Pipfile.lock` | true | `pipenv install` creates/updates `Pipfile.lock`, which is parsed. |
| `pixi.toml` | `pixi.lock` | true/false | handled by **createPixiBom**. If the lock file is missing and `--install-deps` is true, it is generated. |

Transitive dependencies are included whenever a lock file with dependency information is parsed or when package installation and `pip freeze` succeed. When `--no-install-deps` is used and no lock file is available, only direct dependencies are listed.

## CBOM generation

A cryptography BOM can be produced for Python using either the `cbom` alias or the `--include-crypto` option. The alias automatically enables evidence and formulation data and sets the CycloneDX specification to 1.6.

## Environment controls

Environment variables can adjust the behaviour when packages are installed:

- `PIP_INSTALL_ARGS` – extra arguments passed to `pip install`
- `PIP_TARGET` – target directory for pip installations
- `PYPI_URL` – custom Python package index

Refer to `docs/ENV.md` for full details.
