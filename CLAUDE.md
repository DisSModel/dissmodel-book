# CLAUDE.md — context for Claude Code sessions in this repo

## What this repo is

`dissmodel-book` is the **technical reference** book for the DisSModel
ecosystem (a Python-native alternative to TerraME/LUCCME). It is NOT the
didactic textbook — that's the sibling repo `geospatial-modeling-python`.
Do not duplicate that book's pedagogical content here; this book is
reference-oriented (installation, API, architecture, migration).

## Source of truth

The satellite package repos live one level up, as siblings:

```
../../dissmodel/{dissmodel,dissmodel-ca,dissmodel-abm,dissmodel-sysdyn,
                 disslucc-continuous,disslucc-discrete,disscube,
                 brmangue-dissmodel,brmangue-qgis,
                 dissmodel-platform,dissmodel-configs}
```

**Every factual claim, code snippet, CLI command, or config example in a
chapter must be grounded in something that actually exists in one of
these repos** — a README, a docstring, a test, an example script, or a
config file. Never invent a function signature, a CLI flag, or a config
key. If you can't find something in the source repos, leave a `*TODO:*`
marker instead of guessing.

## Chapter → repo mapping

| Chapter | File | Source repo(s) |
|---|---|---|
| 1 | part1-core/ch01_why_dissmodel.md | dissmodel (README, trajectory) |
| 2 | part1-core/ch02_core_concepts.md | dissmodel/src (core/, executor/, geo/) |
| 3 | part1-core/ch03_getting_started.md | dissmodel/examples |
| 4 | part2-paradigms/ch04_ca.md | dissmodel-ca |
| 5 | part2-paradigms/ch05_abm.md | dissmodel-abm |
| 6 | part2-paradigms/ch06_sysdyn.md | dissmodel-sysdyn |
| 7 | part3-domain/ch07_disslucc.md | disslucc-continuous, disslucc-discrete |
| 8 | part3-domain/ch08_coastal_case_study.md | brmangue-dissmodel |
| 9 | part4-infra/ch09_platform.md | dissmodel-platform, dissmodel-configs, brmangue-qgis |
| 10 | part4-infra/ch10_disscube.md | disscube |
| 11 | part5-reference/ch11_migration.md | all repos (cross-cutting) |
| 12 | part5-reference/ch12_architecture_contributing.md | all repos (cross-cutting) |

## Writing conventions

- Chapter titles are **concept-first** ("Cellular Automata", not
  "dissmodel-ca"). The package name is linked once, right under the
  title, not in the title itself.
- Language: **English**.
- Each chapter keeps: Learning objectives → numbered sections → Exercises
  → Summary. Don't drop these sections when filling in a TODO.
- Prefer real, runnable examples pulled from `examples/cli`,
  `examples/notebooks`, or `tests/` in the source repos over invented
  ones.
- When documenting a TerraME/LUCCME equivalence, only claim an
  equivalence if it's stated or clearly implied in the source repo's own
  docs (e.g. brmangue-dissmodel's golden-output validation). Don't assert
  numerical parity that hasn't been demonstrated.

## Workflow

1. Work one chapter at a time, on branch `chapter/<slug>` (see main README).
2. Before writing: read the relevant source repo(s) end to end — README,
   `pyproject.toml`/`src` layout, `examples/`, and `tests/` — don't rely
   on a partial grep.
3. Replace `*TODO:*` markers with real content; keep the ones you can't
   ground in source material, and say explicitly what's missing.
4. Run `mkdocs build --strict` before considering a chapter done — it
   catches broken internal links and pages that fell out of `nav` in
   `mkdocs.yml`.
5. Don't touch chapter numbering, `mkdocs.yml` nav structure, or the
   part groupings without flagging it — that structure was deliberately
   decided and finalized.

## Out of scope for this repo

- Rewriting `geospatial-modeling-python` content.
- Modifying any file inside the sibling package repos (`../../dissmodel/*`)
  — this repo only reads them, never writes to them.
