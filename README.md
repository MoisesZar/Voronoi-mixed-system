# Voronoi Microstructure Generator for Exchange Bias Simulations

Python/Ubermag tool to generate a **2D columnar Voronoi microstructure** for
micromagnetic simulations of exchange bias in granular magnetic systems.

The script builds the geometry, classifies grains into three size categories
(single-domain, ferromagnetic multidomain, antiferromagnetic multidomain),
optimizes the grain size distribution with **Simulated Annealing**, and
returns all the Ubermag objects needed to run the simulation:

- `df.Mesh` with per-cell subregions
- anisotropy dictionary `K_dict`
- random easy-axis field `u`
- exchange dictionary `Adict` (with sign changes at grain boundaries)

> ⚠️ This script **prepares** the micromagnetic system. It does not run the
> field-cooling or hysteresis-loop protocols. Those are driven separately
> once the system is assembled.

---

## Context: why this tool

Exchange bias in granular systems depends on the interplay between grains of
different sizes and magnetic phases. This tool isolates the **geometric**
contribution by generating microstructures where:

- all phases share the same internal properties (`Ms`, `K`, `A`);
- the only difference between phases is the **sign of the exchange coupling**
  between neighboring grains.

This makes it possible to test whether grain-size geometry alone can produce
the experimentally observed exchange-bias phenomenology.

---

## Physics

### Grain classification

Grains are classified by their **equivalent spherical diameter**

    d_eq = (6 V / π)^(1/3)

where `V` is the grain volume, into three categories:

| Category | Condition | Meaning |
|---|---|---|
| `mono` (SD) | `d_eq < t_mono` | single-domain |
| `FM` (MD) | `t_mono ≤ d_eq < t_AFM` | ferromagnetic multidomain |
| `AFM` (MD) | `d_eq ≥ t_AFM` | antiferromagnetic multidomain |

Default thresholds:

    t_mono = Dc = 10 nm
    t_AFM  = 1.4 · Dc = 14 nm

The critical single-domain diameter and the domain-wall width are:

    Dc    = 72 √(A K) / (μ₀ Ms²)
    δ     = π √(A / K)

### Exchange coupling rule

The exchange constant between two neighboring grains depends on their
categories:

| Pair | `mixed` interaction |
|---|---|
| mono – mono | +A |
| mono – FM   | +A |
| FM – FM     | +A |
| any – AFM   | −A |
| AFM – AFM   | −A |

Inside a single Voronoi grain, the exchange is always `+A`.

### Internal properties (identical for all phases)

    Ms = 1e5 A/m
    Bc = 200 mT              →  K = Bc · Ms / 2
    A  = (μ₀ Dc / 72)² · Ms³ / Bc · 2
    d  = 0.5 nm              (simulation cell)

Derived lengths:

    l_ex     = √(2A / (μ₀ Ms²))       ≈ 0.7 nm
    l_domain = π √(A / K)             ≈ domain-wall width
    Tc       = 2.88 · d · A / k_B     ≈ 31.8 K

---

## How the microstructure is built

1. **Random seeds** are placed in the simulation box
   (`n_grains` seeds, columnar or full 3D).
2. **Voronoi tessellation** assigns each mesh cell to its nearest seed.
   - `columnar=True`: 2D Voronoi extended along `z` (grains are columns).
   - `columnar=False`: full 3D Voronoi.
3. **Equivalent diameter** `d_eq` is computed for each grain from its volume.
4. **Classification** into `mono` / `FM` / `AFM` using `t_mono` and `t_AFM`.
5. **Simulated Annealing** moves the seeds to minimize an objective function
   that combines:
   - the squared error in the **number of grains per category**, and
   - the mean squared log-ratio between actual and target diameters.
6. The optimized seeds are used to build:
   - a `df.Mesh` with one **subregion per cell (or column)**,
   - a random easy-axis field `u` per grain,
   - a `K_dict` (anisotropy per subregion),
   - an `Adict` (exchange per pair of neighboring subregions).

---
MoisesZar
moisesg.zarzoza@uclm.es

## Requirements

- Python 3.9+
- [Ubermag](https://ubermag.github.io/) (`oommfc`, `discretisedfield`, `micromagneticmodel`)
- OOMMF installed and available in `PATH`
  (see [OOMMF installation guide](https://math.nist.gov/oommf/))
- `numpy`, `scipy`

```bash
pip install -r requirements.txt

