# Chapter 7: DisSLUCC — Land Use and Cover Change Modeling

*Part III — Domain Modeling: Land Use & Coastal Systems*

## Learning objectives

- Understand what "DisSLUCC" means as a family of LUCC modeling approaches
- Situate DisSLUCC relative to the original LUCCME (INPE/CCST)
- Choose between continuous and discrete allocation for a given problem
- Run a minimal model of each kind

## 7.1 What DisSLUCC is

**DisSLUCC** is not a single package — it's the name for the pair of
libraries that implement spatially explicit Land Use and Cover Change
(LUCC) modeling on top of `dissmodel`, mirroring the two allocation
philosophies historically supported by LUCCME:

| Approach | Package | Style | Unit of allocation |
|---|---|---|---|
| Continuous | `disslucc-continuous` | LUCCME-like | Area/percentage per cell |
| Discrete | `disslucc-discrete` | CLUE-S-like | One land use per cell |

Both packages are built directly on top of DisSModel and depend on it as a
regular dependency, following the same additive-package philosophy as
`dissmodel-ca` and `dissmodel-abm`.

## 7.2 Mapping to the original ecosystem

| Original ecosystem (INPE/CCST) | LambdaGeo ecosystem | Role |
|---|---|---|
| TerraME | `dissmodel` | Generic framework for dynamic spatial modeling |
| LUCCME | `disslucc-continuous` | Domain-specific environment for continuous LUCC modeling |
| — | `disslucc-discrete` | CLUE-S-like discrete allocation (no direct 1:1 LUCCME equivalent, but same problem family) |
| TerraLib | `geopandas`/`shapely` | Geographic data handling |
| FillCell | `disslucc_continuous.io` | Cellular space preparation utilities |

## 7.3 Continuous allocation (`disslucc-continuous`)

Implements spatially explicit components for **continuous** LUCC (area or
percentage per cell), equivalent to the core LUCCME components.

*TODO: expand with a real execution example.*

## 7.4 Discrete allocation (`disslucc-discrete`)

Implements a **CLUE-S-like allocation algorithm** with logistic regression
for potential estimation, following the philosophy described by Verburg
et al. (2002). Focuses on **discrete** land use change — one land use per
cell.

### Running a simulation (Moju example)

```bash
python src/disslucc_discrete/executors/clue_s_vector_executor.py run \
  --input data/cs_moju.zip \
  --output outputs/resultado_moju.gpkg \
  --toml examples/moju_model.toml \
  --param demand_csv=examples/data/demand_moju.csv \
  --param n_steps=6
```

### Configuration (model.toml)

Model behavior is defined in a TOML file: land use types, logistic
regression coefficients (betas), elasticities, and the transition matrix.

```toml
[model]
land_use_types = ["f", "d", "o"]
region_attr    = "region"

[[model.potential]]
const      = -2.3418
elasticity = 0.0
[model.potential.betas]
media_decl = -0.0272
dist_br    = 3.1031

[model.allocation]
max_difference   = 10.0
max_iteration    = 1000
factor_iteration = 0.0001
```

### Demand (demand.csv)

One column per land use type, one row per time step:

```csv
f,d,o
5706,205,3
5658,253,3
5611,300,3
```

### Lab6 validation (TerraME)

*TODO: expand the validation section against the TerraME reference lab.*

## 7.5 Choosing between continuous and discrete

*TODO: write a decision guide — when does a research question call for
percentage-per-cell allocation vs. a single dominant land use per cell?*

## Exercises

*TODO: write exercises.*

## Summary

*TODO: summarize the chapter's key takeaways.*
