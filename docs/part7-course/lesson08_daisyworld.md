# Lesson 8 — Daisyworld

*Part VII — TerraME Compatibility Course*

## Lovelock's questions

James Lovelock, a NASA atmospheric chemist analyzing the Martian atmosphere, asked himself: why has the
Earth's surface temperature stayed within a narrow range for the last 3.6 billion years, while the Sun's
heat output increased by about 25% over that period? (the "faint young Sun paradox"). And why does the
oxygen in our atmosphere stay near 21%, while Mars and Venus have atmospheres dominated by CO₂, close to
chemical equilibrium?

Lovelock's answer: the Earth's atmosphere is in an **unnatural low-entropy state**, because life actively
uses it as a source, depository, and transporter of raw materials. A lifeless planet would have an
atmosphere determined by physics and chemistry alone, close to equilibrium — like Mars and Venus.

## Gaia Hypothesis

This led Lovelock to formulate the Gaia Hypothesis: the Earth functions as a strongly interacting system,
with a homeostatic control of the biosphere maintaining environmental conditions favorable to life — a
cybernetic feedback loop (active, but non-teleological — there is no intent, only feedback). Central idea:
biotic factors feed back to control abiotic factors, which in turn determine biological possibilities.

## Daisyworld: the minimal Gaia model

Daisyworld is a hypothetical planet with dark soil, and two species of daisies: black (low albedo, absorb
solar energy, warm the planet) and white (high albedo, reflect solar energy, cool the planet). The model
shows how life can regulate climate with no central coordination:

- When the planet is **cold**: black daisies (which locally warm more) grow more, the planet becomes
  darker, albedo drops, the planet absorbs more sunlight and warms up.
- When the planet is **warm**: white daisies grow more, the planet becomes lighter, albedo rises, and the
  planet cools down.

The result is a self-regulating system: the planet's temperature stays close to the optimum for the
daisies, even with large variations in solar luminosity.

## The equations

Fraction of area covered by black daisies (`αb`) and white daisies (`αw`):

```
dαb/dt = αb (αg · β(Tb) − γ)
dαw/dt = αw (αg · β(Tw) − γ)
```

where `αg` is the fraction of bare soil available (`1 − αb − αw`), `β(T)` is a function that is zero at
5 °C, rises to a maximum of 1 at 22.5 °C, and falls back to zero at 40 °C (the daisies' "growth curve"),
and `γ` is the death rate.

Energy balance (energy arriving must equal energy radiated):

```
S·L·(1 − A) = σ·T⁴
```

- `S`: solar constant; `L`: solar luminosity (the parameter varied in the experiment); `A`: mean
  reflectivity of the planet (albedo); `σ`: Stefan-Boltzmann constant; `T`: effective temperature.

Because different regions of the planet (soil, black daisies, white daisies) sit at different
temperatures, there is heat flow between them, controlled by a parameter `q` (the larger `q`, the larger
the temperature differences between regions; `q = 0` means instantaneous mixing, the whole planet at the
same temperature).

## The classic experiment

1. Seed the planet with a mix of light and dark daisies.
2. Slowly increase solar luminosity (as the real Sun did over 4.6 billion years).
3. Observe: on a lifeless planet, temperature rises monotonically with luminosity. On Daisyworld,
   temperature stays nearly constant over a wide range of luminosity — life is regulating the climate.
4. A variant: let the model reach equilibrium, apply a sudden change in solar input, and observe how
   Daisyworld restores its temperature.

## Practice in DisSModel

The `sysdyn_daisyworld.ipynb` notebook implements exactly these stocks (black daisies, white daisies, bare
soil) and the same energy-balance equation.

## TerraME vs DisSModel: side-by-side code

*Note: in the TerraME ecosystem, `Daisyworld.lua` lives in the `ca` repository, not `sysdyn` — even though
it is a stock-based model (TerraME organized examples by package somewhat less strictly than DisSModel).*

**TerraME** (`ca/lua/Daisyworld.lua`) — the helper functions (`daisyGrowthRate`, `calcTemp`,
`tempNearDaisy`) are `local function`s at the top of the file, and the state update lives inside the
`Timer`:

```lua
local function daisyGrowthRate(tempK)
    local temp = tempK - 273
    if temp > 5.0 and temp < 40.0 then
        return 1 - 0.003265 * (22.5 - temp)^2
    end
    return 0.0
end

Daisyworld = Model{
    whiteArea = 0.40, blackArea = 0.273, emptyArea = 0.327,
    whiteAlbedo = 0.75, blackAlbedo = 0.25, soilAlbedo = 0.50, decayRate = 0.3,
    init = function(model)
        model.timer = Timer{
            Event{action = function()
                model.aveTemp = calcTemp(model.sunLuminosity, model:planetAlbedo())
                model.tempNearWhite = tempNearDaisy(model.aveTemp, model:planetAlbedo(), model.whiteAlbedo)
                model.whiteGrowthRate = daisyGrowthRate(model.tempNearWhite) * model.emptyArea
                model.whiteArea = model.whiteArea + model.whiteArea *
                    (model.whiteGrowthRate - model.decayRate)
                -- (same logic repeated for blackArea)
                model.emptyArea = model.planetArea - (model.blackArea + model.whiteArea)
            end}
        }
    end
}
```

**DisSModel** (`dissmodel-sysdyn/src/.../daisyworld.py`) — the same helper functions become private
module-level functions (`_` prefix), and the same heat-transfer constant (`Q = 2.06e9`) and
Stefan-Boltzmann constant (`SIGMA = 5.67e-8`) reappear:

```python
_SIGMA      = 5.67e-8
_SOLAR_FLUX = 3668.0
_Q          = 2.06e9

def _daisy_growth_rate(temp_k: float) -> float:
    temp_c = temp_k - 273.0
    if 5.0 < temp_c < 40.0:
        return max(0.0, 1.0 - 0.003265 * (22.5 - temp_c) ** 2)
    return 0.0

class Daisyworld(Model):
    def execute(self) -> None:
        pa = self._planet_albedo()
        self.ave_temp = _planet_temp(self.sun_luminosity, pa)

        temp_white   = _local_temp(self.ave_temp, pa, self.white_albedo)
        white_growth = _daisy_growth_rate(temp_white) * self.empty_area
        self.white_area += self.white_area * (white_growth - self.decay_rate)

        temp_black   = _local_temp(self.ave_temp, pa, self.black_albedo)
        black_growth = _daisy_growth_rate(temp_black) * self.empty_area
        self.black_area += self.black_area * (black_growth - self.decay_rate)

        self.empty_area = self.planet_area - (self.white_area + self.black_area)
```

The physical constants (Stefan-Boltzmann, solar flux, heat-transfer coefficient) and the daisy
growth-curve formula (`1 − 0.003265 × (22.5 − T)²`) were copied exactly — including the same "magic
number" `0.003265` from the original Lua.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 4 — System Dynamics](../part2-paradigms/ch04_sysdyn.md)
(`sysdyn_daisyworld.ipynb` notebook)
