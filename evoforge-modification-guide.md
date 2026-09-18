# EvoForge — Modification Guide
**Version:** v2.2  
**Package:** `src/evoforge/`  
**Entry point:** `evoforge.cli:main`  
**Dependencies:** `numpy`, `pygame`, `colorsys` (stdlib)

---

## Who This Document Is For

Engineers who want to change agent behavior, evolution parameters, simulation dynamics, or the rendering system. This is not a usage guide. This document tells you where the code lives and exactly where to make each class of change.

---

## Architecture Overview

```
cli.py          → App(config, seed) → mainloop
app.py          → App class — pygame event loop, pause/save/load, FPS
world.py        → World — simulation step, agent lifecycle, plant system
  genome.py     → Genome (neural net + traits), mutation, crossover
  entities.py   → Agent, Plant data classes
  config.py     → WorldConfig — all simulation parameters
  spatial.py    → SpatialHash — fast proximity queries
  mathutil.py   → math utilities (clamp, wrap_angle, sigmoid, tanh)
render.py       → Renderer, Camera — pygame rendering
telemetry.py    → Telemetry — population time series
```

---

## Configuration — `config.py`

`WorldConfig` is the single source of all tunable simulation parameters. It is a frozen dataclass that can be serialized to/from JSON for save files.

```python
@dataclass(slots=True)
class WorldConfig:
    width: int = 1400
    height: int = 900
    initial_agents: int = 220
    initial_plants: int = 700
    max_agents: int = 1800
    max_plants: int = 3500
    plant_spawn_rate: float = 3.2       # plants per step (before modifiers)
    plant_energy_min: float = 18.0
    plant_energy_max: float = 42.0
    cell_size: int = 64                 # SpatialHash cell size (pixels)
    wrap_world: bool = True             # torus topology
    day_length: int = 2400              # steps per day cycle
    mutation_rate: float = 0.09        # probability per gene per reproduction
    mutation_scale: float = 0.18       # magnitude of mutations
    crossover_rate: float = 0.35       # probability of sexual reproduction vs asexual
    telemetry_interval: int = 30       # steps between telemetry snapshots
```

All of these can be overridden via CLI flags. To add a new config field:

1. Add it to `WorldConfig` with a default.
2. Add a CLI flag in `cli.py`.
3. Use `config.your_field` wherever needed in `world.py`.

---

## Neural Network — `genome.py`

### Architecture Constants

```python
INPUTS = 14    # sensor vector size
HIDDEN = 12    # hidden layer neurons
OUTPUTS = 6    # motor output size (4 used, 2 currently unused)
```

**To change network size**, update these constants. `INPUTS` must match the sensor vector built in `World.sense()`. `OUTPUTS` must match the motor assignments in `World.think()`.

### `Traits` (frozen dataclass, `slots=True`)

Twelve heritable physical traits:

| Trait | Range | Effect |
|---|---|---|
| `radius` | 2.6–10.0 | Body size, reach, energy capacity |
| `max_speed` | 0.35–3.8 | Maximum movement speed |
| `turn_rate` | 0.025–0.34 | Steering agility |
| `vision_range` | 30.0–230.0 | Sensor reach |
| `metabolism` | 0.45–2.0 | Movement cost multiplier |
| `fertility` | 0.35–1.85 | Reproduction threshold and cooldown |
| `aggression` | 0.0–1.0 | Attack effectiveness modifier |
| `diet` | 0.0–1.0 | 0=herbivore, 1=carnivore |
| `efficiency` | 0.45–1.4 | Energy gain multiplier |
| `color_h` | 0.0–1.0 (wraps) | Hue — visual identification, lineage signal |
| `color_s` | 0.35–1.0 | Color saturation |
| `color_v` | 0.45–1.0 | Color brightness |

**To add a new trait:**

1. Add it to `Traits` with a range-clamped `clamp_all()` entry.
2. Add random initialization in `Genome.random()`.
3. Decide if it should affect mutation via log-normal (multiplicative) or additive: `color_h` is additive, all others use `value *= exp(normal(0, scale))`.
4. Use the trait in the relevant `World` method.

### `Genome` (dataclass, `slots=True`)

Two-layer fully connected network: `(HIDDEN, INPUTS)` weight matrix `w1`, bias `b1`, `(OUTPUTS, HIDDEN)` weight matrix `w2`, bias `b2`. Activation: `tanh` at both layers.

```python
def forward(self, inputs: np.ndarray) -> np.ndarray:
    hidden = np.tanh(self.w1 @ inputs + self.b1)
    return np.tanh(self.w2 @ hidden + self.b2)
```

**To change the activation function:**
```python
# Replace tanh with sigmoid in the hidden layer:
hidden = 1.0 / (1.0 + np.exp(-np.clip(self.w1 @ inputs + self.b1, -30.0, 30.0)))
```

**To add a third hidden layer:**
1. Add `HIDDEN2 = 8` constant.
2. Add `w3` and `b3` arrays to `Genome`.
3. Update `random()`, `mutate()`, `crossover()`, `to_dict()`, `from_dict()`.
4. Update `forward()`.

### Mutation

Per-trait: log-normal multiplicative mutation (`value *= exp(normal(0, scale))`). `color_h` uses additive (`value += normal(0, scale*0.2)`). Per-weight: additive Gaussian, clipped to `[-5.0, 5.0]`.

`mutation_rate` is the probability per gene/weight. `mutation_scale` is the magnitude.

### Crossover

`Genome.crossover(a, b, rng)`: per-trait uniform crossover (50/50 per trait). Per-weight uniform crossover (50/50 per element). A new lineage is created when the child's `color_h` diverges by more than `0.085` from the parent, or with 0.8% random probability.

---

## Sensor System — `World.sense(agent)`

Builds the 14-element input vector:

| Index | Signal |
|---|---|
| 0 | Energy level (normalized, -1 to 1) |
| 1 | Nearest plant distance (normalized by vision, -1 if none) |
| 2 | Sin of angle to nearest plant |
| 3 | Cos of angle to nearest plant |
| 4 | Nearest prey distance |
| 5 | Sin of angle to nearest prey |
| 6 | Nearest threat distance |
| 7 | Sin of angle to nearest threat |
| 8 | Local density (normalized) |
| 9 | Boundary signal X |
| 10 | Boundary signal Y |
| 11 | Sin of day phase |
| 12 | Cos of day phase |
| 13 | Random noise |

**To add a new sensor input:**

1. Increment `INPUTS` in `genome.py`.
2. Grow the `inputs` array in `sense()` and assign the new index.
3. Existing agents loaded from save files will have mismatched network weights — adding inputs requires re-seeding the simulation.

### Motor Outputs — `World.think(agent)`

```python
outputs = agent.genome.forward(self.sense(agent))
agent.turn = float(outputs[0])
agent.throttle = float((outputs[1] + 1.0) * 0.5)   # mapped to [0, 1]
agent.attack = float((outputs[2] + 1.0) * 0.5)
agent.reproduce_signal = float((outputs[3] + 1.0) * 0.5)
# outputs[4] and outputs[5] are unused — available for new behaviors
```

**To use the spare outputs** (e.g. a communication signal):

```python
agent.signal = float((outputs[4] + 1.0) * 0.5)
```

Then use `agent.signal` in `sense()` as an input for nearby agents (requires adding it to the agent dataclass and the sensor vector).

---

## Movement Cost — `World.move(agent)`

```python
movement_cost = (
    0.008
    + traits.metabolism * 0.007
    + speed * speed * (0.0045 + traits.radius * 0.0009)
    + traits.vision_range * 0.000006
)
agent.energy -= movement_cost
```

Components: base cost + metabolism overhead + speed² cost (larger radius = more expensive movement) + vision overhead.

**To change movement cost scaling:**
```python
movement_cost = (
    0.005                                    # lower base cost
    + traits.metabolism * 0.005
    + speed * speed * (0.003 + traits.radius * 0.0007)
    + traits.vision_range * 0.000004
)
```

---

## Feeding System

### Herbivory — `eat_plants(agent)`

Agents eat one plant per step (first within `reach = radius + 4.0`). Energy gained = `plant.energy × (1 - diet) × efficiency`. Low `diet` trait = herbivore. If `herbivory < 0.08`, skips entirely.

### Carnivory — `attack_agents(agent)`

Attack succeeds based on: `advantage = aggression × diet × (radius/prey.radius) × attack_signal`. Defense = `0.35 + prey.aggression × 0.35`. Damage = `max(0, (advantage - defense) × 9.0 + 0.8)`. Attack cost = `0.12 + radius × 0.015`. If prey dies, attacker gains `prey.max_energy × 0.32 × diet × efficiency`.

**To change the damage formula:**
```python
damage = max(0.0, (advantage - defense) * 12.0)   # increase damage scaling
```

---

## Reproduction — `maybe_reproduce(agent)`

Requirements:
- Population below `max_agents`
- Age >= `int(90 / fertility)` (high fertility = shorter maturity)
- Energy >= `max_energy × (0.68 / fertility)` (high fertility = lower threshold)
- `reproduce_signal >= 0.52`

Sexual reproduction: if a nearby agent exists (within 35px) and `rng.random() < crossover_rate`, crossover is used. Otherwise, asexual (mutated self copy).

Birth cost: `min(energy × 0.46, child_radius × 12 + 38)`. Child spawns near parent with slight angle randomness.

**To change reproductive cost:**
```python
birth_cost = min(agent.energy * 0.38, child_genome.traits.radius * 10.0 + 30.0)
```

---

## Plant System — `World.step()`

Plant spawning accumulates fractionally each step:

```python
self.plant_spawn_accumulator += (
    config.plant_spawn_rate
    * (0.45 + self.daylight)          # 0.45–0.90 multiplier (night/day)
    * weather_multiplier               # 0.3 (blight) / 1.0 (calm) / 2.4 (bloom)
    * max(0.0, 1.0 - len(self.plants) / config.max_plants)  # density cap
)
```

When an agent dies, a plant spawns at its location with 38% probability, energy = `min(45.0, max_energy × 0.12)`.

---

## Weather System — `_update_weather()`

Three states: `"calm"`, `"bloom"` (plant × 2.4), `"blight"` (plant × 0.3). Transitions are random with timers. Plant spawn rate is the main effect.

**To add a new weather state** (e.g. `"storm"` reducing agent speed):

1. Add `"storm"` to the `weather_multiplier` dict with a plant multiplier.
2. Use `self.weather == "storm"` in `move()` to apply a speed penalty.
3. Add the transition in `_update_weather()`.

---

## Save / Load — `world.py`

`World.save(path)` serializes to JSON: config, seed, step count, all agents (with full genomes), all plants, all lineages, records. `World.load(path)` reconstructs from JSON.

`Genome.to_dict()` / `from_dict()` serialize numpy arrays as nested lists. This is readable but not compact — for large populations consider numpy binary format.

---

## Spatial Hash — `spatial.py`

`SpatialHash(cell_size)` divides the world into a grid. `rebuild()` is called every step. `query(x, y, radius)` returns all items in cells overlapping the bounding box — results include items outside the exact radius, so callers do distance-squared checks afterward.

`cell_size=64` (default) is set in `WorldConfig`. Larger cells = faster rebuild, slower queries. For dense populations, smaller cells may help query speed.

---

## Renderer — `render.py`

`Renderer` draws agents (circle + vision arc + direction line), plants (dot), event flash effects (birth/kill/death), HUD (telemetry graph, stats, lineage legend). Colors are computed from `color_h/s/v` traits via `colorsys.hsv_to_rgb`.

**To add a new visual element**, find the relevant draw loop in `Renderer.draw()` and add pygame draw calls. Agent and plant positions come from `world.agents` and `world.plants`.

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Change world size | `WorldConfig.width` / `height` in `config.py` |
| Change initial population | `WorldConfig.initial_agents` / `initial_plants` |
| Change mutation rate or scale | `WorldConfig.mutation_rate` / `mutation_scale` |
| Change crossover probability | `WorldConfig.crossover_rate` |
| Add a new heritable trait | `Traits` dataclass + `clamp_all()` + `Genome.random()` + usage in `world.py` |
| Change network size | `INPUTS` / `HIDDEN` / `OUTPUTS` constants in `genome.py` |
| Change activation function | `Genome.forward()` |
| Add a third hidden layer | `Genome` fields + `forward()` + `mutate()` + `crossover()` + `to_dict/from_dict` |
| Add a new sensor input | `INPUTS` + `World.sense()` new index |
| Use spare motor outputs (4, 5) | `World.think()` — assign `outputs[4]` / `outputs[5]` |
| Change movement cost | `World.move()` `movement_cost` formula |
| Change damage formula | `World.attack_agents()` `damage` calculation |
| Change reproduction cost | `World.maybe_reproduce()` `birth_cost` formula |
| Change plant spawn dynamics | `World.step()` `plant_spawn_accumulator` formula |
| Add a new weather state | `_update_weather()` + `weather_multiplier` dict + optional agent effect in `move()` |
| Add a new visual element | `Renderer.draw()` in `render.py` |

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
