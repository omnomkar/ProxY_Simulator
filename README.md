# ProxY_Simulator

Agent-based simulation of proxy distribution under censorship, built to evaluate how distribution strategies hold up against adversaries that observe assignments and block what they find.

Supporting code for **"The Game Has Changed: Revisiting Proxy Distribution and Game Theory"** — , Omkar Fulsundar, Hassan Fares, Nicholas Hopper (University of Minnesota), FOCI 2026.
[Paper](https://www.petsymposium.org/foci/2026/foci-2026-0003.pdf) · [Canonical lab repository](https://github.com/hoppernj/ProxySimulator)

---

## Background

The proxy distribution problem: a distributor hands out proxies (bridges, relays, VPN endpoints) to censored clients, while a censor poses as a client to enumerate and block them. Give out too many and the censor learns them all; give out too few and honest users go unconnected.

The paper revisits the game-theoretic framework of *Enemy at the Gateways* (Nasr et al., 2019) and extends it in three directions the original model didn't cover:

- **Ephemeral proxies with high churn** — modern schemes (SpotProxy-style cloud spot VMs, Snowflake volunteers) rotate proxies continuously rather than treating them as long-lived.
- **Stronger and more varied censors** — not one rational adversary, but several operating simultaneously against different client populations, with different budgets and targeting logic.
- **Collateral damage** — censors block innocent infrastructure, and that cost has to appear in the model.

This repository holds the simulation code used to explore those questions.

---

## What's in here

| Module | What it does |
|---|---|
| `MultiCensor_Simulations/` | Primary simulator. Multiple censors act concurrently against partitioned client populations. This is the module to run. |
| `Minimized_Spotproxy_Version/` | Single-censor baseline. Same model without client partitioning — kept as the control condition. |
| `sim_core/` | Live reverse proxy over Docker containers, demonstrating backend rotation against real HTTP traffic rather than in simulation. |

---

## The model

Simulation state lives in a Django ORM over SQLite, so every step of a run is a queryable database rather than in-memory objects — assignment history, block times, and per-client state survive the run and can be inspected afterward.

**Entities** (`assignments/models.py`)

- `Proxy` — address, active/blocked flags, creation and block timestamps. Lifetime is derived from the gap between the two.
- `Client` — address, whether it is a censor agent in disguise, count of blocked proxies it has observed, accumulated credits, and (in `MultiCensor_Simulations`) an indexed censor-group label.
- `Assignment` — the client↔proxy edge. Every distribution decision writes one.

**Distribution strategies** (`--distributor`)

Two scored, adaptive strategies:

| Strategy | Behavior |
|---|---|
| `strict` | Heavily penalizes clients with a history of burned proxies. Conservative; protects the pool at the cost of connectivity. |
| `kind` | Uniform weights. Serves more clients, exposes more proxies. |

Scoring, per candidate proxy and requesting client:

```
score(P, C) = α₁·U − α₂·R − α₃·B − α₅·L
```

where `U` is the proxy's current usage count, `R` the client's request history, `B` the blocked proxies the client has already encountered, and `L` a location penalty. The α weights are the entire difference between `strict` and `kind` — see `scripts/config_basic.py`.

Three static baselines: `broadcast` (every client gets every proxy), `random` (n proxies sampled per client), `fixed` (round-robin).

**Censors** (`scripts/Censor.py`)

| Censor | Behavior |
|---|---|
| `OptimalCensor` | Blocks one proxy every 10 steps. Baseline budget. |
| `AggresiveCensor` | Doubles that budget. |
| `TargetedCensor` | Ranks proxies by honest-user count and blocks the top decile — an adversary that has learned which proxies matter. |
| `SnowflakeCensor` | Blocks a fraction of the volunteer pool on an interval, modeling the churn-heavy Snowflake case. |
| `MultiCensor` | Composes the above. Each client group faces its own censor; blocks are deduplicated across groups before being applied. |

`MultiCensor` is what makes the primary module distinct: client group A and group B experience different adversaries in the same run, drawing from the same shared proxy pool. Availability is logged per group so the asymmetry is visible.

**Dynamics per step**

Censors select and block. Affected clients increment their blocked-proxy count and immediately re-request. Independently, collateral damage fires with a per-strategy probability, blocking an uninvolved proxy. Every 10 steps a rejuvenation pass migrates surviving proxies to new addresses and clears their blocked state — the ephemeral-infrastructure behavior the paper adds to the original framework. New clients and proxies arrive on fixed cadences. Clients earn credits for each step spent on an unblocked proxy and spend them to request replacements, so reconnection is rate-limited rather than free.

A ZigZag detector flags two suspicious patterns during distribution: a client reassigned to a proxy it already held, and a proxy reused across many clients in a short window.

**Output**

Each run writes `results/minimal_results.csv`:

```
step, nonblocked_proxy_ratio, proxy_count, connected_overall, connected_A, connected_B, avg_proxy_lifetime
```

and prints average client wait time (steps between losing a proxy and being reconnected) and total collateral proxies blocked.

---

## Running it

### Primary simulator

```bash
cd MultiCensor_Simulations
pip3 install -r requirements.txt
python manage.py makemigrations
python manage.py migrate

PYTHONPATH=. python scripts/run_simulation_minimal.py \
    --distributor strict \
    --mode dynamic \
    --censor optimal
```

| Flag | Options | Default |
|---|---|---|
| `--distributor` | `strict`, `kind`, `broadcast`, `random`, `fixed` | `strict` |
| `--mode` | `dynamic`, `static` | `dynamic` |
| `--censor` | `optimal`, `targeted`, `snowflake` | `optimal` |

Run from the module root — the results path is relative. Simulation length, censor-agent ratio, block intervals, and the α weight profiles are all in `scripts/config_basic.py`.

`scripts/run_simulation.py` is the earlier control implementation, retained for reference and not invoked by the entry point.

### Live proxy demo (`sim_core`)

Requires a running Docker daemon.

```bash
cd sim_core
pip3 install -r requirements.txt
python3 Minimized_VMs.py        # -d for debug logging
```

Starts an aiohttp reverse proxy on port 8080 (override with `MINIMIZED_PORT`) in front of six nginx containers, rotating the active backend every 15 seconds. Requests are forwarded transparently with headers preserved; `/healthz` reports liveness. Containers are created on startup and torn down on exit.

---

## Layout

```
MultiCensor_Simulations/
  assignments/models.py       Proxy, Client, Assignment
  scripts/
    run_simulation_minimal.py Entry point: step loop, metrics, CSV output
    Censor.py                 Adversary models incl. MultiCensor composition
    simulation_utils.py       Proxy scoring, assignment, credit economy
    config_basic.py           Durations, ratios, α weight profiles
Minimized_Spotproxy_Version/  Single-censor control
sim_core/Minimized_VMs.py     Live rotating reverse proxy over Docker
```

---

## Attribution

`sim_core/Minimized_VMs.py` was written by [Hassan Fares](https://github.com/hassanfarescodes).

The canonical artifact accompanying the publication is maintained by the Hopper Lab at
[github.com/hoppernj/ProxySimulator](https://github.com/hoppernj/ProxySimulator). This repository is a working copy.

## Citation

```bibtex
@inproceedings{fares2026game,
  title     = {The Game Has Changed: Revisiting Proxy Distribution and Game Theory},
  author    = {Fares, Hassan and Fulsundar, Omkar and Hopper, Nicholas},
  booktitle = {Free and Open Communications on the Internet (FOCI)},
  year      = {2026}
}
```
