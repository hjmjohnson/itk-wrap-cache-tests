# itk-wrap-cache-tests

Standalone validation suite for ITK's CastXML two-level cache and SQLite pkl
database ([ITK PR #6533](https://github.com/InsightSoftwareConsortium/ITK/pull/6533)).
Tests the wrapping scripts **by path** against any ITK checkout — no ITK build
required.

## Run (Linux / macOS)

```bash
export ITK_SOURCE_DIR=/path/to/ITK   # default: ../ITK
python -m pytest
```

Requires Python ≥ 3.11 (ITK's floor) and `pytest`.

## Run (Windows, PowerShell)

```powershell
# one-time setup
winget install Python.Python.3.13 Git.Git   # if not already present
git clone https://github.com/hjmjohnson/itk-wrap-cache-tests
git clone --depth 1 --branch enh/igenerator-sqlite-pkl `
    --filter=blob:none --sparse https://github.com/hjmjohnson/ITK
cd ITK; git sparse-checkout set Wrapping/Generators; cd ..
pip install pytest

# run
cd itk-wrap-cache-tests
$env:ITK_SOURCE_DIR = (Resolve-Path ..\ITK)
python -m pytest
```

Only `Wrapping/Generators/` is needed from ITK — the sparse checkout is
a few MB.  No compiler, CMake, or ITK build is required.  The fake
castxml is a `.cmd` shim on Windows; `smoke_realbuild.sh` (L3) is
Linux-only and not part of the pytest run.

## Layers

| File | Layer | What it proves |
|---|---|---|
| `tests/test_unit_cache.py` | L0 | key/manifest/parse pure functions, Windows path unescaping, incdir fingerprints |
| `tests/test_unit_pkl_db.py` | L0 | schema, WAL (+NFS fallback), upsert, reader self-containment |
| `tests/test_integration.py` | L1 | cold/warm/header-edit/shadowing/upgrade/corruption/eviction/bypass/multi-root with an instrumented fake castxml |
| `tests/test_pyi_generator.py` | L1 | `--prune` gating (external-project safety), DB self-heal, guards |
| `tests/test_concurrency.py` | L2 | 32 parallel pkl writers, keyset-DELETE, racing same-key stores |
| `tests/test_relocate.py` | L4-sim | tar round-trip to a new absolute path — models `actions/cache` / Azure `Cache@2` transport |
| `smoke_realbuild.sh` | L3 | optional real-castxml double build (Linux, needs an ITK build tree) |

The fake castxml (`fake_castxml.py`) logs every invocation, so tests assert
exact subprocess counts: cold = 2 (`-E` + full), L2-hit = 1, L1-hit = 0.
