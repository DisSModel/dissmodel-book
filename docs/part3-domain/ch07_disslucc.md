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
percentage per cell), equivalent to the core LUCCME components, following
the three-pillar Demand/Potential/Allocation philosophy described by
Verburg et al. (2006).

**Demand** — the magnitude of change to allocate per time step:

```python
from disslucc_continuous import DemandPreComputedValues, load_demand_csv

demand = DemandPreComputedValues(
    annual_demand  = load_demand_csv("demand.csv", ["f", "d", "outros"]),
    land_use_types = ["f", "d", "outros"],
)
```

**Potential** — per-cell suitability to change, from spatial driving
factors via `PotentialLinearRegression`:

```python
from disslucc_continuous import PotentialLinearRegression, RegressionSpec

potential = PotentialLinearRegression(
    gdf              = gdf,
    land_use_types   = ["f", "d", "outros"],
    land_use_no_data = "outros",
    potential_data   = [[
        RegressionSpec(const=0.7392, betas={
            "assentamen": -0.2193, "uc_us": 0.1754,
            "dist_riobr": 2.388e-7, "fertilidad": -0.1313,
        }),
        RegressionSpec(const=0.267, betas={
            "rodovias": -9.922e-7, "assentamen": 0.2294,
        }),
        RegressionSpec(const=0.0),  # "outros" — no betas
    ]],
)
```

**Allocation** — spatially distributes demand according to potential via
`AllocationClueLike`:

```python
from disslucc_continuous import AllocationClueLike, AllocationSpec

AllocationClueLike(
    gdf             = gdf,
    demand          = demand,
    potential       = potential,
    land_use_types  = ["f", "d", "outros"],
    static          = {"f": -1, "d": -1, "outros": 1},
    complementar_lu = "f",
    cell_area       = 25.0,  # km²
    allocation_data = [[
        AllocationSpec(static=-1, min_value=0, max_value=1, min_change=0, max_change=1),
        AllocationSpec(static=-1, min_value=0, max_value=1, min_change=0, max_change=1),
        AllocationSpec(static=1,  min_value=0, max_value=1, min_change=0, max_change=1),
    ]],
)
```

`PotentialLinearRegression` and `AllocationClueLike` each have a
discrete-package counterpart (`PotentialDLogisticRegression`,
`AllocationDClueSLike` in 7.4) — same role, different algorithm for a
different allocation philosophy.

Running it end-to-end via the CLI (vector substrate, over the `csAC`
dataset shipped in the repo):

```bash
python lab1_vector.py run \
  --input data/input/csAC.zip \
  --param interactive=True

# with calibrated coefficients from a TOML file
python lab1_raster.py run \
  --input data/input/csAC.zip \
  --toml  examples/model.toml
```

The `lucc_benchmark` executor runs vector and raster substrates in a
single pass and compares both against a TerraME/LUCCME reference result,
producing `report.md` (MAE/RMSE/match% per band) and `scatter.png`:

```bash
python -m disslucc_continuous.executors.lucc_benchmark_executor run \
  --input  examples/data/input/csAC.zip \
  --output ./benchmark/ \
  --param  demand_csv=examples/data/input/examples_demand_lab1.csv \
  --param  terrame_reference=benchmark/data/LUCCME_Lab1_2014.zip \
  --param  n_steps=6 \
  --param  tolerance=0.01
```

The automated test suite (`tests/test_benchmark_validation.py`) asserts
MAE/RMSE below `tolerance=0.01` for both `Vector_vs_TerraME` and
`Raster_vs_TerraME` — this is the "benchmark-first validation" principle
the README calls out explicitly: never trust a production run without
first validating it against the TerraME/LUCCME reference.

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

`disslucc-discrete`'s primary validation strategy is cell-by-cell parity
against the original TerraME/LuccME reference implementation — **Lab6**,
run on the Moju dataset (1999–2004). Reproducing the parity check:

```bash
python src/disslucc_discrete/executors/lucc_validation_executor.py run \
  --input  data/cs_moju.zip \
  --output outputs/validation \
  --param  terrame_reference=benchmark/data/Lab6_2004.zip
```

This writes `report.md`, `scatter.png`, `map.png`, and
`lab6_python_2004.zip` to `outputs/validation/`. The integration test
(`tests/test_validation_lab6.py`) instantiates `LuccValidationExecutor`,
runs the simulation over `data/cs_moju.zip`, and asserts:

| Metric | Expected |
|---|---|
| Accuracy | 100.00% |
| Cohen's κ (Kappa) | 1.0000 |
| F1 Score | 1.0000 |
| FP / FN | 0 / 0 |

The README states this reproduces **100% numerical parity at the cell
level** against TerraME/LuccME (Lab6, Moju dataset), at roughly
65 ms/step — and the test runs automatically on every CI build:

```bash
pytest tests/test_validation_lab6.py -v
```

This is the strongest TerraME-equivalence claim in the whole DisSModel
ecosystem: unlike `brmangue-dissmodel`'s continuous numerical comparison
(Chapter 8) or `disslucc-continuous`'s MAE/RMSE tolerance-based benchmark
(7.3), Lab6 parity is exact — 100% accuracy, κ = 1.0000 — because
discrete land-use allocation is a categorical, not continuous, quantity.

## 7.5 Choosing between continuous and discrete

The two packages implement the same Demand/Potential/Allocation
philosophy with different algorithms at each step:

| Component | `disslucc-continuous` | `disslucc-discrete` |
|---|---|---|
| Potential | `PotentialLinearRegression` (or `PotentialCSpatialLagRegression`, `PotentialCSampleBased`) | `PotentialDLogisticRegression` (or `PotentialDNeighSimpleRule`, `PotentialDSampleBased`) |
| Allocation | `AllocationClueLike` (or `AllocationClueLikeSaturation`) | `AllocationDClueSLike` (competition-based CLUE-S) |
| Reference validation | MAE/RMSE tolerance vs. TerraME/LUCCME (7.3) | 100% exact cell-level parity vs. TerraME/LuccME Lab6 (7.4) |

Use **continuous** (`disslucc-continuous`) when the research question is
about *how much* land changes use in a cell — e.g. modeling gradual
deforestation as a shrinking forest percentage, or partial urbanization
of a rural cell. This is the closer analogue to LUCCME's original
component design (linear regression potential + CLUE-like allocation) and
is the appropriate choice whenever cells can legitimately hold more than
one land use simultaneously.

Use **discrete** (`disslucc-discrete`) when the research question is
about *which single* land use dominates a cell — CLUE-S-style regional
allocation where each cell converts wholesale from one category to
another (logistic-regression potential + competition-based allocation).
This is the right choice when ground-truth data or the validation target
(as in Lab6/Moju) is itself categorical — a classified land-cover map,
not a fractional-cover raster — since only a categorical model can be
checked for *exact* cell-level parity the way Lab6 is.

If you're unsure, look at your calibration/validation dataset first: if
it's expressed as one class label per cell, discrete is the model that
can be validated against it exactly; if it's expressed as area/percentage
per cell, continuous is the only one that can represent it without
lossy discretization.

## Exercises

1. Run the continuous vector example over the `csAC` dataset
   (`python lab1_vector.py run --input data/input/csAC.zip --param
   interactive=True`) and separately the discrete Moju example
   (`python src/disslucc_discrete/executors/clue_s_vector_executor.py run
   --input data/cs_moju.zip --output outputs/resultado_moju.gpkg --toml
   examples/moju_model.toml --param demand_csv=examples/data/demand_moju.csv
   --param n_steps=6`). Inspect each output — one is a
   percentage-per-cell field, the other a categorical land-use column.
   Confirm which is which.
2. `PotentialLinearRegression` (continuous) and
   `PotentialDLogisticRegression` (discrete) both take driving-factor
   `betas` per land use. Explain, from the model names alone, why one
   fits a linear regression and the other a logistic regression — what
   does each potential component need to predict?
3. Reproduce the Lab6 parity check (7.4) and read the generated
   `report.md`. Which metric would change first if the Python
   implementation drifted from TerraME's — accuracy, kappa, or F1 — and
   why would a discrete model make all three move together?
4. `disslucc-continuous`'s `lucc_benchmark` accepts a `tolerance`
   parameter (default checked at `0.01` in
   `tests/test_benchmark_validation.py`), while `disslucc-discrete`'s
   Lab6 test asserts exact equality (100.00% accuracy). Why does a
   tolerance make sense for the continuous package but not for the
   discrete one?

## Summary

DisSLUCC is two packages sharing one philosophy: Demand (how much change
to allocate), Potential (which cells are suitable), and Allocation
(spatially distributing the change) — the same three-pillar structure
LUCCME established, now implemented on top of `dissmodel`'s
`ModelExecutor` contract instead of TerraME/Lua. `disslucc-continuous`
answers "how much land use changes per cell" with linear-regression
potential and CLUE-like allocation, validated against TerraME/LUCCME
within an MAE/RMSE tolerance. `disslucc-discrete` answers "which land use
dominates per cell" with logistic-regression potential and
competition-based CLUE-S allocation, and reaches a stronger validation
bar — 100% exact cell-level parity against the original TerraME/LuccME
Lab6 reference on the Moju dataset, automated in CI. Neither package
modifies `dissmodel`'s core; both are additive, `ModelExecutor`-based
packages exactly like `dissmodel-ca` and `dissmodel-sysdyn` — the choice
between them is a modeling decision about your data (fractional vs.
categorical), not an architectural one.
