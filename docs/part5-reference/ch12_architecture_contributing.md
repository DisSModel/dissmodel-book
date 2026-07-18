# Chapter 12: Architecture decisions and how to contribute

*Part V — Migration & Reference*

## Learning objectives

- Understand the design principles guiding the whole ecosystem
- Know how to propose a new satellite package
- Learn the contribution workflow (issues, PRs, tests, docs)

## 15.1 Design principles observed across the ecosystem

1. **Minimal core, additive extensions.** `dissmodel-ca`, `dissmodel-abm`,
   `dissmodel-sysdyn` depend on the core as regular packages — none
   modifies the core directly.
2. **Protective layers over the substrate.** `Society`/`Agent` (Ch. 5) is
   the canonical example: model code never touches `self.gdf` directly.
3. **Configuration decoupled from code.** `dissmodel-configs` (Ch. 9)
   treats parameterization as an external, PR-versioned registry, not
   code embedded in the executor.
4. **One repository, one responsibility.** Each paradigm, each
   application domain, and each operational layer (platform, configs,
   plugin) lives in its own repository.

## 15.2 How to propose a new satellite package

*TODO: write a guide — naming convention (`dissmodel-<paradigm>` or
`diss<domain>`), expected minimal structure (`pyproject.toml`, `src/`,
`examples/{cli,streamlit,notebooks}`, `tests/`, `docs/`, `mkdocs.yml`),
and the platform registration process via `dissmodel-configs`.*

## 15.3 Contribution workflow

*TODO: expand based on the main `dissmodel` repository's `CONTRIBUTING.md`.*

## 15.4 Known risks and areas of attention

- **Bus factor**: maintenance concentrated among few maintainers — newer
  projects should document this explicitly for funders/reviewers.
- **Test coverage**: the core has ~79% coverage; satellite packages don't
  yet expose this metric in a standardized way.
- **`dissmodel-platform` is in MVP phase**: no security hardening —
  should be clearly flagged for anyone evaluating production use.
- **`disscube` is Alpha**: the declarative API is still evolving.

## Summary

*TODO: summarize the chapter's key takeaways.*
