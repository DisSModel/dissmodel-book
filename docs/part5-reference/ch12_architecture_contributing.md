# Chapter 12: Architecture decisions and how to contribute

*Part V — Migration & Reference*

## Learning objectives

- Understand the design principles guiding the whole ecosystem
- Know how to propose a new satellite package
- Learn the contribution workflow (issues, PRs, tests, docs)

## 12.1 Design principles observed across the ecosystem

1. **Minimal core, additive extensions.** `dissmodel-ca`, `dissmodel-abm`,
   `dissmodel-sysdyn` depend on the core as regular packages — none
   modifies the core directly.
2. **Protective layers over the substrate.** `Society`/`Agent` (Ch. 6) is
   the canonical example: model code never touches `self.gdf` directly.
3. **Configuration decoupled from code.** `dissmodel-configs` (Ch. 9)
   treats parameterization as an external, PR-versioned registry, not
   code embedded in the executor.
4. **One repository, one responsibility.** Each paradigm, each
   application domain, and each operational layer (platform, configs,
   plugin) lives in its own repository.

## 12.2 How to propose a new satellite package

**Naming.** The ecosystem doesn't use one single naming rule — it's
close to consistent, not exact:

| Package | Pattern |
|---|---|
| `dissmodel-ca`, `dissmodel-sysdyn`, `dissmodel-abm` | `dissmodel-<paradigm>` |
| `disslucc-continuous`, `disslucc-discrete` | `disslucc-<allocation-style>` (a domain family of its own) |
| `brmangue-dissmodel` | `<domain>-dissmodel` (reversed order) |
| `disscube` | standalone, no `dissmodel`/domain prefix |
| `dissmodel-platform`, `dissmodel-configs` | `dissmodel-<infra-role>` |

Pick whichever reads most naturally for what you're building: a new
*paradigm* extension (like CA/sysdyn) fits `dissmodel-<paradigm>`; a new
*application domain* built on top of `dissmodel` (like BR-MANGUE) can
follow either `dissmodel-<domain>` or `<domain>-dissmodel` — the
ecosystem itself isn't strict about the order.

**Minimal structure**, based on what every satellite package that has
one actually ships (not all of them have every piece — see the gaps
noted below):

```
your-package/
  pyproject.toml     # name, dependencies (dissmodel as a regular dep), version
  src/your_package/
    models/           # science layer only — Model/SpatialModel/RasterModel subclasses
    executors/         # infrastructure layer, if you expose a ModelExecutor (Chapter 2)
  examples/
    cli/                # or executors' own CLI via run_cli — see Chapter 3, 3.3
    streamlit/
    notebooks/
  tests/
  docs/ + mkdocs.yml   # present in dissmodel-ca, dissmodel-sysdyn, disscube;
                        # dissmodel-abm ships docs/ (agent.md) without mkdocs.yml;
                        # absent entirely in disslucc-discrete and brmangue-dissmodel
```

`dissmodel-platform` is the one exception worth calling out: it has no
`pyproject.toml` at all, because it's a deployed service (Docker Compose
+ FastAPI), not a `pip`-installable package — its "installation" is
`git clone` + `docker compose up` (Chapter 9, 9.3), not `pip install`.

**Registering with the platform.** A new package becomes runnable on the
platform only by adding a TOML file to `dissmodel-configs/models/`
following the `<model>_<substrate>.toml` convention and structure from
Chapter 9 (9.5) — `executor_module`, `name`, `class`, `package` (a
`git+https://` URL), and a `[model.parameters]` table. That file is
merged via a PR to `dissmodel-configs`; there is no other registration
path.

## 12.3 Contribution workflow

Based on the main `dissmodel` repository's `CONTRIBUTING.md` (the
canonical reference — satellite packages are expected to follow the same
shape):

**Reporting bugs.** Open a GitHub issue with a descriptive title, steps
to reproduce, expected vs. actual behavior, and environment details (OS,
Python version, DisSModel version).

**Suggesting enhancements.** Open an issue tagged "enhancement";
explain why the feature is useful and how it should work.

**Pull requests.**
1. Fork the repository, branch from `main`.
2. Add tests for any new code.
3. Ensure the test suite passes (`pytest`).
4. Follow the project's coding style.
5. Open the PR.

**Development setup:**

```bash
git clone https://github.com/DisSModel/dissmodel.git
cd dissmodel
python -m venv venv && source venv/bin/activate
pip install -e ".[dev]"
pytest tests/
```

**Coding standards.** PEP 8, type hints where possible, NumPy-style
docstrings for new functions/classes.

**Docstring examples are executable.** This is the one contribution
detail worth calling out explicitly: NumPy-style docstring examples using
`>>>` prompts are run as doctests in CI (`pytest --doctest-modules
dissmodel`) and must be fully self-contained — every name they use must
be defined within the example itself, and expected output must match
exactly. If an example needs objects from a broader context (an existing
`GeoDataFrame`, `Environment`, or model instance), use a plain
` ```python ` fenced block instead of `>>>` — both render correctly in
the mkdocstrings-generated API reference, but only the doctest form is
checked in CI. Writing a `>>>` example that isn't actually runnable will
fail CI, not just look sloppy.

## 12.4 Known risks and areas of attention

- **Bus factor**: maintenance concentrated among few maintainers — newer
  projects should document this explicitly for funders/reviewers.
- **Test coverage**: the core has ~79% coverage; satellite packages don't
  yet expose this metric in a standardized way.
- **`dissmodel-platform` is in MVP phase**: no security hardening —
  should be clearly flagged for anyone evaluating production use.
- **`disscube` is Alpha**: the declarative API is still evolving.

## Summary

DisSModel's architecture is held together by convention more than by
enforcement: a minimal, dependency-only core; satellite packages that
extend it additively rather than modifying it; configuration
(`dissmodel-configs`) kept out of executor code entirely; and one
repository per paradigm/domain/operational layer. None of this is unique
per package — `dissmodel-ca`, `dissmodel-sysdyn`, `dissmodel-abm`,
`disslucc-continuous`/`disslucc-discrete`, `brmangue-dissmodel`, and
`disscube` all reuse the same `Model`/`SpatialModel`/`RasterModel`/
`ModelExecutor` contracts from Chapter 2, which is precisely what makes a
new satellite package addable without touching `dissmodel` itself —
register a TOML in `dissmodel-configs` and it's runnable on the platform.
Contributing follows the same standard open-source shape (issues, PRs,
tests, PEP 8) with one CI-enforced detail specific to this codebase:
`>>>` docstring examples are executed as doctests and must be
self-contained, or CI fails. The honest caveats belong in this chapter
too, not hidden: the platform is MVP-stage with no security hardening,
`disscube`'s declarative API is still Alpha, satellite packages don't yet
report test coverage in a standardized way, and `dissmodel-abm` itself is
explicit about what it doesn't cover yet — no raster substrate, no social
networks, no state machines (Chapter 6, 6.6). Anyone evaluating this
ecosystem for production use should treat those gaps as current state,
not oversight.
